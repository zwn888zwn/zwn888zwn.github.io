---
layout    : post
title     : "JAVA使用JNI调用C++代码"
date      : 2020-04-12
lastupdate: 2020-04-12
categories: java
---

  1，编写一个本地方法hello（），方法具体由C++代码实现

![在这里插入图片描述](/assets/img/2020-04-12-java-cni-c-zh/20200412202831913.png)



2，javah生成方法头文件，也可由idea快速生成，具体配置请参考链接

https://blog.csdn.net/ldkjsdty/article/details/103104638
![在这里插入图片描述](/assets/img/2020-04-12-java-cni-c-zh/2.png)
生成的头文件：
![在这里插入图片描述](/assets/img/2020-04-12-java-cni-c-zh/3.png)



 3，打开VS2019，创建DLL项目，并在属性页设置JDK头文件和刚刚我们生成头文件目录
![在这里插入图片描述](/assets/img/2020-04-12-java-cni-c-zh/4.png)
注意，配置是release，平台选X86
![在这里插入图片描述](/assets/img/2020-04-12-java-cni-c-zh/5.png)
IDE页面环境也选择下
![在这里插入图片描述](/assets/img/2020-04-12-java-cni-c-zh/20200412203707554.png)



4，把我们生成的头文件复制进“头文件”文件夹下
![在这里插入图片描述](/assets/img/2020-04-12-java-cni-c-zh/6.png)



5，打开cpp，分别添加
#include "JNITest_JNIHelloWorld.h"
#include "jni.h"
#include "jni_md.h" 代码，并且实现头文件中的方法

```cpp
#include <iostream>
#include "JNITest_JNIHelloWorld.h"
#include "jni.h"
#include "jni_md.h"

using namespace std;


JNIEXPORT void JNICALL Java_JNITest_JNIHelloWorld_hello
(JNIEnv *, jclass) {
	cout << "hello world" << endl;
}
```



6，生成解决方案
DLL文件生成到如下目录
![在这里插入图片描述](/assets/img/2020-04-12-java-cni-c-zh/20200412204119331.png)



7，IDEA配置库路径

-Djava.library.path=D:\cProject\JINTest\Dll1\x64\Release
![在这里插入图片描述](/assets/img/2020-04-12-java-cni-c-zh/7.png)



8，java代码，System.loadLibrary("Dll1");加载库文件

```java
package JNITest;

public class JNIHelloWorld {
    public static native void hello();

    static {
        System.loadLibrary("Dll1");
    }

    public static void main(String[] args) {
        hello();
    }
}
```


9，运行成功！
![在这里插入图片描述](/assets/img/2020-04-12-java-cni-c-zh/8.png)



## 参考链接：

https://blog.csdn.net/ldkjsdty/article/details/103104638
https://blog.csdn.net/qq_28875681/article/details/69912943
https://blog.csdn.net/qq_22899021/article/details/80798234