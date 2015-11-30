---
layout: post
title: iOS使用Socket实现客户端-服务器双向通信
date:   2015-11-29 10:18:00
categories: iOS Socket
---


在视频客户端的开发过程中，遇到了这样一个需求，同在一个局域网的两个设备A和B，A/B可能是iOS/安卓系统的平板/手机/机顶盒，A发现一个视频比较精彩，需要将其分享到B进行播放（类似Airplay实现的效果，但不局限于支持Airplay的iOS设备）。实现这样的效果的最为简单高效的方式是Socket。

> **Socket通信和我们常见的HTTP请求的区别**
> 
>Socket和HTTP并不是对等的概念，HTTP是应用层的协议，而Socket是对TCP/IP进行封装的接口，也就是说，HTTP是协议，而Socket是接口。严格对等的比较是NSURLRequest vs Socket或HTTP vs TCP/IP。我们平时浏览网页，都是通过HTTP进行的，但是有些场合，HTTP并不适用，比如实现双向通信，服务器端往客户端发信息，某些对于实时性，效率要求高的场合等，这些就需要使用Socket。这里有一份[详细的对比测试](https://www.pubnub.com/blog/2015-01-05-websockets-vs-rest-api-understanding-the-difference/)


###搭建一个Socket服务器

由于Socket通信是双向的，我们需要客户端+服务器端，服务器端的搭建可以参考Example里面的EchoServer。我们接下来要实现的内容是接受来自客户端的连接请求并建立连接。然后将客户端发出的信息原封不动返回给客户端。而我们的服务器也会在iOS设备上运行。

####引入GCDAsyncSocket库

这里使用CocoaPods，新建iPhone单页面工程（CLEchoServer）然后`pod init`进行初始化，并添加CocoaAsyncSocket库，Podfile代码如下：
```
target 'CLEchoServer' do

use_frameworks!
pod 'CocoaAsyncSocket' 

end
```

然后`pod install`进行安装，完成后打开对应的workspace文件。

在根ViewController中引入：

```
@import CocoaAsyncSocket;
```

####实现代码

服务器要实现接收客户端的连接请求，首先需要新建一个GCDAsyncSocket对象，新建对象的方法为：

```
_listenSocket = [[GCDAsyncSocket alloc] initWithDelegate:self delegateQueue:dispatch_get_main_queue()];
```

这个方法的两个参数是delegate对象（self），和delegate执行的线程（这里是主线程）。

新建完成后，要启动这个GCDAsyncSocket，方法是：

```
NSError *error = nil;
if (![_listenSocket acceptOnPort:port error:&error]) {
	NSLog(@"Start Error %@", error);
	return;
}
        
NSLog(@"Echo server started on port %li", (long)port);
```

这里启动的语句我们可以放到一个UIButton的点击事件中执行，而port可以从界面中的UITextfield中获取。

至此，一个具备接收客户端连接请求的服务器就已经搭建好了，我们可以将其在模拟器或真机上运行。


###搭建一个Socket客户端

为了验证服务器是否已经正常运行，我们需要搭建客户端（这里先假设我们搭建的服务器的IP为192.168.1.105，端口为9688）。

客户端也是基于CocoaAsyncSocket进行的，所以我们先要重复服务器中引入这个库的步骤，这里假设客户端工程名称位CLSocketClient。

然后，和服务器一样，我们在根ViewController中添加：
```
@import CocoaAsyncSocket;
```

同服务器类似，我们先要建立一个GCDAsyncSocket对象（这里服务器和客户端均为此类型对象），

```
_requestSocket = [[GCDAsyncSocket alloc] initWithDelegate:self delegateQueue:dispatch_get_main_queue()];
```

然后启动连接

```
NSError *err = nil;
if (![_requestSocket connectToHost:@"192.168.12.105" onPort:9688 error:&err])
{
	NSLog(@"连接失败: %@", err);
}
```

这里的IP地址为服务器的IP地址，端口为服务器GCDAsyncSocket对象对应的端口。


然后把这个语句添加到一个UIButton的点击事件中执行。

此外，我们需要实现连接成功的Delegate方法

```
- (void)socket:(GCDAsyncSocket *)sender didConnectToHost:(NSString *)host port:(UInt16)port
{
    NSLog(@"服务器已连接");
}
```

至此具备连接功能的客户端已经搭建完成，只要我们服务器正常运行，执行连接后控制台便会显示『服务器已连接』。


###发送数据

光有连接当然是不够的，我们最终的目的是实现双向通信，也就是客户端即可以发送信息给服务器端，服务器端也可以发送信息给客户端。

首先考虑的是添加这样一个功能，当客户端连上服务器的时候，服务器返回一条信息『欢迎你连接本服务器』。

打开CLEchoServer工程，然后实现客户端连接成功后，服务器端的Delegate：

```
- (void)socket:(GCDAsyncSocket *)sock didAcceptNewSocket:(GCDAsyncSocket *)newSocket
{
	NSString *host = [newSocket connectedHost];
    UInt16 port = [newSocket connectedPort];
    NSLog(@"发现新连接 %@:%hu",host, port);
    NSString *welcomeMsg = @"欢迎你连接本服务器\n";
    NSData *welcomeData = [welcomeMsg dataUsingEncoding:NSUTF8StringEncoding];
	[newSocket writeData:welcomeData withTimeout:-1 0];
}
```

这里我们先把连接本服务器的客户端的IP和端口打印出来，然后向客户端发送『欢迎你连接本服务器』的欢迎信息，这里除了信息外，还有两个参数，超时时长和tag可以和客户端进行商定，用于区分不同消息类型。客户端接收信息的同时也会收到此参数。

回到客户端，我们需要在连接成功后，监听来自服务器的欢迎信息，方法是

```
- (void)socket:(GCDAsyncSocket *)sender didConnectToHost:(NSString *)host port:(UInt16)port
{
	NSLog(@"连接成功");
	[sender readDataWithTimeout:-1 tag:0];
}
```

这样就完成了服务器向客户端发送消息的过程。

同理，客户端向服务器发送消息的过程也是服务器调用readDataWithTimeout方法监听客户端信息，然后客户端调用writeData方法发送数据，服务器端的对应Delegate方法便能接收信息。

但是这个方式实现的功能也有一定缺陷，我们需要事先知道服务器的地址，而我们在将设备连接到到同一Wi-Fi后，通常是不会去查设备的IP地址的，那我们如何实现搜索局域网的设备并连接呢？

未完待续...
To be continued...