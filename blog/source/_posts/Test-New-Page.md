---
title: iOS中事件的传递
date: 2017-04-21 10:34:59
tags: [iOS]
categories: [Blogs]
---



### 1.事件的基本认识
- 响应者对象
```objc
1.只有继承了UIResponder的对象才能接收并处理事件。我们称之为“响应者对象” 
2.因为 UIResponder 内部实现 ‘触碰事件’ ‘加速器事件’ ‘远程控制事件’等。
3.UIApplication、UIViewController、UIView都继承自UIResponder，因此它们都是响应者对象，都能够接收并处理事件

```
- UITouch 对象 和 UIEvent对象
```objc
如：当手指触碰屏幕时
- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
// 先了解这些参数的概念
// touches
这是一个集合，且NSSet 是一个无序的集合，里面存放的是UITouch对象（触碰对象）。
// event
当手指触碰屏幕产生的UIEvent事件

```
- UITouch
```
// 1.概念
1.1 当用户用一根手指触摸屏幕时，会创建一个与手指相关联的UITouch对象
1.2 一根手指对应一个UITouch对象

// 2.作用
2.1保存着跟手指相关的信息，比如触摸的位置、时间、阶段
2.2当手指移动时，系统会更新同一个UITouch对象，使之能够一直保存该手指在的触摸位置
2.3当手指离开屏幕时，系统会销毁相应的UITouch对象

```
- UIEvent
```objc
// 1.概念
1.1 每产生一个事件，就会产生一个UIEvent对象
1.2 事件对象，记录事件产生的时刻和类型

```


### 2.事件的产生和传递
如何传递?
```
1.当产生一个事件后，系统会把该事件放UIApplication的事件队列中。
2.UIAppliaction会取出最前面的事件，即哪个事件先触碰就先处理谁。
3.然后把事件往下传递，先传递给主窗口。
4.主窗口会在视图层次结构中地往下传递，找最合适的View来处理该事件
如何找到最合适的View？
4.1 判断自己能否处理事件，如果能 往下判断。
4.2 判断该触碰点是否在自己的身上，如果在 往下判断
4.3 从后到前的顺序遍历子控件（目的是获取最前面的View）,重复4.1、4.2的步骤。
4.4 如果没有找到，则返回自己。
```


## 3.hitTest: withEvent:方法的底层实现
- 什么时候调用
> 当一个事件发生，会先调用这个方法。即 - 当事件传递给当前View时就会调用这个方法

- 作用是什么?
> 目的是为了寻找最合适的View 来处理事件。

- 底层实现
```objc
// 当一个事件产生 -> 
// point : 事件的触碰点（位置）
// event : 当前的触碰事件
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event{
    // 1.当前的view能否接受事件处理 ? 如果是 -> 继续判断
    if (self.userInteractionEnabled == NO | self.hidden == YES | (self.alpha <= 0.01) ) { （不能处理事件的三个原因）
        return nil;
    }
    // 2.该触摸点是否在当前的Vies上 ? 如果是 -> 继续判断
    if ([self pointInside:point withEvent:event] == NO) {
        return nil;
    }
    // 3.遍历子控件（从后面开始遍历 -> 因为后添加的在View最上面）
    NSInteger count = self.subviews.count;
    for (NSInteger index = count - 1; index > 0; index--) {
        // 3.1取出子控件
        UIView *childView = self.subviews[index];
        // 3.2取出当前点 -> 坐标系转换
        CGPoint childViewPoint = [self convertPoint:point toView:childView];
        // 4.重复 (递归)
        UIView *fitView = [childView hitTest:childViewPoint withEvent:event];
        if (fitView) {
            return fitView;
        }
    }
    // 5.如果遍历完了 都没有合适的View处理该事件 则返回给父类 往上传
    return [super hitTest:point withEvent:event];
}
```
- 应用场景
> 重写该方法，根据某个范围的触碰点，指定某个View处理该事件。

- -(BOOL)pointInside:(CGPoint)point withEvent:(UIEvent *)event方法
> 作用:判断触摸点在不在当前的View上.
什么时候调用:在hitTest方法当中会自动调用这个方法.
注意:point,必须得要跟当前View同一个坐标系.
这个方法 ： --> 可以决定哪个触碰点能接受事件 （如：在实现图文混排的时候，指定点击特殊文字的时候才能处理事件，可以使用这个方法）

- 转换坐标系
> CGPoint childViewPoint = [self convertPoint:point toView:childView]; //-> 把point这个点 转换到 childView 的坐标系 上 -> 返回 childViewPoint 。

## 4.事件的响应

事件如何响应?
```
1.当一个事件产生后，经过上面一系列的传递，传递给最合适的View来处理该事件。

2.如果这个View实现了touchs方法来做具体的事情，即确定要处理该事件。如果没有实现，那么该事件顺着响应者链条一层一层往上传递。

3.传递给上一个响应者,接着就会判断上一个响应者是否实现touches方法,是否要处理该事件。
如何找到上一个响应者?
3.1 如果当前的View是控制器的View,，那么控制器就是它的上一个响应者。
3.2 如果当前的View不是控制器的View，那么它的父控件就是它的上一个响应者。
3.3 如果在视图层次结构中的最高的视图都没有处理该事件，那么它就会交给主窗口Window处理
3.4 如果主窗口Window也没有处理该事件，那么它就会交给UIApplication处理。
3.5 如果UIApplication也没有处理该事件，那么该事件就会被丢弃。
```

- 作用
>   灵活地利用多个View做事件的处理。
>   利用响应者链条 能让多个控件 处理  一个触摸事件


