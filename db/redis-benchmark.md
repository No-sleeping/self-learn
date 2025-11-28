https://github.com/redis/redis/blob/6.0/src/redis-benchmark.c

# 1、数据结构

## 1.1  client

```c_cpp
typedef struct _client {
    redisContext *context;
    sds obuf;  // 输出缓冲区
    char **randptr;         /* Pointers to :rand: strings inside the command buf */
    size_t randlen;         /* Number of pointers in client->randptr */
    size_t randfree;        /* Number of unused pointers in client->randptr */
    char **stagptr;         /* Pointers to slot hashtags (cluster mode only) */
    size_t staglen;         /* Number of pointers in client->stagptr */
    size_t stagfree;        /* Number of unused pointers in client->stagptr */
    size_t written;         /* Bytes of 'obuf' already written */  //已写入字节数
    long long start;        /* Start time of a request */
    long long latency;      /* Request latency */
// 待处理请求数（待消费的回复）
    int pending;            /* Number of pending requests (replies to consume) */
    int prefix_pending;     /* If non-zero, number of pending prefix commands. Commands
                               such as auth and select are prefixed to the pipeline of
                               benchmark commands and discarded after the first send. */
    int prefixlen;          /* Size in bytes of the pending prefix commands */
    int thread_id;
    struct clusterNode *cluster_node;
    int slots_last_update;
} *client;
```

## 1.2  config

```c_cpp
static struct config {
    aeEventLoop *el;
    const char *hostip;
    int hostport;
    const char *hostsocket;
    int numclients;
    int liveclients;
    int requests;
    int requests_issued;
    int requests_finished;
    int keysize;
    int datasize;
    int randomkeys;
    int randomkeys_keyspacelen;
    int keepalive;
  // ----------------------- -P
    int pipeline;
    int showerrors;
    long long start;
    long long totlatency;
    long long *latency;
    const char *title;
    list *clients;
    int quiet;
    int csv;
    int loop;
    int idlemode;
    int dbnum;
    sds dbnumstr;
    char *tests;
    char *auth;
    const char *user;
    int precision;
    int num_threads;
    struct benchmarkThread **threads;
    int cluster_mode;
    int cluster_node_count;
    struct clusterNode **cluster_nodes;
    struct redisConfig *redis_config;
    int is_fetching_slots;
    int is_updating_slots;
    int slots_last_update;
    int enable_tracking;
    /* Thread mutexes to be used as fallbacks by atomicvar.h */
    pthread_mutex_t requests_issued_mutex;
    pthread_mutex_t requests_finished_mutex;
    pthread_mutex_t liveclients_mutex;
    pthread_mutex_t is_fetching_slots_mutex;
    pthread_mutex_t is_updating_slots_mutex;
    pthread_mutex_t updating_slots_mutex;
    pthread_mutex_t slots_last_update_mutex;
} config;
```

# 2、流程

## 2.1 main

main()
  ├─ 初始化默认配置
  ├─ parseOptions() - 解析命令行参数
  ├─ 创建事件循环 aeCreateEventLoop()
  ├─ 集群模式处理
  │   ├─ fetchClusterConfiguration() - 获取集群配置
  │   └─ 为每个master节点获取配置
  ├─ 初始化多线程 (如果启用)-----------------
  │   └─ initBenchmarkThreads()
  ├─ 执行基准测试
  │   ├─ 自定义命令测试 (如果有参数)
  │   └─ 默认测试套件 (PING, SET, GET, LPUSH等)
  └─ 清理资源

### 2.1.1 benchmark

```c_cpp
static void benchmark(char *title, char *cmd, int len) {
    // 1. 初始化统计
    config.requests_issued = 0;
    config.requests_finished = 0;
    
    // 2. 初始化线程 (多线程模式)
    if (config.num_threads) 
        initBenchmarkThreads();
    
    // 3. 创建第一个客户端
    c = createClient(cmd, len, NULL, thread_id);
    
    // 4. 创建所有客户端连接
    createMissingClients(c);
    
    // 5. 启动事件循环
    config.start = mstime();
    if (!config.num_threads) 
        aeMain(config.el);  // 单线程模式
    else 
        startBenchmarkThreads();  // 多线程模式
    
    // 6. 计算总延迟
    config.totlatency = mstime() - config.start;
    
    // 7. 显示报告
    showLatencyReport();
    
    // 8. 清理
    freeAllClients();
}

```

