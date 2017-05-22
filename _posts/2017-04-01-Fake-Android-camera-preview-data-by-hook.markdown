---
layout: post
title:  "Fake Android camera preview data by hook(通过hook技术伪造Android摄像头前景数据)"
date:   2017-04-01 15:22:01 +0800
categories: Tech Android
---

3年前在CSDN上写过一篇文Android hook system API —— 修改系统相机preview data，当时是做了一个POC, 能够直接在4.0的root手机上通过安装一个app来篡改支付宝的扫描二维码数据，比如扫描A的支付账户，实际上扫描到我的支付账户。当时做完这个原型，博文只写了一半，因为有事情耽搁了就没有写完整。

具体原理和思路是利用linux的ptrace函数hook进Android的系统进程mediaservice的libcameraservice.so, 拿到底层的一份camera拷贝数据，并修改为我自己的二维码，并且不会影响系统的摄像头前景数据（因为只是修改给应用的拷贝数据）。

最近有这样一个需求，想用这种方法来破解一些公司的手机拍照打卡系统，使得使用者可以用一张照片来欺骗上层应用，从而不需要真正跑到公司现场去拍照打卡。于是翻出以前的博客和程序，稍微修改完善了一番，然而却发现并没有伪造成功。后来扫了一下系统源码，究其原因，是因为摄像头的拍照和扫描前景的底层程序走了两个不同的分支，拍照时系统进程并没有走进我的hook函数。最终，拍照的分支也没有找到好的点来hook，所以最后调研算是失败了。如果有大神能有更好的办法，可以评论里讨论研究下。

不过这次的调研也算完善了以前的hook扫描前景的程序，使得能够兼容至少2.3到4.4之间的所有版本，整理于此。

有兴趣往下读的网友，建议先读一下CSDN的旧文，大体思路还是一致的。

这次完善版本的hook点不再是libbinder的MemoryBase的构造函数，而直接是memcpy函数。原因是MemoryBase的实现依赖于智能指针，Android智能指针并没有在NDK中导出，我以前的实现是直接在4.0 source code的环境里编译的hook函数，这种实现方式兼容性很差，几乎只能兼容4.0的手机。其实memcpy是个更好的hook点，因为这是libc的函数，只要用ndk的低版本库进行编译，就能兼容以后所有版本。

```c
void CameraService::Client::copyFrameAndPostCopiedFrame(  
        int32_t msgType, const sp<ICameraClient>& client,  
        const sp<IMemoryHeap>& heap, size_t offset, size_t size,  
        camera_frame_metadata_t *metadata) {  
  
    sp<MemoryHeapBase> previewBuffer;  
  
    if (mPreviewBuffer == 0) {  
        mPreviewBuffer = new MemoryHeapBase(size, 0, NULL);  
    } else if (size > mPreviewBuffer->virtualSize()) {  
        mPreviewBuffer.clear();  
        mPreviewBuffer = new MemoryHeapBase(size, 0, NULL);  
    }  
    if (mPreviewBuffer == 0) {  
        LOGE("failed to allocate space for preview buffer");  
        mLock.unlock();  
        return;  
    }  
    previewBuffer = mPreviewBuffer;  
  
    //hook point
    memcpy(previewBuffer->base(), (uint8_t *)heap->base() + offset, size);  
  
    sp<MemoryBase> frame = new MemoryBase(previewBuffer, 0, size);  
    if (frame == 0) {  
        LOGE("failed to allocate space for frame callback");  
        mLock.unlock();  
        return;  
    }  
  
    mLock.unlock();  
    client->dataCallback(msgType, frame, metadata);  
}
```

```c
void *my_memcpy(void *_dst, const void *_src, unsigned len)
{
     LOGE("I hook here:%d",len);
     return memcpy(dst,src,len);
}
```
这里得到的camera数据是yuv420sp的，len等于w*h*1.5，关于yuv420sp格式以及如何提取yuv分量可以参考网上的做法，这类资料很多。

