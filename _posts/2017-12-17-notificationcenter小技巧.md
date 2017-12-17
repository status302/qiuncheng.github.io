---
layout:     post
title:      "NotificationCenter小技巧"
subtitle:   "\"自从 Swift3 将 Notification.Name 更改了之后，没有了代码提示，大多数情况下还是把多余的提示删掉...，有没有觉得很繁琐。\""
date:       2017-12-17 12:30:00
author:     "vsccw"
header-img: "img/post-bg-2015.jpg"
tags:
    - iOS小技巧
    - iOS
---

> 开发者时间那么宝贵，提高开发效率才是保证啊。

# 问题

在笔者参与开发的 `Swift` 项目中，创建 `Notification.Name` 的方式一直是这样的：

```swift
class NotificationClass {
    
    static let testNotificationName = Notification.Name(rawValue: "com.vsccw.notification.name")
    
    func testPostNotification() {
        NotificationCenter.default.post(name: NotificationClass.testNotificationName, object: nil, userInfo: ["content": "xxx"])
    }
}
```

然后 observe 方式是这样的：

```swift
class ObserveClass {
    
    init() {
        NotificationCenter.default.addObserver(self, selector: #selector(receiveTestNotification(_:)), name: NotificationClass.testNotificationName, object: nil)
    }
    
    deinit {
        NotificationCenter.default.removeObserver(self)
    }
    
    @objc private func receiveTestNotification(_ noti: Notification) {
        /// do something here.
    }
}
```
这样其实是没有什么问题的，而且可以算得上是一种清晰明确的方式，不过还是存在两个问题：

1. `NotificationClass` 可以表示*发出者*，而通知的名字，就是我们的静态属性  `testNotificationName`， 不过这往往造成了一个困扰，开发者还必须要记住，名字为`testNotificationName`的通知的*发出者*是谁，如果我们在开发中突然只记住这个通知名字，而不知道他的发出者，也就是类名，那么还必须要去其文件中查找一下。这样一来就大大降低了开发效率了。
2. 我们在使用的时候是没有代码提示的，仅仅是在我们敲出了 *发送者类名* 之后才可以看到提示。这又再一次降低了效率。

# 解决方式

那么笔者在这里就针对以上两个问题，对创建 Notification.Name 进行一点点改造，以找出一种优雅的方式，来处理这个问题。

### 1. 利用 `Swift` 中强大的 `extension` 特性，直接：

```
extension Notification.Name {
    
    static let testNotificationName = Notification.Name(rawValue: "com.vsccw.notification.name")
}
```
这样我们在使用的时候可以在 `<#T##NSNotification.Name?#>` 里面直接点击 `enter` 然后 `delete` 掉 `?` 直接输入 `.` 就会有提示了.

不过这里明显存在一个弊端, 那就是系统的所有 Notification 都是以这种方式进行的，如下：

```
extension NSNotification.Name {
    public static let UIWindowDidBecomeVisible: NSNotification.Name
}
```
那么当你使用这种方式，输入 `.` 的时候也必然会出现很多系统的通知名字。所以很多团队在这里使用前缀的方式来进行区别。不得不承认，名字越来越长啊。

### 2. 也是使用 `extension` 的方式, 不过在 `extension` 里面使用 `struct` 进行包装一下.

```swift
extension Notification.Name {
    struct SomeWork {
        static let testNotificationName = Notification.Name(rawValue: "com.vsccw.notification.name")
    }
}
```
那么这个时候在使用的时候直接这样：

```swift
/// 添加通知
NotificationCenter.default.addObserver(self, selector: #selector(receiveTestNotification(_:)), name: NSNotification.Name.SomeWork.testNotificationName, object: nil)

/// 发送通知
NotificationCenter.default.post(name: NSNotification.Name.SomeWork.testNotificationName, object: nil, userInfo: [:])
```

虽然 Name看起来很长，不过这也是没有办法的事情，毕竟iOS开发中名字长也是常有的事情。

### 3. 针对笔者项目的问题，可以直接将 `Notification.Name` 写在类外，而且还可以免去使用 static 静态属性。

```swift
let testNotificationName = Notification.Name(rawValue: "com.vsccw.notification.name")

class NotificationClass {
    
    func testPostNotification() {
        NotificationCenter.default.post(name: testNotificationName, object: nil, userInfo: ["content": "xxx"])
    }
}
```
这也不失为一种好的处理方式, 但为了让其含义更加具体, 还是避免不了使用很长的名字才行.

# 总结
在笔者提出的3种方式中，笔者更加倾向于1、2中，毕竟笔者项目中使用的那个方式是在不是很友好。对于第3种，笔者在阅读他人的代码中，也多次看到，综合考虑一下，还是要将推荐度 -1， 只是因为多了删除和记忆工作。

如果你有更好的处理方式，也请及时下方评论，我们一起探讨。😃

