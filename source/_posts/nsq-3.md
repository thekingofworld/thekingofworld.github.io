---
title: 消息队列NSQ源码分析（四）：nsqlookupd
categories:
    - 消息队列
tags:
    - nsq
---
本篇为NSQ源码分析的第四篇，主要分析nsq服务端发现机制的源码，即nsqlookupd

<!-- more -->

## 概述

nsqlookupd主要用来管理生产者实例（即nsqd）的注册与发现。nsqd实例启动时可以指定nsqlookupd实例地址，启动后会以一定时间间隔上报实例信息到nsqlookupd。消费者客户端通过查询nsqlookupd即可实现动态发现当前生产指定topic的nsqd实例。下面我们将从启动过程、生产者注册、生产者发现等3个模块来讲述nsqlookupd的具体实现。

## 启动过程

我们从`apps/nsqlookupd/main.go`开始，可以看到其代码结构与之前的nsqd类似，使用了program包装，同时实现了Init、Start、Stop等方法，具体看下Start方法的执行过程：
```go
func (p *program) Start() error {
	opts := nsqlookupd.NewOptions()

	flagSet := nsqlookupdFlagSet(opts)
	flagSet.Parse(os.Args[1:])

	var cfg map[string]interface{}
	configFile := flagSet.Lookup("config").Value.String()
	if configFile != "" {
		_, err := toml.DecodeFile(configFile, &cfg)
		if err != nil {
			logFatal("failed to load config file %s - %s", configFile, err)
		}
	}

	options.Resolve(opts, flagSet, cfg)
	nsqlookupd, err := nsqlookupd.New(opts)
	if err != nil {
		logFatal("failed to instantiate nsqlookupd", err)
	}
	p.nsqlookupd = nsqlookupd

	go func() {
		err := p.nsqlookupd.Main()
		if err != nil {
			p.Stop()
			os.Exit(1)
		}
	}()

	return nil
}
```
大致流程仍然与nsqd类似：解析启动参数，创建nsqlookupd实例、执行nsqlookupd实例的Main函数，接着我们看下nsqlookupd.New与nsqlookupd.Main两个函数：
```go
func New(opts *Options) (*NSQLookupd, error) {
	var err error

	if opts.Logger == nil {
		opts.Logger = log.New(os.Stderr, opts.LogPrefix, log.Ldate|log.Ltime|log.Lmicroseconds)
	}
	l := &NSQLookupd{
		opts: opts,
		DB:   NewRegistrationDB(),
	}

	l.logf(LOG_INFO, version.String("nsqlookupd"))

	l.tcpListener, err = net.Listen("tcp", opts.TCPAddress)
	if err != nil {
		return nil, fmt.Errorf("listen (%s) failed - %s", opts.TCPAddress, err)
	}
	l.httpListener, err = net.Listen("tcp", opts.HTTPAddress)
	if err != nil {
		return nil, fmt.Errorf("listen (%s) failed - %s", opts.TCPAddress, err)
	}

	return l, nil
}
```
nsqlookupd.New用来根据启动参数创建nsqd实例，同时开启tcp、http两个端口监听，接着我们看下nsqlookupd.Main的内容：
```go
func (l *NSQLookupd) Main() error {
	ctx := &Context{l}

	exitCh := make(chan error)
	var once sync.Once
	exitFunc := func(err error) {
		once.Do(func() {
			if err != nil {
				l.logf(LOG_FATAL, "%s", err)
			}
			exitCh <- err
		})
	}

	tcpServer := &tcpServer{ctx: ctx}
	l.waitGroup.Wrap(func() {
		exitFunc(protocol.TCPServer(l.tcpListener, tcpServer, l.logf))
	})
	httpServer := newHTTPServer(ctx)
	l.waitGroup.Wrap(func() {
		exitFunc(http_api.Serve(l.httpListener, httpServer, "HTTP", l.logf))
	})

	err := <-exitCh
	return err
}
```
通过上述代码可以看出，nsqlookupd.Main主要包含逻辑有：
1、开启单独协程对tcp连接进行处理
2、开启单独协程对http连接进行处理

## 生产者注册

生产者实例信息注册通过tcp连接通信来完成，关于连接处理、通信这一块大致的流程与nsqd类似，主要就是接受nsqd发送过来的信息，这些信息主要包括topic、channel、nsqd实例地址、tcp端口号、http端口号等，然后nsqlookupd会将这些信息放置在内存中的一个RegistrationDB结构中并维护，具体实现代码在`nsqlookupd/lookup_protocol_v1.go`与`registration_db.go`中，因整个过程实现较为简单，有兴趣的同学可以直接查看源码，这里就不再详细描述了。

## 生产者发现

生产者的发现则是通过http接口查询上述RegistrationDB结构中的信息来实现，消费者客户端会定时轮询该http接口以支持动态发现生产指定topic的nsqd实例，具体实现代码可查看`nsqlookupd/http.go`。