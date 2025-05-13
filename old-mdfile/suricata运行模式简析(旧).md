### suricata简析

#### 二、suricata-5.0.3

##### 1、定义

**suircata是一款支持IDS、IPS和NSM的入侵检测系统**

​       **IDS**：英文“Intrusion Detection Systems”的缩写，中文意思是“**入侵检测系统**”。依照一定的安全策略，通过软、硬件，对网络、系统的运行状况进行监视，尽可能发现各种攻击企图、攻击行为或者攻击结果，以保证网络系统资源的机密性、完整性和可用性。
　　**IPS：**是英文“Intrusion Prevention System”的缩写，中文意思是"**入侵防御系统"**。随着网络攻击技术的不断提高和网络安全漏洞的不断发现，传统防火墙技术加传统IDS的技术，已经无法应对一些安全威胁。在这种情况下，IPS技术应运而生，IPS技术可以深度感知并检测流经的数据流量，对恶意报文进行丢弃以阻断攻击，对滥用报文进行限流以保护网络带宽资源。
　　**NSM**：英文“network security monitoring”的缩写，中文意思是“网络安全监控”。

##### 2、运行模式

　suricata的基本组成。**Suricata**是由所谓的**线程**（threads）、**线程模块** （thread-modules）和**队列**（queues）组成。Suricata是一个多线程的程序，因此在同一时刻会有多个线程在工作。线程模块是依据 功能来划分的，比如一个模块用于解析数据包，另一个模块用于检测数据包等。每个数据包可能会有多个不同的线程进行处理，队列就是用于将数据包从一个线程传 递到另一个线程。与此同时，一个线程可以拥有多个线程模块，但是在某一时刻只有一个模块在运行。

​	Suricata支持多种运行模式。运行模式决定了不同的线程如何用于IDS：**single, workers, autofp**；![image-20200901113832013](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20200901113832013.png)

​       图中的RunMode Type并不是配置文件中的runmodes选项，而是后面的Custom Mode也就是自定义模式才可以在此处设置

**注：数据包处理线程（packet processing thread1）的工作模式可以参照【 线程Threading】**

###### 2.1 workers

​        通常，`workers`**运行模式执行最佳**。在这种模式下，NIC /驱动程序可确保在Suricata的处理线程上适当地平衡数据包。然后，每个数据包处理线程都包含完整的数据包管道。

