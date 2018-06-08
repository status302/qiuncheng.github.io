---
layout:     article
title:      "iOS URL特殊编码小技巧"
subtitle:   "\"你们的 APP 中如果用户的用户名中包含了&/*等等，字符的时候，你们是怎么处理包含该用户名的链接的？\""
date:       2017-12-27 00:38:00
key:        1008
tags:
    - iOS小技巧
    - iOS
---

> 说到 URL 编码问题，iOS 开发经常打交道，有人说使用系统默认的`urlString.addingPercentEncoding(withAllowedCharacters: CharacterSet)` 方法就可以，那么就真的可以吗？
<!--more-->

## 问题
在这里我们结合一个 `URL` 来进行实验一下。 对于这个 `https://api.vsccw.com/user.html?username=吴彦祖&gender=1&age=38` URL 字符串来说，如果我们直接使用去初始化一个 `URL(string: String)` 的话，100%会初始化失败，为什么呢？里面包含了中文，有同学会说，我们在浏览器里面就经常输入带有中文的链接，那是因为浏览器将该URL编码了一下。  
所以我们需要将该URL中查询的参数编码一下。。`Swift4 String` 就已经提供了 `addingPercentEncoding` 方法, 所以我们在这里使用 `.urlQueryAllowed` 即可，关于其他 `CharacterSet`, 想了解的同学可以[参考官方文档](https://developer.apple.com/documentation/foundation/nscharacterset)，得到的结果 `https://api.vsccw.com/user.html?username=%E5%90%B4%E5%BD%A6%E7%A5%96&gender=1&age=38` 果然没有了中文, 初始化 URL 成功. 

但是如果用户名是 `https://api.vsccw.com/user.html?username=吴彦祖&孙俪=%邓超/&gender=1&age=38`，我们在使用上面那种方式试试吧。没毛病，我们得到的结果是这样的 `https://api.vsccw.com/user.html?username=%E5%90%B4%E5%BD%A6%E7%A5%96&%E5%AD%99%E4%BF%AA=%25%E9%82%93%E8%B6%85/&gender=1&age=38` 依旧没有中文, 但是我们仔细观察一下参数。得到的结果如下所示：

```swift
["username"] = xxx,
["%E5%AD%99%E4%BF%AA"] = xxx,
["gender"] = xxx,
["age"] = xxx
```

我们期望得到的正确的参数应该是
```swift
["usename"] = xxx,
["gender"] = xxx,
["age"] = xxx
```

莫名奇妙多了一个参数，其实第二个参数就是因为用户名中包含了&和=之后的结果，那这样就导致了一些问题啊，首先用户名都无法正确显示，其次如果 js 那边获取查询参数是按照 & 分隔符的话，那么这种情况下肯定是很大的问题啊，会莫名其妙多出一些东西。

## 分析

那么对于这种情况我们应该怎么解决呢？首先第一个想法肯定是将&和=也编码了，但是使用系统提供的 `addingPercentEncoding` 显然做不到啊, 你可以试试其他的 `CharacterSet`, 完全略过 `&` 和 `=`, 所以这个时候就需要我们自己来编码那些可能出现 `&` 和 `=` 的地方了。

## 解决

笔者之前看过 `Alamofire` 的源码, 其中就有使用`CFURLCreateStringByAddingPercentEscapes` 来编码链接的字符串（如下图所示），所以我们只需要也使用这种方式，将name里面的字符串也相应编码一下即可。
![](https://user-images.githubusercontent.com/9990834/34905112-4a2624a6-f88d-11e7-9424-075bfebcba4e.png)

解决结果代码如下：

```swift
extension String {
    var usernameURLEncodedString: String {
    	/// 初始的字符串，需要将其转换为 CFString
        let originString = self as CFString
        /// 这些字符必须要使用 %xx 的形式展示
        let escapedString =  "/%&=?$#+-~@<>|\\*,.()[]{}^!" as CFString
        /// 获取该结果，并以UTF8形式编码
        let encodedString = CFURLCreateStringByAddingPercentEscapes(nil, originString, nil, escapedString, CFStringBuiltInEncodings.UTF8.rawValue) as String
        return encodedString
    }
}
```
参考如图：
![](https://user-images.githubusercontent.com/9990834/34905248-a984bde8-f88f-11e7-832f-9845afb7cfc4.png)

## 参考

- [关于URL编码
](http://www.ruanyifeng.com/blog/2010/02/url_encoding.html)
- [nscharacterset](https://developer.apple.com/documentation/foundation/nscharacterset)
- [Alamofire \[Issue #206\]](https://github.com/Alamofire/Alamofire/commit/28df82ec9c0115f43ee7f3dc371cf7a0cb84a587)

