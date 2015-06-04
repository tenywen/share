## nsqd 

**nsqd扮演的是接收，分发消息角色。nsqd中有三个重要的结构topic,channel,client。nsqd可以比作一台电视，topic是其中某个频道。channel就是位置，占到一个位置就可以看电视节目。几个人同时占一个位置时，只能大家轮流看。所以当有多个client同时sub一个channel时，同一个message只有一个client收到。**

**topic**

每个topic创建时，都会go messagePump(),它负责将client发送的message分发给每个channel。

**channel**

每个channel创建时，都会go messagePump(),它负责接收message,然后push到channel.clientMsgChan。channel.clientMsgChan是channel暴露给client唯一的go-chan。

channel维护两个队列InFlightQueue和DeferredQueue。client 发布的有timeout的message会被channel push到DeferredQueue等待timeout后处理

**client**

nsqd会为主动连接的conn创建client。client从channel.clientMsgChan中得到message。当client接收到消息后，会将message在再次放入到channel的InFlightQueue。以防止由于网络问题，client没有收到message。只有channel收到client发送的FIN时，才会删除message。

=====================================================================

### 1. nsqd的main()函数在apps/nsqd/nsqd.go.

main()函数的主要工作：

* 创建nsqd。
* 监听端口。

		func main() {
			// 设置默认配置
			// 从stdin读取配置
			...

			// 设置捕获终端信号 
			...

			// resolve ops from config
			... 

			// 创建nsqd 
			nsqd := nsqd.NewNSQD(opts) 

			// 看起来好像是读取opts.DataPath路径的文件。实际上就是为nsqd的配置文件创建topics和channels
			nsqd.LoadMetadata() 

			// 记录topics/channels 
			// topics 记录topic.Name,topic.IsPaused()
			// channel 记录channel.Name,channel.IsPaused()
			err := nsqd.PersistMetadata()

			// Main主要做三件事 
			// 1.监听 tcp/http/https 端口 
			// 2.为每个连接到nsqd的conn建立一个client
			// 3.client负责处理conn传输的命令
			nsqd.Main()

			// 阻塞！等待终端信号 
			<- signalChan 

			// 等待nsqd.Main()所有的wg.Done()完成
			nsqd.Exit()
		}

### 2. nsqd.LoadMetadata()在nsqd/nsqd.go.
* 读取dat文件,检查nsqd中是否存在dat中的topic和channel,没有则创建

		func (n *NSQD) LoadMetadata() {
			n.setFlag(flagLoading, true)
			defer n.setFlag(flagLoading, false)
			fn := fmt.Sprintf(path.Join(n.opts.DataPath, "nsqd.%d.dat"), n.opts.ID)
			data, err := ioutil.ReadFile(fn)
			if err != nil {
				return
			}

			js, err := simplejson.NewJson(data)
			if err != nil {
				return
			}

			// 从json中获得topics	
			topics, err := js.Get("topics").Array()
			if err != nil {
				return
			}

			// 如果nsqd中没有该topic则创建
			for ti := range topics {
				topicJs := js.Get("topics").GetIndex(ti)

				topicName, err := topicJs.Get("name").String()
				if err != nil {
					return
				}

				topic := n.GetTopic(topicName)
		   		paused, _ := topicJs.Get("paused").Bool()
		   		// 我的理解是如果nsqd刚启动的时候，设置了pause=true，那么就会永远阻塞在这里
		   		if paused {
			   		topic.Pause()
		   		}

	   			channels, err := topicJs.Get("channels").Array()
		   		if err != nil {
			   		return
		   		}

	   			// 没有该channel，则创建
	   			for ci := range channels {
					channelJs := topicJs.Get("channels").GetIndex(ci)
					channelName, err := channelJs.Get("name").String()
					if err != nil {
						return
					}
					channel := topic.GetChannel(channelName)

					paused, _ = channelJs.Get("paused").Bool() 
					// nsqd启动时，设置了paused=true，那么就玩完了，因为此时连client都没有!
					if paused {
						channel.Pause()
					}
				}	
			}
		}

