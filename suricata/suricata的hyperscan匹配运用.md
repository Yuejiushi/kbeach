# suricata的hyperscan运用

## 一、hyperscan注册

### 1.1 MpmHSRegister

```less
void MpmHSRegister(void)
{
    mpm_table[MPM_HS].name = "hs";
    mpm_table[MPM_HS].InitCtx = SCHSInitCtx;
    mpm_table[MPM_HS].InitThreadCtx = SCHSInitThreadCtx;
    mpm_table[MPM_HS].DestroyCtx = SCHSDestroyCtx;
    mpm_table[MPM_HS].DestroyThreadCtx = SCHSDestroyThreadCtx;
    mpm_table[MPM_HS].AddPattern = SCHSAddPatternCS;
    mpm_table[MPM_HS].AddPatternNocase = SCHSAddPatternCI;
    mpm_table[MPM_HS].Prepare = SCHSPreparePatterns;
    mpm_table[MPM_HS].Search = SCHSSearch;
    mpm_table[MPM_HS].PrintCtx = SCHSPrintInfo;
    mpm_table[MPM_HS].PrintThreadCtx = SCHSPrintSearchStats;
    mpm_table[MPM_HS].RegisterUnittests = SCHSRegisterTests;

    /* Set Hyperscan memory allocators */
    SCHSSetAllocators();
}
```

### 1.2 SCHSSetAllocators

我之所以在此处特意标记该函数，是因为此处 hs_set_allocator 是可以设置hyperscan内部对于内存的malloc/free方式的，或许可以用dpdk的分配来提速。

```less
static void SCHSSetAllocators(void)
{
    hs_error_t err = hs_set_allocator(SCHSMalloc, SCHSFree);
    if (err != HS_SUCCESS) {
        FatalError("Failed to set Hyperscan allocator.");
    }
}
```

### 1.3 注册逻辑路线

```less
PostConfLoadedSetup
	MpmTableSetup();   <-----
	SpmTableSetup();
		MpmACRegister();
		MpmACBSRegister();
		MpmACTileRegister();
		MpmHSRegister();    <-----
```

### 1.4 代码总体逻辑

```less
// 《suricata规则匹配源码数量》中3.3.2.1
DetectEngineReload
	SigLoadSignatures
		SigGroupBuild
					|
					|
					|
SigAddressPrepareStage4   PrefilterSetupRuleGroup  PatternMatchPrepareGroup   MpmStorePrepareBuffer  MpmStoreSetup
											   PatternMatchPrepareGroup PrepareMpms MpmStorePrepareBufferAppLayer  MpmStoreSetup
											   PatternMatchPrepareGroup PrepareMpms MpmStorePrepareBufferPkt  MpmStoreSetup
											   PatternMatchPrepareGroup PrepareMpms MpmStorePrepareBufferFrame  MpmStoreSetup
					|
					|
					|
PopulateMpmHelperAddPattern
	MpmAddPatternCI
	MpmAddPatternCS
		mpm_table[mpm_ctx->mpm_type].AddPatternNocase	--> SCHSAddPatternCI
		mpm_table[mpm_ctx->mpm_type].AddPattern  		--> SCHSAddPatternCS
					|
					|
					|
SCHSAddPatternCI / SCHSAddPatternCS
	SCHSAddPattern // ctx->init_hash
		SCHSPattern *p = SCHSAllocPattern(mpm_ctx);
		p->id = pid;
		p->sids[0] = sid; // p->sids[p->sids_size] = sid;
		p->flags = flags;
		p->len = patlen; // 数据长度
		memcpy(p->original_pat, pat, patlen); // 规则 pat == cd->content 即是规则的 content 字段。
		SCHSInitHashAdd(ctx, p)
		mpm_ctx->pattern_cnt++;
```



## 二、SCHSAddPattern

### 2.0 DetectPrefilterSetup （FP设置）

