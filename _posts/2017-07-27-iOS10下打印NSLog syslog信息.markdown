---
layout: post
title: iOS10下打印NSLog syslog信息
date: 2017-07-27 15:32:24.000000000 +09:00
---

在iO10以前的版本越狱机器可以用socat方便的打印出NSLog信息，但在iOS10下由于日志系统发生了改变，以前一些打印方法失效。

The logging system has changed in iOS 10. Apple now uses what it calls "Unified Logging". Below are links to a brief overview, as well as a WWDC session covering the topic:

https://developer.apple.com/reference/os/logging

https://developer.apple.com/videos/play/wwdc2016/721/

Note that Apple claims that it is much more efficient (as it does not write to disk), and so it appears that there is no need to ever turn it off.



iOS10下可用deviceconsole这个工具打印系统日志，安装方法从git上下载源码https://github.com/MegaCookie/deviceconsole并编译，将编译好的二进制文件，放到/usr/local/bin下并保证执行权限。

deviceconsole是需要通过usb来连接手机，在终端中运行deviceconsole，并连上手机，操作说明如下：

```
Usage: deviceconsole [options]

Options:
-i | --case-insensitiveMake filters case-insensitive
-f | --filter Filter include by single word occurrences (case-sensitive)
-x | --exclude Filter exclude by single word occurrences (case-sensitive)
-p | --process Filter by process name (case-sensitive)
-u | --udid Show only logs from a specific device
-s | --simulator Show logs from iOS Simulator
--debugInclude connect/disconnect messages in standard out
--use-separatorsSkip a line between each line
--force-colorForce colored text
Control-C to disconnect

```

比如打印iOSRETargetApp这个app的信息，可用deviceconsole -i -f iOSRETargetApp命令，打印结果如下：

```
dxs-Mac-mini:bin tdx$ deviceconsole -i -f iOSRETargetApp
Mar3 10:14:22 sjjvenumediaserverd(AudioToolbox)[269]:1067: pid 1103(iOSRETargetApp)
Mar3 10:14:25 sjjvenuiOSRETargetApp[1103]:addButtonTapped
Mar3 10:14:27 sjjvenuiOSRETargetApp[1103]:addButtonTapped
Mar3 10:14:27 sjjvenuiOSRETargetApp[1103]:addButtonTapped
Mar3 10:14:28 sjjvenuiOSRETargetApp[1103]:addButtonTapped
Mar3 10:14:55 sjjvenuiOSRETargetApp[1103]:addButtonTapped
Mar3 10:14:56 sjjvenuiOSRETargetApp[1103]:addButtonTapped
Mar3 10:15:37 sjjvenumediaserverd(AudioToolbox)[269]:1067: pid 1103(iOSRETargetApp)
Mar3 10:15:38 sjjvenumediaserverd(AudioToolbox)[269]:1067: pid 1103(iOSRETargetApp)
Mar3 10:17:00 sjjvenuSpringBoard[317]:Process exited: ->
Mar3 10:17:10 sjjvenuiOSRETargetApp(Flex.dylib)[1244]:FLXX: Loaded patches from: /var/mobile/Library/Application Support/Flex3/patches.plist
Mar3 10:17:10 sjjvenuiOSRETargetApp(libsystem_trace.dylib)[1244]:subsystem: com.apple.siri, category: Intents, enable_level: 1, persist_level: 1, default_ttl: 0, info_ttl: 0, debug_ttl: 0, generate_symptoms: 0, enable_oversize: 0, privacy_setting: 0, enable_private_data: 0
Mar3 10:17:10 sjjvenuiOSRETargetApp(iOSREHookerTweak.dylib)[1244]:====== Find Short C Function!
Mar3 10:17:10 sjjvenuiOSRETargetApp(iOSREHookerTweak.dylib)[1244]:[iOSREHookerTweak] Tweak.xm:51DEBUG:Hello Test HBLogDebug
```
