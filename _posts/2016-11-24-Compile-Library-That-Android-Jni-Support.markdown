---
layout: post
title:  "Compile Library That Android Jni Support"
date:   2016-11-23 15:40:05
categories: Android
---

在开发android jni的程序中，有些功能需要用到第三方的library，网上可以直接下载library，但通常这些lib都是pc platform，无法直接被android项目调用。因此就需要下载源代码手动编译成android platform可以用的lib包。

具体做法如下：

 1. 新建文件夹jni，将src下需要的源代码拷贝到jni中
 2. 编写Android.mk编译文件，加入src file和一些依赖，指定编译的类型:static/shared library
 3. 编写Application.mk文件，指定编译的参数，注意这些编译参数应和android项目中的参数一致

    ~~~~
    Application.mk(例子)
    # This needs to be defined to get the right header directories for egl / etc
    APP_PLATFORM := android-19
    
     # This needs to be defined to avoid compile errors like:
     # Error: selected processor does not support ARM mode `ldrex r0,[r3]'
    APP_ABI := armeabi-v7a
    
    # These are needed for CImg lib
    APP_STL := gnustl_static
    APP_CPPFLAGS += -fexceptions
    ~~~~

 4. 进入jni目录中，执行ndk-build，编译完成后lib会生成在../obj/local/armeabi-v7a/
 5. 编译后的lib就可以拷贝到android项目中，由jni调用，注意还应拷贝相关的.h头文件

注意：有些代码pc上可以支持，但android平台上未必有对应的类库或方法，因此可能需要手动修改才能编译通过
