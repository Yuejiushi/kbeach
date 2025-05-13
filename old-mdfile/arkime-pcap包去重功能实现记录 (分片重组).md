

####                                                                       pcap包去重功能实现记录

##### 一、抓包模式确定与分析

在dps中一共五种抓包方式：

```less
   //抓取模式的hash表存储
    HASH_INIT(s_, readersHash, moloch_string_hash, moloch_string_cmp);
    moloch_readers_add("libpcap-file", reader_libpcapfile_init);
    moloch_readers_add("libpcap", reader_libpcap_init);
   // DPS所常用的模式
    moloch_readers_add("tpacketv3", reader_tpacketv3_init);
    moloch_readers_add("null", reader_null_init);
    moloch_readers_add("greread", reader_greonly_init);//add by yb
```

根据梳理：dps目前所使用的抓包模式为"tpacketv3"：以下为相关的代码解析所需结构定义：

![image-20201217101122133](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201217101122133.png)

![image-20201217104431482](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201217104431482.png)

![image-20201217102022409](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201217102022409.png)

对应：LOCAL MolochTPacketV3_t  **infos**[MAX_INTERFACES];

![image-20201217104830325](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201217104830325.png)

```less
//下属代码解析做使用到的libpcap结构原型：
//当然也有最主要的结构体 pcap_t结构，但是因为其太长，此处不予展示
typedef struct pcap pcap_t；

struct bpf_program {
    u_int bf_len;
    struct bpf_insn *bf_insns;
};


/* 创建TPACKET_V3环形缓冲区时对应的配置参数结构
 * 备注： tpacket_req3结构是tpacket_req结构的超集，实际可以统一使用本结构去设置所有版本的环形缓冲区，V1/V2版本会自动忽略多余的字段
 */
struct tpacket_req3 {
    unsigned int    tp_block_size;      // 每个连续内存块的最小尺寸(必须是 PAGE_SIZE * 2^n )
    unsigned int    tp_block_nr;        // 内存块数量
    unsigned int    tp_frame_size;      // 每个帧的大小(虽然V3中的帧长是可变的，但创建时还是会传入一个最大的允许值)
    unsigned int    tp_frame_nr;        // 帧的总个数(必须等于 每个内存块中的帧数量*内存块数量)
    unsigned int    tp_retire_blk_tov;  // 内存块的寿命(ms)，超时后即使内存块没有被数据填入也会被内核停用，0意味着不设超时
    unsigned int    tp_sizeof_priv;     // 每个内存块中私有空间大小，0意味着不设私有空间
    unsigned int    tp_feature_req_word;// 标志位集合(目前就支持1个标志 TP_FT_REQ_FILL_RXHASH)
}

 
// 开启PACKET_MMAP时libpcap对V1、V2、V3版本环形缓冲区的统一管理结构
union thdr {
    struct tpacket_hdr          *h1;
    struct tpacket2_hdr         *h2;
    struct tpacket_block_desc   *h3;     
    void *raw;
}


// TPACKET_V3环形缓冲区每个帧的头部结构
// 每个帧也有一个与之关联的头部，头部结构为tpacket3_hdr，其中tp_next_offset字段指向同一个内存块中的下一个帧。
struct tpacket3_hdr {
    __u32       tp_next_offset; // 指向同一个内存块中的下一个帧
    __u32       tp_sec;         // 时间戳(s)
    __u32       tp_nsec;        // 时间戳(ns)
    __u32       tp_snaplen;     // 捕获到的帧实际长度 （去除帧头）
    __u32       tp_len;         // 帧的理论长度
    __u32       tp_status;      // 帧的状态
    __u16       tp_mac;         // 以太网MAC字段距离帧头的偏移量
    __u16       tp_net;
    union {
        struct tpacket_hdr_variant1 hv1;    // 包含vlan信息的子结构  下示：
    };
    __u8        tp_padding[8];
}

 struct tpacket_hdr_variant1{
	uint32_t 	tp_rxhash
	uint32_t 	tp_vlan_tci
}

struct iovec {
	 /* Starting address (内存起始地址）*/
   void  *iov_base;   
   
    /* Number of bytes to transfer（这块内存长度） */
   size_t iov_len;    
};


struct packet_mreq
{
   int mr_ifindex; /* 接口索引号 */
   unsigned short mr_type; /* 动作 */
   unsigned short mr_alen; /* 地址长度 */
   unsigned char mr_address[8]; /* 物理层地址 */
};

struct sock_fprog      
/*Required for SO_ATTACH_FILTER. */
{
    unsigned short  len;   
    /*Number of filter blocks */
    struct sock_filter   *filter;
};
// sock_filter是一个很冷门的结构，google几乎没有其使用的源码出现
struct sock_filter {	/* Filter block */
	__u16	code;   /* Actual filter code */
	__u8	jt;	/* Jump true */
	__u8	jf;	/* Jump false */
	__u32	k;      /* Generic multiuse field */
};

// 数据链路层的头信息通常定义在 sockaddr_ll　的结构体中
struct sockaddr_ll {
               unsigned short sll_family; /* Always AF_PACKET */
               unsigned short sll_protocol; /* Physical-layer protocol */
               int sll_ifindex; /* Interface number */
               unsigned short sll_hatype; /* ARP hardware type */
               unsigned char sll_pkttype; /* Packet type */
               unsigned char sll_halen; /* Length of address */
               unsigned char sll_addr[8]; /* Physical-layer address */
};
         sll_protocol : 标准以太网协议类型，按网络字节顺序。定义在中。
         sll_ifindex: interface索引，0 匹配所有的网络接口卡； 
         sll_hatype: ARP 硬件地址类型(hardware address type) 定义在中,常用 ARPHRD_ETHER
         sll_pkttype: 包含了packet类型。
                 PACK_HOST                  包地址为本地主机地址。
                 PACK_BROADCAST    物理层广播包。
                 PACK_MULTICAST      发送到物理层多播地址的包。
                 PACK_OTHERHOST    发往其它在混杂模式下被设备捕获的主机的包。
                 PACK_OUTGOING        本地回环包；
         sll_addr 和 ssl_halen 包含了物理层地址和其长度
```