### 2.1.2 createClient

<br/>

```c_cpp
static client createClient(char *cmd, size_t len, client from, int thread_id) {
    int j;
    int is_cluster_client = (config.cluster_mode && thread_id >= 0);
    client c = zmalloc(sizeof(struct _client));

    const char *ip = NULL;
    int port = 0;
    c->cluster_node = NULL;
    if (config.hostsocket == NULL || is_cluster_client) {
        if (!is_cluster_client) {
            ip = config.hostip;
            port = config.hostport;
        } else {
            int node_idx = 0;
            if (config.num_threads < config.cluster_node_count)
                node_idx = config.liveclients % config.cluster_node_count;
            else
                node_idx = thread_id % config.cluster_node_count;
            clusterNode *node = config.cluster_nodes[node_idx];
            assert(node != NULL);
            ip = (const char *) node->ip;
            port = node->port;
            c->cluster_node = node;
        }
        c->context = redisConnectNonBlock(ip,port);
    } else {
        c->context = redisConnectUnixNonBlock(config.hostsocket);
    }
    if (c->context->err) {
        fprintf(stderr,"Could not connect to Redis at ");
        if (config.hostsocket == NULL || is_cluster_client)
            fprintf(stderr,"%s:%d: %s\n",ip,port,c->context->errstr);
        else
            fprintf(stderr,"%s: %s\n",config.hostsocket,c->context->errstr);
        exit(1);
    }
    c->thread_id = thread_id;
    /* Suppress hiredis cleanup of unused buffers for max speed. */
    c->context->reader->maxbuf = 0;
 // ----------------------------------------------------------------------------
    /* Build the request buffer:
     * Queue N requests accordingly to the pipeline size, or simply clone
     * the example client buffer. */
  // -
    c->obuf = sdsempty();
    /* Prefix the request buffer with AUTH and/or SELECT commands, if applicable.
     * These commands are discarded after the first response, so if the client is
     * reused the commands will not be used again. */
    c->prefix_pending = 0;
    if (config.auth) {
        char *buf = NULL;
        int len;
        if (config.user == NULL)
            len = redisFormatCommand(&buf, "AUTH %s", config.auth);
        else
            len = redisFormatCommand(&buf, "AUTH %s %s",
                                     config.user, config.auth);
        c->obuf = sdscatlen(c->obuf, buf, len);
        free(buf);
        c->prefix_pending++;
    }

    if (config.enable_tracking) {
        char *buf = NULL;
        int len = redisFormatCommand(&buf, "CLIENT TRACKING on");
        c->obuf = sdscatlen(c->obuf, buf, len);
        free(buf);
        c->prefix_pending++;
    }

    /* If a DB number different than zero is selected, prefix our request
     * buffer with the SELECT command, that will be discarded the first
     * time the replies are received, so if the client is reused the
     * SELECT command will not be used again. */
    if (config.dbnum != 0 && !is_cluster_client) {
        c->obuf = sdscatprintf(c->obuf,"*2\r\n$6\r\nSELECT\r\n$%d\r\n%s\r\n",
            (int)sdslen(config.dbnumstr),config.dbnumstr);
        c->prefix_pending++;
    }
    c->prefixlen = sdslen(c->obuf);
    /* Append the request itself. */
    if (from) {
        c->obuf = sdscatlen(c->obuf,
            from->obuf+from->prefixlen,
            sdslen(from->obuf)-from->prefixlen);
    } else {
        for (j = 0; j < config.pipeline; j++)
            c->obuf = sdscatlen(c->obuf,cmd,len);
    }

    c->written = 0;
    c->pending = config.pipeline+c->prefix_pending;
    c->randptr = NULL;
    c->randlen = 0;
    c->stagptr = NULL;
    c->staglen = 0;

    /* Find substrings in the output buffer that need to be randomized. */
    if (config.randomkeys) {
        if (from) {
            c->randlen = from->randlen;
            c->randfree = 0;
            c->randptr = zmalloc(sizeof(char*)*c->randlen);
            /* copy the offsets. */
            for (j = 0; j < (int)c->randlen; j++) {
                c->randptr[j] = c->obuf + (from->randptr[j]-from->obuf);
                /* Adjust for the different select prefix length. */
                c->randptr[j] += c->prefixlen - from->prefixlen;
            }
        } else {
            char *p = c->obuf;

            c->randlen = 0;
            c->randfree = RANDPTR_INITIAL_SIZE;
            c->randptr = zmalloc(sizeof(char*)*c->randfree);
            while ((p = strstr(p,"__rand_int__")) != NULL) {
                if (c->randfree == 0) {
                    c->randptr = zrealloc(c->randptr,sizeof(char*)*c->randlen*2);
                    c->randfree += c->randlen;
                }
                c->randptr[c->randlen++] = p;
                c->randfree--;
                p += 12; /* 12 is strlen("__rand_int__). */
            }
        }
    }
    /* If cluster mode is enabled, set slot hashtags pointers. */
    if (config.cluster_mode) {
        if (from) {
            c->staglen = from->staglen;
            c->stagfree = 0;
            c->stagptr = zmalloc(sizeof(char*)*c->staglen);
            /* copy the offsets. */
            for (j = 0; j < (int)c->staglen; j++) {
                c->stagptr[j] = c->obuf + (from->stagptr[j]-from->obuf);
                /* Adjust for the different select prefix length. */
                c->stagptr[j] += c->prefixlen - from->prefixlen;
            }
        } else {
            char *p = c->obuf;

            c->staglen = 0;
            c->stagfree = RANDPTR_INITIAL_SIZE;
            c->stagptr = zmalloc(sizeof(char*)*c->stagfree);
            while ((p = strstr(p,"{tag}")) != NULL) {
                if (c->stagfree == 0) {
                    c->stagptr = zrealloc(c->stagptr,
                                          sizeof(char*) * c->staglen*2);
                    c->stagfree += c->staglen;
                }
                c->stagptr[c->staglen++] = p;
                c->stagfree--;
                p += 5; /* 5 is strlen("{tag}"). */
            }
        }
    }
    aeEventLoop *el = NULL;
    if (thread_id < 0) el = config.el;
    else {
        benchmarkThread *thread = config.threads[thread_id];
        el = thread->el;
    }
  // -------------------------------------------------
  // 注册写事件处理器。
  // 原子操作加入客户端列表
    if (config.idlemode == 0)
        aeCreateFileEvent(el,c->context->fd,AE_WRITABLE,writeHandler,c);
    listAddNodeTail(config.clients,c);
    atomicIncr(config.liveclients, 1);
    atomicGet(config.slots_last_update, c->slots_last_update);
    return c;
}
```