```less
    SigMatch *sm = DetectGetLastSM(s);
    if (sm == NULL) {
        SCLogError("prefilter needs preceding match");
        SCReturnInt(-1);
    }

    /* if the sig match is content, prefilter should act like
     * 'fast_pattern' w/o options. */
    if (sm->type == DETECT_CONTENT) {
        DetectContentData *cd = (DetectContentData *)sm->ctx;
        if ((cd->flags & DETECT_CONTENT_NEGATED) &&
                ((cd->flags & DETECT_CONTENT_DISTANCE) ||
                 (cd->flags & DETECT_CONTENT_WITHIN) ||
                 (cd->flags & DETECT_CONTENT_OFFSET) ||
                 (cd->flags & DETECT_CONTENT_DEPTH)))
        {
            SCLogError("prefilter; cannot be "
                       "used with negated content, along with relative modifiers");
            SCReturnInt(-1);
        }
        cd->flags |= DETECT_CONTENT_FAST_PATTERN; // 快速模式
    } else {
        if (sigmatch_table[sm->type].SupportsPrefilter == NULL) {
            SCLogError("prefilter is not supported for %s", sigmatch_table[sm->type].name);
            SCReturnInt(-1);
        }
        s->flags |= SIG_FLAG_PREFILTER; // 预过滤模式

        /* make sure setup function runs for this type. */
        de_ctx->sm_types_prefilter[sm->type] = true;
    }

    s->init_data->prefilter_sm = sm;

    SCReturnInt(0);
```



### 2.1 pid

```less
// 线索
	cd->id = res->cd->id;
	cd->id = max_id++;

// 位于 DetectSetFastPatternAndItsId
SigGroupBuild
	 DetectSetFastPatternAndItsId
	 ………………
	 SigAddressPrepareStage4(); //  SCHSPattern 的设置

```

#### 2.1.1 DetectSetFastPatternAndItsId

**找出引擎中所有标志的FP及其各自的内容id。**


```less
int DetectSetFastPatternAndItsId(DetectEngineCtx *de_ctx)
{
    /*
    
    		作用1：设置规则的MPM：etrieveFPForSig(de_ctx, s)
    			  	包含了设置MPM中的cd：DetectContentData *cd = (DetectContentData *)s->init_data->mpm_sm->ctx;

    
    */
    uint32_t cnt = 0;
    for (Signature *s = de_ctx->sig_list; s != NULL; s = s->next) {
        // 对应处理“prefilter”字段
        if (s->flags & SIG_FLAG_PREFILTER) // sigmatch_table[DETECT_PREFILTER].name = "prefilter";
            continue;

        RetrieveFPForSig(de_ctx, s); // 下述 ，处理快速模式 fast_pattern
        if (s->init_data->mpm_sm != NULL) {
            s->flags |= SIG_FLAG_PREFILTER; // 也标志预过滤
            cnt++;
        }
    }
    /* no mpm rules */  // cnt 记录了一共有多少条FP规则需要
    if (cnt == 0)
        return 0;
	
   
    
    
    /*
    
    		作用2设置RetrieveFPForSig(de_ctx, s)规则的MPM
    
    */
    HashListTable *ht =
            HashListTableInit(4096, PatternChopHashFunc, PatternChopCompareFunc, PatternFreeFunc);
    BUG_ON(ht == NULL);
    PatIntId max_id = 0;

    for (Signature *s = de_ctx->sig_list; s != NULL; s = s->next) {
        if (s->init_data->mpm_sm == NULL) // 如果规则不存在多模匹配则返回
            continue;

        const int sm_list = s->init_data->mpm_sm_list;
        BUG_ON(sm_list == -1);

        DetectContentData *cd = (DetectContentData *)s->init_data->mpm_sm->ctx;

        DetectPatternTracker lookup = { .cd = cd, .sm_list = sm_list, .cnt = 0, .mpm = 0 };
        DetectPatternTracker *res = HashListTableLookup(ht, &lookup, 0);
        if (res) {
            res->cnt++;
            res->mpm += ((cd->flags & DETECT_CONTENT_MPM) != 0);

            cd->id = res->cd->id;
            SCLogDebug("%u: res id %u cnt %u", s->id, res->cd->id, res->cnt);
        } else {
            DetectPatternTracker *add = SCCalloc(1, sizeof(*add));
            BUG_ON(add == NULL);
            add->cd = cd;
            add->sm_list = sm_list;
            add->cnt = 1;
            add->mpm = ((cd->flags & DETECT_CONTENT_MPM) != 0);
            HashListTableAdd(ht, (void *)add, 0);

            cd->id = max_id++;
            SCLogDebug("%u: add id %u cnt %u", s->id, add->cd->id, add->cnt);
        }
    }

    HashListTableFree(ht);

    return 0;
}
```