```less
void reader_tpacketv3_init(char *UNUSED(name))
{
    int i;
    // 此处用以确定key-value中的tpacketv3BlockSize字段值：8388608
    int blocksize = moloch_config_int(NULL, "tpacketv3BlockSize", 1<<21, 1<<16, 1U<<31);
    // 此处用以确定key-value中的tpacketv3NumThreads字段值：2
    numThreads = moloch_config_int(NULL, "tpacketv3NumThreads", 2, 1, 6);

    if (blocksize % getpagesize() != 0) {   //getpagesize:返回一分页的大小，单位为字节(byte)
        DPS_LOGEXIT("block size %d not divisible by pagesize %d", blocksize, getpagesize());
    }
	// 猜测：snapLen设置的是数据包抓取长度为snaplen, 如果不设置默认将会是68字节
    if (blocksize % config.snapLen != 0) {
        DPS_LOGEXIT("block size %d not divisible by %u", blocksize, config.snapLen);
    }link_type)

    moloch_packet_set_dltsnap(DLT_EN10MB, config.snapLen);
	//于正式的抓包开始前创建一个pcap_t *dpcap
    pcap_t *dpcap = pcap_open_dead(pcapFileHeader.dlt, pcapFileHeader.snaplen);
	// 判断config.in配置文件中是否配置了相关的bpf式子：如果存在，则进行相关的编译设置
    if (config.bpf) {
        if (pcap_compile(dpcap, &bpf, config.bpf, 1, PCAP_NETMASK_UNKNOWN) == -1) {
            DPS_LOGEXIT("ERROR - Couldn't compile filter: '%s' with %s", config.bpf, pcap_geterr(dpcap));
        }
    }

    int fanout_group_id = moloch_config_int(NULL, "tpacketv3ClusterId", 0x0000, 0x0000, 0xffff);
	//MAX_INTERFACES == 32 由该参数我们也可以知道dps的最大接口配置数量是32个；
    for (i = 0; i < MAX_INTERFACES && config.interface[i]; i++) {
        // LOCAL MolochTPacketV3_t infos[MAX_INTERFACES];
        // MolochTPacketV3_t结构可以查看上面截图：
        MOLOCH_LOCK_INIT(infos[i].lock);
		// if_nametoindex()：参数为网络接口名称字符串；若该接口存在，则返回相应的索引，否则返回0 
         // 为下面的  mreq.mr_ifindex赋值
        int ifindex = if_nametoindex(config.interface[i]);

        infos[i].fd = socket(AF_PACKET, SOCK_RAW, 0);

        int version = TPACKET_V3;
        if (setsockopt(infos[i].fd, SOL_PACKET, PACKET_VERSION, &version, sizeof(version)) < 0)
            DPS_LOGEXIT("Error setting TPACKET_V3, might need a newer kernel: %s", strerror(errno));


        memset(&infos[i].req, 0, sizeof(infos[i].req));
        // 以下都是在填充初始化infos中的：tpacket_req3结构
        infos[i].req.tp_block_size = blocksize;  // 每个连续内存块的最小尺寸(必须是 PAGE_SIZE * 2^n )
        infos[i].req.tp_block_nr = numThreads * 64;  // 内存块数量
        infos[i].req.tp_frame_size = config.snapLen; // 每个帧（数据传输一次的最小单位）的大小
        infos[i].req.tp_frame_nr = (blocksize * infos[i].req.tp_block_nr) / infos[i].req.tp_frame_size;//帧的总个数(必须等于 每个内存块中的帧数量*内存块数量)  内存总的大小除以帧大小
        infos[i].req.tp_retire_blk_tov = 60;  // 内存块的寿命(ms)，超时后即使内存块没有被数据填入也会被内核停用，0意味着不设超时
        infos[i].req.tp_feature_req_word = 0; // 标志位集合(目前就支持1个标志 TP_FT_REQ_FILL_RXHASH)  Rx环-功能请求位
        if (setsockopt(infos[i].fd, SOL_PACKET, PACKET_RX_RING, &infos[i].req, sizeof(infos[i].req)) < 0)
            DPS_LOGEXIT("Error setting PACKET_RX_RING: %s", strerror(errno));
		// 结构详情可以看上述代码块
        struct packet_mreq      mreq;
        memset(&mreq, 0, sizeof(mreq));
        mreq.mr_ifindex = ifindex; // 网络接口映射
        mreq.mr_type = PACKET_MR_PROMISC; //设置混杂模式
        // setsockopt成功0失败-1
        // setsockopt()用来设置参数s 所指定的socket 状态
        if (setsockopt(infos[i].fd, SOL_PACKET, PACKET_ADD_MEMBERSHIP, &mreq, sizeof(mreq)) < 0)
            DPS_LOGEXIT("Error setting PROMISC: %s", strerror(errno));
		// 配置文件中出现bpf时才会调用
        if (config.bpf) {
            struct sock_fprog       fcode;
            fcode.len = bpf.bf_len;
            fcode.filter = (struct sock_filter*)bpf.bf_insns;
            if (setsockopt(infos[i].fd, SOL_SOCKET, SO_ATTACH_FILTER, &fcode, sizeof(fcode)) < 0)
                DPS_LOGEXIT("Error setting SO_ATTACH_FILTER: %s", strerror(errno));
        }
		// 内存映射函数（共享内存）：
         // 下述
         // infos[i].map记录了映射标志（用以标识区分不同的映射空间）
        infos[i].map = mmap64(NULL, infos[i].req.tp_block_size * infos[i].req.tp_block_nr,
            PROT_READ | PROT_WRITE, MAP_SHARED | MAP_LOCKED, infos[i].fd, 0);
        // uint64_t readerMem; 记录所需读取的内存大小：内存块大小*内存块数量
        readerMem += infos[i].req.tp_block_size * infos[i].req.tp_block_nr;

        if (unlikely(infos[i].map == MAP_FAILED)) {
            DPS_LOGEXIT("ERROR - MMap64 failure in reader_tpacketv3_init, %d: %s", errno, strerror(errno));
        }
        // 为infos[i].rd(struct iovec)分配内存大小== 内存块数量 * iovec结构大小 
        // 每个内存块对应一个iovec结构，用以记录内存使用大小与起始位置
        infos[i].rd = malloc(infos[i].req.tp_block_nr * sizeof(struct iovec));
        // 更新所需读取的内存大小：
        readerMem += infos[i].req.tp_block_nr * sizeof(struct iovec);
        
        uint16_t j;
        // 为每块内存进行赋值，记录infos[i].map（内存映射）
        for (j = 0; j < infos[i].req.tp_block_nr; j++) {
            infos[i].rd[j].iov_base = infos[i].map + (j * infos[i].req.tp_block_size);
            // 内存长度等于内存块的大小==blocksize==8388608b==8M
            infos[i].rd[j].iov_len = infos[i].req.tp_block_size;
        }

        struct sockaddr_ll ll;
        memset(&ll, 0, sizeof(ll));
        ll.sll_family = PF_PACKET; // 默认都是该标志
        // #define ETH_P_ALL       0x0003
        // linux自身有两种从数据链路层接收分组：
        //为fd=socket(PF_PACKET,SOCK_RAW,htons(ETH_P_ALL));
        ll.sll_protocol = htons(ETH_P_ALL);
        // 见上面的初始化：int ifindex = if_nametoindex(config.interface[i]);
        // 存储了"网卡索引"
        ll.sll_ifindex = ifindex;
	    //infos[i].fd是由setsockopt所创建的套接字

        if (bind(infos[i].fd, (struct sockaddr*)&ll, sizeof(ll)) < 0)
            DPS_LOGEXIT("Error binding %s: %s", config.interface[i], strerror(errno));
        //  int fanout_group_id = moloch_config_int(NULL, "tpacketv3ClusterId", 0x0000, 0x0000, 0xffff);
        //  DPS中基本未使用该字段：tpacketv3ClusterId
        if (fanout_group_id != 0) {
            int fanout_type = PACKET_FANOUT_HASH;
            int fanout_arg = ((fanout_group_id+i) | (fanout_type << 16));
            if(setsockopt(infos[i].fd, SOL_PACKET, PACKET_FANOUT, &fanout_arg, sizeof(fanout_arg)) < 0)
                DPS_LOGEXIT("Error setting packet fanout parameters: (%d,%s)", fanout_group_id, strerror(errno));
        }
    }
	// 验证累加后的‘i’值是否复合MAX_INTERFACES（最大网卡数量）值
    if (i == MAX_INTERFACES) {
        DPS_LOGEXIT("Only support up to %d interfaces", MAX_INTERFACES);
    }

    moloch_reader_start = reader_tpacketv3_start;
    moloch_reader_stop = reader_tpacketv3_stop;
    moloch_reader_stats = reader_tpacketv3_stats;
}
```



```less
mmap函数说明:
（1）参数 addr 指明文件描述字fd指定的文件在进程地址空间内的映射区的开始地址，必须是页面对齐的地址，通常设为 NULL,让内核去选择开始地址。任何情况下，mmap 的返回值为内存映射区的开始地址。

（2）参数 length 指明文件需要被映射的字节长度。off 指明文件的偏移量。通常 off 设为 0 。
如果 len 不是页面的倍数，它将被扩大为页面的倍数。扩充的部分通常被系统置为 0 ，而且对其修改并不影响到文件。
off 同样必须是页面的倍数。通过 sysconf(_SC_PAGE_SIZE) 可以获得页面的大小。

（3）参数 prot 指明映射区的保护权限。通常有以下 4 种。通常是 PROT_READ | PROT_WRITE 。
PROT_READ 可读
PROT_WRITE 可写
PROT_EXEC 可执行
PROT_NONE 不能被访问

（4）参数 flag 指明映射区的属性。取值有以下几种。MAP_PRIVATE 与 MAP_SHARED 必选其一，MAP_FIXED 为可选项。
MAP_PRIVATE 指明对映射区数据的修改不会影响到真正的文件。
MAP_SHARED 指明对映射区数据的修改，多个共享该映射区的进程都可以看见，而且会反映到实际的文件。
MAP_FIXED 要求 mmap 的返回值必须等于 addr 。如果不指定 MAP_FIXED 并且 addr 不为 NULL ，则对 addr 的处理取决于具体实现。考虑到可移植性，addr 通常设为 NULL ，不指定 MAP_FIXED。

当 mmap 成功返回时,fd 就可以关闭，这并不影响创建的映射区。
```



