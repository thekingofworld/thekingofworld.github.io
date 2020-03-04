---
title: 消息队列NSQ源码分析（二）：go-nsq
categories:
    - 消息队列
tags:
    - nsq
---
本篇为NSQ源码分析的第二篇，主要分析nsq的客户端源码，即go-nsq。文章主要分为3个部分，第一部分主要讲述代码的整体架构设计以及一些配置项；第二部分结合源码分析***生产者***的整个生命周期；第三部分结合源码分析***消费者***的整个生命周期。

<!-- more -->

## 代码架构

### 整体设计

先上两个图：
![](/img/go-nsq.jpg) ![](/img/go-nsq-architecture.png)
上面左图列出了`go-nsq`客户端包中的所有文件，右图是我整理的代码架构设计图，左图中的核心文件基本在右图中都有所体现。从右图可以看出，整体结构大致可以分为两层，上层的话主要对外暴露了两个概念：生产者和消费者；下层主要涉及与nsqd的连接以及协议处理；其中生产者和消费者没有直接的交互，两者只与底层连接进行通信，通过command的形式发送请求，以delegate的形式接收server端的响应，message也就是以这种方式传达给消费者的，同时message也将底层连接绑定为它的delegate，通过这种方式来进行消息的requeue，即重发（消息处理失败时会触发）。

### 配置项介绍

我们使用各种第三方组件时通常都会传入一些配置项进行初始化，nsq也不例外，上图中没有将config配置标记出来，但不代表它不重要。在深入了解生产者、消费者源码之前，这里有必要单独对config中包含的配置项做一个简单的介绍。

