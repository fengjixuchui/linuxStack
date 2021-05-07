# 传统网络
在传统的网络数据包接收设计中，数据从外界过来到达网卡最终到达应用层程序中解析经过了如图的一段过程
1. 报文到达网卡设备
2. 网卡通过DMA将报文写入`ring buffer`
3. 网卡发送硬中断通知CPU
4. CPU根据中断调用相应的中断函数最终调用网卡驱动中的函数，先禁用网卡的中断，接着启动软中断
5. 内核线程`ksoftirqd`收到软中断进行处理，调用中断函数最终调用网卡驱动中的`poll函数`，从`ring buffer`中读取数据并填充到`skb`结构中
6. `skb`发往协议栈然后经过各种处理后拷贝到`socket接收队列`中由用户态程序处理


![32ee5cf8-3b19-4387-885e-f093adf97c4c.jpg](linux网络(六)_files/32ee5cf8-3b19-4387-885e-f093adf97c4c.jpg)


这儿需要关注的前置知识点就是`DMA`技术。


## DMA
计算机上的不同`I/O`设备之间的数据通过总线来传输，但是并非就是说任意的设备都能主动发起数据传输，因此设备被划分成了`主设备`比如`CPU`和`从设备`比如`磁盘`，只有`主设备`可以主动发起数据传输，因此最早`主设备`去判断是否有数据需要传输是通过程序轮询的方式。那`从设备`自己有数据了想要`主设备`传输怎么办？后来就有了`硬中断`，由`从设备`发起一个信号通知`主设备`发起数据传输，也就是由`主设备`读取而不是`从设备`推送。


那问题来了`从设备`直接和`从设备`数据交互可以吗？如果可以的话这显然就违背了设计初衷，但是在计算机里能作为`主设备`的也只有`CPU`一个，如果每一次数据传输都要`cpu`参与其中并等待完成的话，那`cpu`其实大部分时间都耗费在`I/O`等待上实在是太浪费时间了，因此需要再引入一个`主设备`来分担一部分`cpu`的压力，这就导致了`DMA`技术的诞生。
> 这儿并不是说`cpu`就完全不参与的，只是`cpu`的耗费近乎于0


`DMA`技术就是在主板上新加了芯片，当进行`内存`和`I/O设备`数据传输的时候直接通过`DMAC`设备处理，这个设备既是`主设备`也是`从设备`，`cpu`会先去设置`DMAC`的配置然后其会变成`idle`状态等待其余`I/O设备`通过主板上的一个额外连线发起数据传输请求，等`DMAC`处理完数据后发起一个`中断`让`cpu`处理一下后续的事情，然后重新回到`idle`状态。


可以看出来`DMA`技术十分适合大数据量传输的场景其中最典型的场景就是网卡了，在网卡驱动加载的过程中，会调用`dma_alloc_coherent`在`内核空间`里建立环形缓冲区`ring buffer`
```
rxdr->desc = dma_alloc_coherent(&pdev->dev, rxdr->size, &rxdr->dma,
                    GFP_KERNEL);
```
一共会建两个，一个是`接收环形缓冲区`另一个是`发送环形缓冲区`，他们通常被称为`DMA环形缓冲区`，对于多核`cpu`来说每个核都有一个自己`接收/发送环形缓冲区`。


回到传统网络上来可以看出来，一个数据包从网卡一直到用户态就得经过一次DMA和至少2次内存拷贝，还涉及到内存的分配和释放，这显然是十分消耗性能的，更不用说还有内核态用户态切换，这些其实都还是小问题因为最大的问题出现在`协议栈`上，现在非常多网卡都支持`RSS`队列，这个技术是为了实现数据包并行捕获，然而问题是`协议栈`针对数据包的处理是串行的，这就导致硬件的并行能力完全无法发挥，在这种情况下绕过`协议栈`处理报文成了研究的方向。


