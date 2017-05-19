---
layout: post
title:  "Android studio 2.2 自动编译打包NDK shared library 和 executable 二进制文件到APK"
date:   2017-05-10 13:38:01 +0800
categories: jekyll update
---
最近用Android studio 2.2写NDK executable时发现IDE并没有将编译生成的executable二进制文件自动打包进APK，调研一番后，整理于此。

首先，右键侧边栏的app，添加assets目录。

在Android.mk文件上，加上TARGET_OUT这一项，指向assets的文件路径。

```make
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

LOCAL_SRC_FILES:= \
    inject.c 


LOCAL_C_INCLUDES := $(LOCAL_PATH)
LOCAL_MODULE:= injector
LOCAL_LDLIBS :=-llog
TARGET_OUT =./src/main/assets
LOCAL_CFLAGS := -DANDROID

include $(BUILD_EXECUTABLE)


include $(CLEAR_VARS)


sources := \
    hook.cc


LOCAL_SRC_FILES := $(sources)

LOCAL_C_INCLUDES := $(LOCAL_PATH)

LOCAL_MODULE:= hook
LOCAL_LDLIBS :=-llog
TARGET_OUT =./src/main/assets
include $(BUILD_SHARED_LIBRARY)
```

同时修改build.gradle 添加externalNativeBuild target, 示例如下。

```gradle
apply plugin: 'com.android.application'

android {
    compileSdkVersion 25
    buildToolsVersion "25.0.2"
    defaultConfig {
        applicationId "com.vito.research.camerahook"
        minSdkVersion 10
        targetSdkVersion 25
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
        ndk {
            abiFilters 'armeabi'
        }
        externalNativeBuild {
            ndkBuild {
                targets 'injector', 'hook'
            }
        }
    }
    externalNativeBuild {
        ndkBuild {
            path file("src\\main\\java\\jni\\Android.mk")
        }
    }
}
```
defaultConfig里的abiFilters是设置目标编译的平台，不设置的话，Android studio会默认全部平台都编译一遍，很慢。defaultConfig里的externalNativeBuild target是对应Android.mk里定义的module，defaultConfig之外的externalNativeBuild  path是指定Android.mk的文件路径。

这样编译完的所有二进制文件会自动打包进apk的assets中。

然后，你可以在java代码中吧这些文件从assets中读写到手机文件系统里，再通过命令行来调用。

```java
private void copyDataToExePath(String srcFileName, String strOutFileName) throws IOException {
    InputStream myInput;
    OutputStream myOutput = new FileOutputStream(strOutFileName);
    myInput = getAssets().open(srcFileName);
    byte[] buffer = new byte[1024];
    int length = myInput.read(buffer);
    while (length > 0) {
        myOutput.write(buffer, 0, length);
        length = myInput.read(buffer);
    }
    myOutput.flush();
    myInput.close();
    myOutput.close();
}

private String runLocalUserCommand(String command) {
    String result = "";
    try {
        Process p = Runtime.getRuntime().exec("ps");
        DataInputStream inputStream = new DataInputStream(p.getInputStream());
        OutputStream outputStream = p.getOutputStream();

        DataOutputStream dataOutputStream = new DataOutputStream(outputStream);
        dataOutputStream.writeBytes(command + "\n");
        dataOutputStream.writeBytes("exit\n");
        dataOutputStream.flush();
        p.waitFor();

        byte[] buffer = new byte[1024];
        while (inputStream.read(buffer) > 0) {
            String s = new String(buffer);
            result = result + s;
        }
        dataOutputStream.close();
        outputStream.close();
    } catch (Throwable t) {
        t.printStackTrace();
    }
    return result;
}
copyDataToExePath("test", "data/data"+getPackageName()+"/"+"test");
runLocalUserCommand("data/data"+getPackageName()+"/"+"test");
```

这样的一个android executable的编译运行环境就搭好了。
