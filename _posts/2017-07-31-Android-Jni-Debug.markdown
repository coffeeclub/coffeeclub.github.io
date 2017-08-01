---
layout: post
title:  "Android Jni Debug With Studio"
date:   2017-07-31 13:00:05
categories: Android
---

### Jni Debug with Studio

Jni代码执行出错，又找不到问题所在，若是可以像c++项目一样可以进行调试，对于开发者来说就方便很多。
下面就是使用Android Studio进行调试native代码的方法：

1. 若希望在studio中直接调试，必须使用gradle支持编译jni代码，即使用com.android.tools.build:gradle-experimental插件编写build.gradle。
2. 使用gradle build apk，编译后的jni代码可以在app/build/intermediates/binaries/路径下找到，若最后build apk成功，表示我们的build.gradle配置正确。
  
   注：若之前已经使用android.mk编译过jni代码，可以将jniLibs下面的so删除，以防止.so无法更新
3. jni调试：
  * 点击 run->debug->Edit Configuration
  * 弹出的对话框中在Debugger中选择Native,默认是调试java代码，配置完成后点击debug就可以开始调试了，可以在jni代码中设置断点

  注：java代码和jni代码同一时间只能选择一种进行调试


### Jni的那些坑

* stl = "gnustl_static"，该变量指定c++ stl使用的版本，但是gnustl_static的stl部分函数没有如to_string，若编译报错可以改成"c++_shared"
* 编译上层的stl库一定要与依赖的so编译时的stl库相同，否则链接会报错，因两者stl不能通用
* 同样道理，设置abiFilters时也应注意，相同的ABI才能互相依赖，若so库的ABI与本地jni的ABI不同，也会出现各种错误
* 使用com.android.tools.build:gradle-experimental插件时，若gradle报错，可以尝试在赋值时加上=，目测新版已无此问题