### 2.1.3 writeHandler

```c_cpp
static void writeHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    client c = privdata;
    UNUSED(el);
    UNUSED(fd);
    UNUSED(mask);

    /* Initialize request when nothing was written. */
    if (c->written == 0) {
        /* Enforce upper bound to number of requests. */
        int requests_issued = 0;
      //  ---------
      // 线程安全地获取并递增 config.requests_issued
      //返回递增前的值到 requests_issued
      //多线程环境下保证请求计数准确
        atomicGetIncr(config.requests_issued, requests_issued, 1);
      // 所有发送的请求都已经处理完毕，释放客户端。
        if (requests_issued >= config.requests) {
            freeClient(c);
            return;
        }

        /* Really initialize: randomize keys and set start time. */
        if (config.randomkeys) randomizeClientKey(c);
        if (config.cluster_mode && c->staglen > 0) setClusterKeyHashTag(c);
      //  记录时间戳
        atomicGet(config.slots_last_update, c->slots_last_update);
        c->start = ustime();
        c->latency = -1;
    }
  // 第二阶段：数据写入
    if (sdslen(c->obuf) > c->written) {
      // c->obuf 缓冲区即 pipline存命令位置
        void *ptr = c->obuf + c->written;
       // 分片写入 到 内核 TCP 发送缓冲区
        ssize_t nwritten = write(c->context->fd,ptr,sdslen(c->obuf)-c->written);
      //EPIPE: 连接已关闭（对端关闭）.不打印错误（正常情况）
      //EAGAIN/EWOULDBLOCK: 缓冲区满.但这里不应该发生，因为只在可写时触发
      //ECONNRESET: 连接被重置.打印错误并释放客户端
        if (nwritten == -1) {
            if (errno != EPIPE)
                fprintf(stderr, "Writing to socket: %s\n", strerror(errno));
            freeClient(c);
            return;
        }
        c->written += nwritten;
      // 删除写事件监听
      //注册读事件监听
        if (sdslen(c->obuf) == c->written) {
            aeDeleteFileEvent(el,c->context->fd,AE_WRITABLE);
            aeCreateFileEvent(el,c->context->fd,AE_READABLE,readHandler,c);
        }
    }
}
```

