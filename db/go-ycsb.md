https://github.com/pingcap/go-ycsb/tree/v1.0.1

● redis/db.go：Redis 驱动主逻辑

● core/client.go：客户端核心逻辑

● core/workload.go：工作负载生成

● pkg/properties/properties.go：配置管理

# redis/db.go

<br/>

确定redis建出来个各种参数、默认值。

eg：

```golang

redisMaxRetries            = "redis.max_retries"                 //!defalut == 0

redisMinRetryBackoff       = "redis.min_retry_backoff"                 //!!!!defalut == 8ms

redisMaxRetryBackoff       = "redis.max_retry_backoff"                 //!!!!defalut == 512ms

redisDialTimeout           = "redis.dial_timeout"                 //!!!!defalut == 5s

redisReadTimeout           = "redis.read_timeout"                 //!!!!defalut == 3s

redisWriteTimeout          = "redis.write_timeout"                 //!!!!defalut == 3s

redisPoolSize              = "redis.pool_size"                 //!!!!defalut == 0
```

<br/>

```golang
const HASH_DATATYPE string = "hash"
const STRING_DATATYPE string = "string"
const JSON_DATATYPE string = "json"

type redis struct {
    client     redisClient  // Redis 客户端接口
    mode       string       // 运行模式 (single/cluster)
    datatype   string       // 数据类型 (hash/string/json)
    fieldcount int64        // 字段数量
}

const (
	redisMode                  = "redis.mode"                 //!!!!
	redisModeDefault           = "single"
	redisDatatype              = "redis.datatype"
	redisDatatypeDefault       = "hash"
	redisNetwork               = "redis.network"
	redisNetworkDefault        = "tcp"
	redisAddr                  = "redis.addr"
	redisAddrDefault           = "localhost:6379"
	redisUsername              = "redis.username"
	redisPassword              = "redis.password"
	redisDB                    = "redis.db"                 //!!!!
	redisMaxRedirects          = "redis.max_redirects"                 //!!!!
	redisReadOnly              = "redis.read_only"
	redisRouteByLatency        = "redis.route_by_latency"                 //!!!!
	redisRouteRandomly         = "redis.route_randomly"                 //!!!!


	redisPoolSizeDefault       = 0                 //!!!! defalut == 0
	redisMinIdleConns          = "redis.min_idle_conns"
	redisMaxIdleConns          = "redis.max_idle_conns"
	redisMaxConnAge            = "redis.max_conn_age"
	redisPoolTimeout           = "redis.pool_timeout"                 //!!!!
	redisIdleTimeout           = "redis.idle_timeout"                 //!!!!
)

//JSON 类型: 使用 JSON.GET 命令，通过 pipeline 批量获取字段
//Hash 类型: 使用 HMGET 命令一次获取多个字段
//String 类型: 使用 GET 获取整个 JSON 字符串，然后反序列化
func (r *redis) Read(ctx context.Context, table string, key string, fields []string) (data map[string][]byte, err error) {
	data = make(map[string][]byte, len(fields))
	err = nil
	switch r.datatype {
	case JSON_DATATYPE:
		cmds := make([]*goredis.Cmd, len(fields))
		pipe := r.client.Pipeline()
		for pos, fieldName := range fields {
			cmds[pos] = pipe.Do(ctx, JSON_GET, getKeyName(table, key), getFieldJsonPath(fieldName))
		}
		_, err = pipe.Exec(ctx)
		if err != nil {
			return
		}
		var s string = ""
		for pos, fieldName := range fields {
			s, err = cmds[pos].Text()
			if err != nil {
				return
			}
			data[fieldName] = []byte(s)
		}
	case HASH_DATATYPE:
		args := make([]interface{}, 0, len(fields)+2)
		args = append(args, HMGET, getKeyName(table, key))
		for _, fieldName := range fields {
			args = append(args, fieldName)
		}
		sliceReply, errI := r.client.Do(ctx, args...).StringSlice()
		if errI != nil {
			return
		}
		for pos, slicePos := range sliceReply {
			data[fields[pos]] = []byte(slicePos)
		}
	case STRING_DATATYPE:
		fallthrough
	default:
		{
			var res string = ""
			res, err = r.client.Get(ctx, getKeyName(table, key)).Result()
			if err != nil {
				return
			}
			err = json.Unmarshal([]byte(res), &data)
			return
		}
	}
	return

}


//全量更新: 当更新字段数等于总字段数时
//部分更新:
//JSON/Hash: 直接更新指定字段
//String: 先读取原值，合并后再写入
func (r *redis) Update(ctx context.Context, table string, key string, values map[string][]byte) (err error) {
	// check if it's full update. If yes then we can avoid reading the previous value on string datype
	fullUpdate := false
	if int64(len(values)) == r.fieldcount {
		fullUpdate = true
	}
	err = nil
	switch r.datatype {
	case JSON_DATATYPE:
		cmds := make([]*goredis.Cmd, 0, len(values))
		pipe := r.client.Pipeline()
		for fieldName, bytes := range values {
			cmd := pipe.Do(ctx, JSON_SET, getKeyName(table, key), getFieldJsonPath(fieldName), jsonEscape(bytes))
			cmds = append(cmds, cmd)
		}
		_, err = pipe.Exec(ctx)
		if err != nil {
			return
		}
		for _, cmd := range cmds {
			err = cmd.Err()
			if err != nil {
				return
			}
		}
	case HASH_DATATYPE:
		args := make([]interface{}, 0, 2*len(values)+2)
		args = append(args, HSET, getKeyName(table, key))
		for fieldName, bytes := range values {
			args = append(args, fieldName, string(bytes))
		}
		err = r.client.Do(ctx, args...).Err()
	case STRING_DATATYPE:
		fallthrough
	default:
		{
			var encodedJson = make([]byte, 0)
			if fullUpdate {
				encodedJson, err = json.Marshal(values)
				if err != nil {
					return err
				}
			} else {
				var initialEncodedJson string = ""
				initialEncodedJson, err = r.client.Get(ctx, getKeyName(table, key)).Result()
				if err != nil {
					return
				}
				err, encodedJson = mergeEncodedJsonWithMap(initialEncodedJson, values)
				if err != nil {
					return
				}
			}
			return r.client.Set(ctx, getKeyName(table, key), string(encodedJson), 0).Err()
		}
	}
	return
}


//JSON 类型: JSON.SET key . json_data
//Hash 类型: HSET key field1 value1 field2 value2 ...
//String 类型: SET key json_data

func (r *redis) Insert(ctx context.Context, table string, key string, values map[string][]byte) (err error) {
	data, err := json.Marshal(values)
	if err != nil {
		return err
	}
	switch r.datatype {
	case JSON_DATATYPE:
		err = r.client.Do(ctx, JSON_SET, getKeyName(table, key), ".", string(data)).Err()
	case HASH_DATATYPE:
		args := make([]interface{}, 0, 2*len(values)+2)
		args = append(args, HSET, getKeyName(table, key))
		for fieldName, bytes := range values {
			args = append(args, fieldName, string(bytes))
		}
		err = r.client.Do(ctx, args...).Err()
	case STRING_DATATYPE:
		fallthrough
	default:
		err = r.client.Set(ctx, getKeyName(table, key), string(data), 0).Err()
	}
	return
}



```

