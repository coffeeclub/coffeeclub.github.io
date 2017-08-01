---
layout: post
title:  "Android Jni Integrated With Studio"
date:   2016-11-23 15:40:05
categories: Android
---

java支持使用c/c++代码编写native方法，并可由特定的接口方便的调用，并传递数据，也就是java native interface


* **环境搭建**

  目前主流的android编程普遍使用gradle，因此可以使用android studio并安装ndk和gradle，来支持jni编程

* **使用android studio方便的编写jni代码**

  由于android studio默认的gradle插件对jni代码的支持不好，需要在build.gradle中设置jni的src为空，这样可以使ide不会自动build jni代码，同时使用自己重写的Android.mk(ndk的编译文件)来手动编译jni代码。

  但是这样的方式在编写c++代码时没有语法检查和提示，并且不能debug native代码，非常不方便。此时我们可以使用gradle的gradle-experimental插件：
  
  * [Experimental Plugin User Guide](http://tools.android.com/tech-docs/new-build-system/gradle-experimental#TOC-Latest-Version)
  * [Experimental Plugin与Android.mk的映射](https://www.twilio.com/blog/2016/03/building-native-android-libraries-with-the-latest-experimental-android-plugin.html)

* **使用android studio编译jni，gradle配置步骤如下**
  1. 首先修改工程gradle版本，使用gradle-experimental，可以方便的支持新的ndk属性

     ~~~~
     buildscript {  
        repositories {  
          jcenter()  
        }  
        dependencies {  
          classpath 'com.android.tools.build:gradle-experimental:0.9.3'  
     
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
     ~~~~

  2. 修改app中的build.gradle，以下是一个例子:

     ~~~~
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
		  //multi header dirs
                  headers.srcDirs=["src/main/jniLibs/include/soil1","src/main/jniLibs/include/soil2"]
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
          compileSdkVersion = 25
          buildToolsVersion = '25.0.2'
     
          defaultConfig.with {
              applicationId = 'com.example.test.nativetest'
              minSdkVersion.apiLevel = 19
              targetSdkVersion.apiLevel = 25
              versionCode = 1
              versionName = '1.0'
          }
          ndk {
              moduleName = 'xx'
              abiFilters.addAll(["armeabi-v7a"])
              stl = "gnustl_static"
              //stl="c++_shared"     //another stl library , please choose one
              ldLibs.addAll(['GLESv3', 'EGL', 'OpenMAXAL', 'android', 'OpenSLES', 'z', 'log'])
              cppFlags.addAll(['-fexceptions', '-std=c++11', '-fsigned-char'])
              platformVersion 21   //ndk platform build version
           }

          buildTypes {
              release {
                  minifyEnabled = false
                  proguardFiles.add(file('proguard-android.txt'))
                  ndk{
                      debuggable true  //support native debug 
                  }
              }
         
          }
    
             //sourceSets.main {
              // jniLibs.srcDir 'src/main/libs'
             // jni.srcDirs = [] //disable automatic ndk-build call with auto-generated Android.mk         
             //}

          sources.main {
              jni {
                  source {
                       //srcDirs =[]           //若此处将source设成null，仍然使用自己的android.mk手动编译
                       srcDir "src/main/jni"
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
     ~~~~
  
  3. 配置完成后编写native代码，编译时直接点击studio中的run或debug即可，编译后的jni代码可以在app/build/intermediates/binaries/路径下找到，若运行成功，表示我们的配置生效了
 
* **使用Android.mk手动编译jni代码**

  若不希望使用studio编译，也可以编写android.mk，然后使用命令行ndk-build来编译，此方法可以看到更详细的错误信息，查找编译问题时比较方便。Android.mk文件是一个ndk支持的编译文件，使用简单的格式达到编译c++代码的目的，具体语法可参考[ndk官方文档](https://developer.android.com/ndk/guides/android_mk.html)

{% highlight java %}
android.mk:

LOCAL_PATH := $(call my-dir)

include $(LOCAL_PATH)/../jniLibs/another.mk

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

LOCAL_STATIC_LIBRARIES := png soil
LOCAL_SHARED_LIBRARIES := another

LOCAL_MODULE    := xx
LOCAL_ARM_MODE := arm
LOCAL_SRC_FILES  := xx.cpp xx2.cpp
LOCAL_EXPORT_C_INCLUDES := xx.h xx2.h $(LOCAL_PATH)/../include
LOCAL_C_INCLUDES += $(LOCAL_PATH)/../jniLibs/include

LOCAL_LDLIBS += -llog
LOCAL_LDLIBS +=-landroid

include $(BUILD_SHARED_LIBRARY)
{% endhighlight %}

{% highlight java %}
Application.mk

# This needs to be defined to get the right header directories for egl / etc
APP_PLATFORM   := android-21

# This needs to be defined to avoid compile errors like:
# Error: selected processor does not support ARM mode `ldrex r0,[r3]'
APP_ABI       := armeabi-v7a

# These are needed for CImg lib
APP_STL := gnustl_static
APP_CPPFLAGS += -fexceptions -std=c++11 -fsigned-char
{% endhighlight %}