### 2.1.4 aeMain

```c_cpp
// ae.c 中的事件循环
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_BEFORE_SLEEP);
    }
}

// 处理事件
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
    // 1. 调用 epoll_wait/select 等待事件-----------------------------
    numevents = aeApiPoll(eventLoop, tvp);
    
    // 2. 处理文件事件
    for (j = 0; j < numevents; j++) {
        aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
        
        if (fe->mask & AE_READABLE) {
            fe->rfileProc(eventLoop, fd, fe->clientData, mask);
        }
        if (fe->mask & AE_WRITABLE) {
            fe->wfileProc(eventLoop, fd, fe->clientData, mask);
        }
    }
}


```

### 2.1.5 readHandler

```c_cpp
static void readHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
    client c = privdata;
    void *reply = NULL;
    UNUSED(el);
    UNUSED(fd);
    UNUSED(mask);

    /* Calculate latency only for the first read event. This means that the
     * server already sent the reply and we need to parse it. Parsing overhead
     * is not part of the latency, so calculate it only once, here. */
    if (c->latency < 0) c->latency = ustime()-(c->start);

    if (redisBufferRead(c->context) != REDIS_OK) {
        fprintf(stderr,"Error: %s\n",c->context->errstr);
        exit(1);
    } else {
        while(c->pending) { //  已经有发送，但是未得到返回
            if (redisGetReply(c->context,&reply) != REDIS_OK) {//被限流这里会阻塞，即时延会增长------------------ 
                fprintf(stderr,"Error: %s\n",c->context->errstr);
                exit(1);
            }
            if (reply != NULL) {
                if (reply == (void*)REDIS_REPLY_ERROR) {
                    fprintf(stderr,"Unexpected error reply, exiting...\n");
                    exit(1);
                }
                redisReply *r = reply;
                int is_err = (r->type == REDIS_REPLY_ERROR);

                if (is_err && config.showerrors) {
                    /* TODO: static lasterr_time not thread-safe */
                    static time_t lasterr_time = 0;
                    time_t now = time(NULL);
                    if (lasterr_time != now) {
                        lasterr_time = now;
                        if (c->cluster_node) {
                            printf("Error from server %s:%d: %s\n",
                                   c->cluster_node->ip,
                                   c->cluster_node->port,
                                   r->str);
                        } else printf("Error from server: %s\n", r->str);
                    }
                }

                /* Try to update slots configuration if reply error is
                 * MOVED/ASK/CLUSTERDOWN and the key(s) used by the command
                 * contain(s) the slot hash tag. */
                if (is_err && c->cluster_node && c->staglen) {
                    int fetch_slots = 0, do_wait = 0;
                    if (!strncmp(r->str,"MOVED",5) || !strncmp(r->str,"ASK",3))
                        fetch_slots = 1;
                    else if (!strncmp(r->str,"CLUSTERDOWN",11)) {
                        /* Usually the cluster is able to recover itself after
                         * a CLUSTERDOWN error, so try to sleep one second
                         * before requesting the new configuration. */
                        fetch_slots = 1;
                        do_wait = 1;
                        printf("Error from server %s:%d: %s\n",
                               c->cluster_node->ip,
                               c->cluster_node->port,
                               r->str);
                    }
                    if (do_wait) sleep(1);
                    if (fetch_slots && !fetchClusterSlotsConfiguration(c))
                        exit(1);
                }

                freeReplyObject(reply);
                /* This is an OK for prefix commands such as auth and select.*/
                if (c->prefix_pending > 0) {
                    c->prefix_pending--;
                    c->pending--;
                    /* Discard prefix commands on first response.*/
                    if (c->prefixlen > 0) {
                        size_t j;
                        sdsrange(c->obuf, c->prefixlen, -1);
                        /* We also need to fix the pointers to the strings
                        * we need to randomize. */
                        for (j = 0; j < c->randlen; j++)
                            c->randptr[j] -= c->prefixlen;
                        /* Fix the pointers to the slot hash tags */
                        for (j = 0; j < c->staglen; j++)
                            c->stagptr[j] -= c->prefixlen;
                        c->prefixlen = 0;
                    }
                    continue;
                }
                int requests_finished = 0;
                atomicGetIncr(config.requests_finished, requests_finished, 1);
                if (requests_finished < config.requests)
                    config.latency[requests_finished] = c->latency;
                c->pending--;
                if (c->pending == 0) {
                    clientDone(c);
                    break;
                }
            } else {
                break;
            }
        }
    }
}
```

