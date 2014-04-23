---
layout: post
title:  "superview的生命周期"
date:   2014-04-23 15:57:39
categories: ios
---

在工程里发现一个有意思的KVO问题，当在对象ViewB中注册对象ViewA属性的KVO观察者，当释放对象ViewB的时候，superview会变成nil.

对象ViewB中定义弱引用_superScrollView来引用self.superview
<pre><code>
@interface B()
{
    __weak UIScrollView *_superScrollView;
}
@end
</code></pre>

赋值，并注册观察者监视属性
<pre><code>
UIScrollView * scrollView = (UIScrollView *)self.superview;
if (![scrollView isKindOfClass:[UIScrollView class]]) {
        return;
}
 _superScrollView = scrollView;
 
   [_superScrollView addObserver:self
                       forKeyPath:@"contentInset"
                          options:NSKeyValueObservingOptionNew
                          context:nil];
</code></pre>

当释放对象ViewB的时候出现，_superScrollView为nil，从而导致无法移除观察者
<pre><code>
- (void)dealloc
{
		//此时_superScrollView为nil，无法移除观察者
        [_superScrollView removeObserver:self forKeyPath:@"contentInset"];
        _superScrollView = nil;
}
</code></pre>

对代码作调试分析，ViewB的superview在ViewB释放前就已经释放掉,从而定位到对象ViewA的dealloc函数

<pre><code>
- (void)dealloc
{
    _tabScrollView = nil;
    _mainScrollView = nil;
    
    for (WaterFlowView *waterFlowView in _tableViews){
        waterFlowView.dataSource = nil;
        waterFlowView.delegate = nil;
    }
    _tableViews = nil;
｝
</code></pre>

_mainScrollView是waterFlowView的父View，waterFlowView是ViewB的父View，但在执行dealloc过程中，_mainScrollView被先释放掉，从而导致在释放ViewB的时候，superview为nil。

所以在view释放的时候应该遵循先加载后释放，后加载先释放的规律，才能确保dealloc的时候 superview的生命周期正常