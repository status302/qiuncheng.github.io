---
layout:     article
title:      "Swift3 字符串限制处理"
subtitle:   " \"iOS开发中你遇到过表情被拆开的情况吗？\""
date:       2017-02-12 12:00:00
key:        1003
tags:
    - iOS
    - Swift
---

> 做过iOS开发的可能都遇到过这个需要，『限制字符串的个数』，无论是微信还是支付宝，都有过这种限制，那么究竟该怎么限制呢？ <!--more-->
 
目前我在项目遇到过这个情形，具体体现在用户昵称的长度限制和用户账号、以及个性签名，都限制用户输入字符的长度，当然在上传这些的时候后端也会相应验证字符长度，如果太长，则会收到参数错误的提示。我们目前做到如下：假如限制的字符数为30，那么当用户在输入的过程中，会动态监测当前输入字符的长度（当然这里的长度并不是字符的个数，Swift中的字符在一个字符串中并不一定占用相同的内存空间数量，这个长度和字符的编码有关系，我在下面会给出我们计算长度的方式），如何当前字符长度+即将输入字符长度大于所需的字符长度，那么直接剪裁断，当然我们首先在这里将输入的String断成一个个substring, 这样做的原因主要是考虑到有的『看上去是一个的字符』并不是由一个单独字符组成，而是由多位字符组成。
> 参考：http://unicode.org/emoji/charts-beta/emoji-zwj-sequences.html 这里给出了unicode编码的即将支持的字符的组成。咱们可以拿来做一个参考，从这个网页中也可以看到，一个**字符**还可以由多个『字符』组成。

那么这个“一个**字符**还可以由多个『字符』组成”的情况具体在iOS上是怎么体现出来的呢？见下图：
![当前字符由三个字符组成](http://7xk67j.com1.z0.glb.clouddn.com/EmotionString.jpg)

所以在这里就会出现问题了？如果剪裁时候把一个字符剪裁成两个或者三个了怎么办？那么究竟要怎么样去裁剪呢？当然这里肯定要保证字符的完成性，所以裁剪的时候肯定不可以直接去使用`characters`这个了。原因如上图，那么还有什么更好的方式呢？

有：Swift中提供了强大的字符串处理能力，在这里可以使用`String`标准库中提供的`enumerateSubstrings` 使用了该方法后可以获得String的所有substring，而且不会出现上图中出现的情况。
我在这里写了个extension，具体代码如下：
```swift
extension String {
    /// 返回unicode编码的长度
    var length: Int {
        return unicodeScalars.count
    }
    /// 返回当前String 的所有substring数组
    var sequences: [String] {
        var arr = [String]()
        enumerateSubstrings(in: startIndex..<endIndex, options: .byComposedCharacterSequences) { (substring, _, _, _) in
            if let str = substring { arr += [str] }
        }
        return arr
    }
    /// 截取字符串操作
    ///
    /// - Parameters:
    ///   - toLength: 要截取的字符串的length, 编码为：unicode, 即： `unicodeScalars.count`
    /// - Returns: 返回的截取好的字符串
    func substring(toLength: Int) -> String {
        guard toLength < length else {
                return self
        }
        var results = String()
        for index in 0 ..< sequences.count {
            if results.length + sequences[index].length <= toLength {
                results.append(sequences[index])
            }
            else {
                return results
            }
        }
        return self
    }
}
```
具体涉及到如何在`textFiled`或者`textView`中去做限制字符操作请参考：  
1. [iOS中UITextField的字数限制](http://www.jianshu.com/p/2d1c06f2dfa4)
2. [代码地址(Playground形式)](https://github.com/qiuncheng/posted-articles-in-blog/tree/master/Demos/Emotions.playground)