#### 2.1.2 RetrieveFPForSig

```less
// 这个函数很复杂，源码可以自行查看
简而言之，如果你的规则种存在字段需要进行预过滤MPM，则会设置对应规则的 s->init_data->mpm_sm 多模匹配链表；
// 一个字段是否需要进行MPM匹配，取决于你的注册，例如 tls.cert.hashes.s， 如果你把对应的下面预过滤函数去掉，那么规则加载时，DetectSetFastPatternAndItsId 就不存在该相应的cnt++; 你可以跟踪对应sig的%p地址来确认。
// 规则如下：alert ip any any <> any any (msg:"2045009-tls.cert.hashes.s";tls.cert.hashes.s;content:"8c90bd2ab3aee60bd0eab786b01ae4b1cc57ef22";rule.type:"link";sid:1030;gid:1030;unit:"unit123";group_name:"x509_03.pcap";taskid:"wyz+LLC+Len+Flag+TTL+Float";)
 	DetectAppLayerMpmRegister2("tls.cert.hashes.s", SIG_FLAG_TOSERVER, 2, 
            PrefilterGenericMpmRegister, GetData, ALPROTO_TLS, TLS_CERT_DONE);
    DetectAppLayerMpmRegister2("tls.cert.hashes.s", SIG_FLAG_TOCLIENT, 2,
            PrefilterGenericMpmRegister, GetData, ALPROTO_TLS, TLS_CERT_DONE);

```





### 2.2 sid

```less
/* sid 是规则加载到de_ctx经过排序后的序号(唯一) */

第一步：规则解析时：
	《suricata规则匹配源码数量》中2.2
LoadSignatures
  SigLoadSignatures
	ProcessSigFiles  ------------------------------------------------------------------- step 1
		SigInitHelper
		{
        	Signature *sig = SigAlloc();
        	int ret = SigParse(de_ctx, sig, sigstr, dir, &parser);
        	sig->num = de_ctx->signum; // 规则序号
    		de_ctx->signum++;// 规则数增加
		}

第二步：重排序
					------------------------------------------------------------------- step 2
	SCSigRegisterSignatureOrderingFuncs(de_ctx); // 设置比较方法，可以注册多个，first比较为0这next；
	SCSigOrderSignatures(de_ctx); // 开始排序，如果排序函数cmp_func_list里面的比较函数比较不出来，就根据SCSigLessThan的sid来进行从小到大的排序。
 	SCSigSignatureOrderingModuleCleanup(de_ctx);


第三步：重排序后重新更新规则序号
LoadSignatures
  SigLoadSignatures
	SigGroupBuild 	 ------------------------------------------------------------------- step 3
	{
        // 对规则序号进行重排序
        de_ctx->signum = 0;
    	while (s != NULL) {
        	s->num = de_ctx->signum++;
        	s = s->next;
    	}
	}

由此，sum确定；
```







## 三、hyperscan的编译

### 2.1 编译模式

hs_compile_ext_multi 函数进行编译，设置相应匹配拓展参数

#### 2.1.1 SCHSPreparePatterns

```less
mpm_table[MPM_HS].Prepare = SCHSPreparePatterns;
```



### 2.2 多工作线程的暂存空间