### 2.1.6 redisGetReply

```c_cpp
int redisGetReply(redisContext *c, void **reply) {
    int wdone = 0;
    void *aux = NULL;

    /* Try to read pending replies */
    if (redisNextInBandReplyFromReader(c,&aux) == REDIS_ERR)
        return REDIS_ERR;

    /* For the blocking context, flush output buffer and read reply */
  // ------------进入阻塞模式处理
    if (aux == NULL && c->flags & REDIS_BLOCK) {
        /* Write until done */
      // 写入命令
        do {
            if (redisBufferWrite(c,&wdone) == REDIS_ERR)
                return REDIS_ERR;
        } while (!wdone);

        /* Read until there is a reply */
        do {
            if (redisBufferRead(c) == REDIS_ERR)
                return REDIS_ERR;

            if (redisNextInBandReplyFromReader(c,&aux) == REDIS_ERR)
                return REDIS_ERR;
        } while (aux == NULL);
    }

    /* Set reply or free it if we were passed NULL */
    if (reply != NULL) {
        *reply = aux;
    } else {
        freeReplyObject(aux);
    }

    return REDIS_OK;
}
```

### 2.1.7 redisBufferWrite

```c_cpp
int redisBufferWrite(redisContext *c, int *done) {
    if (c->err)
        return REDIS_ERR;

    if (sdslen(c->obuf) > 0) {  // ← 限流时这里总是 > 0
        int nwritten = c->funcs->write(c);  // ← 核心：尝试写入
        
        if (nwritten < 0) {
            return REDIS_ERR;  // ← 限流时频繁到这里（EAGAIN）
        } else if (nwritten > 0) {
            // 部分写入成功，调整缓冲区
            if (nwritten == (signed)sdslen(c->obuf)) {
                sdsfree(c->obuf);
                c->obuf = sdsempty();
            } else {
                sdsrange(c->obuf,nwritten,-1);  // ← 限流时频繁执行
            }
        }
        // nwritten == 0 的情况：什么都没写入
    }
    
    if (done != NULL) 
        *done = (sdslen(c->obuf) == 0);  // ← 限流时 done = 0
    return REDIS_OK;
}

```

### 2.1.8 redisBufferRead

```c_cpp
int redisBufferRead(redisContext *c) {
    char buf[1024*16];
    int nread;

    if (c->err)
        return REDIS_ERR;

    nread = c->funcs->read(c, buf, sizeof(buf));  // ← 尝试读取
    
    if (nread > 0) {
        // 有数据，喂给解析器
        if (redisReaderFeed(c->reader, buf, nread) != REDIS_OK) {
            __redisSetError(c, c->reader->err, c->reader->errstr);
            return REDIS_ERR;
        }
    } else if (nread < 0) {
        return REDIS_ERR;  // ← 限流时：没数据可读，直接返回
    }
    // nread == 0: EOF
    return REDIS_OK;
}
```

```
时间线分析（假设 pipeline=1, numclients=50）:

正常情况（无限流）:
T=0ms:    50个客户端同时发送请求
T=1ms:    服务端处理完，返回响应
T=1ms:    readHandler 读取响应，requests_finished += 50
T=1ms:    客户端立即发送下一轮请求
...
QPS = 50000/s

限流情况（CPU 100%）:
T=0ms:    50个客户端同时发送请求
          requests_issued = 50 ✓
          
T=1ms:    服务端只处理了 10 个请求
          返回 10 个响应
          
T=1ms:    readHandler 读取 10 个响应
          requests_finished = 10
          这 10 个客户端发送下一轮请求
          requests_issued = 60
          
T=2ms:    服务端又处理了 10 个请求（旧的）
          返回 10 个响应
          
T=2ms:    requests_finished = 20
          又有 10 个客户端发送新请求
          requests_issued = 70
          
...持续积压...

T=100ms:  requests_finished = 500
          requests_issued = 1000+
          
QPS = 500/0.1 = 5000/s  ← 下降 90%！

```