##### 二、start/stop/stats函数解析

###### 1、reader_tpacketv3_start

```less
 moloch_reader_start = reader_tpacketv3_start;
    moloch_reader_stop = reader_tpacketv3_stop;
    moloch_reader_stats = reader_tpacketv3_stats;


//头文件：#include <sys/types.h>   #include <sys/socket.h>
//定义函数：int getsockopt(int s, int level, int optname, void* optval, socklen_t* optlen);
//函数说明：getsockopt()会将参数s 所指定的socket 状态返回. 参数optname 代表欲取得何种选项状态, 而参数optval 则指向欲保存结果的内存地址, 参数optlen 则为该空间的大小. 参数level、optname 请参考setsockopt().
//返回值：成功则返回0, 若有错误则返回-1, 错误原因存于errno
//错误代码：
1、EBADF 参数s 并非合法的socket 处理代码
2、ENOTSOCK 参数s 为一文件描述词, 非socket
3、ENOPROTOOPT 参数optname 指定的选项不正确
4、EFAULT 参数optval 指针指向无法存取的内存空间

// 位于reader_tpacketv3_thread函数：struct pollfd pfd
// 每一个pollfd结构体指定了一个被监视的文件描述符，可以传递多个结构体，指示poll()监视多个文件描述符。每个结构体的events域是监视该文件描述符的事件掩码，由用户来设置这个域。revents域是文件描述符的操作结果事件掩码，内核在调用返回时设置这个域。
struct pollfd {
int fd;         /* 文件描述符 */
short events;         /* 等待的事件 */
short revents;       /* 实际发生了的事件 */
} ; 
//下属图片中POLLERR才对
```

<img src="C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201217153152433.png" alt="image-20201217153152433" style="zoom:50%;" />

```less
void reader_tpacketv3_start() {
    int i, t;
    char name[100];
    // 对应一个网卡两个抓包线程
    for (i = 0; i < MAX_INTERFACES && config.interface[i]; i++) {  // MAX_INTERFACES=32
        for (t = 0; t < numThreads; t++) {  // numThreads由ini文件中配置所得：2
            snprintf(name, sizeof(name), "moloch-af3%d-%d", i, t);
            // 第三个参数(gpointer)(long)i其实是第二个参数func函数的参数
            // 所以我们以此来看 i -> 代表：哪一个网卡的抓包，线程
            g_thread_unref(g_thread_new(name, &reader_tpacketv3_thread, (gpointer)(long)i));
        }
    }
}
/******************************************************************************/
void reader_tpacketv3_stop()
{
    int i;
    for (i = 0; i < MAX_INTERFACES && config.interface[i]; i++) {
        close(infos[i].fd);
        infos[i].fd == 0;//add by zy
    }
}
/******************************************************************************/
int reader_tpacketv3_stats(MolochReaderStats_t *stats)
{
    MOLOCH_LOCK(gStats);

    int i;

    struct tpacket_stats_v3 tpstats;
    for (i = 0; i < MAX_INTERFACES && config.interface[i]; i++) {
        socklen_t len = sizeof(tpstats);
        // getsockopt()会将参数s 所指定的socket 状态返回. 参数optname 代表欲取得何种选项状态, 而参数optval 则指向欲保存结果的内存地址, 参数optlen 则为该空间的大小. 参数level、optname 请参考setsockopt().
        getsockopt(infos[i].fd, SOL_PACKET, PACKET_STATISTICS, &tpstats, &len); //这里的隐患是，fd资源可能被释放，但仍然进行了统计。

        gStats.dropped += tpstats.tp_drops;
        gStats.total += tpstats.tp_packets;
    }
    DPS_LOG_INFO("gStats.drop = %llu,gStats.total= %llu\n", gStats.dropped, gStats.total);
    *stats = gStats;
    MOLOCH_UNLOCK(gStats);
    return 0;
}
```

当进入到 reader_tpacketv3_thread 时，就意味着进入了流程：packet --> session -> block的packetQ --> 全局pcaketQ

![image-20201217162049826](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201217162049826.png)

对应的15个线程：下面一共24（+2）个线程，相关的线程定义如下图：

```less
Thread-1 //主线程 在main函数执行时启动
Thread-3-[dps-notify]   //监听线程 由moloch_notify_init创建 
Thread-5-[moloch-stats]  //数据统计线程 由esSession_plugin_init或moloch_db_init创建
Thread-6-[moloch-pkt0]   //packet处理线程 由moloch_packet_init创建
Thread-7...19-[moloch-pkt1...pkt13]   
Thread-20-[moloch-pkt14]   //packet处理线程 由moloch_packet_init创建 共15个
Thread-22-[dps-linkstat]   //链路数据线程 由linkStat_plugin_init创建
Thread-26-[moloch-simple]  //写线程 由writer_simple_init创建
// 位于reader_tpacketv3_start下：
Thread-27-[moloch-af30-0]
Thread-28-[moloch-af30-1] 
```

![image-20201217162821461](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201217162821461.png)



###### 2、reader_tpacketv3_thread函数

此处使用了IO复用模型：

​			IO复用模型，Linux下的select、poll和epoll就是干这个的。将用户socket对应的fd注册进epoll，然后epoll帮你监听哪些socket上有消息到达，这样就避免了大量的无用操作。此时的socket应该采用**非阻塞模式**。
这样，整个过程只在调用select、poll、epoll这些调用的时候才会阻塞，收发客户消息是不会阻塞的，整个进程或者线程就被充分利用起来，这就是**事件驱动**，所谓的reactor模式。