```less
SigGroupBuild
	DetectMpmPrepareAppMpms
	DetectMpmPrepareFrameMpms
	DetectMpmPreparePktMpms
	DetectMpmPrepareBuiltinMpms
		MpmStoreSetup
			SCHSPreparePatterns   // alloc分配内存

DetectEngineThreadCtxInit
	ThreadCtxDoInit
		PatternMatchThreadPrepare(&det_ctx->mtc, de_ctx->mpm_matcher);
		PatternMatchThreadPrepare(&det_ctx->mtcs, de_ctx->mpm_matcher);
		PatternMatchThreadPrepare(&det_ctx->mtcu, de_ctx->mpm_matcher);
			MpmInitThreadCtx(mpm_thread_ctx, mpm_matcher);	
				mpm_table[matcher].InitThreadCtx(NULL, mpm_thread_ctx);
					SCHSInitThreadCtx // 各个工作线程clone暂存空间
```

在多线程或并发任务中使用 Hyperscan 时，有两种常见的方法来为每个线程分配 `scratch` 区域：

1. **每个线程独立调用 `hs_alloc_scratch`**：
    - **优点**：每个线程调用 `hs_alloc_scratch` 会根据当前数据库的要求为该线程分配一个全新的、独立的 `scratch` 区域。这种方法确保每个 `scratch` 区域都是独立分配的，没有任何共享的部分。
    - **缺点**：如果线程数量很多，调用 `hs_alloc_scratch` 的开销可能较大，尤其是在初始化阶段。每次调用都涉及动态内存分配，这在性能上可能不如克隆快。
2. **一个线程调用 `hs_alloc_scratch`，其它线程使用 `hs_clone_scratch`**：
    - **优点**：在这种方法中，一个线程首先使用 `hs_alloc_scratch` 分配一个 `scratch` 区域，接着其它线程使用 `hs_clone_scratch` 克隆这个 `scratch` 区域。`hs_clone_scratch` 通常比 `hs_alloc_scratch` 更快，因为它可以直接复制已经分配好的内存，而不是重新计算和分配所有资源。
    - **缺点**：克隆操作依赖于第一个分配的 `scratch` 区域，可能有一定的初始依赖性。如果第一个分配的 `scratch` 区域有特殊的初始化步骤，克隆后的 `scratch` 可能需要额外的设置。

**性能比较**

- **`hs_alloc_scratch` 每个线程独立分配**：这是最安全的方式，每个 `scratch` 区域完全独立。适合在初始化阶段性能要求不高，或需要高度独立性的场景。
- **`hs_clone_scratch` 克隆 `scratch` 区域**：克隆操作通常比重新分配快，因为它避免了重复计算和内存分配的开销。如果你有很多线程，并且希望减少初始化时间，使用 `hs_clone_scratch` 会更高效。

**实际选择**

- **线程数量较少或初始化时间不敏感**：使用 `hs_alloc_scratch` 为每个线程独立分配 `scratch` 区域。
- **线程数量较多，且希望优化初始化时间**：可以考虑先用 `hs_alloc_scratch` 分配一个 `scratch`，然后使用 `hs_clone_scratch` 进行克隆。

在大多数情况下，如果你需要为多个线程分配 `scratch` 区域，使用 `hs_clone_scratch` 会更快，因为它减少了重复的内存分配和初始化开销。



#### 2.2.1 hs暂存空间内存分配SCHSPreparePatterns

**作用和功能**

- **内存分配**：`hs_alloc_scratch` 会根据传入的 Hyperscan 数据库（`hs_database_t`）的需求，分配足够的内存空间以存储在匹配过程中需要的临时数据。
- **匹配操作的准备**：在调用 `hs_scan` 或 `hs_scan_vector` 等匹配函数之前，必须为每个线程或并发任务分配一个独立的 scratch 区域。这是因为这些函数在匹配过程中会使用 scratch 区域来保存状态和中间结果。
- **线程安全**：为了确保线程安全，每个线程应该有自己专用的 scratch 区域，这样不同线程在并发匹配时不会互相干扰