```go
// Config is a struct of NSQ options
//
// The only valid way to create a Config is via NewConfig, using a struct literal will panic.
// After Config is passed into a high-level type (like Consumer, Producer, etc.) the values are no
// longer mutable (they are copied).
// Use Set(option string, value interface{}) as an alternate way to set parameters

// 这里主要说明的是Config是以结构体的形式存在，同时它只能通过NewConfig方法进行创建，不能以字面量的形式直接创建。
// 当Config被传入生产者或消费者来进行初始化的时候，Config不再可变，因为这时Config已经被生产者和消费者复制了一份

type Config struct {
	
	// 用来判断配置是否已经初始化，通过NewConfig方法创建的Config该字段为true，如果直接用字面量创建的话该字段为false，因为是未导出的，
	// 初始化生产者和消费者时如果为false会直接panic
	initialized bool

	// 用来设置配置项的默认值
	configHandlers []configHandler 
	
	// 建立TCP连接的超时时间
	DialTimeout time.Duration `opt:"dial_timeout" default:"1s"`

	// TCP连接的读写超时时间
	ReadTimeout  time.Duration `opt:"read_timeout" min:"100ms" max:"5m" default:"60s"`
	WriteTimeout time.Duration `opt:"write_timeout" min:"100ms" max:"5m" default:"1s"`

	// 建立TCP连接的本地地址
	LocalAddr net.Addr `opt:"local_addr"`

	// 消费者轮询lookupd的时间间隔，来获取指定topic最新的生产者nsqd实例地址，当消费者时直连nsqd的时候，此配置项表示重连的间隔时间
	LookupdPollInterval time.Duration `opt:"lookupd_poll_interval" min:"10ms" max:"5m" default:"60s"`
	// 当多个消费者重启的时候，会等待一个随机因子的时间，然后再发送请求，用来减少并发的请求量
	LookupdPollJitter   float64       `opt:"lookupd_poll_jitter" min:"0" max:"1" default:"0.3"`

	// 消息处理失败时重新入队的最大延迟时间
	MaxRequeueDelay     time.Duration `opt:"max_requeue_delay" min:"0" max:"60m" default:"15m"`
	// 消息处理失败时重新入队的延迟时间
	DefaultRequeueDelay time.Duration `opt:"default_requeue_delay" min:"0" max:"60m" default:"90s"`

	// 退避的策略，NSQ采用的PUSH模型，因此需要有一定的策略来进行流量控制
	BackoffStrategy BackoffStrategy `opt:"backoff_strategy" default:"exponential"`
	// 退避的最大时间，设置为0表示不进行退避
	MaxBackoffDuration time.Duration `opt:"max_backoff_duration" min:"0" max:"60m" default:"2m"`
	// 退避的时间单位
	BackoffMultiplier time.Duration `opt:"backoff_multiplier" min:"0" max:"60m" default:"1s"`

	// 消费者处理一条消息的最大尝试次数
	MaxAttempts uint16 `opt:"max_attempts" min:"0" max:"65535" default:"5"`

	// rdy的闲置超时时间，意思是消费者与指定nsqd实例对应的连接上没有消息到来的闲置时间
	LowRdyIdleTimeout time.Duration `opt:"low_rdy_idle_timeout" min:"1s" max:"5m" default:"10s"`
	// 距上一次发送rdy的超时时间
	LowRdyTimeout time.Duration `opt:"low_rdy_timeout" min:"1s" max:"5m" default:"30s"`
	// 消费者重新计算各连接rdy数的时间间隔
	RDYRedistributeInterval time.Duration `opt:"rdy_redistribute_interval" min:"1ms" max:"5s" default:"5s"`

	// 用来标识客户端
	// UserAgent is in the spirit of HTTP (default: "<client_library_name>/<version>")
	ClientID  string `opt:"client_id"` // (defaults: short hostname)
	Hostname  string `opt:"hostname"`
	UserAgent string `opt:"user_agent"`

	// 心跳包时间间隔，必须小于读超时时间
	HeartbeatInterval time.Duration `opt:"heartbeat_interval" default:"30s"`
	// 采样率，设置后只有一定比例的消息会发送给该客户端，比如设置为10，那么本来会发送100条消息给你，现在只会采样10条发给你
	SampleRate int32 `opt:"sample_rate" min:"0" max:"99"`

	// TLS配置
	TlsV1     bool        `opt:"tls_v1"`
	TlsConfig *tls.Config `opt:"tls_config"`

	// 消息压缩配置项
	Deflate      bool `opt:"deflate"`
	DeflateLevel int  `opt:"deflate_level" min:"1" max:"9" default:"6"`
	Snappy       bool `opt:"snappy"`

	// 缓冲大小，nsq服务端会为客户端连接设置一个缓冲buffer，用来缓冲消息
	OutputBufferSize int64 `opt:"output_buffer_size" default:"16384"`
	// 缓冲刷新的超时时间，超过该时间间隔后缓冲的消息会直接发送给客户端，设置为0表示不使用缓冲，需要注意的是，缓冲超时时间设置较小会对
	// CPU产生比较大的影响
	OutputBufferTimeout time.Duration `opt:"output_buffer_timeout" default:"250ms"`

	// 客户端最大的并发消息处理数量，此配置项比较重要，稍后讲消费者的时候会详细分析
	MaxInFlight int `opt:"max_in_flight" min:"0" default:"1"`

	// 用于服务端确认该条消息超时的时间，即超过该时间服务端则认为超时，会重发
	MsgTimeout time.Duration `opt:"msg_timeout" min:"0"`

	// 用于认证的密钥
	AuthSecret string `opt:"auth_secret"`
}
```

## 生产者的视角

### 生产者实例创建

与生产者相关的代码主要在`producer.go`文件中，其中`Producer`结构体即为生产者对象的结构，如下：
```go
type Producer struct {
	id     int64        //实例id
	addr   string       //nsqd地址
	conn   producerConn //以接口的形式持有的底层连接
	config Config       //配置，各项含义上面已有介绍

	logger   []logger      //用来打印日志的实例对象
	logLvl   LogLevel      //日志等级
	logGuard sync.RWMutex  //设置日志等级和实例对象时需要加锁

	responseChan chan []byte  //接收响应的管道
	errorChan    chan []byte  //接收错误的管道
	closeChan    chan int     //接收关闭信号的管道

	transactionChan chan *ProducerTransaction   //接收发消息任务的管道
	transactions    []*ProducerTransaction      //当前发送中的消息
	state           int32                       //当前状态，states.go中有定义

	concurrentProducers int32           //当前并发发送消息的生产者数量
	stopFlag            int32           //停止标识
	exitChan            chan int        //接收退出信号的管道
	wg                  sync.WaitGroup
	guard               sync.Mutex
}
```

