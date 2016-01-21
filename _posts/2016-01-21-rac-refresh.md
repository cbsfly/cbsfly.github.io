---
layout: post
title: "利用RAC一句话实现上拉刷新下拉刷新"
description: 
category: iOS
tags: [RAC,刷新]
modified: 2016-01-21

imagefeature: pic6.jpeg
comments: true

---

最近在研究上拉刷新下拉刷新，有点小心得和大家分享下。

首先我是先在网上搜博客看，发现要不就是用原生UIRefreshControl要不就是第三方库的教程，这都不是我想要的（好的开源库推荐[MJRefresh](https://github.com/CoderMJLee/MJRefresh)）。

于是我开始看大神们的源代码，了解了下原理。

#刷新原理

首先上拉下拉刷新肯定是基于UIScrollView的基础上的，包括UITableView其实也是一个UIScrollView。而实现上拉下拉刷新的原理就是UIScrollView中的代理方法
	
	- (void)scrollViewDidScroll:(UIScrollView *)scrollView;// any offset changes

这个方法是在你的scrollView滚动时就会执行的方法。

这里有必要说明下scrollView的两个属性，一个是contentSize，一个是contentOffset。这里上图片讲解可能直观一点。

![挂了？！](https://github.com/cbsfly/cbsfly.github.io/raw/master/images/gallery1/scrollview.png)

*PS：不会用mac的画图工具比较丑见谅。*

看图，这里的绿色框就是我们的手机屏幕，我们的UI都是呈现在手机屏幕上的，那么黑色的框就是contentSize。就是说，虽然手机屏幕只有一点大，但是我们的scrollView并不是只有一点大的，这个属性是可以设置的，而我们滚动scrollView其实就是滚动黑色框，这样看到的界面就会不一样了。而图上标注的红点就是contentOffset。contentOffset是一个CGPoint，代表当前屏幕所在位置左上角相对于scrollView.contentSize左上角的横纵坐标值。

了解了scrollView我们就来说刷新。在
`- (void)scrollViewDidScroll:(UIScrollView *)scrollView;// any offset changes`方法中我们应该监听contentOffset的值，如果他的纵坐标到某个点以上我们就执行刷新数据，移动到某个点以下我们就执行加载数据。具体看代码可能更好理解。

	- (void)scrollViewDidScroll:(UIScrollView *)scrollView 
	{
    if ([scrollView isEqual:_tableView]) {
        if (scrollView.contentOffset.y < -50 ) {
        //下拉刷新方法
        }
        if (scrollView.contentOffset.y > 800 ) {
        //上拉加载方法
        }
      }
    }
    
原理很简单，但是实现的话还是有很多细节需要考虑的，比如`scrollView.contentOffset.y < -50`的情况是很多的，当用户快速上拉到-100时，可能在-51执行一次刷新方法，在-52执行一次，-53执行一次。。。。。。这肯定是不合理的。

所以我们需要节流。这里提两个我的个人观点，一个是开线程执行刷新办法，如果线程存在不会新开一个，这样可以保证刷新方法执行一次。还有一种是用KVO监听用户手指松开的动作，松开的时候再刷新，或者` - (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate`用这个代理方法，他会帮你监听手指是否松开。这两个方法我都没试过，大家可以自己尝试下，有错希望反馈给我，共同进步。

但是如果仅仅是判定松开我觉得是满足不了需求的。我现在希望就是当用户快下滑到底部时就自动加载新数据，那我应该怎么实现呢？总不能用户一拉到底不松手就不加载吧。

#RAC一个方法实现刷新

我想到了RAC。这个想法可能有点非主流，所以肯定有逻辑考虑不周的地方，希望各位指出。这里我先宏定义了下，为了缩短代码

	#define VIEWHEIGHT self.view.frame.size.height

然后就是RAC了。

    [[[RACObserve(self.tableView, contentOffset) map:^id(id value) {
        if (self.tableView.contentOffset.y < -50) {
            return @"1";
        }
        if (self.tableView.contentOffset.y > self.tableView.contentSize.height - VIEWHEIGHT * 1.5 && self.tableView.contentSize.height - VIEWHEIGHT * 1.5 > 0) {
            return @"2";
        }else{
            return @"0";
        }
    }] distinctUntilChanged] subscribeNext:^(id x) {
        debugLog(@"%@", x);
        if ([x integerValue] == 1) {
            [self netWork];
        }else if ([x integerValue] == 2){
            [self loadMoreData];
        }
    }];
    
这里是用了RAC KVO的写法，不会的可以点[我文章](http://cbsfly.github.io/ios/rac1/)复习下。首先写了一个通知监听tableView的contentOffset，如果发生变化立刻进入map产生的映射中执行map中的方法。我给情况分了类，如果用户下拉，返回1，如果上拉快到底部时返回2。并且在映射完成后用了`distinctUntilChanged`属性，当我的映射值不产生变化时是不会传递映射值的。这样当用户拉倒需要刷新的位置，只会发一个信号给订阅者，只会执行一次刷新数据的方法，这样我所有的需求就迎刃而解了。

如果有更好的办法希望大家给我指出，谢谢大家。