```less
// 根据reader_tpacketv3_start函数我们可以知道
// 
LOCAL void *reader_tpacketv3_thread(gpointer infov)
{
    long info = (long)infov;
    struct pollfd pfd; //pollfd结构体指定了一个被监视的文件描述符，可以传递多个结构体，指示poll()监视多个文件描述符
    int pos = -1;

    memset(&pfd, 0, sizeof(pfd));
    pfd.fd = infos[info].fd; //文件描述符
    pfd.events = POLLIN | POLLERR; //等待的事件：有数据可读或者报错
    pfd.revents = 0;  // 内核在调用返回时设置这个域，用以指示实际发生了的事件；

    MolochPacketBatch_t batch; // 创建一个存储多个packet的“批次”容器
    moloch_packet_batch_init(&batch);

    while (!config.quitting) {
        // 因为接下来即将对infos[info]进行相关操作，局部变量：pos起标志定位的作用（集体到了哪一个infos[info]）
        if (pos == -1) {
            MOLOCH_LOCK(infos[info].lock);
            // 我对pos的理解还没有很深刻，我将其输出截图在下方：
            pos = infos[info].nextPos; //指向下一个infos[info]结构
            infos[info].nextPos = (infos[info].nextPos + 1) % infos[info].req.tp_block_nr; //// 内存块数量
            MOLOCH_UNLOCK(infos[info].lock);
        }
/*struct tpacket_block_desc {
        __u32 version;
         __u32 offset_to_priv;
        union tpacket_bd_header_u hdr;
 }*/

        //联系之前多分配的内存块
        /* infos[info].rd[pos].iov_base所指向的是之前所列举的结构：
        struct iovec {
	 			//Starting address (内存起始地址）
  	 			void  *iov_base;   
    			//Number of bytes to transfer（这块内存长度） 
   				size_t iov_len;    
		};
        union tpacket_bd_header_u {
			struct tpacket_hdr_v1 bh1;
		};		
		*/

        struct tpacket_block_desc *tbd = infos[info].rd[pos].iov_base; // 内存起始地址传递给内存块的头部结构
        
        if (config.level == TRACE_LEVEL) {
            int i;
            int cnt = 0;
            int waiting = 0;
		   // struct iovec定义了一个向量元素。通常，这个结构用作一个多元素的数组。对于每一个传输的元素，指针成员iov_base指向一个缓冲区，这个缓冲区是存放的是readv所接收的数据或是writev将要发送的数据。成员iov_len在各种情况下分别确定了接收的最大长度以及实际写入的长度。
            for (i = 0; i < (int)infos[info].req.tp_block_nr; i++) {
                //将iov_base所指向的缓冲区传递给stbd结构(一个内存块)
                struct tpacket_block_desc* stbd = infos[info].rd[i].iov_base;  // 内存起始地址传递给内存块的头部结构
                if (stbd->hdr.bh1.block_status & TP_STATUS_USER) {
                    cnt++;
                    waiting += stbd->hdr.bh1.num_pkts;
                }
            }

            DPS_LOG_TRACE("Stats pos:%d info:%ld status:%x waiting:%d total cnt:%d total waiting:%d", pos, info, tbd->hdr.bh1.block_status, tbd->hdr.bh1.num_pkts, cnt, waiting);
        }

        // Wait until the block is owned by moloch
        // 此处的状态我都成列在了下3的代码块中
        if ((tbd->hdr.bh1.block_status & TP_STATUS_USER) == 0) {
            // 进行轮询管理多个描述符，根据描述符的状态进行处理
            // nfds:用来指定第一个参数数组元素个数

            poll(&pfd, 1, -1);
            continue;
        }
		// TPACKET_V3环形缓冲区为每个帧的头部结构
        struct tpacket3_hdr *th;

        th = (struct tpacket3_hdr *) ((uint8_t *) tbd + tbd->hdr.bh1.offset_to_first_pkt);
        uint16_t p;
        //开始保存所抓取到的数据包：并且存储于packet中；

        for (p = 0; p < tbd->hdr.bh1.num_pkts; p++) {
            // 必须满足：blk_len <= tp_block_size

            if (unlikely(th->tp_snaplen != th->tp_len)) {
                DPS_LOGEXIT("ERROR - Moloch requires full packet captures caplen: %d pktlen: %d\n"
                    "See https://molo.ch/faq#moloch_requires_full_packet_captures_error",
                    th->tp_snaplen, th->tp_len);
            }
			// 开始丰富相关的packet信息，此处已经完成了抓取数据流程：
             // tp_mac：以太网MAC字段距离帧头的偏移量
            MolochPacket_t *packet = MOLOCH_TYPE_ALLOC0(MolochPacket_t);
            packet->pkt           = (u_char *)th + th->tp_mac; //从mac地址后开始取数据包的信息
            packet->pktlen        = th->tp_len;  // 帧的理论长度 此时的理论长度其实和实际长度就是一样的了
            packet->ts.tv_sec     = th->tp_sec;
            packet->ts.tv_usec    = th->tp_nsec/1000;
            packet->readerPos     = info;  // 本次函数的参数定位：long info = (long)infov;
		   
            // 链路数据统计：总的数据长度 + 数据包数量
            linkStat[info].totalBytes[0] += packet->pktlen;
            linkStat[info].totalPkt++;
		   // tp_status：帧状态
    
            if ((th->tp_status & TP_STATUS_VLAN_VALID) && th->hv1.tp_vlan_tci) {
                packet->vlan = th->hv1.tp_vlan_tci & 0xfff; //uint32_t 	tp_vlan_tci
            }
            // config.trafficStats：open traffic stats 标记：打开流量统计
            if (!config.trafficStats)
                moloch_packet_batch(&batch, packet);

            th = (struct tpacket3_hdr*)((uint8_t*)th + th->tp_next_offset);
        }
        moloch_packet_batch_flush(&batch);
        tbd->hdr.bh1.block_status = TP_STATUS_KERNEL;
        pos = -1;
    }
    return NULL;
}
```

结构测试记录：

![image-20201217181851760](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201217181851760.png)

![image-20201218094335328](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201218094335328.png)

<img src="C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201218120328536.png" alt="image-20201218120328536" style="zoom:80%;" />

###### 3、2中所需结构细节

此处用以记录上述 2 中所需确认的结构细节

```less
/*****************************************************************/

/* Rx ring - header status */
#define TP_STATUS_KERNEL		      0  //内核态
#define TP_STATUS_USER			(1 << 0) //用户态
#define TP_STATUS_COPY			(1 << 1)
#define TP_STATUS_LOSING		(1 << 2)
#define TP_STATUS_CSUMNOTREADY		(1 << 3)
#define TP_STATUS_VLAN_VALID		(1 << 4) /* auxdata has valid tp_vlan_tci */
#define TP_STATUS_BLK_TMO		(1 << 5)
#define TP_STATUS_VLAN_TPID_VALID	(1 << 6) /* auxdata has valid tp_vlan_tpid */
#define TP_STATUS_CSUM_VALID		(1 << 7)
/* Tx ring - header status */
#define TP_STATUS_AVAILABLE	      0
#define TP_STATUS_SEND_REQUEST	(1 << 0)
#define TP_STATUS_SENDING	(1 << 1)
#define TP_STATUS_WRONG_FORMAT	(1 << 2)
/* Rx and Tx ring - header status */
#define TP_STATUS_TS_SOFTWARE		(1 << 29)
#define TP_STATUS_TS_SYS_HARDWARE	(1 << 30) /* deprecated, never set */
#define TP_STATUS_TS_RAW_HARDWARE	(1 << 31)
/* Rx ring - feature request bits */
#define TP_FT_REQ_FILL_RXHASH	0x1
/*****************************************************************/

struct tpacket_bd_ts {
	unsigned int ts_sec;
	union {
		unsigned int ts_usec;
		unsigned int ts_nsec;
	};
};

struct tpacket_hdr_v1 {
	__u32	block_status;
	__u32	num_pkts;
	__u32	offset_to_first_pkt;
	/* Number of valid bytes (including padding)
	 * blk_len <= tp_block_size
	 */
	__u32	blk_len;
	/*
	 * Quite a few uses of sequence number:
	 * 1. Make sure cache flush etc worked.
	 *    Well, one can argue - why not use the increasing ts below?
	 *    But look at 2. below first.
	 * 2. When you pass around blocks to other user space decoders,
	 *    you can see which blk[s] is[are] outstanding etc.
	 * 3. Validate kernel code.
	 */
	__aligned_u64	seq_num; //  sequence number 序列号
	/*
	 * ts_last_pkt:
	 *
	 * Case 1.	Block has 'N'(N >=1) packets and TMO'd(timed out)
	 *		ts_last_pkt == 'time-stamp of last packet' and NOT the
	 *		time when the timer fired and the block was closed.
	 *		By providing the ts of the last packet we can absolutely
	 *		guarantee that time-stamp wise, the first packet in the
	 *		next block will never precede the last packet of the
	 *		previous block.
	 * Case 2.	Block has zero packets and TMO'd
	 *		ts_last_pkt = time when the timer fired and the block
	 *		was closed.
	 * Case 3.	Block has 'N' packets and NO TMO.
	 *		ts_last_pkt = time-stamp of the last pkt in the block.
	 *
	 * ts_first_pkt:
	 *		Is always the time-stamp when the block was opened.
	 *		Case a)	ZERO packets
	 *			No packets to deal with but atleast you know the
	 *			time-interval of this block.
	 *		Case b) Non-zero packets
	 *			Use the ts of the first packet in the block.
	 *
	 */
	struct tpacket_bd_ts	ts_first_pkt, ts_last_pkt;
};

union tpacket_bd_header_u {
	struct tpacket_hdr_v1 bh1;
};

struct tpacket_block_desc {
	__u32 version;
	__u32 offset_to_priv;
	union tpacket_bd_header_u hdr;
};
```

