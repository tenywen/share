## grpc-go

grpc-go利用proto buff和http2封装的rpc。底层的传输层使用http2。 

###proto buff 
xxx.proto定义的service 在生成xxx.pb.go还会额外被封装一层包含conn的结构 

### client 

* client基本行为，建立连接，然后生成client 
	
		// 调用newHTTP2Client
		conn,err := grpc.Dail(serverAddr, opts)  
		// client 为proto buff 的封装。包含conn结构
		client := pb.NewXXXClient(conn)


* newHTTP2Client 

		// 设置默认的dialer
		if opts.Dialer == nil {
			opts.Dialer = func(addr string, timeout time.Duration) (net.Conn,error) {
				return net.DialTimeout("tcp",addr,timeout)
			}
		}
		conn,connErr := opts.Dialer(addr,timeout)
		
		// send http2 header for server 
		n,err := conn.Write(clientPreface)	
		
		// 创建http2 reader,writer  
		framer := newFramer(conn) 

		// 创建http2Client 
		t := &http2Client{
			conn: 			conn,
			framer: 		framer,  
			// client streams起始id为1
			nextId:			1,
			activeStreams: 	make(map[uint32]*Stream), 
			// only one 
			writableChan:	make(chan int,1),
		}

		// http2Client控制信息处理
		go t.controller()

		// 同时只能有一个writer 进入transport	
		t.writableChan <- 0

		// 接受in message
		// for {
		//     frame,err := t.framer.readFrame() 
		// 	   switch frame := frame.(type) {
		//         case *....
		//     }
		// }
		go t.reader()

* client建立之后，就调用某个具体rpc了 

protobuff 中定义的service xxx

service xxx 被proto buff 封装为 XXXClient{conn *grpc.ClientConn}	

grpc.ClientConn 再次被封装一层 ClientConn{transport transport.ClientTransport} 

ClientTransport 为Interface{} 对应的实际结构为 http2Client.这才是grpc中实际存储conn得信息

但http2Client中数据的读写却是framer 

httpClient 在client很重要，所以展示结构

		type http2Client struct {
			target string   // server name/addr
			conn   net.Conn // underlying communication channel
			nextID uint32   // the next stream ID to be used

			// writableChan synchronizes write access to the transport.
			// A writer acquires the write lock by sending a value on writableChan
			// and releases it by receiving from writableChan.
			writableChan chan int
			// shutdownChan is closed when Close is called.
			// Blocking operations should select on shutdownChan to avoid
			// blocking forever after Close.
			// TODO(zhaoq): Maybe have a channel context?
			shutdownChan chan struct{}
			// errorChan is closed to notify the I/O error to the caller.
			errorChan chan struct{}

			framer *framer
			hBuf   *bytes.Buffer  // the buffer for HPACK encoding
			hEnc   *hpack.Encoder // HPACK encoder

			// controlBuf delivers all the control related tasks (e.g., window
			// updates, reset streams, and various settings) to the controller.
			controlBuf *recvBuffer
			fc         *inFlow
			// sendQuotaPool provides flow control to outbound message.
			sendQuotaPool *quotaPool
			// streamsQuota limits the max number of concurrent streams.
			streamsQuota *quotaPool

			// The scheme used: https if TLS is on, http otherwise.
			scheme string

			authCreds []credentials.Credentials

			mu            sync.Mutex     // guard the following variables
			state         transportState // the state of underlying connection
			activeStreams map[uint32]*Stream
			// The max number of concurrent streams
			maxStreams int
			// the per-stream outbound flow control window size set by the peer.
			streamSendQuota uint32
		}


