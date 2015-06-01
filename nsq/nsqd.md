## nsqd
### 1. nsqd的main()函数在apps/nsqd/nsqd.go.

main()函数的主要工作：

* 创建nsqd。
* 监听端口，为每个连接创建client

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

			// 看起来好像是读取opts.DataPath路径的文件。其实是创建topics。
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
#### (1).NewTopic()在nsqd/topic.go.
*	创建topic对象，开启goroutine处理chans.

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

				for i,channel := range chans {
					chanMsg := msg 
					// 当channel数大于1时，拷贝message发送给每个channel
					if i > 0 {
						chanMsg = NewMessage(msg.ID,msg.Body)
					}
					// PutMessage最终调用diskQueue.writeOne()chanMsg写成文件
					channel.PutMessage(chanMsg)
				}
			}
    	}

#### (2).NewChannel()在nsqd/channel.go.
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
				// 连接到nsqd的client会处理c.clientMsgChan
				c.clientMsgChan <- msg
				atomic.StoreInt32(&c.bufferedCount, 0)
			}
		}


### 3. nsqd.Main() 在nsqd/nsqd.go
* 主要功能为监听端口，为每个连接创建client.可以通过tcp，http，https连接到nsqd，这里只分析tcp.

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
* 解析clientConn协议，并执行协议 

		func (p *protocolV2) IOLoop(conn net.Conn) error {
			clientID := atomic.AddInt64(&p.ctx.nsqd.clientIDSequence, 1)
			client := newClientV2(clientID, conn, p.ctx)

			messagePumpStartedChan := make(chan bool)
			go p.messagePump(client, messagePumpStartedChan)
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
