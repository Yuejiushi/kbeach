# suricata中StreamingBuffer的操作处理

## 一、结构详细

对于StreamingBuffer的管理，suricata中也是使用得**红黑树**：

```less
RB_HEAD(SBB, StreamingBufferBlock);
RB_PROTOTYPE(SBB, StreamingBufferBlock, rb, SBBCompare);
```

### 1、StreamingBufferBlock 结构体

```less
typedef struct StreamingBufferBlock {
    uint64_t offset;
    RB_ENTRY(StreamingBufferBlock) rb;
    uint32_t len;
} __attribute__((__packed__)) StreamingBufferBlock;
```

### 2、StreamingBufferRegion 结构体

该结构体主要用以**存储重组之后的buffer**。

```less
typedef struct StreamingBufferRegion_ {
    uint8_t *buf;           /**< memory block for reassembly */
    uint32_t buf_size;      /**< size of memory block */
    uint32_t buf_offset;    /**< how far we are in buf_size */
    uint64_t stream_offset; /**< stream offset of this region */
    struct StreamingBufferRegion_ *next;
} StreamingBufferRegion;
```

### 3、StreamingBuffer 结构体

综合结构体

```c
typedef struct StreamingBuffer_ {
    StreamingBufferRegion region; /* 存储还原的数据 */
    struct SBB sbb_tree;          /**< 存储Blocks的红黑树 */
    StreamingBufferBlock *head;   /**< 指向上面红黑树的头部，始终保持树的最小节点位置RB_MIN */
    uint32_t sbb_size;            /**< SBB涵盖的数据大小 */
    uint16_t regions;
    uint16_t max_regions;
} StreamingBuffer;
```