```less
err = hs_compile_ext_multi((const char *const *)cd->expressions, cd->flags,  // 编译
                               cd->ids, (const hs_expr_ext_t *const *)cd->ext,
                               cd->pattern_cnt, HS_MODE_BLOCK, NULL, &pd->hs_db,
                               &compile_err);

SCMutexLock(&g_scratch_proto_mutex);
err = hs_alloc_scratch(pd->hs_db, &g_scratch_proto);  // 暂存空间分配
SCMutexUnlock(&g_scratch_proto_mutex);
```



#### 2.2.2 hs暂存空间内存拷贝SCHSInitThreadCtx

`hs_clone_scratch` 的主要作用是为一个已经分配好的 scratch 区域创建一个副本。这个副本是一个新的 scratch 区域，包含与原区域相同的初始状态，但它占用了独立的内存空间。这意味着在克隆之后，两个 scratch 区域是完全独立的，对其中一个的操作不会影响另一个。

**实现原理**

- **内存分配**：`hs_clone_scratch` 会为新的 scratch 区域申请一块新的内存空间。
- **数据拷贝**：将原 scratch 区域的数据拷贝到新分配的区域中。
- **独立使用**：克隆后的 scratch 区域可以独立于原区域使用，适用于多线程环境下的并行操作。

```less
SCMutexLock(&g_scratch_proto_mutex);
hs_error_t err = hs_clone_scratch(g_scratch_proto, (hs_scratch_t **)&ctx->scratch);
SCMutexUnlock(&g_scratch_proto_mutex);
```





## 四、hyperscan的扫描

高效使用 Hyperscan 的方法结合了从编译、硬件优化到实际应用场景的多方面考虑。以下是一些关键策略：

### 4.1 优化策略

- **选择合适的编译模式**：
    - Hyperscan 提供了三种主要的编译模式：**流式模式**、**块模式**、**矢量模式**。根据你的应用场景，选择适当的模式至关重要。例如，流式模式适合处理连续流的数据，如网络流量，而块模式适合一次性处理整个数据块(Hyperscan读志)。
- **使用扩展参数进行优化**：
    - 可以通过 `hs_compile_ext_multi` 函数设置 `min_offset`、`max_offset`、`min_length` 等扩展参数，控制模式匹配的行为。利用这些参数可以减少无效匹配，提高匹配效率(Hyperscan读志)。
- **启用指令集优化**：
    - Hyperscan 能够利用现代 x86 处理器上的 SIMD 指令集（如 SSE、AVX、AVX2）来提升性能。确保在编译时针对你的硬件平台进行优化，例如使用 `hs_platform_info_t` 结构体来指导编译过程，使其充分利用 CPU 的特性(Hyperscan读志)。
- **并行处理**:
    - 如果处理的数据量非常大，考虑将数据分割成多个块，并使用多线程并行处理每个块。



### 4.2 hs_scan扫描处理(SCHSSearch)

#### 4.2.1 调用addr

以x509的ke-usage为例子

##### 1、预过滤注册回调函数

```less
    DetectAppLayerMpmRegister2("cert.key_usage", SIG_FLAG_TOCLIENT, 2,
            PrefilterMpmCertKeyUsageRegister, NULL, ALPROTO_TLS, TLS_CERT_DONE);

```



##### 2、预过滤回调函数

```less
static int PrefilterMpmCertKeyUsageRegister(DetectEngineCtx *de_ctx, SigGroupHead *sgh, MpmCtx *mpm_ctx,
        const DetectBufferMpmRegistry *mpm_reg, int list_id)
{
    PrefilterMpmCertKeyUsage *pectx = SCCalloc(1, sizeof(*pectx));
    if (pectx == NULL)
        return -1;
    pectx->list_id = list_id;
    pectx->mpm_ctx = mpm_ctx;
    pectx->transforms = &mpm_reg->transforms;

    return PrefilterAppendTxEngine(de_ctx, sgh, /*预过滤HS调用函数 PrefilterStateCertKeyUsage*/,
            mpm_reg->app_v2.alproto, mpm_reg->app_v2.tx_min_progress,
            pectx, PrefilterMpmCertKeyUsageFree, mpm_reg->pname);
}

```