# core/client.go

```golang
type worker struct {
	p               *properties.Properties
	workDB          ycsb.DB
	workload        ycsb.Workload
	doTransactions  bool
	doBatch         bool
	batchSize       int
	opCount         int64
	targetOpsPerMs  float64
	threadID        int
	targetOpsTickNs int64
	opsDone         int64
}
func newWorker(p *properties.Properties, threadID int, threadCount int, workload ycsb.Workload, db ycsb.DB) *worker {
	w := new(worker)
	w.p = p
	w.doTransactions = p.GetBool(prop.DoTransactions, true)
	w.batchSize = p.GetInt(prop.BatchSize, prop.DefaultBatchSize)
	if w.batchSize > 1 {
		w.doBatch = true
	}
	w.threadID = threadID
	w.workload = workload
	w.workDB = db

	var totalOpCount int64
	if w.doTransactions {
		totalOpCount = p.GetInt64(prop.OperationCount, 0)
	} else {
		if _, ok := p.Get(prop.InsertCount); ok {
			totalOpCount = p.GetInt64(prop.InsertCount, 0)
		} else {
			totalOpCount = p.GetInt64(prop.RecordCount, 0)
		}
	}

	if totalOpCount < int64(threadCount) {
		fmt.Printf("totalOpCount(%s/%s/%s): %d should be bigger than threadCount: %d",
			prop.OperationCount,
			prop.InsertCount,
			prop.RecordCount,
			totalOpCount,
			threadCount)

		os.Exit(-1)
	}

	w.opCount = totalOpCount / int64(threadCount)
	if threadID < int(totalOpCount%int64(threadCount)) {
		w.opCount++
	}

	targetPerThreadPerms := float64(-1)
	if v := p.GetInt64(prop.Target, 0); v > 0 {
		targetPerThread := float64(v) / float64(threadCount)
		targetPerThreadPerms = targetPerThread / 1000.0
	}

	if targetPerThreadPerms > 0 {
		w.targetOpsPerMs = targetPerThreadPerms
		w.targetOpsTickNs = int64(1000000.0 / w.targetOpsPerMs)
	}

	return w
}

///  设置target 主要限流函数。
func (w *worker) throttle(ctx context.Context, startTime time.Time) {
	if w.targetOpsPerMs <= 0 {
		return
	}
// 发压时会进行等待
	d := time.Duration(w.opsDone * w.targetOpsTickNs)
	d = startTime.Add(d).Sub(time.Now())
	if d < 0 {
		return
	}
	select {
	case <-ctx.Done():
	case <-time.After(d):
	}
}



//  ------ client
type Client struct {
	p        *properties.Properties
	workload ycsb.Workload
	db       ycsb.DB
}
// Run runs the workload to the target DB, and blocks until all workers end.
func (c *Client) Run(ctx context.Context) {
  //创建 WaitGroup 用于同步所有工作线程
//从配置中获取线程数量（默认为1）
//预先添加计数器
	var wg sync.WaitGroup
	threadCount := c.p.GetInt(prop.ThreadCount, 1)

	wg.Add(threadCount)
// 这是一个独立的 goroutine，负责性能指标的收集和输出
  measureCtx, measureCancel := context.WithCancel(ctx)
	measureCh := make(chan struct{}, 1)
	go func() {
		defer func() {
			measureCh <- struct{}{}
		}()
		// load stage no need to warm up 
      // 只在事务模式（DoTransactions=true）下进行预热，加载数据阶段不需要
		if c.p.GetBool(prop.DoTransactions, true) {
			dur := c.p.GetInt64(prop.WarmUpTime, 0)
			select {
			case <-ctx.Done():
				return
			case <-time.After(time.Duration(dur) * time.Second):
			}
		}
		// finish warming up
      // 结束预热，开始正式统计
		measurement.EnableWarmUp(false)
// 定期输出统计信息 定期报告：默认每10秒输出一次性能统计
		dur := c.p.GetInt64(prop.LogInterval, 10)
		t := time.NewTicker(time.Duration(dur) * time.Second)
		defer t.Stop()

		for {
			select {
			case <-t.C:
				measurement.Summary()
			case <-measureCtx.Done():
				return
			}
		}
	}()

	for i := 0; i < threadCount; i++ {
		go func(threadId int) {
			defer wg.Done()

			w := newWorker(c.p, threadId, threadCount, c.workload, c.db)
			ctx := c.workload.InitThread(ctx, threadId, threadCount)
			ctx = c.db.InitThread(ctx, threadId, threadCount)
			w.run(ctx) // 这里创建worker 执行函数。每个线程每个worker。
			c.db.CleanupThread(ctx)
			c.workload.CleanupThread(ctx)
		}(i)
	}
// 等待所有工作线程完成
	wg.Wait()
  // 数据加载完成后的分析
	if !c.p.GetBool(prop.DoTransactions, true) {
		// when loading is finished, try to analyze table if possible.
		if analyzeDB, ok := c.db.(ycsb.AnalyzeDB); ok {
			analyzeDB.Analyze(ctx, c.p.GetString(prop.TableName, prop.TableNameDefault))
		}
	}
	measureCancel()
	<-measureCh
}

```
