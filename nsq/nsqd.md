#### nsqd
#####1. nsqd的main()函数在apps/nsqd/nsqd.go.
main()函数的主要工作：

* 创建nsqd。
* 监听端口，为每个连接创建client
	


	func main() {

		// 设置默认配置

		// 从stdin读取新配置并修改。是version，则显示版本号之后,退出main
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
-----------------------------------

#####2. nsqd.LoadMetadata()在nsqd/nsqd.go.
*	根据config设置读取dat文件,检查nsqd中是否存在dat中的topic和channel,没有则创建

	
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
######	[1]NewTopic()在nsqd/topic.go.
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

######[2].NewChannel()在nsqd/channel.go.
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