IO复用函数原型：

```less
int poll(struct pollfd fdarray[], nfds_t nfds, int timeout);
int select(int maxfdp1, fd_set restrict readfds, fd_set *restrict expectfds, struct timeval restrict tvptr);

　　其中poll函数中，结构pollfd如下：
　　struct pollfd{
　　　　int fd； //file descriptor
　　　　short event；//event of interest on fd
　　　  short revent；//event that occurred on fd

　　}
```



###### 4、IP分片

![image-20220817115848093](C:\Users\月久矢\AppData\Roaming\Typora\typora-user-images\image-20220817115848093.png)

###### 网络字节序对比

![image-20210317141741853](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20210317141741853.png)

网络号是网段中的第一个地址192.168.8.0，广播地址是网段中的最后一个地址192.168.8.255，这两个地址是不能配置在计算机主机上的

###### struct结构详情

![image-20201208154307460](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201208154307460.png)

```c
#include <linux/ip.h>
#include <linux/tcp.h>
tcp头部数据结构

struct tcphdr {
 __be16 source;     //16位源端口号
 __be16 dest;       //16位目的端口号

                      //每个tcp段都包源和目的端口号，用于寻找发送端和接受端的应用进程。这两个端口号加上ip报头中的源ip和目的ip，来确定一个唯一的TCP连接。
 __be32 seq;        //此次发送的数据在整个报文段中的起始字节数。此序号用来标识从tcp发送端向tcp接受端发送的数据字节流，seq表示在这个报文段中的第一个数据字节。如果将字节流看做在两个应用程序间的单向流动，则tcp用序号对每个字节进行计数。32 bit的无符号数。为了安全起见，它的初始值是一个随机生成的数，它到达2的32次方-1后又从零开始。
 __be32 ack_seq;    //是下一个期望接收的字节，确认序号应当是上次已成功接收的序号+1，只有ack标志为1时确认序号字段才有效。一旦一个连接已经建立了，ack总是=1
#if defined(__LITTLE_ENDIAN_BITFIELD) //小端
 __u16 res1:4, // 保留位
  doff:4,  //tcp头部长度，指明了在tcp头部中包含了多少个32位的字。由于options域的长度是可变的，所以整个tcp头部的长度也是变化的。4bit可表示最大值15，故15*32=480bit=60字节，所以tcp首部最长60字节。然后，没有任选字段，正常的长度是20字节
  fin:1, //发端完成发送任务
  syn:1, //同步序号用来发起一个连接
  rst:1, //重建连接
  psh:1, //接收方应该尽快将这个报文段交给应用层
  ack:1,  //一旦一个连接已经建立了，ack总是=1

  urg:1, //紧急指针有效
  ece:1, 
  cwr:1;
#elif defined(__BIG_ENDIAN_BITFIELD)
 __u16 doff:4,
  res1:4,
  cwr:1,
  ece:1,
  urg:1,
  ack:1,
  psh:1,
  rst:1,
  syn:1,
  fin:1;
#else
#error "Adjust your <asm/byteorder.h> defines"
#endif 
 __be16 window;  //窗口大小，单位字节数，指接收端正期望接受的字节，16bit，故窗口大小最大为16bit=1111 1111 1111 1111（二进制）=65535（十进制）字节
 __sum16 check; //校验和校验的是整个tcp报文段，包括tcp首部和tcp数据，这是一个强制性的字段，一定是由发端计算和存储，并由收端进行验证。
 __be16 urg_ptr;
};

typedef uint32_t tcp_seq;
struct tcphdr {
		uint16_t th_sport;		/* source port */
		uint16_t th_dport;		/* destination port */
    //序号：四字节，TCP是面向字节的，它会为每一个字节编一个号码，其中这里的序号是本报文段发送的数据的第一个字节的序号。
		tcp_seq	  th_seq;		/* sequence number */
    //确认序号：四字节，是期望收到下一个报文段第一个字节的序号，同时是对上一组序号的确认。例如：A向B发送数据，序号为200，报文段长200字节。那么B向A恢复的确认序号就是401，表示需要401开头的数据，同时确认收到前面的数据。
		tcp_seq	  th_ack;		/* acknowledgement number */
	#if BYTE_ORDER == LITTLE_ENDIAN
		/*LINTED non-portable bitfields*/
		uint8_t  th_x2:4,		/* (unused) */
			  th_off:4;		/* data offset */
	#endif
	#if BYTE_ORDER == BIG_ENDIAN
		/*LINTED non-portable bitfields*/
		uint8_t  th_off:4,		/* data offset */
			  th_x2:4;		/* (unused) */
	#endif
		uint8_t  th_flags;
	#define	TH_FIN	  0x01
	#define	TH_SYN	  0x02
	#define	TH_RST	  0x04
	#define	TH_PUSH	  0x08
	#define	TH_ACK	  0x10
	#define	TH_URG	  0x20
	#define	TH_ECE	  0x40
	#define	TH_CWR	  0x80
		uint16_t th_win;			/* window */
		uint16_t th_sum;			/* checksum */  //校验和校验的是整个tcp报文段，包括tcp首部和tcp数据，这是一个强制性的字段，一定是由发端计算和存储，并由收端进行验证。
		uint16_t th_urp;			/* urgent pointer */
	} __packed;




//IP头部，总长度20字节
typedef struct _ip_hdr
{
	#if LITTLE_ENDIAN
	unsigned char ihl:4;     //首部长度
	unsigned char version:4, //版本 
	#else
	unsigned char version:4, //版本
	unsigned char ihl:4;     //首部长度
	#endif
	unsigned char tos;       //服务类型
	unsigned short tot_len;  //总长度
	unsigned short id;       //标志
	unsigned short frag_off; //分片偏移
	unsigned char ttl;       //生存时间
	unsigned char protocol;  //协议
	unsigned short chk_sum;  //检验和
	struct in_addr srcaddr;  //源IP地址
	struct in_addr dstaddr;  //目的IP地址
}ip_hdr;

//IP报头
typedef struct ip
{
	u_int ip_v:4; //version(版本)
	u_int ip_hl:4; //header length(报头长度)
	u_char ip_tos; // 服务类型----网络控制和优先权：3位优先权字段(现已被忽略) + 4位TOS字段 + 1位保留字段(须为0)。4位TOS字段分别表示最小延时、最大吞吐量、最高可靠性、最小费用，其中最多有一个能置为1。应用程序根据实际需要来设置 TOS值，如ssh和telnet这样的登录程序需要的是最小延时的服务，文件传输ftp需要的是最大吞吐量的服务
	u_short ip_len;//长度
	u_short ip_id; //标识段，标识是第几个ip包
	u_short ip_off;//标志段，与标记字段和分片偏移字段一起用于IP报文的分片，这个字段还将在同一原始文件被分片的报文上打上相同的标记，以便接收设备可以识别出属于同一个报文的分片
//#define    IP_DF 0x4000            //是否分片，第二位为1(不可分包) 
//#define    IP_MF 0x2000            /* more fragments flag *///
//#define    IP_OFFMASK 0x1fff       //段移位或后继分片，即二三位为00(可分包,此为最后一个包)或01(还有后继包)
	u_char ip_ttl; // 生命周期
	u_char ip_p;  //上层使用协议，即传输协议
	u_short ip_sum; //校验和  当接受到IP数据包时，要检查IP头是否正确，则对IP头进行检验
	struct in_addr ip_src; //源ip地址
	struct in_addr ip_dst; //目的ip地址
}IP_HEADER;
struct ip{
    unsigned char version:4;       // 版本
    unsigned char hlen:4;        // 首部长度
    unsigned char tos;             // 服务类型
    unsigned short len;         // 总长度
    unsigned short id;            // 标识符
    unsigned short offset;        // 标志和片偏移
    unsigned char ttl;            // 生存时间
    unsigned char protocol;     // 协议
    unsigned short checksum;    // 校验和
    struct in_addr ipsrc;        // 32位源ip地址
    struct in_addr ipdst;       // 32位目的ip地址
};

struct in_addr {
    in_addr_t s_addr;
};
/*
表示一个32位的IPv4地址。
in_addr_t一般为32位的unsigned int，其字节顺序为网络字节序，即该无符号数采用大端字节序。其中每8位表示一个IP地址中的一个数值。打印的时候可以调用inet_ntoa()函数将其转换为char*类型。头文件为：#include <arpa/inet.h>
*/
```

