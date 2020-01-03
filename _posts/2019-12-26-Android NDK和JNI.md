---

layout:     post
title:      Android NDK和JNI
subtitle:   Android NDK和JNI
date:       2019-12-26
author:     skaleto
catalog: true
tags:
    - java，jni

---



# Android NDK和JNI



## 引言

近期项目需要在Android平板上跑一些底层算法，不可避免的会遇到需要集成Android NDK，封装底层so库的需求，由于之前完全没有接触过NDK相关知识，在这里做一个整理。



## Android NDK

### 简介

NDK（Native Develop Kit），是google提供的用于安卓平台上native开发的工具包，使用NDK可以方便地将native实现的C/C++代码集成到安卓app中，Android SDK和NDK的区别和联系可以从下面的图中看出。上层应用通过JNI调用native层封装的库文件，并通过NDK调用系统库完成整个过程。

![img](https://upload-images.jianshu.io/upload_images/5713484-36140dc3f1a04ca2.png?imageMogr2/auto-orient/strip|imageView2/2/w/657/format/webp)

使用NDK有如下的优势：

- 更好的可移植性，方便应用在不同平台上运行
- 更高的可复用性，封装好的动态库、静态库可以提供给多个应用使用
- 在计算密集型的应用中（例如游戏），使用native方法可以得到更好的性能



### 主要组件

- 库文件，动态库(.so【linux】、.dll【windows】)或静态库(.a【linux】、.lib【windows】)

```
动态库：编译时不直接整合到目标文件中，仅在运行时生效，对于一些算法库或功能库，一般都使用动态库，方便更新和升级
静态库：编译时直接整合到目标文件中，对于需要集成对外提供能力的情况下，使用静态库比较方便
```

- JNI，通过JNI连接java应用层和native层
- ABI(Application Binary Interface)，ABI定义了机器码和系统如何交互的逻辑，即不同CPU架构下二进制文件如何运行在相应系统，目前ABI支持32-bit ARM, AArch64, x86, and x86-64这四种



### 快速搭建NDK开发工程

有了Android Studio的帮助，搭建NDK开发工程实属简单

1. 开发工具下载，一般Android Studio下载后会默认进行SDK的下载，并且国内有了developer.android.google.cn网站，SDK的下载速度并不慢，那么我们只需要再下载配套的NDK就可以了。在SDK Manager中，找到SDK Tools，下载LLDB（native安卓应用调试工具）、NDK、CMake（是否需要取决于你native库的构建方式，用的比较多的就是CMake）

![image-20200103093611658](..\img\ndk&jni\tools.png)

2. 创建Android native C++工程

![image-20200103094450168](..\img\ndk&jni\new_project.png)

3. Done！Android Studio会默认给你生成样例代码，修改CMakeList可以编辑你的native库编译方式和依赖内容，native-lib为NDK中JNI的实现代码

![image-20200103104435625](..\img\ndk&jni\example.png)



NDK的介绍就到这里，其实它就是一个安卓平台上的native开发工具，没有太大的难度，接下来要介绍的就是实现native开发，将安卓层与native层连接起来的强大框架——JNI



## JNI（Java Native Interface）

![image-20200103111355785](..\img\ndk&jni\native_method.png)

Java编写的代码，只能运行在Java虚拟机中，这样的特性，使得Java代码可以运行在各种平台环境中，但是也带来了问题，不同平台的底层实现往往是不一样的，思考上图的场景，在我们需要调用创建文件的底层能力时，事实上操作系统本身也做了不同的事情，但是这些事情都通过了统一的标准，因此在上层看到，处理方式没有什么不同。那么Java代码是如何实现方便统一的调用底层能力呢，经过很长时间的发展，也有不少人提出了底层能力调用的java规范，但大多群雄割据，各有各的优劣，直到JNI的出现统一了标准。

![image-20200103112745456](..\img\ndk&jni\history.png)



JNI是从Java1.1开始提供的本地接口规范，使代码可以方便地在各个平台间移植。



### JNI基本实现

#### Step1. 生成java.h头文件

将native方法在某个java类中定义好，使用javac将该类编译为class文件

```
tips：
1. 如果你的java类中依赖了其他包中的类，需要将这些类作为javac的参数一起编译，否则编译不会通过
2. 如果编译过程中报错提示字符编码不支持，那么大概率你是在windows上编译的，需要用javac -encoding UTF8指定编译的字符集
```

得到class文件后，使用javah命令生成.h文件，该文件即为你需要在C++层进行实现的头文件

```
tips：
在使用javah命令时，不能在class文件的同目录进行生成，你需要返回到class包名的根目录，并将包名输入完整后才能正常生成（如下图）
```

![image-20200103115214188](..\img\ndk&jni\testjni.png)



#### Step2.  将.h文件集成到C++代码中

创建完成Android native C++项目后，会同时出现cpp目录和java目录，此时就可以把头文件放到cpp目录下进行集成了。

这里有一些tips，当你直接将头文件放入cpp目录下时，一般会报错提示找不到<jni.h>，那是因为IDE还没有把你的文件当成项目的一部分，你需要创建一个cpp文件来实现这个头文件，同时，将你的cpp文件加入到CMakeList文件中，使IDE在编译时将你的代码加入到项目中

![image-20200103140154480](..\img\ndk&jni\cmake.png)



#### Step3. 编译运行

做完上述步骤后，就可以进行编译运行了，编译之后，native代码会打包成so包（前提是native库的配置和上图一样是SHARED，代表使用的是动态库）



### JNI基本用法

我们来看一下生成的native方法的声明和实现

```c
//java层方法定义
public native void test();
//c++层头文件中的声明
JNIEXPORT void JNICALL Java_com_example_myapplication_TestJni_test
  (JNIEnv *, jobject);
//c++层方法实现
extern "C" JNIEXPORT void JNICALL
Java_com_example_myapplication_TestJni_test(JNIEnv *jniEnv, jobject obj) {
    
}
```

java层定义改方法为void，没有入参，但是发现在jni方法中，默认带入了jniEnv指针和一个jobject，这俩是什么呢

#### JNIEnv *

JNIEnv*指向了native方法中的函数表，所有需要通过jni操作java层的函数都在这个里面作了定义

![这里写图片描述](https://img-blog.csdn.net/2018042117553366)

![image-20200103142501603](E:\skaleto.github.io\img\ndk&jni\jniEnv.png)



#### jobject

传入的是当前native方法对应的对象实例，此对象实例提供给c++层，使c++层可以访问该对象实例中的成员变量和方法，如果native方法定义为static，那么此处的jobject就会变为jclass，即可以直接通过class来访问。

#### 自定义参数

如果我们的native方法传入了java中的一些自定义的对象，此时的jni函数会相应地多出几个对象来

```c
//java层方法定义
public native void test(Point p);
//c++层头文件中的声明
JNIEXPORT void JNICALL Java_com_example_myapplication_TestJni_test
  (JNIEnv *, jobject, jobject);
//c++层方法实现
extern "C" JNIEXPORT void JNICALL
Java_com_example_myapplication_TestJni_test(JNIEnv *jniEnv, jobject obj, jobject point) {
    
}
```

可以看到，我们向test方法中传入了自定义类实例Point，在native侧可以看到多出了一个jobject，但是他并不知道是何种类型的对象，如果需要访问Point中的某些变量，我们就需要通过反射来获取

```c++
extern "C" JNIEXPORT void JNICALL
Java_com_example_myapplication_TestJni_test(JNIEnv *jniEnv, jobject obj, jobject point) {
	//获得point对象的class
    jclass pointClass = env->GetObjectClass(point);
    //获得该class中的x_pixel成员变量的，类型为int
	jfieldID x_pixelFieldID = env->GetFieldID(pointClass, "x_pixel", "I");
    //获得该成员变量的值
	jint hFOV = env->GetIntField(point, x_pixelFieldID);   
}
```

#### 数据和方法类型签名

上面获得某个对象中的成员变量，通过反射获取，除此之外，还需要指定变量的类型签名，例如上面的“I”，代表变量为int型，具体有如下几种

- 基本数据类型

| Signature | Java    | Native   |
| :-------- | :------ | :------- |
| B         | byte    | jbyte    |
| C         | char    | jchar    |
| D         | double  | jdouble  |
| F         | float   | jfloat   |
| I         | int     | jint     |
| S         | short   | jshort   |
| J         | long    | jlong    |
| Z         | boolean | jboolean |
| V         | void    | void     |

- 数组类型

  在前面加个"["即可，例如"byte[]"对应的签名就是"[B"

- 其他类型

  "L+完整包名"，例如String类型对应签名为"Ljava/lang/String"

- 函数签名

  (入参类型签名)返回类型签名，例如"double foo(int[] i,String s)"对应签名为"([I;Ljava/lang/String;)B"



### JNI引用

- 全局引用

  作用域可在多线程中，并且需要手动释放

  ```c++
  jobject NewGlobalRef(JNIEnv *env, jobject object);
  void DeleteGlobalRef(JNIEnv *env, jobject globalRef);
  ```

- 局部引用

  局部引用的作用域仅在当前线程中，方法调用结束后会被自动释放，但在Android中局部引用表的默认上限是512，也就是说，在一个方法中，不要创建过多的局部引用，并且要及时使用jniEnv->DeleteLocalRef删除

  ```c++
  jobject NewLocalRef(JNIEnv *env, jobject object);
  void DeleteLocalRef(JNIEnv *env, jobject localRef);
  ```

- 全局弱引用

  作用域可在多线程中，在GC回收前有效，需要手动释放，因此全局弱引用随时可能失效，需要在使用前判断是否已经失效

  ```c++
  jweak NewWeakGlobalRef(JNIEnv *env, jobject object);
  void DeleteWeakGlobalRef(JNIEnv *env, jweak obj);
  ```



### JNI异常

#### 异常检查

一般在JNI调用中，可能会发生错误，抛出异常，但是本地代码不会停止，需要我们手动检查是否有异常并及时处理

```c++
//检查是否有异常发生，如果有，改函数会返回JNI_TRUE
env->ExceptionCheck()
//清除异常信息
env->ExceptionClear()
```

#### 抛出异常

如果发生了异常，但是需要上层来处理，那么就可以将异常抛出

```c++
//ExceptionOccurred会返回一个jthrowable对象
env->Throw(env->ExceptionOccurred())
```

如果我们需要抛出特定类型的异常，可以使用ThrowNew

```c++
jclass cls=env->FindClass("../../../CustomException")
env->ThrowNew(cls,msg)
```



### JNI多线程

对于下面这种情况，在每个jni方法中，jniEnv都仅对当前线程有效，那么如果我们需要在别的地方调用Jni方法怎么办呢？

![image-20200103163700239](..\img\ndk&jni\multi-thread.png)

能想到的第一个方式就是将jniEnv保存为全局变量，在需要调用的地方使用，但事实上jniEnv作用域仅在该jni方法中，方法结束后就被销毁了。所以我们需要将生成jniEnv的上层对象给保存下来。

```c++
JavaVM *g_javaVM;
//保存javaVM全局变量
env->GetJavaVM(&g_javaVM);
```

通过g_javaVM我们可以再创建一个JniEnv

```c++
JNIEnv *jniEnv = nullptr;
jint ret = g_javaVM->GetEnv((void **) &jniEnv, JNI_VERSION_1_6);
```

如果当前创建JniEnv的线程已经不是之前的线程，那么此时的ret结果将是JNI_EDETACHED，表明当前线程没有被链接到JVM，因此需要将线程链接起来

```c++
if (ret == JNI_EDETACHED) {
    if (g_javaVM->AttachCurrentThread(&jniEnv, nullptr) != JNI_OK) {
        return;
    }
}
```

相应的，在使用完该jniEnv后，需要将当前线程从JVM断开

```c++
g_javaVM->DetachCurrentThread()
```





## 后记

至此，NDK+JNI开发目前遇到的相关知识点都在文中整理了，后续遇到其他的坑会再补充进来