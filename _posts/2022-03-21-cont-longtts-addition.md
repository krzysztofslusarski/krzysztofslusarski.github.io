---
layout: default
title:  "[Java][Profiling][JVM] JVM thread state changes by example"
date:   2022-03-21 02:51:30 +0100
---

# [Java][Profiling][JVM] JVM thread state changes by example

This article is a continuation of [this one](https://krzysztofslusarski.github.io/2021/08/22/cont-longtts.html){:target="_blank"},
where we have studied following code of JVM sources: 

File ```io_util.c```:

```c
#define BUF_SIZE 8192

void
writeBytes(JNIEnv *env, jobject this, jbyteArray bytes,
           jint off, jint len, jboolean append, jfieldID fid)
{
    ...
    if (len == 0) {
        return;
    } else if (len > BUF_SIZE) {
        buf = malloc(len);
        if (buf == NULL) {
            JNU_ThrowOutOfMemoryError(env, NULL);
            return;
        }
    } else {
        buf = stackBuf;
    }
    (*env)->GetByteArrayRegion(env, bytes, off, len, (jbyte *)buf);
    ...
    if (buf != stackBuf) {
        free(buf);
    }
}
```

I also mentioned that there are **four** essential states of a single JVM thread:

> There are 4 essential states:
> 
> 
> 
> _thread_new         : Just started, but not executed init. code yet (most likely still in OS init code) 
> 
> _thread_in_native   : In native code. This is a safepoint region, since all oops will be in jobject handles
> 
> _thread_in_vm       : Executing in the vm
> 
> _thread_in_Java     : Executing either interpreted or compiled Java code (or could be in a stub)

However, there are some understatements.

* Is a whole ```writeBytes``` is done in ```_thread_in_vm```?
* Which call from the JVM code can cause a page fault?

We didn't see the answer for the second question in the previous articles since JVM was compiled **without debug symbols**.

## Our own JVM from sources

Let's try to compile our **own version** of JVM from sources **with debug symbols**. To do it we need to execute (if you're doing
it for the first time you may need to install some dependencies, check the 
[instruction](https://github.com/openjdk/jdk11u-dev/blob/master/doc/building.md) first):

```shell
git clone --depth 1 https://github.com/openjdk/jdk11u-dev.git
cd jdk11u-dev
bash configure --with-debug-level=slowdebug
make images
```

After a success compilation you should see:
```
Creating support/demos/image/jfc/J2Ddemo/J2Ddemo.jar
Creating support/classlist.jar
Creating images/jmods/jdk.jlink.jmod
Creating images/jmods/java.base.jmod
Creating jdk image
Stopping sjavac server
Finished building target 'images' in configuration 'linux-x86_64-normal-server-slowdebug'
```

Let's run our reproduction ```Test``` from previous article with our custom JVM.

```shell
./build/linux-x86_64-normal-server-slowdebug/jdk/bin/java -Xmx700M -XX:+AlwaysPreTouch -XX:+SafepointTimeout -XX:SafepointTimeoutDelay=100 -XX:+UnlockDiagnosticVMOptions -XX:+AbortVMOnSafepointTimeout Temp
```

This time our ```hs_err``` has much more details:

```
Current thread (0x00007f12b401e800):  JavaThread "main" [_thread_in_vm, id=3325, stack(0x00007f12bd06b000,0x00007f12bd16c000)]

Stack: [0x00007f12bd06b000,0x00007f12bd16c000],  sp=0x00007f12bd168148,  free space=1012k
Native frames: (J=compiled Java code, A=aot compiled Java code, j=interpreted, Vv=VM code, C=native code)
C  [libc.so.6+0x15baa1]  __memmove_ssse3_back+0x1b11
V  [libjvm.so+0x30bd40]  Copy::conjoint_jbytes(void const*, void*, unsigned long)+0x2b
V  [libjvm.so+0x30b6a8]  void AccessInternal::arraycopy_conjoint<signed char>(signed char*, signed char*, unsigned long)+0x2b
V  [libjvm.so+0xc03380]  EnableIf<(((!HasDecorator<64ul, 4ul>::value)&&(!(HasDecorator<64ul, 33554432ul>::value&&RawAccessBarrierArrayCopy::IsHeapWordSized<signed char>::value)))&&(!HasDecorator<64ul, 67108864ul>::value))&&(!HasDecorator<64ul, 134217728ul>::value), void>::type RawAccessBarrierArrayCopy::arraycopy<64ul, signed char>(arrayOopDesc*, unsigned long, signed char*, arrayOopDesc*, unsigned long, signed char*, unsigned long)+0x6d
V  [libjvm.so+0xc03218]  bool RawAccessBarrier<64ul>::arraycopy<signed char>(arrayOopDesc*, unsigned long, signed char*, arrayOopDesc*, unsigned long, signed char*, unsigned long)+0x48
V  [libjvm.so+0xc03119]  EnableIf<HasDecorator<2642000ul, 4096ul>::value&&AccessInternal::PreRuntimeDispatch::CanHardwireRaw<2642000ul>::value, bool>::type AccessInternal::PreRuntimeDispatch::arraycopy<2642000ul, signed char>(arrayOopDesc*, unsigned long, signed char*, arrayOopDesc*, unsigned long, signed char*, unsigned long)+0x48
V  [libjvm.so+0xc02fd2]  EnableIf<!HasDecorator<2637904ul, 4096ul>::value, bool>::type AccessInternal::PreRuntimeDispatch::arraycopy<2637904ul, signed char>(arrayOopDesc*, unsigned long, signed char*, arrayOopDesc*, unsigned long, signed char*, unsigned long)+0x59
V  [libjvm.so+0xc02e96]  bool AccessInternal::arraycopy_reduce_types<2637904ul, signed char>(arrayOopDesc*, unsigned long, signed char*, arrayOopDesc*, unsigned long, signed char*, unsigned long)+0x48
V  [libjvm.so+0xc02cf6]  bool AccessInternal::arraycopy<2621440ul, signed char>(arrayOopDesc*, unsigned long, signed char const*, arrayOopDesc*, unsigned long, signed char*, unsigned long)+0x50
V  [libjvm.so+0xc02a56]  void Access<2621440ul>::arraycopy<signed char>(arrayOopDesc*, unsigned long, signed char const*, arrayOopDesc*, unsigned long, signed char*, unsigned long)+0x4d
V  [libjvm.so+0xcc85aa]  void ArrayAccess<0ul>::arraycopy_to_native<signed char>(arrayOopDesc*, unsigned long, signed char*, unsigned long)+0x47
V  [libjvm.so+0xcbe07e]  jni_GetByteArrayRegion+0x2f4
C  [libjava.so+0x1e858]  writeBytes+0x15d
C  [libjava.so+0x10d86]  Java_java_io_FileOutputStream_writeBytes+0x5b
j  java.io.FileOutputStream.writeBytes([BIIZ)V+0 java.base
j  java.io.FileOutputStream.write([BII)V+16 java.base
j  java.io.BufferedOutputStream.write([BII)V+20 java.base
j  java.io.FilterOutputStream.write([B)V+5 java.base
j  Temp.main([Ljava/lang/String;)V+58
v  ~StubRoutines::call_stub
V  [libjvm.so+0xbebc5d]  JavaCalls::call_helper(JavaValue*, methodHandle const&, JavaCallArguments*, Thread*)+0x647
V  [libjvm.so+0x10a88dc]  os::os_exception_wrapper(void (*)(JavaValue*, methodHandle const&, JavaCallArguments*, Thread*), JavaValue*, methodHandle const&, JavaCallArguments*, Thread*)+0x32
V  [libjvm.so+0xbeb614]  JavaCalls::call(JavaValue*, methodHandle const&, JavaCallArguments*, Thread*)+0xa8
V  [libjvm.so+0xc96122]  jni_invoke_static(JNIEnv_*, JavaValue*, _jobject*, JNICallType, _jmethodID*, JNI_ArgumentPusher*, Thread*)+0x1f0
V  [libjvm.so+0xcad204]  jni_CallStaticVoidMethod+0x36a
C  [libjli.so+0x5062]  JavaMain+0xcf7
```

Now we can see that the ```writeBytes``` executes ```jni_GetByteArrayRegion``` which in the end calls ```__memmove_ssse3_back```.
So it's not the ```malloc```, that make sense. The ```malloc``` returns a pointer returned by the OS. The page fault happens when you
first time **touch the page**.

## Thread state changes

Now let's try to understand how the thread state changes over time. Is a whole ```writeBytes``` executed in
```_thread_in_vm```? Or maybe we go from ```_thread_in_Java``` to ``` _thread_in_native``` and then we hit ```_thread_in_vm```?

We will use such a program to track it:

```java
import java.io.BufferedOutputStream;
import java.io.FileOutputStream;

class Temp {
    public static void main(String[] args) throws Exception {
        byte[] arr = new byte[300 * 1024 * 1024];
        for (int i = 0; i < arr.length; i++) {
            arr[i] = (byte) i;
        }

        Thread.sleep(3000);
        try (BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream("/home/<username>/tmp.tmp"))) {
            bos.write(arr);
        }
    }
}
```

Let's customize our JVM a bit and add custom debug ```printf```. 

File ```io_util.c```:

```c
#define BUF_SIZE 8192

void
writeBytes(JNIEnv *env, jobject this, jbyteArray bytes,
           jint off, jint len, jboolean append, jfieldID fid)
{
    printf( "WriteBytes start\n" );
    ...
    if (len == 0) {
        return;
    } else if (len > BUF_SIZE) {
        printf( "Malloc start\n" );
        buf = malloc(len);
        printf( "Malloc ended\n" );
        if (buf == NULL) {
            JNU_ThrowOutOfMemoryError(env, NULL);
            return;
        }
    } else {
        buf = stackBuf;
    }
    printf( "GetByteArrayRegion start\n" );
    (*env)->GetByteArrayRegion(env, bytes, off, len, (jbyte *)buf);
    printf( "GetByteArrayRegion ended\n" );
    ...
    if (buf != stackBuf) {
        free(buf);
    }
    printf( "WriteBytes end\n" );
}
```

And let's modify ```thread.hpp```:

```c
  JavaThreadState thread_state() const           { return _thread_state; }
  void set_thread_state(JavaThreadState s)       {
    printf( "New thread state (%ld) %d -> %d: \n", os::current_thread_id(),  _thread_state, s );
    assert(current_or_null() == NULL || current_or_null() == this,
           "state change should only be called by the current thread");
    _thread_state = s;
  }
```

The output:

```
WriteBytes start
New thread state (346625) 4 -> 5: 
New thread state (346625) 5 -> 6: 
New thread state (346625) 6 -> 7: 
New thread state (346625) 7 -> 4: 
Malloc start
Malloc ended
GetByteArrayRegion start
New thread state (346625) 4 -> 5: 
New thread state (346625) 5 -> 6: 
New thread state (346625) 6 -> 7: 
New thread state (346625) 7 -> 4: 
GetByteArrayRegion ended
New thread state (346625) 4 -> 5: 
New thread state (346625) 5 -> 6: 
New thread state (346625) 6 -> 7: 
New thread state (346625) 7 -> 4: 
New thread state (346625) 4 -> 5: 
New thread state (346625) 5 -> 6: 
New thread state (346625) 6 -> 7: 
New thread state (346625) 7 -> 4: 
New thread state (346625) 4 -> 5: 
New thread state (346625) 5 -> 6: 
New thread state (346625) 6 -> 7: 
New thread state (346625) 7 -> 4: 
New thread state (346625) 4 -> 5: 
New thread state (346625) 5 -> 6: 
New thread state (346625) 6 -> 7: 
New thread state (346625) 7 -> 4: 
WriteBytes end
```

Now we have a bunch of **magic numbers**, the dictionary for them is in the ```globalDefinitions.hpp```:

```c
enum JavaThreadState {
  _thread_uninitialized     =  0, // should never happen (missing initialization)
  _thread_new               =  2, // just starting up, i.e., in process of being initialized
  _thread_new_trans         =  3, // corresponding transition state (not used, included for completness)
  _thread_in_native         =  4, // running in native code
  _thread_in_native_trans   =  5, // corresponding transition state
  _thread_in_vm             =  6, // running in VM
  _thread_in_vm_trans       =  7, // corresponding transition state
  _thread_in_Java           =  8, // running in Java or in stub code
  _thread_in_Java_trans     =  9, // corresponding transition state (not used, included for completness)
  _thread_blocked           = 10, // blocked in vm
  _thread_blocked_trans     = 11, // corresponding transition state
  _thread_max_state         = 12  // maximum thread state+1 - used for statistics allocation
};
```

Our thread state path looks like this (I skipped the ```_trans``` states):

```
_thread_in_native
WriteBytes start 
_thread_in_vm  
_thread_in_native 
Malloc start
Malloc ended
GetByteArrayRegion start 
_thread_in_vm  
_thread_in_native 
GetByteArrayRegion ended 
_thread_in_vm  
_thread_in_native  
_thread_in_vm  
_thread_in_native  
_thread_in_vm  
_thread_in_native  
_thread_in_vm  
_thread_in_native 
WriteBytes end
```

So the **answer** is that the ```writeBytes``` executes code in ```_thread_in_native``` and switches to ```_thread_in_vm``` 
when needed.