# DPDK
在`DPDK`以前依然有过绕过`协议栈`的技术，比如`libpcap`，`libpcap-mmap`和`PF_RING`，前两者旁路了数据链路层绕过了`协议栈`，但是过程中依旧存在多次拷贝的问题，而`PF_RING`则和`DPDK`的效果一致，都是数据包零拷贝且旁路了内核，即用户态直接共享了`DMA缓冲区`，但是底层实现上还有有所区别。


## UIO
`DPDK`用了linux的`UIO`的机制，这个技术简单来说就是用户态的驱动，而产生该技术的原因是对于非常多设备来说创建一个内核态驱动实际上是有点过分的，因为他们真正需要的只是处理中断的方式和提供对设备内存的访问而已，而设备的控制逻辑其实不必要在内核态，因为这些逻辑中并不需要用到内核态的其他资源，如果滥用的话反而会使得内核变得危险。因此将设备驱动设计到了用户态，通过`mmap()`来访问到设备内核内存，通过针对`/dev/uioX`的`read()`来读取中断。
`DPDK`的`UIO`驱动屏蔽了网卡发出`硬中断`，然后通过在用户态采用主动轮询的方式来代替中断，这种模式就是`PMD`。
> `DPDK`会卸载原生的网卡驱动，因此在绑了`DPDK`网卡后，其余程序将不再能收到这个网卡的流量


# XDP
如果说先前的技术都在探讨如何绕过`协议栈`来提高数据报文的处理速度，那`XDP`就比较特殊了，它重新创建了一个数据处理平面也就是预处理的模式，在数据报文进入到协议栈之前就进行处理。
在流程上来说，`XDP`并没有接管网卡，而是在正常的`DMA`流程以后增加了一个处理平面，在这个平面中可以直接操作`raw packet`，但是这个功能需要驱动支持，即驱动在收包流程中分配`skb`以前能够主动调用`ebpf`程序进行处理


![31ceb0b7-b070-43b3-9a21-942d049b3c7a.png](linux网络(六)_files/31ceb0b7-b070-43b3-9a21-942d049b3c7a.png)
XDP包含有三种模式：
1. Native模式：`XDP_FLAGS_DRV_MOD`默认的工作模式，XDP作为网驱的hook调用，这个模式需要驱动支持，支持情况如下：[https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md#xdp](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md#xdp)
2. Offloaded模式：`XDP_FLAGS_HW_MODE`需要硬件级支持即智能网卡，数据报文的处理不依赖本机CPU而是依赖网卡本身的芯片NPU，效率最高但是一些BPF的辅助函数可能无法支持到
3. Generic模式：`XDP_FLAGS_SKB_MODE`最低效的工作模式，作为内核模块安插在协议栈中，对驱动和硬件都无要求因此不建议使用


通过ip link命令可以看一下当前xdp的模式：
```
ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 xdp qdisc pfifo_fast state UP mode DEFAULT group default qlen 1000
generic/SKB (xdpgeneric), native/driver (xdp), hardware offload (xdpoffload)
```
# 结语
就结论来讲，`XDP`和`DPDK`侧重的方向其实并不相同，前者注重在于提供一个`可编程数据报文`的功能，而后者则完全侧重于高速收发数据包并提供出针对数据包广泛的逻辑功能支持，就性能来讲`XDP`确实要比`DPDK`的应用面更加广泛，但是也因为是内核态功能导致自身门槛更高，反观`DPDK`拥有更完善的生态和优越的开发环境更有`intel`背书，因此很难说会被取代掉。


# 参考文章
* [linux报文高速捕获技术对比--napi/libpcap/afpacket/pfring/dpdk/xdp](https://blog.csdn.net/gengzhikui1992/article/details/103142848)
* [LINUX网络子系统中DMA机制的实现](https://club.perfma.com/article/663987)
* [Illustrated Guide to Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/eng/2016/10/11/monitoring-tuning-linux-networking-stack-receiving-data-illustrated/)
* [The Userspace I/O HOWTO](https://www.kernel.org/doc/html/latest/driver-api/uio-howto.html)
* [UIO: user-space drivers](https://lwn.net/Articles/232575/)
* [XDPeXpress Data Path](https://www.iovisor.org/technology/xdp)