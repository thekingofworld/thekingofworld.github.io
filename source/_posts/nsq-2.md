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
New函数用来根据启动参数创建nsqd实例，

## 客户端连接处理、通信协议

## 接收消息

## 投递消息

### 直接投递

### 延迟投递

### 流量控制

##持久化