nsqd.LoadMetadata()和nsqd.Main()函数中均会涉及到NewTopic()和NewChannel().
#### (1).NewTopic()和messagePump()在nsqd/topic.go.
*	创建topic对象，开启goroutine处理chans.
* 	topic.messagePump()将topic的数据分发给channels

		func NewTopic(topicName string, ...) *Topic {
			t := &Topic{}  
			// 处理topic里面的chans
			go t.messagePump() 
			return t
		}

    	func (t *Topic)messagePump() {
			for {
				select {
					case msg = <- memoryMsgChan: // 获得内存中的message
					case buf = <- backendChan:  // 获得 backend队列的message 
						msg,err = decodeMessage(buf)
					case <- t.ChannelUpdateChan: // 更新channel  
						continue
					case pause := <- t.PauseChan: 	// 暂停或者开始
					continue 
					case <- t.exitChan: // 退出
						goto exit
				}

				// topic 的数据分发给channels
				for i,channel := range chans {
					chanMsg := msg 
					// 当channel数大于1时，拷贝message发送给每个channel
					if i > 0 {
						chanMsg = NewMessage(msg.ID,msg.Body)
					}
					// channel.memoryMsgChan容量未满，则chanMsg 放入到channel.memoryMsgChan。否则存入c.backend
					channel.PutMessage(chanMsg)
				}
			}
    	}

#### (2).NewChannel()和messagePump()在nsqd/channel.go.
*	创建channel对象，开启goroutine处理chans 

    	func (c *Channel)NewChannel(topicName string,channelName string,...) *Channel{
			c := &Channel{}
			go c.messagePump()	
		}

    	func (c *Channel) messagePump() {
			for {
				select {
					case msg = <- c.memoryMsgChan: 
					case buf = <- c.backend.ReadChan():
						msg,err = decodeMessage(buff)
					case <- c.exitChan:  // 退出
						goto exit
				}

				atomic.StoreInt32(&c.bufferedCount, 1) 
				// c.clientMsgChan会随机分配数据给连接到channel的clients
				c.clientMsgChan <- msg
				atomic.StoreInt32(&c.bufferedCount, 0)
			}
		}


### 3. nsqd.Main() 在nsqd/nsqd.go
* 主要功能为监听端口，为每个连接创建client。可以通过tcp，http，https连接到nsqd，这里只分析tcp。

		func (n *NSQD) Main() {
			// 监听tcp
			tcpListener, err := net.Listen("tcp", n.opts.TCPAddress)
			if err != nil {
				os.Exit(1)
			}
			
			n.Lock()
			n.tcpListener = tcpListener
			n.Unlock()
			tcpServer := &tcpServer{ctx: ctx}
			// goroutine
			n.waitGroup.Wrap(func() {
				protocol.TCPServer(n.tcpListener, tcpServer, n.opts.Logger)
			})

			// 监听http 
			...
			
			// 监听https
			... 

			// nsqd为每个channel建立一个queueWorkerLoop(),专门处理channel InFlightQueue和DeferredQueue 的message
			n.waitGroup.Wrap(func() { n.queueScanLoop() })
		}

#### TCPServer()在internal/protocol/tcp_server.go 
* 连接的实际处理函数 

		func TCPServer(listener net.Listener, handler TCPHandler, l app.Logger) {
			for {
				clientConn, err := listener.Accept()
				if err != nil {
					if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
						continue
					}
					break
				}
				// handler函数
				go handler.Handle(clientConn)
			}
		}

##### Handle()在nsqd/tcp.go
* 检查协议版本，调用IOLoop()
		
		func (p *tcpServer) Handle(clientConn net.Conn) {

			// 封装又封装。。。
			// 循环处理clinetConn数据 
			err = prot.IOLoop(clientConn)
			if err != nil {
				return
			}
		}