对象中的各个字段含义在上面都进行了注释，接下来我们看下创建生产者实例的方法：`NewProducer`，该方法同样位于`producer.go`文件中，如下：
```go
func NewProducer(addr string, config *Config) (*Producer, error) {
	config.assertInitialized()
	err := config.Validate()
	if err != nil {
		return nil, err
	}

	p := &Producer{
		id: atomic.AddInt64(&instCount, 1),

		addr:   addr,
		config: *config,

		logger: make([]logger, int(LogLevelMax+1)),
		logLvl: LogLevelInfo,

		transactionChan: make(chan *ProducerTransaction),
		exitChan:        make(chan int),
		responseChan:    make(chan []byte),
		errorChan:       make(chan []byte),
	}

	// Set default logger for all log levels
	l := log.New(os.Stderr, "", log.Flags())
	for index, _ := range p.logger {
		p.logger[index] = l
	}
	return p, nil
}
```

从上述代码中可以看出，生产者的创建过程基本就是：
1. 验证传入的配置
2. 实例化生产者对应的结构体
3. 设置日志实例
4. 返回生产者实例

### 发送消息

了解了生产者创建的过程之后，我们来看一下生产者是怎么发送消息的。对外暴露的发送消息的方法有6个，如下：
```go
// 异步发送
func (w *Producer) PublishAsync(topic string, body []byte, doneChan chan *ProducerTransaction,
	args ...interface{}) error {
	return w.sendCommandAsync(Publish(topic, body), doneChan, args)
}
// 异步发送，支持批量
func (w *Producer) MultiPublishAsync(topic string, body [][]byte, doneChan chan *ProducerTransaction,
	args ...interface{}) error {
	cmd, err := MultiPublish(topic, body)
	if err != nil {
		return err
	}
	return w.sendCommandAsync(cmd, doneChan, args)
}
// 同步发送
func (w *Producer) Publish(topic string, body []byte) error {
	return w.sendCommand(Publish(topic, body))
}
// 同步发送，支持批量
func (w *Producer) MultiPublish(topic string, body [][]byte) error {
	cmd, err := MultiPublish(topic, body)
	if err != nil {
		return err
	}
	return w.sendCommand(cmd)
}
// 延迟发送，同步调用
func (w *Producer) DeferredPublish(topic string, delay time.Duration, body []byte) error {
	return w.sendCommand(DeferredPublish(topic, delay, body))
}
// 延迟发送，异步调用
func (w *Producer) DeferredPublishAsync(topic string, delay time.Duration, body []byte,
	doneChan chan *ProducerTransaction, args ...interface{}) error {
	return w.sendCommandAsync(DeferredPublish(topic, delay, body), doneChan, args)
}
```

从上面6个方法中的具体内容可以看出，发送消息基本上分为两步：

1. 构造command
2. 调用sendCommand或sendCommandAsync

我们先来看看构造command的过程，command的定义与相关方法位于`command.go`文件中，这里主要介绍它的结构以及与发送消息相关的方法，如下：
```go
type Command struct {
	Name   []byte
	Params [][]byte
	Body   []byte
}

func Publish(topic string, body []byte) *Command {
	var params = [][]byte{[]byte(topic)}
	return &Command{[]byte("PUB"), params, body}
}

func DeferredPublish(topic string, delay time.Duration, body []byte) *Command {
	var params = [][]byte{[]byte(topic), []byte(strconv.Itoa(int(delay / time.Millisecond)))}
	return &Command{[]byte("DPUB"), params, body}
}

func MultiPublish(topic string, bodies [][]byte) (*Command, error) {
	var params = [][]byte{[]byte(topic)}

	num := uint32(len(bodies))
	bodySize := 4
	for _, b := range bodies {
		bodySize += len(b) + 4
	}
	body := make([]byte, 0, bodySize)
	buf := bytes.NewBuffer(body)

	err := binary.Write(buf, binary.BigEndian, &num)
	if err != nil {
		return nil, err
	}
	for _, b := range bodies {
		err = binary.Write(buf, binary.BigEndian, int32(len(b)))
		if err != nil {
			return nil, err
		}
		_, err = buf.Write(b)
		if err != nil {
			return nil, err
		}
	}

	return &Command{[]byte("MPUB"), params, buf.Bytes()}, nil
}
```