![image-20201207194604588](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201207194604588.png)

**tcp重组发生于parser/tcp.c，而ip分片重组发生在packet入全局队列前；**

```less
    if ((ip_flags & IP_MF) || ip_off > 0) {  //IP_MF分片标志为'1',或者ip_off=ip_off &= IP_OFFMASK的偏移量是不是大于0（此处主要针对于最后一个分片数据包）
        //ip分片位置，IP_MF标志这是最后一个分片
        //除了最后一片外，其他每个组成数据报的片都要把该比特置 1 。
        moloch_packet_frags4(batch, packet);
        return MOLOCH_PACKET_DONT_PROCESS_OR_FREE;
    }


TCP/IP协议组，分为四个层次：网络接口层、网络层、传输层和应用层。

网络层：IP协议、ICMP协议、ARP协议、RARP协议和BOOTP协议。

传输层：TCP协议与UDP协议。

应用层：HTTP,FTP、TELNET、SMTP、DNS等协议。


在分片的数据中，传输层的首部只会出现在第一个分片中，IP数据报分片后，只有第一片带有传输层首部(UDP或ICMP等)，后续分片只有IP首部和应用数据，到了目的地后根据IP首部中的信息在网络层进行重组，这一步骤对上层透明，即传输层根本不知道IP层发生了分片与重组。而TCP报文段的每个分段中都有TCP首部，到了目的地后根据TCP首部的信息在传输层进行重组。

TCP分段仅发生在发送端，这是因为在传输过程中，TCP分段是先被封装成IP数据报，再封装在以太网帧中被链路所传输的，并且在端到端路径上通常不会有工作在三层以上，即传输层的设备，故TCP分段不会发生在传输路径中间的某个设备中，在发送端TCP传输层分段后，在接收端TCP传输层重组。

IP分片不仅会发生在在使用UDP、ICMP等没有分段功能的传输层协议的数据发送方，更还会发生在传输途中，甚至有可能都会发生，这是因为原本的大数据报被分片后很可能会经过不同MTU大小的链路，一旦链路MTU大于当前IP分片大小，则需要在当前转发设备(如路由器)中再次分片，但是各个分片只有到达目的地后才会在其网络层重组，而不是像其他网络协议，在下一跳就要进行重组。
```

```less
htonl(), ntohl(), htons(), ntohs() 函数
在C/C++写网络程序的时候，往往会遇到字节的网络顺序和主机顺序的问题。这是就可能用到htons(), ntohl(), ntohs()，htons()这4个函数。
网络字节顺序与本地字节顺序之间的转换函数：

htonl()--"Host to Network Long"
ntohl()--"Network to Host Long"
htons()--"Host to Network Short"
ntohs()--"Network to Host Short"
```

###### ip分片进行重组位置

```less
    MolochPacket_t * fpacket;
    MolochFrags_t* frags;		//fragment分片
    char             key[10];

    struct ip * const ip4 = (struct ip*)(packet->pkt + packet->ipOffset);
    memcpy(key, &ip4->ip_src.s_addr, 4);
    memcpy(key+4, &ip4->ip_dst.s_addr, 4);
    memcpy(key+8, &ip4->ip_id, 2); //IPid占16bit，即2字节
    //key值存储了相关的ip源、目的地址与ip_id号
    HASH_FIND(fragh_, fragsHash, key, frags);
    //fragh_为表明前缀，fragsHash：hash表头；key为用以计算的hash值；frags为用来保存搜寻到（或空）的MolochPacket_t结构
    // 如果是第一次收到的分片，run this code
    // frags-----fragment(分片、分段)
    if (!frags) {
        frags = MOLOCH_TYPE_ALLOC0(MolochFrags_t);
        memcpy(frags->key, key, 10);
        frags->secs = packet->ts.tv_sec;
        HASH_ADD(fragh_, fragsHash, key, frags); //将行的分片信息加入到fragsHash哈希表中；
        DLL_PUSH_TAIL(fragl_, &fragsList, frags);  //hash表的桶存储形式是以双链表展开的。由此将新分片结构保存于fragsList链表中
        DLL_INIT(packet_, &frags->packets);  // 处理好分片的存储形式后，还需要初始化定义每个分片的数据保存结构，该部分由MolochPacketHead_t的packet头控制开始，其以单链表的形式存储数据包的具体信息（标志位与传输的data）
        DLL_PUSH_TAIL(packet_, &frags->packets, packet); //将对应抓取到的packet存储到与之对应的分片中；

        if (DLL_COUNT(fragl_, &fragsList) > config.maxFrags) {
            droppedFrags++;
            moloch_packet_frags_free(DLL_PEEK_HEAD(fragl_, &fragsList));  //更新分片计数，并且free对应的fragsList，以方便下次有新的分时可以使用；
        }
        return FALSE;
    } else {
        // 我们需要知道的是，通过hash值所计算的key值，包含三个部分：源ip，目的ip，ip_id,其中ip_id是具有唯一标识型的，具有相同ipid的数据分片将会用来进程“IP重组”，所以其实hash值就已经标志了是不是一个ip分片，而为什么对应的桶（双链表）
        // 中还需要使用对应的单链表MolochPacketHead_t     packets;   是为了方便其后进行ip的数据重组而使用；
        DLL_MOVE_TAIL(fragl_, &fragsList, frags); //如果所遇到的ip数据包其key值计算后包含在原有的hash表中，则将其保存至对应hash桶的双链表分片下的对于分片数据包；
    }

    uint16_t          ip_off = ntohs(ip4->ip_off);
    uint16_t          ip_flags = ip_off & ~IP_OFFMASK;  
    ip_off &= IP_OFFMASK; ///与操作后ip_off存储着ip分片偏移量


    // we might be done once we receive the packets with no flags
    // 可能接受到没有flags的数据包
    if (ip_flags == 0) {
        frags->haveNoFlags = 1;  
    }

    //此处为插入IP分片数据
    // Insert this packet in correct location sorted by offset
    DLL_FOREACH_REVERSE(packet_, &frags->packets, fpacket) {  //从尾部逐个获取packet结构存于fpacket中
        //以下代码可以配和之后的截图理解
        
        struct ip *fip4 = (struct ip*)(fpacket->pkt + fpacket->ipOffset); //将pkt[0:]的ip头数据都存储于结构体中；
        uint16_t fip_off = ntohs(fip4->ip_off) & IP_OFFMASK; //ntohs()--"Network to Host Short"
        if (ip_off >= fip_off) {  // 可以思索以下为什么一个条件比较就可以将ip分片插入对应位置
            // 此处则用到了双向链表中的所存储frags->packets单向对列；用来处理IP分片重组
            DLL_ADD_AFTER(packet_, &frags->packets, fpacket, packet); //找到分片要插入的位置
            break;
        }
    }

//尾部开始读取
   #define DLL_FOREACH_REVERSE(name,head,element) \
       for ((element) = (head)->name##prev; \
           (element) != (void *)(head); \
           (element)=(element)->name##prev)
//
   #define DLL_ADD_AFTER(name,head,after,element) \
        ((element)->name##next            = (after)->name##next, \
        (element)->name##prev            = (after), \
        (after)->name##next->name##prev  = (element), \  //注意想象双向队列上下队列指针方向时相反的，所以此处是此操作
        (after)->name##next              = (element), \
        (head)->name##count++ \
    )


//上面的判断语句之所以成立是因为ip_off进行了偏移量IP_OFFMASK的与操作后，则只保留了ip分片的偏移量数值。之后再与逐个读取(c从尾部开始读取）的packet元素的偏移量进行比较，直到遇到新元素的偏移量大于某一元素时，再进行相关的插入（插入到大于元素的之后）操作。由此，我们可以把该队列想象成一个顺序队列（递增);
```

