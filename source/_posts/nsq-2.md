---
title: 消息队列NSQ源码分析（三）：nsqd
categories:
    - 消息队列
tags:
    - nsq
---
本篇为NSQ源码分析的第三篇，主要分析nsq的服务端源码，即nsqd

<!-- more -->

## 启动过程

首先看一下nsqd的启动过程，我们从`apps/nsqd/main.go`开始，可以看到在main包中声明了`program`结构体，该结构体包装了nsqd实例，提供了Init、Start、Stop方法。接着我们看main函数，主要逻辑就是创建program结构体，然后调用`github.com/judwhite/go-svc`这个库提供的svc.Run方法，该方法底层实现则是调用program的Init、Start方法，引入这个库主要是为了提供创建可在其自身的 Windows 会话中长时间运行的可执行应用程序的支持。现在我们重点看下program的Start方法，简化后的逻辑如下：
```go
func (p *program) Start() error {
	opts := nsqd.NewOptions()
	flagSet := nsqdFlagSet(opts)
	flagSet.Parse(os.Args[1:])
	options.Resolve(opts, flagSet, cfg)
	nsqd, err := nsqd.New(opts)
	if err != nil {
		logFatal("failed to instantiate nsqd - %s", err)
	}
	p.nsqd = nsqd

	err = p.nsqd.LoadMetadata()
	if err != nil {
		logFatal("failed to load metadata - %s", err)
	}
	err = p.nsqd.PersistMetadata()
	if err != nil {
		logFatal("failed to persist metadata - %s", err)
	}

	go func() {
		err := p.nsqd.Main()
		if err != nil {
			p.Stop()
			os.Exit(1)
		}
	}()

	return nil
}
```
大致流程即为解析启动参数、创建nsqd实例、加载之前持久化的元数据（topic、channel等）、执行nsqd实例的main函数，接着我们看下nsqd.New与nsqd.Main两个函数：
```go
func New(opts *Options) (*NSQD, error) {
	var err error
	n := &NSQD{
		startTime:            time.Now(),
		topicMap:             make(map[string]*Topic),
		clients:              make(map[int64]Client),
		exitChan:             make(chan int),
		notifyChan:           make(chan interface{}),
		optsNotificationChan: make(chan struct{}, 1),
		dl:                   dirlock.New(dataPath),
	}
	httpcli := http_api.NewClient(nil, opts.HTTPClientConnectTimeout, opts.HTTPClientRequestTimeout)
	n.ci = clusterinfo.New(n.logf, httpcli)

	n.lookupPeers.Store([]*lookupPeer{})

	n.swapOpts(opts)
	n.errValue.Store(errStore{})

	err = n.dl.Lock()
	if err != nil {
		return nil, fmt.Errorf("--data-path=%s in use (possibly by another instance of nsqd)", dataPath)
	}
	
	n.tcpListener, err = net.Listen("tcp", opts.TCPAddress)
	if err != nil {
		return nil, fmt.Errorf("listen (%s) failed - %s", opts.TCPAddress, err)
	}
	n.httpListener, err = net.Listen("tcp", opts.HTTPAddress)
	if err != nil {
		return nil, fmt.Errorf("listen (%s) failed - %s", opts.HTTPAddress, err)
	}
	if n.tlsConfig != nil && opts.HTTPSAddress != "" {
		n.httpsListener, err = tls.Listen("tcp", opts.HTTPSAddress, n.tlsConfig)
		if err != nil {
			return nil, fmt.Errorf("listen (%s) failed - %s", opts.HTTPSAddress, err)
		}
	}

	return n, nil
}
```
nsqd.New用来根据启动参数创建nsqd实例，同时开启tcp、http两个端口监听，接着我们看下nsqd.Main的内容：
```go
func (n *NSQD) Main() error {
	ctx := &context{n}

	exitCh := make(chan error)
	var once sync.Once
	exitFunc := func(err error) {
		once.Do(func() {
			if err != nil {
				n.logf(LOG_FATAL, "%s", err)
			}
			exitCh <- err
		})
	}

	tcpServer := &tcpServer{ctx: ctx}
	n.waitGroup.Wrap(func() {
		exitFunc(protocol.TCPServer(n.tcpListener, tcpServer, n.logf))
	})
	httpServer := newHTTPServer(ctx, false, n.getOpts().TLSRequired == TLSRequired)
	n.waitGroup.Wrap(func() {
		exitFunc(http_api.Serve(n.httpListener, httpServer, "HTTP", n.logf))
	})
	if n.tlsConfig != nil && n.getOpts().HTTPSAddress != "" {
		httpsServer := newHTTPServer(ctx, true, true)
		n.waitGroup.Wrap(func() {
			exitFunc(http_api.Serve(n.httpsListener, httpsServer, "HTTPS", n.logf))
		})
	}

	n.waitGroup.Wrap(n.queueScanLoop)
	n.waitGroup.Wrap(n.lookupLoop)
	if n.getOpts().StatsdAddress != "" {
		n.waitGroup.Wrap(n.statsdLoop)
	}

	err := <-exitCh
	return err
}
```
通过上述代码可以看出，nsqd.Main主要包含逻辑有：
1、开启单独协程对tcp连接进行处理
2、开启单独协程对http连接进行处理
3、如果存在https设置，则开启单独协程处理https请求
4、开启单独协程执行queueScanLoop，主要用来处理延迟消息以及超时消息，文章后面讲投递消息时会详细描述
5、开启单独协程执行lookupLoop，主要用来与nsqlookupd通信，保持心跳、上报数据等
6、如果配置了StatsdAddress，则开启单独协程执行statsdLoop，主要用来上报内存分配、gc、topic和channel等数据

