---
layout: post
title: iOS AVPlayer播放器学习
date:   2015-09-23 15:01:00
categories: iOS Video AVPlayer
---

原来的播放器是使用MPMoviePlayerController写的，由于iOS9之后，苹果废弃了MPMovie的相关类和接口，考虑把MPMovie换成AVPlayer，同时也系统的学习一下iOS播放器的相关知识。

AVPlayer原本是MPMoviePlayer的底层实现，能实现比MP更多更丰富的自定义内容，iOS9之后，苹果也逐步用其代替MP，包括PIP画中画播放就只能通过AVPlayer完成。AVPlayer只有视频播放画面，开发者需要自己设计相关的UI控制元素。与AVPlayer对应的AVPlayerViewController类封装了iOS自带的一些播放UI元素，如果可以接受默认UI元素也可以选择使用AVPlayerViewController。

要使用AVPlayer，先得引入AVFoundation库及其头文件。

```
#import <AVFoundation/AVFoundation.h>
```

由于AVPlayer只是一个NSObject类，要进行显示还要有个用于显示的AVPlayerLayer类：

```
self.avPlayer = [AVPlayer playerWithURL:[NSURL URLWithString:@"http://183.238.77.5:1935/dvr/30135.stream/playlist.m3u8?DVR"]];
AVPlayerLayer *layer = [AVPlayerLayer playerLayerWithPlayer:self.avPlayer];
layer.frame = CGRectMake(0, 0, 1024, 768);
[self.view.layer addSublayer: layer];
[self.avPlayer play];
```

然后视频就可以播放了。

播放比较简单，下面详细看看这两个类AVPlayerLayer和AVPlayer。

##AVPlayerLayer

###初始化方法

AVPlayerLayer是用于显示播放内容的类，是CALayer的子类。初始化方法有（不推荐使用alloc init方法）：

```
+ (AVPlayerLayer *)playerLayerWithPlayer:(AVPlayer *)player;
```

这个方法如名所示，以AVPlayer来初始化AVPlayerLayer，而且AVPlayerLayer也将这个AVPlayer对象暴露为property，可以手动进行设置，不过一般情况下，我们只会在初始化时指定AVPlayer，不会手动去设置它。

###readyForDisplay

```
@property(nonatomic, readonly, getter=isReadyForDisplay) BOOL readyForDisplay
```

readyForDisplay这个Property则是表示视频是否已经准备好开始播放（视频从载入，到Ready会由NO变为YES，视频卡顿后恢复不会触发其变化），我们可以通过KVO来检测其改变来获取视频是否已准备好播放。

###videoGravity


```
@property(copy) NSString *videoGravity
```

