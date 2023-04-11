# iOS开发实用技巧

[在低版本的Xcode中运行高版本的真机](#在低版本的Xcode中运行高版本的真机)

[清除不使用的Xcode模拟器，节省mac空间](#清除不使用的Xcode模拟器，节省mac空间)

[不要使用stringByAppendingPathComponent:方法做URL的拼接！](#不要使用stringByAppendingPathComponent:方法做URL的拼接！)

## 在低版本的Xcode中运行高版本的真机

* 去 https://github.com/iGhibli/iOS-DeviceSupport/tree/master/DeviceSupport 这里下载你要运行的真机版本

* 将下载好的压缩包解压，将解压后的文件夹拷贝至/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/DeviceSupport

* 重启Xcode

源自：https://stackoverflow.com/questions/67863355/xcode-12-4-unsupported-os-version-after-iphone-ios-update-14-7

## 清除不使用的Xcode模拟器，节省mac空间

`xcrun simctl delete unavailable`

源自：https://stackoverflow.com/questions/34910383/xcode-free-to-clear-devices-folder/34914591#34914591

## 不要使用stringByAppendingPathComponent:方法做URL的拼接！

[苹果文档](https://developer.apple.com/documentation/foundation/nsstring/1417069-stringbyappendingpathcomponent)明确表明`stringByAppendingPathComponent:`只能适用于文件路径，不适用于URL。那么我如果强行使用会有什么后果呢？

比如我使用`https://www.baidu.com`后面拼接一个`search`，希望生成`https://www.baidu.com/search`这样子的URL路径。如果使用

```objective-c
[@"https://www.baidu.com" stringByAppendingPathComponent:@"search"]
```

你会发现拼接后的URL地址为`https:/www.baidu.com/search`，https协议少了一个/，虽然这样子的URL字符串可以生成一个NSURL对象，并且能正常请求（wtf!?）。但是还是不要使用`stringByAppendingPathComponent:`做URL的拼接，会留下会多坑，这样子拼接的NSURL的host属性为nil，并且如果你有一些NSURLProtocol，他们对这些请求的处理逻辑可能也会出现问题。