## 客户端连接处理、通信协议

现在我们来了解一下nsqd实例处理客户端连接的具体过程，上面Main中的代码可以看到，nsqd在tcp连接处理这一块采用的还是go语言中常规的网络模型，即调用accept接收连接，单独开启协程处理该连接，我们从tcp.go中的Handle方法开始，看看连接的具体处理过程：
```go
func (p *tcpServer) Handle(clientConn net.Conn) {
	buf := make([]byte, 4)
	_, err := io.ReadFull(clientConn, buf)
	if err != nil {
		p.ctx.nsqd.logf(LOG_ERROR, "failed to read protocol version - %s", err)
		clientConn.Close()
		return
	}
	protocolMagic := string(buf)

	p.ctx.nsqd.logf(LOG_INFO, "CLIENT(%s): desired protocol magic '%s'",
		clientConn.RemoteAddr(), protocolMagic)

	var prot protocol.Protocol
	switch protocolMagic {
	case "  V2":
		prot = &protocolV2{ctx: p.ctx}
	default:
		protocol.SendFramedResponse(clientConn, frameTypeError, []byte("E_BAD_PROTOCOL"))
		clientConn.Close()
		p.ctx.nsqd.logf(LOG_ERROR, "client(%s) bad protocol magic '%s'",
			clientConn.RemoteAddr(), protocolMagic)
		return
	}

	err = prot.IOLoop(clientConn)
	if err != nil {
		p.ctx.nsqd.logf(LOG_ERROR, "client(%s) - %s", clientConn.RemoteAddr(), err)
		return
	}
}
```
可以看到，处理过程首先会读取4个字节的版本号，并判断当前协议版本是否为V2，不是的话则终止连接，接着创建protocolV2实例，调用IOLoop进入客户端连接的读写循环，我们再看下IOLoop的具体代码：
```go
func (p *protocolV2) IOLoop(conn net.Conn) error {
	var err error
	var line []byte
	var zeroTime time.Time

	clientID := atomic.AddInt64(&p.ctx.nsqd.clientIDSequence, 1)
	client := newClientV2(clientID, conn, p.ctx)
	p.ctx.nsqd.AddClient(client.ID, client)

	messagePumpStartedChan := make(chan bool)
	go p.messagePump(client, messagePumpStartedChan)
	<-messagePumpStartedChan

	for {
		if client.HeartbeatInterval > 0 {
			client.SetReadDeadline(time.Now().Add(client.HeartbeatInterval * 2))
		} else {
			client.SetReadDeadline(zeroTime)
		}

		// ReadSlice does not allocate new space for the data each request
		// ie. the returned slice is only valid until the next call to it
		line, err = client.Reader.ReadSlice('\n')
		if err != nil {
			if err == io.EOF {
				err = nil
			} else {
				err = fmt.Errorf("failed to read command - %s", err)
			}
			break
		}

		// trim the '\n'
		line = line[:len(line)-1]
		// optionally trim the '\r'
		if len(line) > 0 && line[len(line)-1] == '\r' {
			line = line[:len(line)-1]
		}
		params := bytes.Split(line, separatorBytes)

		p.ctx.nsqd.logf(LOG_DEBUG, "PROTOCOL(V2): [%s] %s", client, params)

		var response []byte
		response, err = p.Exec(client, params)
		if err != nil {
			ctx := ""
			if parentErr := err.(protocol.ChildErr).Parent(); parentErr != nil {
				ctx = " - " + parentErr.Error()
			}
			p.ctx.nsqd.logf(LOG_ERROR, "[%s] - %s%s", client, err, ctx)

			sendErr := p.Send(client, frameTypeError, []byte(err.Error()))
			if sendErr != nil {
				p.ctx.nsqd.logf(LOG_ERROR, "[%s] - %s%s", client, sendErr, ctx)
				break
			}

			// errors of type FatalClientErr should forceably close the connection
			if _, ok := err.(*protocol.FatalClientErr); ok {
				break
			}
			continue
		}

		if response != nil {
			err = p.Send(client, frameTypeResponse, response)
			if err != nil {
				err = fmt.Errorf("failed to send response - %s", err)
				break
			}
		}
	}

	p.ctx.nsqd.logf(LOG_INFO, "PROTOCOL(V2): [%s] exiting ioloop", client)
	conn.Close()
	close(client.ExitChan)
	if client.Channel != nil {
		client.Channel.RemoveClient(client.ID)
	}

	p.ctx.nsqd.RemoveClient(client.ID)
	return err
}
```
IOLoop的主要逻辑即为循环读取客户端发送的命令并执行，然后给客户端返回响应，需要注意的是，代码中单独开启了一个协程执行messagePump，该方法主要承担接收消息并分发给客户端、向客户端发送心跳包等功能

