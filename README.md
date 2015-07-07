<br>失业了，没事干读些源码。分析的有错，欢迎指正，免得让我这个菜b误入歧途。感谢!</br>

* 1.nsq 

* 2.grpc-go

===

#### 使用godep 

[godep源码](https://github.com/tools/godep)

[godep依赖](https://github.com/golang/go/wiki/GoGetTools)

		export $GOPATH=workspace 
		cd $GOPATH
		go get github.com/tools/godep
		sudo atp-get install mercurial   


#### go使用proto buff 

[下载](//github.com/google/protobuf/tree/v3.0.0-alpha-3.1)

		$ ./configure 
		$ make 
		$ make check
		$ make install  
		$ go get -a github.com/golang/protobuf/protoc-gen-go 

[依赖] 

		$ sudo apt-get install autoconf
		$ sudo apt-get install libtool

for example 
	
		$ sudo ldconfig  
		# or 
		$ export LD_LIBRARY_PATH=/usr/local/lib

		# from the grpc-common/go dir; invoke protoc
		$ protoc -I ../protos ../protos/helloworld.proto --go_out=plugins=grpc:helloworld

