# iOS开发实用技巧

[在低版本的Xcode中运行高版本的真机](#在低版本的Xcode中运行高版本的真机)

[清除不使用的Xcode模拟器，节省mac空间](#清除不使用的Xcode模拟器，节省mac空间)

## 在低版本的Xcode中运行高版本的真机

* 去https://github.com/iGhibli/iOS-DeviceSupport/tree/master/DeviceSupport这里下载你要运行的真机版本

* 将下载好的压缩包解压，将解压后的文件夹拷贝至/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/DeviceSupport

* 重启Xcode

源自：https://stackoverflow.com/questions/67863355/xcode-12-4-unsupported-os-version-after-iphone-ios-update-14-7

## 清除不使用的Xcode模拟器，节省mac空间

`xcrun simctl delete unavailable`

源自：https://stackoverflow.com/questions/34910383/xcode-free-to-clear-devices-folder/34914591#34914591