## 接收消息

接收消息的过程由客户端发送PUB命令开始，然后再由protocol_v2.go中的IOLoop中读取命令并执行，PUB命令的具体执行过程大致如下：
1、解析PUB命令参数，得到topic名称以及消息内容
2、根据topic名称获取topic实例，调用topic.PutMessage发送消息，PutMessage方法实现上则是往topic内部的消息管道（memoryMsgChan）中发送消息

## 投递消息

### 直接投递

之前我们讲解了nsq中存在的Topic和Channel等概念，因此一条消息的投递会先由Topic分发给Channel，然后再由Channel到client，我们分别看下这两个投递过程

1、Topic -> Channel，每个Topic创建出来的时候（NewTopic方法）都会开启一个额外的协程执行messagePump，我们看看该方法的具体逻辑：
```go
func (t *Topic) messagePump() {
	t.RLock()
	for _, c := range t.channelMap {
		chans = append(chans, c)
	}
	t.RUnlock()

	for {
		select {
		case msg = <-memoryMsgChan:
		case buf = <-backendChan:
			msg, err = decodeMessage(buf)
			if err != nil {
				t.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
				continue
			}
		}

		for i, channel := range chans {
			chanMsg := msg
			if i > 0 {
				chanMsg = NewMessage(msg.ID, msg.Body)
				chanMsg.Timestamp = msg.Timestamp
				chanMsg.deferred = msg.deferred
			}
			if chanMsg.deferred != 0 {
				channel.PutMessageDeferred(chanMsg, chanMsg.deferred)
				continue
			}
			err := channel.PutMessage(chanMsg)
			if err != nil {
				t.ctx.nsqd.logf(LOG_ERROR,
					"TOPIC(%s) ERROR: failed to put msg(%s) to channel(%s) - %s",
					t.name, msg.ID, channel.name, err)
			}
		}
	}

exit:
	t.ctx.nsqd.logf(LOG_INFO, "TOPIC(%s): closing ... messagePump", t.name)
}
```
上面代码经过简化后，只保留了与消息投递有关的代码，可以看到messagePump方法会一直从Topic的消息管道中读取消息，然后将消息分发给当前所有的Channel

2、channel -> client，第一步中topic将消息分发给Channel，实现上是Channel会将消息发给自己的内部管道memoryMsgChan，客户端会从该消息管道中读取消息，从而到达消息分发的目的，具体代码可以看protocol_v2.go中的messagePump方法

### 延迟投递

