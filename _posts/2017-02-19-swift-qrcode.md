---
layout:     article
title:      "关于iOS上二维码的简单处理"
date:       2017-02-19 19:00:00
key:        1002
tags:
    - iOS
    - QRCode
---

## 前言
要试着扫描下面的二维码嘛？  
<img src="http://images.vsccw.com/IMG_2831.JPG" width="200px" />

## 闲言
关于二维码，应该是个烂大街的话题了，同时它的出现也让这个互联网的时代信息传递的更便捷。在这个发展迅速的互联网时代他还能够存活多久对我们来说都是个未知数，不过作为一名开发者，还是有必要去了解下平台下的相关知识。
<!--more-->

## 二维码介绍
> 二维条码是指在一维条码的基础上扩展出另一维具有可读性的条码，使用黑白矩形图案表示二进制数据，被设备扫描后可获取其中所包含的信息。一维条码的宽度记载着数据，而其长度没有记载数据。二维条码的长度、宽度均记载着数据。二维条码有一维条码没有的“定位点”和“容错机制”。容错机制在即使没有辨识到全部的条码、或是说条码有污损时，也可以正确地还原条码上的信息。  
以上抄自[维基百科](https://zh.wikipedia.org/wiki/%E4%BA%8C%E7%B6%AD%E6%A2%9D%E7%A2%BC)

## 原理是什么
我觉得这个是一个深奥又具有研究性的话题，如果你非要纠结的话，我推荐你去看看 **左耳朵耗子大叔的『二维码的生成细节和原理』** 链接放在了文末的参考里。我这一篇是很基础的记录文章。所以不会告诉你原理是什么。
## 为什么写这个
现在网上关于iOS上二维码的东西到处都是，那么我写这篇文章是为什么呢？主要还是记录下自己使用`Swift3`遇到的问题，也算是对自己实现过程的一个回顾。YOLO的项目之前支持iOS7的时候遗留了Zxing，所以这次支持iOS 8+了也算彻底将其移除。
## 二维码的扫描
### 第一步：自定义相机
二维码扫描用到`AVFoundation`自定义相机，关于更多自定义的详情，推荐这篇文章 [30分钟搞定iOS自定义相机](http://www.jianshu.com/p/8b28892bae5a)写的很不错。认真看完真的是30就可以自定义一个相机来。
那么咱们这个就直接来看代码吧
```swift
/// 需要用到的变量
fileprivate var captureSession = AVCaptureSession()
fileprivate var capturePreviewLayer: AVCaptureVideoPreviewLayer?
fileprivate var deviceInput: AVCaptureDeviceInput?
fileprivate var metadataOutput: AVCaptureMetadataOutput?
fileprivate var setupResult = YLScanSetupResult.successed
fileprivate var sessionQueue = DispatchQueue(label: "com.vsccw.qrcode.session.queue", attributes: [], target: nil)
fileprivate var rectOfInteres = CGRect.zero
```

```swift
/// 使用enum表示进入扫描二维码界面相机初始化状态
enum YLScanSetupResult {
    case successed /// 已经获取到相机权限
    case failed   /// 没有获取到权限
    case unknown  /// 状态未知
}
```

```swift
/// 请求相机权限代码，可以放在viewDidLoad: 获取viewDidAppear: 里调用
func authorizationStatus() -> YLScanSetupResult {
    var setupResult = YLScanSetupResult.successed
    let authorizationStatus = AVCaptureDevice.authorizationStatus(forMediaType: AVMediaTypeVideo)
    switch authorizationStatus {
    case .authorized:
        setupResult = YLScanSetupResult.successed
    case .notDetermined:
        sessionQueue.suspend()
        AVCaptureDevice.requestAccess(forMediaType: AVMediaTypeVideo, completionHandler: { [weak self] (granted) in
                if !granted {
                    setupResult = YLScanSetupResult.failed
                }
                self?.sessionQueue.resume()
            })
            break
    case .denied:
        setupResult = YLScanSetupResult.failed
        break
    default:
        setupResult = YLScanSetupResult.unknown
        break
    }
    return setupResult
}
```

```swift
/// 初始化相机session， 在这里应该注意几点：
/// 1. previewLayer也就是预览layer是直接添加在view.layer上的
/// 2. 配置sesssion时候不要忘了-beginConfiguration / -commitConfiguration
fileprivate func configSession() {
    guard setupResult == .successed else { return }
    /// setup session
    captureSession.beginConfiguration()
    do {
        var defaultVedioDevice: AVCaptureDevice?
        if #available(iOS 10.0, *) {
            if let backCameraDevice = AVCaptureDevice.defaultDevice(withDeviceType: AVCaptureDeviceType.builtInWideAngleCamera, mediaType: AVMediaTypeVideo, position: .back) {
                defaultVedioDevice = backCameraDevice
            }
        } else {
            if let cameraDevice = AVCaptureDevice.defaultDevice(withMediaType: AVMediaTypeVideo) {
                defaultVedioDevice = cameraDevice
            }
        }
        let videoDeviceInput = try AVCaptureDeviceInput(device: defaultVedioDevice)

        /// 添加自动对焦功能，否则不容易读取二维码
        /// **添加了自动对焦，反而增大了模糊误差**
        if videoDeviceInput.device.isAutoFocusRangeRestrictionSupported
            && videoDeviceInput.device.isSmoothAutoFocusSupported {
            try videoDeviceInput.device.lockForConfiguration()
            videoDeviceInput.device.focusMode = .autoFocus
            videoDeviceInput.device.unlockForConfiguration()
        }

        if captureSession.canAddInput(videoDeviceInput) {
            captureSession.addInput(videoDeviceInput)
        }
        self.deviceInput = videoDeviceInput
    } catch {
        print("无法添加input.")
        setupResult = .failed
    }
    metadataOutput = AVCaptureMetadataOutput()
    if captureSession.canAddOutput(metadataOutput) {
        captureSession.addOutput(metadataOutput)
    } else {
        setupResult = .failed
        return
    }
    metadataOutput?.setMetadataObjectsDelegate(self, queue: self.sessionQueue)
    /// 指定输出的元数据类型为 二维码 类型
    metadataOutput?.metadataObjectTypes = [AVMetadataObjectTypeQRCode]
    /// 这里的rectOfInterest虽然是CGRect类型的，有几点需要注意：
    /// 1. 默认为CGRect(0, 0, 1, 1)
    /// 2. 上面(0,0,1,1)为从(top left 到 bottom right)，所以这里在计算的时候就要注意下了：大致是这样(y/viewY, x/viewX, height/viewHeight, width/viewWidth)
    metadataOutput?.rectOfInterest = self.rectOfInteres
    
    captureSession.commitConfiguration()
    
    capturePreviewLayer = AVCaptureVideoPreviewLayer(session: captureSession)
    capturePreviewLayer?.frame = view.bounds
    view.layer.insertSublayer(capturePreviewLayer!, at: 0)
}
```
既然准备工作都已经做好，那么就该运行它了。由于session的初始化工作是比较费时的, 所以这里创建了一个queue来异步处理，那么如何启动它呢？
其实很简单，只需要调用`captureSession.startRunning()`即可。
停止的话也只需要调用`captureSession.stopRunning()`方法。

### 第二步，去识别
以上是相机相关，下面简单介绍下如何去读取。主要用到`AVCaptureMetadataOutputObjectsDelegate`的代理方法`func captureOutput(_ captureOutput: AVCaptureOutput!, didOutputMetadataObjects metadataObjects: [Any]!, from connection: AVCaptureConnection!)`
具体实现如下：

```swift
func captureOutput(_ captureOutput: AVCaptureOutput!, didOutputMetadataObjects metadataObjects: [Any]!, from connection: AVCaptureConnection!) {
    /// 相机捕获到的照片里可能有多个元数据信息
    for _supportedBarcode in metadataObjects {
        guard let supportedBarcode = _supportedBarcode as? AVMetadataObject else { return }
        /// 在这里只取了QRCode，而且可能存在多张二维码的情况，我测试了下，多张二维码优先左上角，所以这里只取一张，就直接return
        if supportedBarcode.type == AVMetadataObjectTypeQRCode {
            guard let barcodeObject = self.capturePreviewLayer?.transformedMetadataObject(for: supportedBarcode) as? AVMetadataMachineReadableCodeObject else { return }
            /// resultString就是获取的二维码信息
            self.resultString = barcodeObject.stringValue
            self.stopSessionRunning()
            return
        }
    }
}
```
以上就是二维码的扫描识别，不过要是想做到更精确点的话，比如像微信那样，距离远了自动放大，就需要我们自己去创新了。以上具体的代码请看下面的GitHub地址。
## 二维码的识别
图片上的二维码识别的话相比扫描识别就比较简单了，主要用到CoreImage，但是只支持iOS 8+。
所以在写之前应该`import CoreImage`
具体代码如下：

```swift
typealias CompletionHandler<T> = (T) -> Void
func scanQRCodeFromPhotoLibrary(image: UIImage, completion: CompletionHandler<String?>) {
    guard let cgImage = image.cgImage else {
        completion(nil)
        return
    }
    /// 这里设置了识别的精准程度为High，不过这个可能会有一点耗时。
    if let detector = CIDetector(ofType: CIDetectorTypeQRCode, context: nil, options: [CIDetectorAccuracy: CIDetectorAccuracyHigh]) {
        let features = detector.features(in: CIImage(cgImage: cgImage))
        for feature in features { // 这里实际上可以识别两张二维码，在这里只取第一张（左边或者上边）
            if let qrFeature = feature as? CIQRCodeFeature {
                completion(qrFeature.messageString)
                return
            }
        }
    }
    completion(nil)
}
```

## 二维码的生成
生成的话也是用到CoreImage。所以使用之前也要先`import CoreImage`即可。
具体代码如下：
```swift
 static func beginGenerate(text: String, completion: CompletionHandler<UIImage?>) {
    let strData = text.data(using: .utf8)
    
    let qrFilter = CIFilter(name: "CIQRCodeGenerator")
    /// inputMessage: 表示要转换的data数据，也就是我们的信息String
    qrFilter?.setValue(strData, forKey: "inputMessage")
    /// inputCorrectionLevel: 可以理解成容错率，实际上这个参数是控制输出图片允许错误出现的数目。
    qrFilter?.setValue("H", forKey: "inputCorrectionLevel")
    
    if let ciImage = qrFilter?.outputImage {
        /// 这里生成的是260宽260高的二维码
        let size = CGSize(width: 260, height: 260)
        ///  在iOS 8上有点小问题：会报这个Error： [CIContext initWithOptions:]: unrecognized selector sent to instance xxxx
        /// 原因可能是Xcode8转换Swift代码到OC不成功导致，解决方法也很简单，只需要用OC代码写一个CIContext分类即可，具体解决代码我放在下面。
        let context = CIContext(options: nil)
        var cgImage = context.createCGImage(ciImage, from: ciImage.extent)

        UIGraphicsBeginImageContext(size)
        let cgContext = UIGraphicsGetCurrentContext()
        cgContext?.interpolationQuality = .none
        cgContext?.scaleBy(x: 1.0, y: -1.0)
        cgContext?.draw(cgImage!, in: cgContext!.boundingBoxOfClipPath)
        
        let codeImage = UIGraphicsGetImageFromCurrentImageContext()
        UIGraphicsEndImageContext()
        completion(codeImage)
        cgImage = nil
        return
    }
    completion(nil)
}
```
上面提到问题的解决方式链接放在了最下面。
## 小知识
### 微信二维码长按识别功能
这个很好做的，就是在长按的过程中，对当前的照片进行一次识别，如果有二维码弹出框就显示『识别二维码』按钮，如果没有识别到二维码的话就不显示。我会在Demo中给出相应的实现。
### 给二维码添加上像微信一样的logo
这个也是比较好做的吧，因为在生成二维码的时候使用了UIGraphics..,所以我们在这里只需要将logo绘制到当前的Context上就可以啦。如果你仔细看微信的话，你会发现微信生成的二维码logo周边都有一些白色阴影，那是为了让整个二维码看起来更加融合，所以我这里提供了一个简单的实现方式。
```swift
func getBorderImage(image: UIImage) -> UIImage? {
    let imageView = UIImageView()
    imageView.frame = CGRect(x: 0, y: 0, width: 34, height: 34)
    imageView.layer.borderColor = UIColor.white.cgColor
    /// 这里的border恰好可以做到logo周边的效果
    imageView.layer.borderWidth = 2.0
    imageView.image = image
        
    var currentImage: UIImage? = nil
    UIGraphicsBeginImageContext(CGSize(width: 34.0, height: 34.0))
    if let context = UIGraphicsGetCurrentContext() {
        imageView.layer.render(in: context)
        currentImage = UIGraphicsGetImageFromCurrentImageContext()
    }
    UIGraphicsEndImageContext()
    return currentImage
}
```
然后使用draw方法直接绘制到outputImage上去就可以。
比如：
```swift
let image = YLGenerateQRCode.getBorderImage(image: <#image you give#>)            
if let podfileCGImage = image?.cgImage {
    cgContext?.draw(podfileCGImage, in: cgContext!.boundingBoxOfClipPath.insetBy(dx: (size.width - 34.0) * 0.5, dy: (size.height - 34.0) * 0.5))
}
```

### 绘制不同颜色的二维码
大部分的二维码都是黑白照，不过如果你真的想要彩色的话，也是可以通过CoreImage来实现。
具体如同生成二维码一样，不过这次创建的是一个CIFalseColor。然后指定其color0和color1即可，具体步骤可以参考**生成二维码**。
### 扫描UI界面及动画
这个感觉So easy吧, 不过需要注意下动画的remove就可以啦

## 几点注意
- [rectOfInterest]()
- iOS10申请权限
- iOS8 上CIContext消息传递问题

## 参考及链接
- [左耳朵耗子大叔的『二维码的生成细节和原理』](http://coolshell.cn/articles/10590.html)
- [串哥的这篇『再见ZXing 使用系统原生代码处理QRCode』](http://adad184.com/2015/09/30/goodbye-zxing/)
- [代码在GitHub上](https://github.com/qiuncheng/posted-articles-in-blog/tree/master/Demos/QRCodeDemo)
    代码风格不太好，所以如果看起来不舒服，请谅解。
- [『CIContext initWithOptions:』: unrecognized selector sent to instance xxxx](http://stackoverflow.com/questions/39570644/cicontext-initwithoptions-unrecognized-selector-sent-to-instance-0x170400960)