###### ip分片校验处

```less
    int off = 0;
    struct ip *fip4;

    int payloadLen = 0;
    
    //此处开始校验IP分片数据
    DLL_FOREACH(packet_, &frags->packets, fpacket) {  //从头部逐个获取packet结构存于fpacket中
        fip4 = (struct ip*)(fpacket->pkt + fpacket->ipOffset);
        uint16_t fip_off = ntohs(fip4->ip_off) & IP_OFFMASK;
        if (fip_off != off) 
            break;
        off += fpacket->payloadLen/8;
        payloadLen = MAX(payloadLen, fip_off*8 + fpacket->payloadLen);
    }

    // We have a hole
    // 经过上面的循环读取，如果我们的fpacket队列中还不是头部的话，就证明刚刚的偏移校验中出现了ip分片的丢失现象
    if ((void*)fpacket != (void*)&frags->packets) {
        return FALSE;
    }
	
	// 如果出现的ip分片过于多，则认定为黑客攻击，开始释放相关的内存空间
    // payloadOffset为pqyload的开始位置，payloadLen记录payload的数据长度
    if (payloadLen + packet->payloadOffset >= MOLOCH_PACKET_MAX_LEN) {
        droppedFrags++;
        moloch_packet_frags_free(frags);
        return FALSE;
    }

//第一次循环时，先判断偏移量是不是0（即开头ip分片），之后随着更新偏移量，也会不断校验新来的数据分片是否满足偏移量匹配，如果不满足，则直接break
```

###### ip数据拷贝

```less
 // 开始分配知道长度（ip头+payload）内存空间
    // Now alloc the full packet
    packet->pktlen = packet->payloadOffset + payloadLen; //将最后一个数据包的数据长度加入到总包长中
    uint8_t *pkt = malloc(packet->pktlen);

    // Copy packet header
    // 先拷贝数据包头
    memcpy(pkt, packet->pkt, packet->payloadOffset);
    // 对totalMallocPkt进行自增packet->payloadOffset的大小;
    MOLOCH_THREAD_INCR_NUM(totalMallocPkt, packet->payloadOffset);//alter by zy

    // Fix header of new packet
    // 因为对于packet在经过重组数据信息之后，其对应的字节计数也需要更新；
    fip4 = (struct ip*)(pkt + packet->ipOffset);
    fip4->ip_len = htons(payloadLen + 4*ip4->ip_hl); //ip_hl为首部长度，ip_len为总长度
    fip4->ip_off = 0; // 数据包的偏移地址置为0；

    // Copy payload
    // 开始拷贝payload部分的数据至pkt中
    DLL_FOREACH(packet_, &frags->packets, fpacket) {
        fip4 = (struct ip*)(fpacket->pkt + fpacket->ipOffset);
        uint16_t fip_off = ntohs(fip4->ip_off) & IP_OFFMASK; //fip_off存储了数据偏移量，方便定位拷贝位置

        if (packet->payloadOffset+(fip_off*8) + fpacket->payloadLen <= packet->pktlen) //当（原长度+需要添加的新数据） < (记录的总数据长时)  在此处也体现了pktlen记录的是数据bit长度，而fip_off存储的是字节长度
            memcpy(pkt+packet->payloadOffset+(fip_off*8), fpacket->pkt+fpacket->payloadOffset, fpacket->payloadLen); // 开始根据新payload的长度来递增数据
        else
            DPS_LOG_WARN("WARNING - Not enough room for frag %d > %d", packet->payloadOffset + (fip_off * 8) + fpacket->payloadLen, packet->pktlen);
    }

    // Set all the vars in the current packet to new defraged packet
    // 拷贝标志位为1；则说明该数据包已经拷贝完成，直接进行释放该数据包；
    if (packet->copied) {
        free(packet->pkt);
        //printf("moloch_packet_frags_process free \n");
        MOLOCH_THREAD_INCR_NUM(totalFreePkt, packet->pktlen + sizeof(MolochPacket_t));
    }
    packet->pkt = pkt; //更新数据包指针
    packet->copied = 1; //将数据包“拷贝位”置为1,表示该数据包已经经过拷贝;
    packet->wasfrag = 1; // 数据包“分片确认位”置为1 标记改数据包是分片;
    packet->payloadLen = payloadLen; //更新重组后的数据包长度
    DLL_REMOVE(packet_, &frags->packets, packet); // Remove from list so we don't get freed in frags_free  //删除frags双向链表中的数据包
    moloch_packet_frags_free(frags); //释放frags双向链表内存空间
```



###### ip分片缺失清除代码：

![image-20201224102259375](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201224102259375.png)



###### 5、tcp流重组

###### 如何确定重组包

​		至于收到一个报文后如何确定它是一个"TCP segment"？如果有几个报文的ACK序号都一样，并且这些报文的Sequence Number都不一样，并且后一个Sequence Number为前一个Sequence Number加上前一个报文大小再加上len的话，肯定是TCP segment了，对于没有ACK标志时，则无法判断。

![image-20201226154835477](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201226154835477.png)

```less
typedef uint32_t tcp_seq;
struct tcphdr {
		uint16_t th_sport;		/* source port */
		uint16_t th_dport;		/* destination port */
    //序号：四字节，TCP是面向字节的，它会为每一个字节编一个号码，其中这里的序号是本报文段发送的数据的第一个字节的序号。
		tcp_seq	  th_seq;		/* sequence number */
    //确认序号：四字节，是期望收到下一个报文段第一个字节的序号，同时是对上一组序号的确认。例如：A向B发送数据，序号为200，报文段长200字节。那么B向A恢复的确认序号就是401，表示需要401开头的数据，同时确认收到前面的数据。
		tcp_seq	  th_ack;		/* acknowledgement number */
	#if BYTE_ORDER == LITTLE_ENDIAN
		/*LINTED non-portable bitfields*/
		uint8_t  th_x2:4,		/* (unused) */
    // // tcpip 协议头是变长的，它可以是 0 个字节（虽然这不可能）这取决于协议头中 “th_off” 字段的值（ “th_off * 4 = nTcpHeaderCount”）
			  th_off:4;		/* data offset */
	#endif
	#if BYTE_ORDER == BIG_ENDIAN
		/*LINTED non-portable bitfields*/
		uint8_t  th_off:4,		/* data offset */
			  th_x2:4;		/* (unused) */
	#endif
		uint8_t  th_flags;
	#define	TH_FIN	  0x01   终止位FIN：当该位置一的时候，说明数据已经传输完毕，请求释放连接。
	#define	TH_SYN	  0x02   同步位SYN：当该位为一的时候，说明这是一个请求连接或者连接接受报文，当A向B请求建立连接的时候，SYN=1，ACK=0。当B同意和A建立连接的时候，SYN=1，ACK=1.表示接受连接。
	#define	TH_RST	  0x04   复位位RST：当该位置一的时候，说明TCP连接出现了问题，需要断开连接重新建立连接。在TCP协议中RST表示复位，用来关闭异常的连接，在TCP的设计中它是不可或缺的。发送RST包关闭连接时，不必等缓冲区的包都发出去，直接就丢弃缓存区的包发送RST包。而接收端收到RST包后，也不必发送ACK包来确认。
	#define	TH_PUSH	  0x08   推送为PSH：当该位置一的时候，需要尽快将数据传入上层应用，而不用等整个缓存满了在上传。例如对于tcp分片的最后一片则会使用该标志；
	#define	TH_ACK	  0x10   确认位ACK：当ACK为一的时候，确认序号才为有效的序号。否则确认序号无效。
	#define	TH_URG	  0x20   当该位为一的时候，表示紧急指针启用。说明该报文中有紧急数据，需要尽快传入上层应用，数据的第一位到紧急指针的位置就是紧急数据。
	#define	TH_ECE	  0x40
	#define	TH_CWR	  0x80
		uint16_t th_win;			/* window */
		uint16_t th_sum;			/* checksum */  //校验和校验的是整个tcp报文段，包括tcp首部和tcp数据，这是一个强制性的字段，一定是由发端计算和存储，并由收端进行验证。
		uint16_t th_urp;			/* urgent pointer */
	} __packed;
```