可以看到，构造command的过程就是填充Command结构体，包含了命令名称，命令参数（topic、延迟发送的时间），具体消息内容三个部分。
我们再来看看发送命令的过程，即调用sendCommand或sendCommandAsync，这两个方法位于`producer.go`文件中，如下：
```go
func (w *Producer) sendCommand(cmd *Command) error {
	doneChan := make(chan *ProducerTransaction)
	err := w.sendCommandAsync(cmd, doneChan, nil)
	if err != nil {
		close(doneChan)
		return err
	}
	t := <-doneChan
	return t.Error
}

func (w *Producer) sendCommandAsync(cmd *Command, doneChan chan *ProducerTransaction,
	args []interface{}) error {
	// keep track of how many outstanding producers we're dealing with
	// in order to later ensure that we clean them all up...
	atomic.AddInt32(&w.concurrentProducers, 1)
	defer atomic.AddInt32(&w.concurrentProducers, -1)

	if atomic.LoadInt32(&w.state) != StateConnected {
		err := w.connect()
		if err != nil {
			return err
		}
	}

	t := &ProducerTransaction{
		cmd:      cmd,
		doneChan: doneChan,
		Args:     args,
	}

	select {
	case w.transactionChan <- t:
	case <-w.exitChan:
		return ErrStopped
	}

	return nil
}
```

可以看到，sendCommand最终也是调用了sendCommandAsync，先看下sendCommandAsync方法的三个参数，第一个参数是command，也就是我们上一步构造的命令；第二个参数是一个管道，这个主要是用来支持异步调用，我们发送消息时可以单独创建一个管道，开启一个goroutine来异步接收管道中的返回结果，sendCommandAsync会通过管道将异步调用的结果发送给调用方，当处于同步调用时，我们可以看到sendCommand中内部自己构造了一个channel，然后调用sendCommandAsync，接着等待channel返回值，因为sendCommand中并没有单独开启一个goroutine去异步接收，从而实现了同步调用的效果；第三个参数是异步调用时用户自定义的参数。了解了参数之后，我们来看下函数具体的执行过程：
1. 调用atomic.AddInt32原子性的增加当前并发的生产者数量，通过defer在函数执行完后减掉刚刚递增的数量
2. 判断当前生产者的连接是否有效，如果未连接则调用connect()方法建立连接
3. 构造发送消息的任务并通过管道发送

这里我们会有个很直接的疑问，发送消息的任务通过管道发送之后，谁来处理呢？谁来真正调用底层的连接进行消息的发送呢？答案就在第二步中的connect()方法中，我们不妨来看下connect()方法：
```go
func (w *Producer) connect() error {
	w.guard.Lock()
	defer w.guard.Unlock()

	w.conn = NewConn(w.addr, &w.config, &producerConnDelegate{w})

	_, err := w.conn.Connect()
	if err != nil {
		w.conn.Close()
		return err
	}
	atomic.StoreInt32(&w.state, StateConnected)
	w.closeChan = make(chan int)
	w.wg.Add(1)
	go w.router()

	return nil
}
```
上面的connect方法经过了处理，部分无关紧要的内容已经被略去，可以看到主要流程就是对conn字段的初始化，并调用conn.connect，然后go出一个协程执行w.router()，我们重点看下w.router的具体内容：
```go
func (w *Producer) router() {
	for {
		select {
		case t := <-w.transactionChan:
			w.transactions = append(w.transactions, t)
			err := w.conn.WriteCommand(t.cmd)
			if err != nil {
				w.log(LogLevelError, "(%s) sending command - %s", w.conn.String(), err)
				w.close()
			}
		case data := <-w.responseChan:
			w.popTransaction(FrameTypeResponse, data)
		case data := <-w.errorChan:
			w.popTransaction(FrameTypeError, data)
		case <-w.closeChan:
			goto exit
		case <-w.exitChan:
			goto exit
		}
	}

exit:
	w.transactionCleanup()
	w.wg.Done()
	w.log(LogLevelInfo, "exiting router")
}
```
这下真相大白了，这个goroutine创建了一个死循环，一直接收transactionChan管道里的任务，并通过底层的连接进行发送，可以看到select还有一些其他的case，如responseChan、errorChan，主要还是接收发送消息后的服务端响应以及处理一些错误。

