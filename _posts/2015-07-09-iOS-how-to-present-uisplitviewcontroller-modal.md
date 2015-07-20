---
layout: post
title: 如何以Modal方式弹出UISplitViewController
date:   2015-07-09 10:18:00
categories: iOS
---

开发中遇到这样一个问题，当用户点击某个按键的时候，需要弹出一个后台设置的页面，而这个页面，是使用UISplitViewController来构建的。关键的语句如下：

```
TSSettingViewController *settingVC = [[TSSettingViewController alloc] init];
[self presentViewController:settingVC animated:YES completion:nil];
```

实际运行的时候，iOS8下没有问题，能正常弹出页面，但是到了iOS7上，直接崩溃，提示以下错误：

```
Application tried to present a Split View Controllers modally
```

Google得知，苹果的[官方文档](https://developer.apple.com/library/ios/documentation/WindowsViews/Conceptual/ViewControllerCatalog/Chapters/SplitViewControllers.html)有写：
> "A split view controller must always be the root of any interface you create. In other words, you must always install the view from a UISplitViewController object as the root view of your application’s window."

也就是说，UISplitViewController必须是应用的Window的Root View，之前以Modal方式直接弹出的方式是不对的（iOS8做了改进，不会崩溃，但官方的文档还是不推荐这样用）。

由于应用要兼容iOS7，不得不找办法解决。

一种解决的方法是改写TSSettingViewController，不再继承自UISplitViewController，但是这个方法工作量也比较大，基本就要重新构建原来的页面了。

另一种方法是弹出一个普通的ViewController，这个ViewController的View是原来的UISplitViewController的view，这样的方式是可行的，但总感觉和苹果推荐的处理方法不符。

第三种方法，既然UISplitViewController必须是应用的Window的Root View，那就在它弹出的时候让它成为Root View，弹回去的时候，设置回原来的Root View即可，这样的方式也是最符合苹果的推荐处理方式的。

实际的解决方式并不复杂，我们需要将
```
[self presentViewController:settingVC animated:YES completion:nil];
```
进行替换，使得新的弹出方法即可以将settingVC以类似的动画效果弹出，也可以设置为Window的RootViewController。
首先在TSSettingViewController.h中添加这个新的弹出方法，我们这里命名为：

```
// in TSSettingViewController.h above @end
- (void)presentAsRootViewController;
```

这个方法的实现语句是：

```
// in TSSettingViewController.m above @end
- (void)presentAsRootViewController {
    
    UIWindow *window=[[[UIApplication sharedApplication] delegate] window]; //获取当前Window
    _previousViewController=window.rootViewController;  //获取弹出前的ViewController
    
    UIView *windowSnapShot = [window snapshotViewAfterScreenUpdates:YES];   //创建一个弹出前Window的截图，这是为了模拟Modal方式弹出页面的效果
    window.rootViewController = self;
    
    [window insertSubview:windowSnapShot atIndex:0];
    
    CGRect dstFrame=self.view.frame;
    
    CGSize offset=CGSizeApplyAffineTransform(CGSizeMake(0, 1), window.rootViewController.view.transform);
    offset.width*=self.view.frame.size.width;
    offset.height*=self.view.frame.size.height;
    self.view.frame=CGRectOffset(self.view.frame, offset.width, offset.height); 
    //模拟Modal方式弹出ViewController的动画效果
    [UIView animateWithDuration:0.5
                          delay:0.0
         usingSpringWithDamping:1.0
          initialSpringVelocity:0.0
                        options:UIViewAnimationOptionCurveEaseInOut
                     animations:^{
                         self.view.frame=dstFrame;
                     } completion:^(BOOL finished) {
                         [windowSnapShot removeFromSuperview];
                     }];
}
```

然后将原先调用 `[self presentViewController:settingVC animated:YES completion:nil]`的地方改为`[settingVC presentAsRootViewController]`即可。

最后，我们还要实现UISplitViewContrller的弹回方法，类似弹出方法：

```
// in TSSettingViewController.m above @end
- (void)returnToPreviousViewController {
    if(_previousViewController) {
        
        UIWindow *window=[[[UIApplication sharedApplication] delegate] window];
        
        UIView *windowSnapShot = [window snapshotViewAfterScreenUpdates:YES];
        window.rootViewController = _previousViewController;
        
        [window addSubview:windowSnapShot];
        
        CGSize offset=CGSizeApplyAffineTransform(CGSizeMake(0, 1), window.rootViewController.view.transform);
        offset.width*=windowSnapShot.frame.size.width;
        offset.height*=windowSnapShot.frame.size.height;
        
        CGRect dstFrame=CGRectOffset(windowSnapShot.frame, offset.width, offset.height);
        
        [UIView animateWithDuration:0.5
                              delay:0.0
             usingSpringWithDamping:1.0
              initialSpringVelocity:0.0
                            options:UIViewAnimationOptionCurveEaseInOut
                         animations:^{
                             windowSnapShot.frame=dstFrame;
                         } completion:^(BOOL finished) {
                             [windowSnapShot removeFromSuperview];
                             _previousViewController=nil;
                         }];
    }
}
```

然后替换掉原来的弹出语句就可以了。