```less
LOCAL        int      mProtocolCnt;
MolochProtocol_t      mProtocols[0x100];

#define moloch_mprotocol_register(name, ses, createSessionId, preProcess, process, sFree) moloch_mprotocol_register_internal(name, ses, createSessionId, preProcess, process, sFree, sizeof(MolochSession_t), MOLOCH_API_VERSION)


int moloch_mprotocol_register_internal(char                            *name,
                                       int                              ses,
                                       MolochProtocolCreateSessionId_cb createSessionId,
                                       MolochProtocolPreProcess_cb      preProcess,
                                       MolochProtocolProcess_cb         process,
                                       MolochProtocolSessionFree_cb     sFree,
                                       size_t                           sessionsize,
                                       int                              apiversion)
{
    ……………………
    // 全文只有此处会对mProtocolCnt进行修改；
    int num = ++mProtocolCnt; // Leave 0 empty so we know if not set in code
    mProtocols[num].name = name;
    mProtocols[num].ses = ses;
    mProtocols[num].createSessionId = createSessionId;
    mProtocols[num].preProcess = preProcess;
    mProtocols[num].process = process;
    mProtocols[num].sFree = sFree;
    return num;
}


//tcp.c中
typedef enum {
    SESSION_TCP,
    SESSION_UDP,
    SESSION_ICMP,
    SESSION_SCTP,
    SESSION_ESP,
    SESSION_OTHER,
    SESSION_MAX
} SessionTypes;

void moloch_parser_init()
{
    maxTcpOutOfOrderPackets = moloch_config_int(NULL, "maxTcpOutOfOrderPackets", 256, 64, 10000);

    tcpMProtocol = moloch_mprotocol_register("tcp",
                                             SESSION_TCP,// enum SessionTypes
        // mProtocols[packet->mProtocol].createSessionId(sessionId, packet); packet.c  moloch_packet_process
                                             tcp_create_sessionid,
        // mProtocols[packet->mProtocol].preProcess(session, packet, isNew); packet.c  moloch_packet_process
                                             tcp_pre_process,
        // mProtocols[packet->mProtocol].process(session, packet)  packet.c  moloch_packet_process
                                             tcp_process,
        //  mProtocols[session->mProtocol].sFree(session) packet.c  moloch_session_save  moloch_session_free
                                             tcp_session_free);
}

```

###### 重组代码定位：

![image-20201224101453636](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201224101453636.png)

其中：针对  tcp_sequence_diff  函数：其作用为：定义一个比较TCP序列的顺序

```less
LOCAL int64_t tcp_sequence_diff (int64_t a, int64_t b)
{
    if (a > 0xc0000000 && b < 0x40000000)
        return a + 0x100000000LL - b;

    if (b > 0xc0000000 && a < 0x40000000)
        return a - b - 0x100000000LL;

    return b - a;
}
```

函数参照的项目源代码：[gopacket](https://github.com/google/gopacket)/[tcpassembly](https://github.com/google/gopacket/tree/master/tcpassembly)/**assembly.go** 

```less
Difference定义了一个比较TCP序列的顺序，这种顺序对于滚动是安全的。它返回：
		>0:如果t在s之后
		<0:如果t在s之前
		0：如果t==s
返回的数字是序列差，所以4.difference（8）将返回4。
它通过将uint32空间的第一个四分之一中的任何序列视为在该空间的最后一个四分之一中的任何序列之后来处理滚动，从而包装uint32空间
```

![image-20201224105752664](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201224105752664.png)

对于该函数的验证，所返回的值是两个数字的差值：

![image-20201225141200164](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201225141200164.png)



tcp相关信号量：

```less
Sequence Number是包的序号，用来解决网络包乱序（reordering）问题。
Acknowledgement Number就是ACK——用于确认收到，用来解决不丢包的问题。
Window又叫Advertised-Window，也就是著名的滑动窗口（Sliding Window），用于解决流控的。
TCP Flag，也就是包的类型，主要是用于操控TCP的状态机的。
```

分片pcap实例：前位相对seq后为实际seq

![image-20201226160646748](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201226160646748.png)

![image-20201226164614526](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201226164614526.png)

###### 分片分析：

![tcp分片](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\tcp分片.png)

对于tcp分段的处理：（重复包的处理）

```less
 if (diff == 0) { // 以此判断隐含判断重复数据包（当同一个方向并且位于同一个位置seq时，如果是不同方向时，二者值相同就以为着：这个新来的数据包就是上一次tcp的请求数据）
                if (packet->direction == ftd->packet->direction) { //同一方向  表示是tcp的分段信息
                    if (td->len > ftd->len) { //新来的包大于先前的数据包则替换
                        DLL_ADD_AFTER(td_, tcpData, ftd, td);

                        DLL_REMOVE(td_, tcpData, ftd);
                        moloch_packet_free(ftd->packet);
                        MOLOCH_TYPE_FREE(MolochTcpData_t, ftd);
                        ftd = td;
                    } else { // td->len < ftd->len 新来的包小于先前的数据包则删除td后再直接返回
                        MOLOCH_TYPE_FREE(MolochTcpData_t, td);
                        return 1;
                    }
                    break;
                    // // 经过数据包的抓包查看，目前发现只有一种状况在二者方向相同且当前ack值小于先前的seq值
                } else if (tcp_sequence_diff(ack, ftd->seq) < 0) { //当位于不同的方向，且ack < ftd->seq（新来的包在ftd的前面）
                    printf("确实会出现该状况\n");
                    DLL_ADD_AFTER(td_, tcpData, ftd, td);
                    break;
                }
            } else if (diff > 0) {  // 如果新来的数据包其方向相同时seq值不能与上一个一致，或者方向不一致时不能与前一个包的ack值相同，则证明现在传来的包是提前到达了，则可以直接插入，之后再将其间缺失的包补齐
                DLL_ADD_AFTER(td_, tcpData, ftd, td);
                break;
            }
```

seq与ack的实例：

![image-20201225135837477](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201225135837477.png)

![IMG_20201225_135926](C:\Users\rfx\Documents\Tencent Files\2531589523\FileRecv\IMG_20201225_135926.jpg)

###### packet->direction的解析

​     在TCP数据包的重组过程中，packet->direction时常出现成为判断性元素，所以我们可以回溯查看相关的代码：

![image-20201226155258873](C:\Users\rfx\AppData\Roaming\Typora\typora-user-images\image-20201226155258873.png)

​       由以上的代码我们可以看出，其实针对direction由两部分来确定：session结构中对应的ip1/2是否与源/目地ip相对应；port1/2是否与源/目地端口相对应；如果全部对应，则记为1；反之记为0；





###### <总结>

![image-20220505154222168](C:\Users\月久矢\AppData\Roaming\Typora\typora-user-images\image-20220505154222168.png)

![image-20220505154208758](C:\Users\月久矢\AppData\Roaming\Typora\typora-user-images\image-20220505154208758.png)