videoGravity属性是个可设置，读取的属性，功能是改变视频的缩放模式，有四个可能的值，下面以图为例说明，视频源是16:9的CCTV5直播信号，而playerLayer是4:3的frame，背景色是灰色：
- AVLayerVideoGravityResizeAspect：默认值，保持宽高比，完整显示播放内容 
![Alt text](http://ww1.sinaimg.cn/large/dd869288jw1ewcedem8ukj205j04baa3.jpg)
- AVLayerVideoGravityResizeAspectFill：保持宽高比，填充播放窗口，切割显示播放内容
![Alt text](http://ww4.sinaimg.cn/large/dd869288jw1ewcee42upzj205a04amxc.jpg)
-  AVLayerVideoGravityResize：不保持宽高比，填充播放窗口 
![Alt text](http://ww2.sinaimg.cn/large/dd869288jw1ewced328cqj205a04amx8.jpg)

###videoRect
当前播放视频画面的bounds。只能读不可写。

##AVPlayer

负责显示的AVPlayerLayer提供的属性和接口并不多，视频播放的大部分功能都由AVPlayer提供。

###初始化

AVPlayer有两种初始化方式：通过URL初始化和通过AVPlayerItem初始化。

通过URL方法比较简单直接把URL扔进去就可以了，方法是：

```
- (instancetype)initWithURL:(NSURL *)URL
```

或

```
+ playerWithURL:
```

推荐使用类方法`playerWithURL`进行初始化

通过AVPlayerItem初始化则需要先新建一个AVPlayerItem，然后：

```
+ (id)playerWithPlayerItem:(AVPlayerItem *)item;
```

###播放控制

播放：`play`

暂停：`pause`

播放速度：`rate`，默认为1，如果对应的AVPlayerItem的canPlaySlowForward或canPlayFastForward属性为YES，可以设为低于1或高于1的值以达到快放和慢放的效果。AVPlayerItem的canPlaySlowReverse或canPlayFastReverse为YES时还可以设为负值，打到后退快放，后退慢放的效果。

播放完成后行为：`actionAtItemEnd`，可以设为AVPlayerActionAtItemEndAdvance，AVPlayerActionAtItemEndPause和AVPlayerActionAtItemEndNone三种值，分别表示播放下一个item（如果有的话），暂停和什么都不干。

替换当前播放的Item：
`- replaceCurrentItemWithPlayerItem:`

###时间控制

获取当前时间：`- (CMTime)currentTime;`

快进到某一时间：`- (void)seekToDate:(NSDate *)date;`或`- (void)seekToTime:(CMTime)time;`

##完整的播放器

在开头我们完成了一个最基本的播放器：

```
self.avPlayer = [AVPlayer playerWithURL:[NSURL URLWithString:@"http://183.238.77.5:1935/dvr/30135.stream/playlist.m3u8?DVR"]];
AVPlayerLayer *layer = [AVPlayerLayer playerLayerWithPlayer:self.avPlayer];
layer.frame = CGRectMake(0, 0, 1024, 768);
[self.view.layer addSublayer: layer];
[self.avPlayer play];
```

但是这个的播放器只有简单的播放功能，并没有播放时间，进度条，播放，暂停等功能。

##iOS 支持的视频

在播放之前，我们需要了解AVPlayer到底支持什么格式的音视频。我们可以google苹果的官方文档查看，也可以在代码中加上这一句，

```
NSLog(@"Supported av types %@", [AVURLAsset audiovisualMIMETypes]);
```

会返回这样一个数组：

```
asset type (
    "audio/aacp",
    "video/3gpp2",
    "audio/mpeg3",
    "audio/mp3",
    "audio/x-caf",
    "audio/mpeg",
    "video/quicktime",
    "audio/x-mpeg3",
    "video/mp4",
    "audio/wav",
    "video/avi",
    "audio/scpls",
    "audio/mp4",
    "audio/x-mpg",
    "video/x-m4v",
    "audio/x-wav",
    "audio/x-aiff",
    "application/vnd.apple.mpegurl",
    "video/3gpp",
    "text/vtt",
    "audio/x-mpeg",
    "audio/wave",
    "audio/x-m4r",
    "audio/x-mp3",
    "audio/AMR",
    "audio/aiff",
    "audio/3gpp2",
    "audio/aac",
    "audio/mpg",
    "audio/mpegurl",
    "audio/x-m4b",
    "application/mp4",
    "audio/x-m4p",
    "audio/x-scpls",
    "audio/x-mpegurl",
    "audio/x-aac",
    "audio/3gpp",
    "audio/basic",
    "audio/x-m4a",
    "application/x-mpegurl"
)
```

以一般的HLS为例，对应的是最后的一种"application/x-mpegurl"，具体的格式是视频编码H.264，音频AAC，封装为MPEG-TS，索引文件为M3U8的HLS。

如果不确定某个音视频流是否符合苹果的HLS规范，可以使用mediastreamvalidator工具对视频流进行验证（验证过程中可以Ctrl+C中断验证，以不用读取完所有视频文件，只验证部分视频流）。当然最实际的还是在AVPlayer里面实际播放一下。

##视频可播放状态

播放之前，会有一个加载视频的过程，这个过程根据网络及视频的情况不同会有区别，架设我们播放界面有个播放键，在视频加载完成前，应该是不可点击的，加载完成才变成可点击的状态，这一功能可以通过添加AVPlayer的status进行。这个status跟播放状态无关，只有表示载入状态的三个值：未知（未开始载入AVPlayerStatusUnknown），加载完成（AVPlayerStatusReadyToPlay）和加载失败（AVPlayerStatusFailed）。

添加KVO的语句是：

```
[self.videoPlayer addObserver:self forKeyPath:@"status" options:0 context:NULL];

- (void)observeValueForKeyPath:(NSString*)path ofObject:(id)object change:(NSDictionary*)change context:(void*) context
{
    if ([path isEqualToString:@"status"]) {
        if (self.videoPlayer.status == AVPlayerStatusReadyToPlay) {
            [self.playButton setEnabled:YES];
        } else {
            [self.playButton setEnabled:NO];
            NSLog(@"player Error %@", self.videoPlayer.currentItem.error);
        }
    } 
}

```

##播放和暂停

播放过程少不了暂停和播放控制，对应的方法是

```
[self.player play];
[self.player pause];
```

一般的视频应用都是用的同一个按键来完成此功能，这样我们就需要根据视频的状态来设置按键的状态，正在播放状态下，显示为暂停键，暂停状态下，显示为播放键。实现这一控制可以通过添加播放Rate的KVO进行。


```
[self.videoPlayer addObserver:self forKeyPath:@"rate" options:0 context:NULL];
```

处理方法中比较rate是否为0，为0表示视频已经暂停，按键显示为播放。否则显示为暂停键。

##播放进度条

另一个需要实现的模块是播放进度的Slider，Slider即可以显示当前播放进度，也可以拖动移动到某一进度进行播放。

进度可以通过添加播放进度的KVO来实现，添加的方式有点特殊：

```
    __weak __typeof__(self) weakSelf = self;
    CMTime interval = CMTimeMakeWithSeconds(1.0, NSEC_PER_SEC);
    _playbackTimeObserver = [self.videoPlayer addPeriodicTimeObserverForInterval:interval queue:NULL usingBlock:^(CMTime time) {
        CGFloat sliderValue = CMTimeGetSeconds(self.videoPlayer.currentTime)/ CMTimeGetSeconds(self.videoPlayer.currentItem.duration);
        [weakSelf.progressSlider setValue:sliderValue animated:YES];
        NSTimeInterval currentTime = CMTimeGetSeconds(weakSelf.videoPlayer.currentTime);
        if (currentTime < 0) {
            currentTime = 0;
        }
        NSTimeInterval durationTime = CMTimeGetSeconds(weakSelf.videoPlayer.currentItem.duration);
        NSString *timeText = [weakSelf timeFromSecond:currentTime];
        NSString *durationText = [weakSelf timeFromSecond:durationTime];
        [weakSelf.currentTimeLabel setText:timeText];
        [weakSelf.durationTimeLabel setText:durationText];
    }];
```

这样，一个包含了播放/暂停，播放进度条的基本播放器已经开发完成了。