nsq中支持了消息的延迟投递，我们来看看具体实现的逻辑：
1、每个Topic对应的Channel内部结构中存在一个优先级队列（Channel.deferredPQ，具体实现为最小堆），通过DeferredPublish发送的延迟消息首先会进入到Channel的优先级队列中，队列中每个元素的优先级具体值为延迟的时间
2、nsqd启动时有个queueScanLoop函数，这个函数每隔一定的时间间隔（配置项：QueueScanInterval）会拿到当前所有的channel，取一部分（配置项：QueueScanSelectionCount）进行扫描，并调用Channel的processDeferredQueue方法，processDeferredQueue方法则用来处理延迟消息，会从其对应的优先级队列中取出比当前时间小的那些消息进行发送，发送的过程跟上述的直接投递过程一样，这里就不细讲了

### 超时处理

nsq中的消息投递模型为至少一次，那么就需要引入超时重传与确认机制，我们接着具体看看nsq是如何处理超时消息的：
1、与延迟投递类似，每个Topic对应的Channel内部结构中还存在另一个优先级队列（Channel.inFlightPQ），发送给客户端且还未收到处理结果的消息会存放在队列中，元素的优先级具体值为消息超时的时间
2、上面将延迟投递的时候，有讲到queueScanLoop这个函数，此函数在扫描Channel过程中也会调用Channel的processInFlightQueue方法，processInFlightQueue则会处理Channel中的超时消息，将这些超时消息取出来重新投递

### 流量控制

关于流量控制这一块我们在上一篇讲消费者客户端的时候已经有所描述，即通过发送RDY状态来限制客户端的消息接收，下面我们就来看看nsqd实例在接收客户端发送的RDY值之后会有哪些变化，具体代码就在`nsqd/protocol_v2.go`中：
```go
func (p *protocolV2) RDY(client *clientV2, params [][]byte) ([]byte, error) {
	count := int64(1)
	if len(params) > 1 {
		b10, err := protocol.ByteToBase10(params[1])
		if err != nil {
			return nil, protocol.NewFatalClientErr(err, "E_INVALID",
				fmt.Sprintf("RDY could not parse count %s", params[1]))
		}
		count = int64(b10)
	}

	if count < 0 || count > p.ctx.nsqd.getOpts().MaxRdyCount {
		// this needs to be a fatal error otherwise clients would have
		// inconsistent state
		return nil, protocol.NewFatalClientErr(nil, "E_INVALID",
			fmt.Sprintf("RDY count %d out of range 0-%d", count, p.ctx.nsqd.getOpts().MaxRdyCount))
	}

	client.SetReadyCount(count)

	return nil, nil
}
```
当nsqd接收到客户端发送的RDY命令时，会调用client.SetReadyCount(count)方法，将该客户端ReadyCount设置为count，当客户端触发流量控制时，count即为0，接着我们看投递消息给客户端时的处理逻辑：
```go
func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
	for {
		if subChannel == nil || !client.IsReadyForMessages() {
			// the client is not ready to receive messages...
			memoryMsgChan = nil
			backendMsgChan = nil
			flusherChan = nil
			// force flush
			client.writeLock.Lock()
			err = client.Flush()
			client.writeLock.Unlock()
			if err != nil {
				goto exit
			}
			flushed = true
		}
    
        select {
        case <-client.ReadyStateChan:
        case <-heartbeatChan:
        case b := <-backendMsgChan:
        case msg := <-memoryMsgChan:
        }
	}
}
```
这里我将messagePump其他逻辑都屏蔽掉了，想要突出的是if条件判断中的`!client.IsReadyForMessages()`，当消费者客户端进行流量控制时，会将RDY置为0，此时client的IsReadyForMessages则返回false，整个条件判断逻辑也就为true，然后执行if块中的代码，即将memoryMsgChan和backendMsgChan设置为nil，置为nil后代码之后的select逻辑也就无法从消息管道中获取消息，从而达到限制该客户端接收消息的目的

##持久化

nsq目前有两块持久化数据的逻辑：
1、当nsqd实例存在topic、channel新建、删除等元数据变化的时候，会进行持久化，具体方法可见nsqd.PersistMetadata
2、nsqd优先采用内存存储消息，当堆积的消息超过配置的最大内存消息数时则会进行持久化存储，目前nsq基于文件系统实现了一个先进先出的队列，源码地址：github.com/go-diskqueue