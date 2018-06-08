---
layout:     article
title:      "UIView animation移动视图点击事件小技巧"
subtitle:   "\"既然视图的点击事件我们那么熟悉，为什么不再熟悉一点使用UIView animation移动视图的点击事件呢？你还可以趁机深入一下Core Animation的细微差别。\""
date:       2017-12-19 23:30:00
key:        1006
tags:
    - iOS小技巧
    - iOS
---

> 既然是 UI 开发工程师，那么对点击事件自然是很熟悉的，但是你遇到过处理移动中视图的点击事件了嘛？<!--more-->

在笔者参与的 iOS 开发项目中，有弹幕的需求。既然是弹幕，那么就涉及到 `View` 移动，而且还有*万恶*的点击事件需求。这真的是一件让人头疼的事情，获取 View 的点击事件很方便，无论是使用手势，还是直接使用 `touchBegin`、`touchEnd`... 抑或你直接在继承 `UIControl` 自定义点击事件，再不济你可以将一个 `UIButton` 放在该 `View` 上。😂但是当它移动起来了，就不是那么简单得了。

## 问题
为了展示移动的视图无法获取点击事件，笔者特地做了一个小测试，如下图所示：
![](https://user-images.githubusercontent.com/9990834/34166489-01db592c-e51a-11e7-968c-16979a6c8a9f.gif)
> 上图表示静止情况下，该红色视图的点击事件。具体的代码很简单，就不贴出来了。

![](https://user-images.githubusercontent.com/9990834/34166729-9f73230e-e51a-11e7-9ea0-7e43552a02fc.gif)
> 上图为移动中的视图的点击事件，在 gif 图中也可以看到点击是没有效果的。使用移动的动画也是简单的 UIView 动画，通过改变 y 坐标来实现。如下所示：

```swift
let redView = UIView(frame: CGRect(x: 100, y: 20, width: 100, height: 100))
redView.backgroundColor = UIColor.red
redView.isUserInteractionEnabled = true
view.addSubview(redView)

let tapGesture = UITapGestureRecognizer(target: self, action: #selector(tapGestureAction(_:)))
redView.addGestureRecognizer(tapGesture)

UIView.animate(withDuration: 5.0, delay: 0, options: [.allowUserInteraction, .autoreverse, .repeat], animations: {
    redView.frame.origin.y += (UIScreen.main.bounds.height - 140)
}) { (isFinished) in
    // do somthing      
}
```

## 分析
那么为什么出现这个问题呢？
首先我们要明白，`UIView` 隐式动画与 `Core Animation` 的显示动画的一点点区别，`Core Animation` 动画不改变 `UIView` 的 `frame`，它改变的只是 `layer` 的一些属性。而 UIView 动画作用在 `frame` 上的话，直接改变的是 `UIView` 的 `frame`。  
也就是说，一旦你在 `UIView animation` 里面设置了 `View` 的 `frame`，那么该 `View` 的 `frame` 将会立即生效 (即使该 `View` 视觉上还在移动中)。喜欢动手的同学可以在上面代码的 `redView.frame.origin.y += (UIScreen.main.bounds.height - 140)` 下一行打印一下 `redView` 的 `frame`, 一定是改变之后的 `frame` 大小。

既然使用 `UIView animtion` 改变 `frame` 立即生效的话, 那么点击移动中的该 View 时间一定不会传递到该 `View` 中的, 因为在该响应链中你点击的位置是不在该 `View` 的 `bounds` 范围内的。不过在该 View 即将要移动到终点的时候，还是可以获取到点击时间的，此时该 `View` 的 `bounds` 与最终的 `frame` 有交叉，肯定可以捕捉到该点击事件，(建议感兴趣的同学可以动手试下)。这里涉及到了响应链的传递等, 对这个不太了解的同学可以先看一下文档等。

## 解决
既然点击事件添加到该 `View` 上无法识别，我们可以将该点击事件加在其静止的父 `View` 上，然后根据获取到的点击位置来判断该移动的 `View` 是否包含该位置。

可能这里又会有一个新的疑问：如何检测移动的 `View` 是否包含该位置呢？
其实细心的同学可以早已经见过 `CALayer` 里面提供了 `presentation()`, 它在苹果文档里面的注释如下:
> Returns a copy of the presentation layer object that represents the state of the layer as it currently appears onscreen.

大致意思就是说，该方法会返回该 `layer` 在显示在屏幕上的状态的拷贝。
请抓住关键词：**拷贝**、**显示**、**屏幕**。拷贝是指返回的还是一个 `layer` 啊，显示和屏幕是指当前屏幕显示的样子啊。所以我觉得在这里将其理解为原当前移动状态中 `layer` 的一份拷贝，既然这样，那就可以很轻松获取到移动状态下的 bounds 了啊。然后手动去判断是否包含点击的位置即可。

好了，既然解决方案都已经出来了，那么就请看看解决之后的效果吧。下图所示：
![](https://user-images.githubusercontent.com/9990834/34170040-632ad23e-e524-11e7-93df-37be24b77253.gif)

思路如上所示, 具体的代码如下所示：

```swift
/// 添加手势
let tapGesture = UITapGestureRecognizer(target: self, action: #selector(tapGestureAction(_:)))
view.addGestureRecognizer(tapGesture)
/// 处理手势
@objc private func tapGestureAction(_ gesture: UITapGestureRecognizer) {
let location = gesture.location(in: view)
guard let layer = redView.layer.presentation() else { return }
   if (layer.hitTest(location) != nil) {
   	/// 此时表明已经击中了该移动中的视图。可以在这里处理事情了。 
   }
}
```


## 附录

完整代码如下所示:

```swift
class ViewController: UIViewController {
   
    let redView = UIView(frame: CGRect(x: 100, y: 20, width: 100, height: 100))
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        redView.backgroundColor = UIColor.red
        redView.isUserInteractionEnabled = true
        view.addSubview(redView)

        let tapGesture = UITapGestureRecognizer(target: self, action: #selector(tapGestureAction(_:)))
        view.addGestureRecognizer(tapGesture)
        
        UIView.animate(withDuration: 5.0, delay: 0, options: [.allowUserInteraction, .autoreverse, .repeat], animations: { [weak self] in
            self?.redView.frame.origin.y += (UIScreen.main.bounds.height - 140)
        }) { (isFinished) in
            
        }
    }
    
    @objc private func tapGestureAction(_ gesture: UITapGestureRecognizer) {
        let location = gesture.location(in: view)
        guard let layer = redView.layer.presentation() else { return }
        if (layer.hitTest(location) != nil) {
            let alertController = UIAlertController(title: "击中我了", message: nil, preferredStyle: .alert)
            alertController.addAction(UIAlertAction(title: "😂😂😂😂", style: .default, handler: nil))
            present(alertController, animated: true, completion: nil)
        }
    }
```

## 参考
[深入浅出iOS事件机制](http://zhoon.github.io/ios/2015/04/12/ios-event.html)