### 连接处理
接下来我们讲讲连接处理，这一块的逻辑主要从connect()开始，上面我们分析发送消息的源码时有看到，调用sendCommandAsync时如果producer的状态不等于StateConnected（已连接），则会调用connect()，这里用到了一个lazy connect on publish的技巧，即当发送消息时才真正建立连接。同时上面也有讲到，connect的主要流程是调用NewConn函数对conn字段进行初始化，并调用conn.connect建立连接，我们先来看下NewConn函数的源码：
```go
func NewConn(addr string, config *Config, delegate ConnDelegate) *Conn {
	return &Conn{
		addr: addr,   //地址

		config:   config,   //配置
		delegate: delegate, //委托者模式的需要实现的接口

		maxRdyCount:      2500,  //最大并发消息数
		lastMsgTimestamp: time.Now().UnixNano(), //上一次收到消息时间

		cmdChan:         make(chan *Command),     //接收命令的管道
		msgResponseChan: make(chan *msgResponse), //接收消息响应的管道
		exitChan:        make(chan int),          //退出信号的管道
		drainReady:      make(chan int),          //清空当前未处理的消息
	}
}
```
先来看看NewConn调用，前两个参数是地址和配置项，前面已有介绍，我们看下第3个参数：delegate，这里主要使用了委托者模式，由producer实现ConnDelegate中相应的接口，Conn在接收到服务端发送回来的响应时会通过这种委托者的模式调用delegate对应的方法，我们可以看到上面生产者调用NewConn时传递的第3个参数具体内容为`&producerConnDelegate{w}`，这个结构体主要实现了一些生产者所关心的内容：服务端响应、连接错误、心跳等，其他接口都为空实现，如下：
```go
type producerConnDelegate struct {
	w *Producer
}

func (d *producerConnDelegate) OnResponse(c *Conn, data []byte)       { d.w.onConnResponse(c, data) }
func (d *producerConnDelegate) OnError(c *Conn, data []byte)          { d.w.onConnError(c, data) }
func (d *producerConnDelegate) OnMessage(c *Conn, m *Message)         {}
func (d *producerConnDelegate) OnMessageFinished(c *Conn, m *Message) {}
func (d *producerConnDelegate) OnMessageRequeued(c *Conn, m *Message) {}
func (d *producerConnDelegate) OnBackoff(c *Conn)                     {}
func (d *producerConnDelegate) OnContinue(c *Conn)                    {}
func (d *producerConnDelegate) OnResume(c *Conn)                      {}
func (d *producerConnDelegate) OnIOError(c *Conn, err error)          { d.w.onConnIOError(c, err) }
func (d *producerConnDelegate) OnHeartbeat(c *Conn)                   { d.w.onConnHeartbeat(c) }
func (d *producerConnDelegate) OnClose(c *Conn)                       { d.w.onConnClose(c) }
```
接着我们再来看看conn.Connect()的具体实现，如下：
```go
func (c *Conn) Connect() (*IdentifyResponse, error) {
	dialer := &net.Dialer{
		LocalAddr: c.config.LocalAddr,
		Timeout:   c.config.DialTimeout,
	}

	conn, err := dialer.Dial("tcp", c.addr)
	if err != nil {
		return nil, err
	}
	c.conn = conn.(*net.TCPConn)
	c.r = conn
	c.w = conn

	_, err = c.Write(MagicV2)
	if err != nil {
		c.Close()
		return nil, fmt.Errorf("[%s] failed to write magic - %s", c.addr, err)
	}

	resp, err := c.identify()
	if err != nil {
		return nil, err
	}

	if resp != nil && resp.AuthRequired {
		if c.config.AuthSecret == "" {
			c.log(LogLevelError, "Auth Required")
			return nil, errors.New("Auth Required")
		}
		err := c.auth(c.config.AuthSecret)
		if err != nil {
			c.log(LogLevelError, "Auth Failed %s", err)
			return nil, err
		}
	}

	c.wg.Add(2)
	atomic.StoreInt32(&c.readLoopRunning, 1)
	go c.readLoop()
	go c.writeLoop()
	return resp, nil
}
```
整个函数主要分为4个流程：
1. 建立tcp连接
2. 发送版本号：MagicV2
3. 调用identify，将客户端的一些配置项传递给服务端，同时根据服务端响应进行一些配置项的设置
4. 开启I/O循环
前3步为nsq客户端与服务端建立完整连接的标准流程，没有特别的东西，我们重点关注I/O循环这一块，先来看下readLoop：
```go
func (c *Conn) readLoop() {
	delegate := &connMessageDelegate{c}
	for {
		frameType, data, err := ReadUnpackedResponse(c)
		if err != nil {
			if !strings.Contains(err.Error(), "use of closed network connection") {
				c.delegate.OnIOError(c, err)
			}
			goto exit
		}
		if frameType == FrameTypeResponse && bytes.Equal(data, []byte("_heartbeat_")) {
			c.delegate.OnHeartbeat(c)
			err := c.WriteCommand(Nop())
			if err != nil {
				c.delegate.OnIOError(c, err)
				goto exit
			}
			continue
		}

		switch frameType {
		case FrameTypeResponse:
			c.delegate.OnResponse(c, data)
		case FrameTypeMessage:
			msg, err := DecodeMessage(data)
			if err != nil {
				c.delegate.OnIOError(c, err)
				goto exit
			}
			msg.Delegate = delegate
			msg.NSQDAddress = c.String()
			
			c.delegate.OnMessage(c, msg)
		case FrameTypeError:
			c.delegate.OnError(c, data)
		default:
			c.delegate.OnIOError(c, fmt.Errorf("unknown frame type %d", frameType))
		}
	}
}
```
readLoop先调用ReadUnpackedResponse根据协议读取服务端发送过来的网络包，该函数具体代码在`protocol.go`文件中，主要是对协议上的处理，这里不做细讲，我们继续看接下来的逻辑，接着判断包的类型是不是心跳包，是的话直接返回响应，无须上层的生产者或消费者处理；接着是一个switch分支，有3个case：
1. 包类型为FrameTypeResponse时，表示服务端对之前客户端发送命令的响应，比如生产者发送消息的响应
2. 包类型为FrameTypeMessage时，表示收到消息，主要是消费者的场景会用到
3. 包类型为FrameTypeError时，表示服务端处理发生错误
readLoop的内容基本就是处理服务端心跳包、发回的消息、响应和错误，并调用委托者delegate通知上层的生产者或消费者，接下来我们再来分析writeLoop的源码：
```go
func (c *Conn) writeLoop() {
	for {
		select {
		case <-c.exitChan:
			close(c.drainReady)
			goto exit
		case cmd := <-c.cmdChan:
			err := c.WriteCommand(cmd)
			if err != nil {
				c.close()
				continue
			}
		case resp := <-c.msgResponseChan:
			msgsInFlight := atomic.AddInt64(&c.messagesInFlight, -1)
			if resp.success {
				c.delegate.OnMessageFinished(c, resp.msg)
				c.delegate.OnResume(c)
			} else {
				c.delegate.OnMessageRequeued(c, resp.msg)
				if resp.backoff {
					c.delegate.OnBackoff(c)
				} else {
					c.delegate.OnContinue(c)
				}
			}

			err := c.WriteCommand(resp.cmd)
			if err != nil {
				c.close()
				continue
			}

			if msgsInFlight == 0 &&
				atomic.LoadInt32(&c.closeFlag) == 1 {
				c.close()
				continue
			}
		}
	}

exit:
	c.wg.Done()
}
```
可以看到，writeLoop通过for循环加select处理三种场景：
1. 接收退出信号，清理未处理的消息
2. 从命令管道接收命令，目前这个管道只有conn的onMessageTouch方法在使用，该方法又由message的公共方法Touch调用，主要用来发送touch命令，即重置消息的超时时间
3. 从消息处理结果管道接收结果，将结果发送给nsqd服务端，主要用来通知nsqd服务端该消息是消费完成还是需要重新入队
总结一下writeLoop的内容基本就是将消息处理的结果通过命令的形式发送给nsqd服务端，如重置消息超时时间、消息完成、消息重新入队等。

### 退出

## 消费者的视角