##### 3、预过滤模式(mpm_ctx->mpm_type)HS是如何确定的

由PatternMatchDefaultMatcher在程序初始化时读取yaml文件确定

```less
AppLayerProtoDetectSetup
	uint8_t mpm_matcher = PatternMatchDefaultMatcher();
	for (i = 0; i < FLOW_PROTO_DEFAULT; i++) {
        for (j = 0; j < 2; j++) {
            MpmInitCtx(&alpd_ctx.ctx_ipp[i].ctx_pm[j].mpm_ctx, mpm_matcher);
        }
    }
```

MpmInitCtx会初始化对应  FLOW_PROTO_TCP = 0，FLOW_PROTO_UDP, FLOW_PROTO_ICMP协议流的匹配模式

```less
void MpmInitCtx(MpmCtx *mpm_ctx, uint8_t matcher)
{
    mpm_ctx->mpm_type = matcher; // 确定对应tcp/udp/icmp等流种类的匹配模式
    mpm_table[matcher].InitCtx(mpm_ctx); // 并进行对应模式的初始化函数调用
}
```



##### 4、根据预过滤模式进行(SCHSSearch)调用

```less
static void PrefilterStateCertKeyUsage(DetectEngineThreadCtx *det_ctx, const void *pectx, Packet *p,
        Flow *f, void *txv, const uint64_t idx, const AppLayerTxData *_txd, const uint8_t flags)
{
    SCEnter();

    const PrefilterMpmCertKeyUsage *ctx = (const PrefilterMpmCertKeyUsage *)pectx;
    const MpmCtx *mpm_ctx = ctx->mpm_ctx;
    const int list_id = ctx->list_id;

    const SSLState *ssl_state = (SSLState *)f->alstate;
    if (!ssl_state)
        return;
    uint32_t local_id = 0;
    SSLCertsChain *cert = NULL;
    TAILQ_FOREACH(cert, &ssl_state->server_connp.certs, next)
    {
        struct DetectX509CertGetDataArgs cbdata = { local_id, cert };
        InspectionBuffer *buffer = CertCertKeyUsageGetData(det_ctx, ctx->transforms,
                f, &cbdata, list_id);

        if (buffer != NULL && buffer->inspect_len >= mpm_ctx->minlen) {
            // Search 的调用位置
           	// 其中的mpm_ctx->mpm_type的预过滤匹配模式，由MpmInitCtx进行初始化
            (void)mpm_table[mpm_ctx->mpm_type].Search(mpm_ctx,
                    &det_ctx->mtcu, &det_ctx->pmq,
                    buffer->inspect, buffer->inspect_len);
        }
        local_id++;
    }
}

```

##### 4、SCHSSearch函数

```less
uint32_t SCHSSearch(const MpmCtx *mpm_ctx, MpmThreadCtx *mpm_thread_ctx,
                    PrefilterRuleStore *pmq, const uint8_t *buf, const uint32_t buflen)
{
    uint32_t ret = 0;
    SCHSCtx *ctx = (SCHSCtx *)mpm_ctx->ctx;
    SCHSThreadCtx *hs_thread_ctx = (SCHSThreadCtx *)(mpm_thread_ctx->ctx);
    const PatternDatabase *pd = ctx->pattern_db;

    if (unlikely(buflen == 0)) {
        return 0;
    }

    SCHSCallbackCtx cctx = {.ctx = ctx, .pmq = pmq, .match_count = 0};

    hs_scratch_t *scratch = hs_thread_ctx->scratch;
	// 开始扫面进行匹配处理
    hs_error_t err = hs_scan(pd->hs_db, (const char *)buf, buflen, 0, scratch,
                             SCHSMatchEvent, &cctx);
    if (err != HS_SUCCESS) {
        exit(EXIT_FAILURE);
    } else {
        ret = cctx.match_count;
    }

    return ret;
}
```

