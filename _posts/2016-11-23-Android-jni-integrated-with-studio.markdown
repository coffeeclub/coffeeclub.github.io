---
layout: post
title:  "Android jni integrated with studio"
date:   2016-11-23 15:40:05
categories: 
---

java支持使用c/c++代码编写native方法，并可由特定的接口方便的调用，并传递数据，也就是java native interface


* _环境搭建_

目前主流的android编程普遍使用gradle，因此可以使用android studio并安装ndk和gradle，来支持jni编程

* _使用android studio方便的编写jni代码_

由于android studio默认的ndkbuild文件无法重写，网上查找的一些方法会在build.gradle中设置jni的src为空，这样可以使ide不会自动build jni代码，并用自己重写的Android.mk(ndk的编译文件)来手动编译jni代码。但是这样的方式在编写c++代码时没有语法检查和提示，非常不方便，效率变得很低。于是，查遍资料后，找到一种studio支持的方法，可以自定义一些local参数，share/static library等。只需修改build.gradle文件，添加一些属性即可。之后可以很方便的编写代码，还有关键字提示，快速查看函数定义等。

  1.首先修改工程gradle版本，使用gradle-experimental，可以方便的支持新的ndk属性
   {% highlight java %}
buildscript {  
    repositories {  
        jcenter()  
    }  
    dependencies {  
        classpath 'com.android.tools.build:gradle-experimental:0.7.0'  
     
        // NOTE: Do not place your application dependencies here; they belong  
        // in the individual module build.gradle files  
     }  
}  
allprojects {  
    repositories {  
        jcenter()  
    }  
}  
    
task clean(type: Delete) {  
    delete rootProject.buildDir  
}  
{% endhighlight %}

  2.修改build.gradle，以下是一个例子:
{% highlight java %}
apply plugin: 'com.android.model.application'
model {
    repositories {
         libs(PrebuiltLibraries) {
            // Configure one pre-built lib: static
            nnn {
                // Inform Android Studio where header file dir for this lib
                headers.srcDir "src/main/jniLibs/include"
                // Inform Android Studio where lib is -- each ABI should have a lib file
                binaries.withType(SharedLibraryBinary) {
                    sharedLibraryFile = file("src/main/jniLibs/libnnn.so")
                }
            }
            png {
                headers.srcDir "src/main/jniLibs/include/png"
                binaries.withType(StaticLibraryBinary) {
                    staticLibraryFile = file("src/main/jniLibs/libpng.a")
                }
            }
            soil {
                headers.srcDir "src/main/jniLibs/include"
                binaries.withType(StaticLibraryBinary) {
                    staticLibraryFile = file("src/main/jniLibs/libsoil.a")
                }
            }
            // Configure another pre-built lib: shared;[change to static after Studio supports]
            // static lib generation. USING static lib is supported NOW, for that case,
            // simple change:
            //  SharedLibraryBinary --> StaticLibraryBinary
            //  sharedLibraryFile  --> staticLibraryFile
            //  *.so --> *.a
         }
     } 
     android {
        compileSdkVersion = 23
        buildToolsVersion = '23.0.2'
     
        defaultConfig.with {
            applicationId = 'com.example.test.nativetest'
            minSdkVersion.apiLevel = 19
            targetSdkVersion.apiLevel = 23
            versionCode = 1
            versionName = '1.0'
        }
        ndk {
            moduleName = 'xx'
            abiFilters.addAll(["armeabi-v7a"])
            stl = "gnustl_static"
            ldLibs.addAll(['GLESv3', 'EGL', 'OpenMAXAL', 'android', 'OpenSLES', 'z', 'log'])
            cppFlags.addAll(['-fexceptions'])
        }

        buildTypes {
            release {
                minifyEnabled = false
                proguardFiles.add(file('proguard-android.txt'))
            }
        }

          //sourceSets.main {
           // jniLibs.srcDir 'src/main/libs'
           // jni.srcDirs = [] //disable automatic ndk-build call with auto-generated Android.mk         
           //}

        sources.main {
            jni {
                source {
                         //此处将source设成null，仍然使用自己的android.mk手动编译
                }
                dependencies {
                    library 'nnn' linkage 'shared'
                    library 'png' linkage 'static'
                    library 'soil' linkage 'static'
                }
            }
            jniLibs {
                source {
                    srcDir 'src/main/libs'
                }
            }

         }
     }
}
repositories {
     jcenter()
     flatDir {
         dirs 'src/main/libs'
     }
}
dependencies {
     compile(name: 'xxlib', ext: 'aar')  //此处是依赖的java代码库
     //    compile fileTree(include: ['*.aar'], dir: 'libs')
     //    compile fileTree(include: ['*/*.so'], dir: 'libs')
     testCompile 'junit:junit:4.12'
     compile 'com.android.support:appcompat-v7:23.3.0'
}
{% endhighlight %}

  3.使用ndk编译jni代码

编写android.mk，该文件是一个ndk支持的编译文件，使用简单的格式达到编译c++代码的目的，具体语法可参考ndk官方文档

编写Applacation.mk，该文件可以指定编译环境
{% highlight java %}
android.mk:

LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)

LOCAL_MODULE := png
LOCAL_EXPORT_C_INCLUDES := $(LOCAL_PATH)/../jniLibs/include/png
LOCAL_SRC_FILES := $(LOCAL_PATH)/../jniLibs/libpng.a

include $(PREBUILT_STATIC_LIBRARY)

#include $(CLEAR_VARS)

LOCAL_MODULE := soil
LOCAL_EXPORT_C_INCLUDES := $(LOCAL_PATH)/../jniLibs/include/
LOCAL_SRC_FILES := $(LOCAL_PATH)/../jniLibs/libsoil.a

include $(PREBUILT_STATIC_LIBRARY)

include $(CLEAR_VARS)

LOCAL_STATIC_LIBRARIES := png
LOCAL_STATIC_LIBRARIES += soil

LOCAL_MODULE    := xx
LOCAL_SRC_FILES  := xx.cpp

LOCAL_C_INCLUDES += ../jniLibs/include

LOCAL_LDLIBS += -llog

include $(BUILD_SHARED_LIBRARY)
{% endhighlight %}

{% highlight java %}
Application.mk

# This needs to be defined to get the right header directories for egl / etc
APP_PLATFORM   := android-19

# This needs to be defined to avoid compile errors like:
# Error: selected processor does not support ARM mode `ldrex r0,[r3]'
APP_ABI       := armeabi-v7a

# These are needed for CImg lib
APP_STL := gnustl_static
APP_CPPFLAGS += -fexceptions
{% endhighlight %}