###### IOLoop()在nsqd/protocol_v2.go 
* 解析clientConn协议，并执行协议。

		func (p *protocolV2) IOLoop(conn net.Conn) error {
			clientID := atomic.AddInt64(&p.ctx.nsqd.clientIDSequence, 1)
			client := newClientV2(clientID, conn, p.ctx)

			messagePumpStartedChan := make(chan bool)
			go p.messagePump(client, messagePumpStartedChan)
			// 同步 ！防止messagePump中client处理chan的函数还没有执行
			<-messagePumpStartedChan

			for {
				// 读取一行数据，去除回车换行符
				line, err = client.Reader.ReadSlice('\n')
				if err != nil {
					if atomic.LoadInt32(&client.State) == stateClosing {
						err = nil
					} else {
						err = fmt.Errorf("failed to read command - %s", err)
					}
					break
				}
				line = line[:len(line)-1]
				if len(line) > 0 && line[len(line)-1] == '\r' {
					line = line[:len(line)-1]
				}
				params := bytes.Split(line, separatorBytes)

				// 执行params对于的命令	
				response, err := p.Exec(client, params)
			}
		}


p.messagePump()在nsqd/protocol_v2.go 
* 处理client channels 
	
		func (p *protocolV2) messagePump(client *clientV2, startedChan chan bool) {
			var clientMsgChan chan *Message
			var subChannel *Channel
			var flusherChan <-chan time.Time

			subEventChan := client.SubEventChan
			identifyEventChan := client.IdentifyEventChan
		
			outputBufferTicker := time.NewTicker(client.OutputBufferTimeout)
			heartbeatTicker := time.NewTicker(client.HeartbeatInterval)
			heartbeatChan := heartbeatTicker.C
			msgTimeout := client.MsgTimeout

			flushed := true

			close(startedChan)

			for {
				if subChannel == nil || !client.IsReadyForMessages() { // 刷新client之前的数据
					clientMsgChan = nil
					flusherChan = nil
					client.Lock()
					err = client.Flush()
					client.Unlock()
					if err != nil {
						goto exit
					}
					flushed = true
				} else if flushed { // 刷新之后，等待client连接的channel的数据。
					clientMsgChan = subChannel.clientMsgChan
					flusherChan = nil
				} else { // 设置一个刷新timer
					clientMsgChan = subChannel.clientMsgChan
					flusherChan = outputBufferTicker.C
				}

				select {
					case <-flusherChan: // 通知刷新
						client.Lock()
						err = client.Flush()
						client.Unlock()
						if err != nil {
							goto exit
						}
						flushed = true
					case <-client.ReadyStateChan: // client 准备好了。
					case subChannel = <-subEventChan: // 已经给client 分配了一个channel
						// 一个client 只能分配一次
						subEventChan = nil
					case identifyData := <-identifyEventChan: // 设置identify
						// 只能设置一次
						identifyEventChan = nil

						outputBufferTicker.Stop()
						if identifyData.OutputBufferTimeout > 0 {
							outputBufferTicker = time.NewTicker(identifyData.OutputBufferTimeout)
						}

						heartbeatTicker.Stop()
						heartbeatChan = nil
						if identifyData.HeartbeatInterval > 0 {
							heartbeatTicker = time.NewTicker(identifyData.HeartbeatInterval)
							heartbeatChan = heartbeatTicker.C
						}

						if identifyData.SampleRate > 0 {
							sampleRate = identifyData.SampleRate
						}

						msgTimeout = identifyData.MsgTimeout
					case <-heartbeatChan: // 心跳检查
						err = p.Send(client, frameTypeResponse, heartbeatBytes)
						if err != nil {
							goto exit
						}
					case msg, ok := <-clientMsgChan: // channel的数据
						if !ok {
							goto exit
						}
						// 再次保存message，防止client并没有收到message
						subChannel.StartInFlightTimeout(msg, client.ID, msgTimeout)
						// 计数器 + 1	
						client.SendingMessage()
						err = p.SendMessage(client, msg, &buf)
						if err != nil {
							goto exit
						}
						flushed = false
					case <-client.ExitChan:
						goto exit
					}
				}
			}