![../_images/workers.png](https://suricata.readthedocs.io/en/suricata-5.0.3/_images/workers.png)



###### 2.2 autofp （flow worker）

1）autofp （单一抓包线程）

![../_images/autofp1.png](https://suricata.readthedocs.io/en/suricata-5.0.3/_images/autofp1.png)

**流平衡**同时发生在suricate中

2）autofp （复合抓包线程）

​        用于**处理PCAP文件**，或在某些IPS设置（如NFQ）的情况下， 使用该模式。这里有一个或多个捕获线程，它们捕获数据包并进行数据包解码，然后将其传递给线程。

![../_images/autofp2.png](https://suricata.readthedocs.io/en/suricata-5.0.3/_images/autofp2.png)

**流平衡**同时发生在suricate和硬件/驱动程序中

###### 2.3 single

​        运行模式`single`与`workers`模式**相同**，但是只有一个数据包处理线程。这在开发过程中很有用。

![../_images/single.png](https://suricata.readthedocs.io/en/suricata-5.0.3/_images/single.png)



##### 3、日志类型

对于分析日志的输出，一般会有四个日志文件;

![image-20200903111929405](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20200903111929405.png)

```less
eve.json //这是警报和事件的JSON输出。
fast.log  //该日志包含由警报组成的行输出。
stats.log //启用stats.log时，您可以设置将输出数据写入日志文件的时间（秒）
suricata.log

##其他日志输出
http.log //此日志跟踪所有HTTP流量事件。它包含HTTP请求，主机名，URI和用户代理。此信息将存储在http.log（默认名称，在suricata日志目录中）中。
pcap-log //使用pcap-log选项，您可以将Suricata注册的所有数据包保存在名为_log.pcap_的日志文件中。这样，您可以随时查看所有数据包。在正常模式下，将在default-log-dir中创建一个pcap文件。如果在yaml文件中设置了绝对路径，也可以在其他位置创建它。
alert-debug.log //提供有关警报的补充信息
```

**suricata.log**文件内容实例：

![image-20200904102706443](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20200904102706443.png)

##### 4、动作类型

```less
action-order:
 - pass //如果签名匹配且包含通过，则Suricata停止扫描数据包并跳至所有规则的末尾（仅适用于当前数据包）。
 - drop //这仅涉及IPS /串联模式。如果程序找到匹配的签名（包含丢弃），它将立即停止。数据包将不再发送。
 - reject //这是对数据包的主动拒绝。接收方和发送方都接收拒绝数据包。有两种类型的拒绝数据包将被自动选择。一种为涉及TCP的问题数据包，它将是一个Reset-packet。另一种为所有其他协议的ICMP-error数据包
 - alert //如果签名匹配并包含alert，则包将被视为与任何其他非威胁包一样，除此之外，Suricata将生成一个alert。只有系统管理员才能注意到此警告alert。
```

##### 5、线程Threading

Suricata是多线程的。Suricata使用多个CPU的CPU核心，因此它可以同时处理许多网络数据包。（在单核引擎中，将一次处理一个数据包。）

###### 线程模块：

​        数据包获取；解码和流应用程序层；检测；输出。

```less
##报文获取模块从网络中读取报文。

##解码模块对数据包进行解码，流应用程序(stream application layer)应用层具有三个任务：
      第一：它执行流跟踪，这意味着它将确保采取所有步骤来建立正确的网络连接。
      第二：TCP网络流量以数据包的形式传入。流组装引擎重建原始流。
      最后：检查应用层。分析HTTP和DCERPC。

##检测线程将比较签名(Compares signatures)。可以有多个检测线程，因此它们可以同时运行。

##在输出中，将处理所有警报和事件。
```

![../_images/threading.png](https://suricata.readthedocs.io/en/suricata-5.0.3/_images/threading.png)

| Packet acquisition | 从网络读取数据包   |
| ------------------ | ------------------ |
| Decode             | 解码数据包         |
| Stream app. Layer  | 执行流跟踪和重组   |
| Detect             | 比较签名           |
| Outputs            | 处理所有事件和警报 |

 	大多数计算机具有多个CPU’s/CPU cores 。默认情况下，操作系统确定哪个内核在哪个线程上工作。当一个内核已被占用时，将指定另一个"空闲"内核在该线程上工作。因此，哪个内核在哪个线程上工作可能会不时发生变化。



###### 线程选项：

```less
##(1)
set-cpu-affinity: no
##(2)
detect-thread-ratio: 1.5
##(3)
cpu-affinity:
  - management-cpu-set:
      cpu: [ 0 ]  # include only these cpus in affinity settings
  - receive-cpu-set:
      cpu: [ 0 ]  # include only these cpus in affinity settings
  - worker-cpu-set:
      cpu: [ "all" ]
      mode: "exclusive"
      # Use explicitely 3 threads and don't compute number by using
      # detect-thread-ratio variable:
      # threads: 3
      prio:
        low: [ 0 ]
        medium: [ "1-2" ]
        high: [ 3 ]
        default: "medium"
  - verdict-cpu-set:
      cpu: [ 0 ]
      prio:
        default: "high"
```

解释：

   （1）使用“set-cpu-affinity”此选项，您可以使Suricata为每个线程设置固定核心。在这种情况下，1、2和4位于核心0（零）。每个内核都有自己的检测线程。在核心0上运行的检测线程的优先级比在核心0上运行的其他线程的优先级低。
	    如果要占用这些其他内核，则核心0上的检测线程没有太多要处理的数据包。在其他内核上运行的检测线程将处理更多数据包。仅在将选项设置为“yes”后才是这种情况。（This is only the case after setting the option to ‘yes’.）**但是应该是设置为“no”才能有效**。

![../_images/balancing_workload.png](https://suricata.readthedocs.io/en/suricata-5.0.3/_images/balancing_workload.png)


​       （2）“detect-thread-ratio”选项：检测线程比率将确定检测线程的数量。默认情况下，它将是您计算机上存在的 CPU’s/CPU cores  数量的1.5倍。这将导致具有比CPU / CPU核心更多的检测线程。意味着您超额订购了核心数量。这可能会很方便当必须等待检测线程的时候。其余的检测线程可以变为活动状态。

​       （3） 在选项“ cpufinity”中，您可以设置哪个CPU /内核在哪个线程上工作。此选项中存在线程的相关设置选项：management-, receive-, worker- 和verdict-set。这些都是固定名称，不能更改。
​		对于每个设置选项也都有几个设置选项：cpu，mode和prio。在选项“ cpu”中，您可以设置将运行该集中的线程的 CPU’s/cores。您可以将此选项设置为“all”，也可以使用范围（0-3）或逗号分隔的列表（0,1）来指定相应内核。
​		选项“mode”可以设置为“balanced”或“exclusive”。设置为“balanced”时，可以通过在选项“ cpu”中设置的所有内核来处理各个线程。如果选项“mode”设置为“exclusive”，则每个线程都有固定的核心。
​		如前所述，线程可以具有不同的优先级。在选项“ prio”中，可以为每个线程设置优先级。此优先级可以为 low, medium, high，也可以将优先级设置为“default”。如果未为CPU设置优先级，则将使用“default”设置。默认情况下，Suricata为每个可用的CPU / CPU内核创建一个“detect”（工作线程）线程。

```makefile
##原文参照
		In the option ‘cpu affinity’ you can set which CPU’s/cores work on which thread. In this option there are several sets of threads. 
​		The management-, receive-, worker- and verdict-set. These are fixed names and can not be changed. For each set there are several options: cpu, mode, and prio. 
​		In the option ‘cpu’ you can set the numbers of the CPU’s/cores which will run the threads from that set. You can set this option to ‘all’, use a range (0-3) or a comma separated list (0,1). 
​		The option ‘mode’ can be set to ‘balanced’ or ‘exclusive’. When set to ‘balanced’, the individual threads can be processed by all cores set in the option ‘cpu’. If the option ‘mode’ is set to ‘exclusive’, there will be fixed cores for each thread. 
​		As mentioned before, threads can have different priority’s. In the option ‘prio’ you can set a priority for each thread. This priority can be low, medium, high or you can set the priority to ‘default’. If you do not set a priority for a CPU, than the settings in ‘default’ will count. By default Suricata creates one ‘detect’ (worker) thread per available CPU/CPU core.
```

###### 对于IDS/IPS模式的"cpu-affinity"相关设置

相关的设置可以参看  "线程选项"->(3)  部分；

![image-20200904140707050](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20200904140707050.png)

##### 6、IP Defrag

​		有时，网络数据包看起来是零散的。在某些网络上，它发生的频率比在其他网络上更高。碎片数据包存在许多部分。在Suricata能够准确检查此类数据包之前，必须重新构造这些数据包。这将由Suricata的一个组件完成；碎片整理引擎。碎片整理引擎重构了碎片后的数据包后，该引擎会将重组后的数据包发送到Suricata的其余部分。

​		碎片整理中包含三个选项：max-frags，prealloc和timeout。当Suricata收到数据包的片段时，它将保留在内存中，该数据包的其他片段将很快出现以完成该数据包。但是，有可能碎片没有出现。为了防止Suricata一直等待该数据包（从而使用内存），在一定时间间隔后Suricata会丢弃这些碎片。默认情况下，在60秒后发生。

```less
defrag:
  max-frags: 65535
  prealloc: yes
  timeout: 60
```



##### 7、Flow and Stream handling

###### 7.1 Flow 设置

​		在Suricata中，Flow 非常重要。它们在Suricata内部组织数据的方式中起着重要作用。Flow 有点类似于连接（connection），只是Flow 更为通用。具有相同元组（tuple）（协议，源IP，目标IP，源端口，目标端口）的所有数据包（packet）都属于同一Flow 。属于Flow 的数据包在内部连接到它。**下图也标识了flow与stream的区别**

![../_images/flow.png](https://suricata.readthedocs.io/en/suricata-5.0.3/_images/flow.png)

<img src="https://suricata.readthedocs.io/en/suricata-5.0.3/_images/Tuple1.png" alt="../_images/Tuple1.png"  />

​		跟踪这些flow会占用内存。flow越多，将花费越多的内存。为了控制内存使用量，有几种选择：

```less
flow:
  memcap: 33554432    #用于设置流引擎将使用的最大字节数
  hash_size: 65536    #用于设置哈希表的大小（flow将会组织于hash表内）
  Prealloc: 10000     #用于指示suricate所需要为flow所准备的内存大小。（对于尚未属于流的数据包，Suricata将创建一个新流。这是一个相对昂贵的动作。随之而来的风险是，攻击者/黑客可以在此部分攻击引擎系统。当他们确保计算机收到大量具有不同元组的数据包时，引擎必须进行大量新的处理。这样，攻击者可能会淹没系统。为了减轻引擎的过载，此选项指示Suricata将大量流保留在内存中。这样，Suricata不太容易受到此类攻击。）
```

​		尽管预先分配了内存，但仍然会达到memcap的位置，流程引擎将进入紧急模式。在这种模式下，引擎将利用较短的超时时间。它使流以更积极的方式过期，因此将有更多空间容纳新的流。

​		有两个选项：Emergency_recovery和prune_flows。紧急恢复设置为30。这是预分配流的百分比，在此百分比之后，流引擎将恢复正常（当10000个流中的30％完成时）。

> 如果在紧急模式下，主动超时未达到期望的结果，则此选项是最后的选择。即使它们尚未达到超时，它也会结束一些流量。prune_flows 选项显示每次设置新流时将终止多少流。

```less
emergency_recovery: 30      #Percentage of 1000 prealloc'd flows.
prune_flows: 5              #显示每次设置新流时将终止多少流。
```

###### 7.2 flow 超时

Suricata在内存中保持流的时间由流超时确定。

​		流可以处于不同的状态。Suricata区分TCP的三种流状态和UDP的两种流状态。对于TCP，这些是：new，Established 和Closed；

​		对于UDP，仅new和Established 。对于这些状态中的每一个，Suricata都可以使用不同的超时时间。

​		TCP流程中的：new状态表示三向握手期间的时间段。Established 的状态是三向握手完成时的状态。TCP流程中的Closed状态：有几种方式都可以结束flow，通过复位或“四次挥手”的方式。

​		UDP流中的 new 状态：单向发送数据包的状态。

​		UDP流中的 Established 状态：数据包可以双向发送的状态。

​		在示例配置中，是每个协议的设置。TCP，UDP，ICMP和default （所有其他协议）。

```less
flow-timeouts:

  default:
    new: 30                     ## Time-out in seconds after the last activity in this flow in a New state.
    established: 300            ## Time-out in seconds after the last activity in this flow in a Established state.
    emergency_new: 10           ## Time-out in seconds after the last activity in this flow in a New state during the emergency mode.
    emergency_established: 100  ## Time-out in seconds after the last activity in this flow in a Established state in the emergency mode.
  tcp:
    new: 60
    established: 3600
    closed: 120
    emergency_new: 10
    emergency_established: 300
    emergency_closed: 20
  udp:
    new: 30
    established: 300
    emergency_new: 10
    emergency_established: 100
  icmp:
    new: 30
    established: 300
    emergency_new: 10
    emergency_established: 100
```



###### 7.3 Stream-engine（流引擎）

​		流引擎跟踪TCP连接。该引擎分为两部分：**流跟踪引擎**和**重组引擎**。

​		流跟踪引擎监视连接状态。重组引擎将按原样重建flow ，因此Suricata会识别它。

​		流引擎有两个可以设置的memcap。一台用于流跟踪引擎，一台用于重组引擎。

​		流跟踪引擎将流的信息保留在内存中。有关状态，TCP序列号和TCP窗口的信息。为了保留此信息，它可以利用memcap允许的容量。

​		TCP数据包具有所谓的校验和。这是一个内部代码，可以查看数据包是否已到达良好状态。流引擎将不会处理校验和错误的数据包。可以通过输入“ no”代替“ yes”来取消此选项。

```less
stream:
  memcap: 64mb                # TCP会话跟踪的最大内存使用量（字节）
  checksum_validation: yes    #验证数据包校验和，拒绝无效校验和的数据包。

```

……………………（该部分其它信息可见官网文档10.1.12.3）…………………………