如果要对数据做图像处理，或者用自己预设的图像来替换原始数据，都需要知道图片的w和h。而我们只知道w*h*1.5，怎么办呢？

可以在java层做Hook之前，先获取到本机摄像头支持的所有分辨率组合，然后把这些数据写到本地的文件中。成功hook后，系统的camera服务进程会执行到我们的Hook点my_memcpy，在这里我们可以读取之前写的文件，遍历所有的分辨率组合，找到那个正好w*h*1.5 == len 的组合，即为此次前景数据的长和宽。

java应用层
```java
public void getCameraSupportedSize()
{
        Camera mCamera = Camera.open();

        Camera.Parameters parameters=mCamera.getParameters();

        preSize = parameters.getSupportedPreviewSizes();

        mCamera.stopPreview();
        mCamera.release();
        将获取到分辨率列表序列化写入某个文件中
}
```
Hook点

读文件，反序列化获取手机支持的分辨率列表
```java
for(int i=0;i<supportedListLength;i++)
{
       supported* s = supportedList[i];
       int _size = s->w * s->h * 3 / 2;
       if(_size == len)
       {
         previewWidth = s->w;
         previewHeight = s->h;
         break;
       }
}
```
关于如何Hook, 这是很古老的问题了。在不修改源码或者不篡改app的情况下，只能依赖于ptrace和root。然而4.4后Android内核都是selinux，禁止在别的进程里加载第三方的so，这就会使得ptrace没有效果。部分手机可以通过命令行setenforce 0 将selinux mode设置为passive从而使得这种方法依然能奏效。至少我现在手里的红米Note 4.4.4是可以用setenforce的，在往上5.0, 6.0, 7.0就没有试过了。

话又说回来，像欺骗打卡程序这种需求，完全可以自己去搞个二手的4.4以下的手机来安装公司打卡app。所以，一个看起来落伍的技术还有没有价值还是要看需求是怎样的。

ptrace hook的大体步骤和原理如下：

1.  ptrace_attach 目标进程，修改目标进程pc寄存器的值为0, 使得目标进程中断。

2. 寻找dlopen, dlsym, dlclose函数在目标进程的函数地址。

3. 修改目标进程sp寄存器的值，用来传参，传入我们的需要注入的so的路径，修改pc寄存器的值为dlopen的函数地址，ptace_con使得目标进程恢复运行，执行完成后r0寄存器即为dlopen的返回值，返回so的句柄。

4. 大体同步骤三，不过pc寄存器改为dlsym的函数地址，传参改为my_memcpy，这样就可以找到my_memcpy在目标进程的函数地址。

5. 找到目标进程中got表memcpy的链接地址，替换为我们的My_memcpy。
```c
int main(int argc, char *argv[]) {

    int pid;
    void *handle = NULL;
    long proc = 0;

    pid = atoi(argv[1]);
  
    ptrace_find_dlinfo(pid);

    handle = ptrace_dlopen(pid, "/system/lib/libhook.so",RTLD_NOW);

    proc = (long)ptrace_dlsym(pid, handle, "my_memcpy");

    replace_all_rels(pid, "memcpy", proc, sos);
    ptrace_detach(pid);
    exit(0);
    return 0;
}
```
核心代码很简单，主要是利用了别人写好的库,  这里贴出github里库的链接。

使用的时候要注意2个宏的定义，Android这个宏是肯定需要的，thumb这个宏不同的机器不一样，旧一点的手机可能指令集是用的thumb。后来我稍微修改了一下，删掉了thumb这个宏，通过pc指令的最后一位来动态判断是thumb还是arm。
```c
if (regs.ARM_pc& 1) {
       /* thumb */
       regs.ARM_pc = 0x11;
       regs.ARM_cpsr |=0x30;
} else {
       /* arm */
       regs.ARM_pc= 0;
}

```

最后附上github[项目地址]。

[项目地址]: https://github.com/vito11/CameraHook
