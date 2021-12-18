## 第一章

### 介绍

JNI是原生程序接口。它允许运行在虚拟机内部的Java代码与使用C,C++或者汇编语言编写的应用程序和库进行交互。

JNI不会对底层Java VM的实现产生任何限制。Java VM制造商能够支持JNI而不对VM的其他部分产生任何影响。程序员可以编写原生应用程序或库的一个版本，并且希望它能够和所有支持JNI的Java VM一起工作。

### JNI预览

通过JNI编程，你能够使用原生方法创建、检查和更新Java对象，调用Java方法，捕获和抛出异常，加载类并且获取类信息，执行运行时类型检查。

你可以使用JNI的Invocation API将任意的原生应用程序嵌入Java VM。这使得程序员很容易为现有的应用程序支持Java而并不需要链接VM源代码。

## 第二章

### JNI接口函数和指针

原生代码通过调用JNI函数来访问Java VM功能。通过接口指针来访问JNI函数。接口指针是指向指针的指针。这个指针指向一个指针数组，其中每个都指向一个接口函数。每个接口函数都位于数组中预定义的偏移位置。

![](./images/designa.gif)

JNI接口指针仅在当前线程中可用。因此，原生方法不能将接口指针从一个线程传递到另一个线程。实现JNI的VM可以在JNI接口指针指向的区域中分配和存储线程本地数据。

原生方法接收JNI接口指针作为参数。当从相同的Java线程中多次调用原生方法时，VM保证对原生方法传递相同的接口指针。然而一个原生方法能够从不同的Java线程中被调用，因此可能接收不同的JNI接口指针。

### 编译、加载和链接原生方法

使用System.loadLibrary方法来加载原生方法。下面的例子中，类初始化方法加载了一个特定平台的原生库，其中定义了原生方法f

``` java
package pkg;  

class Cls { 

     native double f(int i, String s); 

     static { 
         System.loadLibrary(“pkg_Cls”); 
     } 

} 
```

System.loadLibrary的参数是程序员任意选择的一个库名。系统遵循一种标准的但特定于平台的方法将库名转换成原生库名。例如Solaris系统将名字pkg_Cls转换为libpkg_Cls.so，而Win32系统将名字pkg_Cls转换为pkg_Cls.dll

程序员可以使用一个库来保存任意数量类所需要的所有本地方法，只要使用相同的类加载器来加载这些类。VM内部为每个类加载器维护一个原生库加载列表。VM制造商应该选择本地库名称，以减少名称冲突的可能性。

如果底层操作系统不支持动态链接，VM必须预链接所有本地方法。在这种情况下，VM完成System.loadLibrary调用而不实际加载库。

程序员也可以调用JNI函数RegisterNatives来注册与类相关联的原生方法。RegisterNatives函数对于静态链接函数特别有用。

### 解析原生方法名

原生方法名由以下组件连接而成：

* Java_前缀
* 完全的类名
* 下划线分隔符_
* 方法名
* 对于重载的本地方法，在两个下划线__后面加上参数签名

VM检查方法名是否与驻留在原生库中的方法相匹配。VM首先查找短名称，没有参数签名的名称。然后查找长名称，带有参数签名的名称。只有当原生方法被另一个原生方法重载时，程序员才需要使用长名称。然而如果原生方法与非原生方法具有相同的名称，这是没有问题的。非原生方法不会驻留在原生库中。

在下面的例子中，原生方法g不必使用长名称进行链接，因为另一个方法g不是原生方法，因此不在原生库中。

``` java
class Cls1 { 

  int g(int i); 

  native int g(double d); 

} 
```

我们采用一个简单的名称修改方案来确保所有的Unicode字符都转换为有效的C函数名。我们使用下划线字段_来替换完全类名中的斜杠/。由于名称和类型描述从不以数字开头，我们可以使用\_0...\_9来表示转义字符。

| 转义字符 |                             表示                             |
| :------: | :----------------------------------------------------------: |
|  _0XXXX  | Unicode字符XXXX，使用小写字符表示非ASC码Unicode字符，例如_0abcd |
|    _1    |                            字符_                             |
|    _2    |                            字符;                             |
|    _3    |                            字符[                             |

在给定的平台上，本地方法和接口API都遵循标准库调用约定。例如UNIX系统使用C调用约定，Win32系统使用__stdcall

### 本地方法参数

JNI接口指针是本地方法的第一个参数。JNI接口指针的类型是JNIEnv。第二个参数取决于本地方法是静态的还是非静态的。非静态本地方法的第二个参数是对对象的引用，静态本地方法的第二个参数是对Java类的引用。

其余参数对应于普通Java方法参数。本地方法调用通过返回值将结果传递回调用程序。

下面的例子演示了使用C函数去实现原生方法f

``` java
package pkg;  

class Cls { 

     native double f(int i, String s); 

     ... 

} 
```

带有长名称Java_pkg_Cls_f_ILjava_lang_String_2的C函数实现本地方法f

``` c
jdouble Java_pkg_Cls_f__ILjava_lang_String_2 (
     JNIEnv *env,        /* interface pointer */
     jobject obj,        /* "this" pointer */
     jint i,             /* argument #1 */
     jstring s)          /* argument #2 */
{
     /* Obtain a C-copy of the Java string */
     const char *str = (*env)->GetStringUTFChars(env, s, 0);

     /* process the string */
     ...

     /* Now we are done with str */
     (*env)->ReleaseStringUTFChars(env, s, str);

     return ...
}
```

注意，我们总是使用接口指针env操作Java对象。下面的例子是C++版本:

``` c++
extern "C" /* specify the C calling convention */  

jdouble Java_pkg_Cls_f__ILjava_lang_String_2 ( 

     JNIEnv *env,        /* interface pointer */ 

     jobject obj,        /* "this" pointer */ 

     jint i,             /* argument #1 */ 

     jstring s)          /* argument #2 */ 

{ 

     const char *str = env->GetStringUTFChars(s, 0); 

     ... 

     env->ReleaseStringUTFChars(s, str); 

     return ... 

} 

```

在C++中额外的间接层和接口指针参数从源代码中消失了。然而其底层机制和C完全相同。在C++中JNI函数被定义为内联成员函数，可以扩展为C对应函数。

### 引用Java对象

基本类型比如整数、字符等在Java和原生代码之间被复制。另一方面，Java对象是通过引用传递的。VM必须持续跟踪所有被传递到原生代码的对象，以便垃圾回收器不会释放这些对象。反之，原生代码必须有一种方式通知VM它不在需要这些对象。此外，垃圾回收器必须能够移动原生代码引用的对象。

### 全局和本地引用

JNI将原生代码使用的对象引用分为两类：本地引用和全局引用。本地引用在原生方法调用期间有效，并且在原生方法返回后自动释放。全局引用在明确释放之前保持有效。

传递给原生方法的对象作为本地引用。JNI函数返回的所有Java对象都是本地引用。JNI允许程序员通过本地引用创建全局引用。期望Java对象的JNI函数同时接受全局和本地引用。原生方法可以返回对于VM的本地或者全局引用作为其结果。

通常，程序员应该在原生方法返回后依靠VM释放所有本地引用。然而有时程序员应该明确释放本地引用。比如考虑以下情况：

* 原生方法访问大型Java对象，从而创建对Java对象的本地引用。原生方法在返回调用者之前执行额外的计算。对大型Java对象的本地引用将防止对象被垃圾回收，即使该对象在剩余的计算中不再使用。
* 原生方法会创建大量的本地引用，尽管并不是所有的本地引用都同时使用。由于VM需要一定的空间来持续跟踪本地引用，创建太多本地引用可能导致系统内存溢出。例如，本地方法循环遍历一个大的对象数组，获取元素作为本地引用，并在每次迭代中对一个元素进行操作。在每次迭代之后，程序员不再需要对数组元素的本地引用。

JNI允许程序员在原生方法中的任何一点手动删除本地引用。为了确保程序员能够手动释放本地引用，JNI函数不允许创建额外的本地引用，除了它们作为返回结果的引用。

本地引用仅在创建它们的线程中有效。原生代码不能将本地引用从一个线程传递到另一个线程。

### 实现本地引用

为了实现本地引用，Java VM为控制从Java到原生方法的每次转换创建一个注册表。注册表将不可移动的本地引用映射为Java对象，并防止对象被垃圾回收。所有传递给原生方法的Java对象(包括作为JNI函数调用返回结果的那些对象)都被自动添加到注册表中。注册表在原生方法返回后被删除，它的所有内容被垃圾回收。

有不同的方法来实现注册表，例如使用表、链表或者散列表。虽然引用计数可以用来避免注册表中的重复内容，但是JNI实现并没有义务检测和折叠重复内容。

注意，仅通过扫描原生堆栈不能准确的实现本地引用。原生代码可以将本地引用保存在全局或者堆数据结构中。

### 访问Java对象

JNI为全局和本地引用提供了一组丰富的访问函数。这意味着无论VM内部如何表示Java对象，相同的原生方法实现都可以工作。这是JNI能够被各种VM实现支持的一个重要原因。

通过引用使用访问函数的开销要高于直接访问C数据结构。在大多数情况下，我们相信Java程序员使用原生方法来执行一些重要任务，这些任务掩盖了接口的开销。

### 访问原始数组

对于包含许多基本数据类型的大型Java对象，这种开销是不能接受的。考虑用于执行向量和矩阵计算的原生方法。遍历Java数组并使用函数调用获取每个元素的效率非常低。

一种解决方案引入了固定的概念，以便原生方法可以要求VM固定数组的内容。原生方法然后接收一个指向元素的指针。然而这种方法有两个条件:

* 垃圾回收必须支持固定
* VM必须在内存中连续存储原始数组。虽然这是大多数原始数组最合理的实现，布尔数组可以通过打包或解包实现。因此依赖于布尔数组精确布局的原生代码是不可移植的。

我们采用折中方案克服了上面的两个问题。

首先我们提供了一组函数在Java数组和原生内存缓冲区之间复制基本数组元素。如果原生方法只需要访问一个大数组中的少量元素则使用这些函数。

其次程序员可以使用另一组函数来获取数组元素的固定版本。请记住这些函数可能需要Java VM执行存储分配和复制。这些函数实际上是否复制数组取决于VM的实现:

* 如果垃圾回收支持固定，并且数组的存储与原生方法所期望的相同则不需要复制。
* 否则数组将被复制到一个不可移动的内存块(比如C堆中)，并执行必要的格式转换。返回一个副本的指针。

最后接口提供了通知VM原生代码不再需要访问数组元素的函数。当调用这些函数时，系统将解除数组的固定，或者将原始数组与其不可移动的副本进行协调并释放副本。

我们提供的方法更加灵活。垃圾回收算法可以对每个给定数组的复制或者固定做出单独的决定。比如，垃圾回收可以复制小对象，但固定大对象。

JNI实现必须确保在多个线程中运行的原生方法能够同时访问同一个数组。比如，JNI可以为每个固定的数组保留一个内部计数器，这样一个线程就不会解除另一个线程固定的数组。注意，JNI不需要为原生方法的独占访问锁定原始数组。同时从不同的线程更新Java数组会导致不确定的结果。

### 访问属性和方法

JNI允许原生代码访问属性和调用Java对象方法。JNI通过符号名和类型签名来标识方法和属性。两步法从名称和签名中计算出定位属性或方法的成本。比如，调用cls类中的方法f时，原生代码首先获取一个方法ID:

``` c
jmethodID mid =      env->GetMethodID(cls, “f”, “(ILjava/lang/String;)D”); 
```

然后，原生代码可以重复使用方法ID而不需要执行方法查找

``` c
jdouble result = env->CallDoubleMethod(obj, mid, 10, str); 
```

属性或方法ID不会阻止VM卸载从其生成ID的类。类被卸载之后属性或方法ID将不可用。因此，如果打算在一段时间内使用属性或者方法ID，原生代码必须确保:

* 保持对底层类的引用
* 或者重新计算属性或方法ID

JNI不会对属性和方法ID如何在内部实现施加任何限制。

### 报告编程错误

JNI不检查编程错误，比如传空指针或者非法参数类型。非法参数类型包括使用普通Java对象而不是Java类对象。JNI不检查这些编程错误的原因如下:

* 强制JNI函数检查所有可能的错误条件会降低正常原生方法的性能
* 在很多情况下，没有足够的运行时类型信息来执行这样的检查

大多数C库函数不能防止编程错误。比如，当printf函数接收到一个无效地址时通常会引发运行时错误，而不是返回错误代码。强制C库函数检查所有可能的错误条件可能导致这样的检查在用户代码中和库中重复进行。

程序员不能向JNI函数传递非法指针或错误类型参数。这样会导致不可预期的后果，包括系统状态损坏或VM崩溃。

### Java异常

JNI允许原生方法引发任意Java异常。原生代码也能处理未处理的Java异常。未处理的Java异常被传递回VM。

### 异常和错误代码

某些JNI函数使用Java异常机制来报告错误条件。在很多情况下，JNI函数通过返回错误代码并且抛出Java异常来报告错误条件。错误代码通常是一个特殊的返回值(比如NULL)，它超出了正常返回值的范围。因此程序员可以:

* 快速检查最后一个JNI调用的返回值，已确定是否发生了错误
* 调用ExceptionOccurred函数以获取包含更详细描述错误条件的异常对象

在两种情况下，程序员需要检查异常而不能首先检查错误代码:

* 调用Java方法的JNI函数返回Java方法的结果。程序员必须调用ExceptionOccurred来检查Java方法执行期间可能发生的异常
* 一些JNI数组访问函数不返回错误代码，但可能抛出ArrayIndexOutOfBoundsException或ArrayStoreException异常

在所有其他情况下，非错误返回值保证没有抛出异常。

### 异步异常

在多线程情况下，当前线程之外的其他线程可能抛出异步异常。异步异常不会立即影响当前线程中原生代码的执行，直到:

* 原生代码调用可能引发同步异常的JNI函数
* 原生代码使用ExceptionOccurred明确的检查同步和异步异常

注意，只有那些可能引发同步异常的JNI函数才会检查异步异常

原生方法应该在必要的地方插入ExceptionOccurred检查(比如在没有其他异常检查的循环中)，以确保当前线程在合理的时间内响应异步异常。

### 异常处理

在原生代码中有两种处理异常的方法:

* 原生方法可以选择立即返回，从而使在初始化原生方法调用的Java代码中抛出异常
* 原生代码可以通过调用ExceptionClear清除异常，然后执行自己的异常处理代码

在异常出现之后，原生代码必须在调用其他JNI函数之前首先清除异常。当存在待处理的异常时，可以安全调用的JNI函数包括:

``` c
  ExceptionOccurred()
  ExceptionDescribe()
  ExceptionClear()
  ExceptionCheck()
  ReleaseStringChars()
  ReleaseStringUTFChars()
  ReleaseStringCritical()
  Release<Type>ArrayElements()
  ReleasePrimitiveArrayCritical()
  DeleteLocalRef()
  DeleteGlobalRef()
  DeleteWeakGlobalRef()
  MonitorExit()
  PushLocalFrame()
  PopLocalFrame()
```

## 第三章

### 基本类型

下面的表格描述Java基本类型和对应的依赖于机器的原生类型

| Java类型 | 原生类型 | 描述       |
| -------- | -------- | ---------- |
| boolean  | jboolean | 无符号8位  |
| byte     | jbyte    | 有符号8位  |
| char     | jchar    | 无符号16位 |
| short    | jshort   | 有符号16位 |
| int      | jint     | 有符号32位 |
| long     | jlong    | 有符号64位 |
| float    | jfloat   | 32位       |
| double   | jdouble  | 64位       |
| void     | void     |            |

为了方便起见提供了以下定义

``` c
#define JNI_FALSE  0 
#define JNI_TRUE   1 
```

jsize整数类型用于描述基本的索引和大小

``` c
typedef jint jsize; 
```

### 引用类型

JNI包含很多对应于不同Java对象类型的引用类型。

![](./images/types4.gif)

在C中，所有其他JNI引用类型都被定义为与jobject相同。比如:

``` c
typedef jobject jclass; 
```

在C++中，JNI引入了一组虚拟类来表示子类型关系。比如:

``` c++
class _jobject {}; 
class _jclass : public _jobject {}; 
... 
typedef _jobject *jobject; 
typedef _jclass *jclass; 
```

### 属性和方法ID

方法和属性ID是常规的C指针类型

``` c
struct _jfieldID;              /* opaque structure */ 
typedef struct _jfieldID *jfieldID;   /* field IDs */ 
 
struct _jmethodID;              /* opaque structure */ 
typedef struct _jmethodID *jmethodID; /* method IDs */ 
```

### 值类型

jvalue联合类型被用作参数数组中的元素类型。

``` c
typedef union jvalue { 
    jboolean z; 
    jbyte    b; 
    jchar    c; 
    jshort   s; 
    jint     i; 
    jlong    j; 
    jfloat   f; 
    jdouble  d; 
    jobject  l; 
} jvalue; 
```

### 类型签名

JNI使用Java VM的类型签名表示。

| 类型签名                  | Java类型 |
| ------------------------- | -------- |
| Z                         | boolean  |
| B                         | byte     |
| C                         | char     |
| S                         | short    |
| I                         | int      |
| J                         | long     |
| F                         | float    |
| D                         | double   |
| L fully-qualified-class ; | 完整类名 |
| [ type                    | 类型数组 |
| ( arg-types ) ret-type    | 方法类型 |

比如，Java方法:

``` java
long f (int n, String s, int[] arr); 
```

对应下面的类型签名:

``` c
(ILjava/lang/String;[I)J 
```

### 改进的UTF-8字符串

JNI使用改进的UTF-8字符串来表示各种字符串类型。改进的UTF-8字符串与Java VM使用的字符串相同。对改进的UTF-8字符串进行编码，以便仅包含非空ASC码字符的字符序列只能用每个字符一个字节表示，但是所有Unicode字符都能被表示。

这种格式与标准UTF-8格式之间有两个不同。首先，使用双字节格式而不是单字节格式对空字符进行编码。这意味着改进的UTF-8字符串永远不会引入空值。其次，只使用标准UTF-8的一字节、二字节和三字节格式。Java VM不识别标准UTF-8的四字节格式，它使用自己的两倍三字节格式代替。

## 第四章

本章作为JNI函数的参考部分，提供了所有JNI函数的完整列表。

注意，"必须"一词用来描述对JNI程序员的限制。比如，当你看到某个JNI函数必须接收一个非空对象时，你有责任确保没有将空传递给JNI函数。因此，JNI实现不需要在该JNI函数中执行空指针检查。

### 接口函数表

每个函数都可以通过JNIEnv参数以固定的偏移量访问。JNIEnv类型是一个指向存储所有JNI函数指针结构体的指针。定义如下:

``` c
typedef const struct JNINativeInterface *JNIEnv; 
```

VM初始化函数表。注意，前三个条目是为了将来与COM兼容而保留的。此外，我们在函数表的开始处保留了许多额外的空条目，以便将来与类相关的JNI操作可以添加在FindClass之后，而不是在表的末尾。

注意，函数表可以在所有JNI接口指针之间共享。

``` c
const struct JNINativeInterface ... = {

    NULL,
    NULL,
    NULL,
    NULL,
    GetVersion,

    DefineClass,
    FindClass,

    FromReflectedMethod,
    FromReflectedField,
    ToReflectedMethod,

    GetSuperclass,
    IsAssignableFrom,

    ToReflectedField,

    Throw,
    ThrowNew,
    ExceptionOccurred,
    ExceptionDescribe,
    ExceptionClear,
    FatalError,

    PushLocalFrame,
    PopLocalFrame,

    NewGlobalRef,
    DeleteGlobalRef,
    DeleteLocalRef,
    IsSameObject,
    NewLocalRef,
    EnsureLocalCapacity,

    AllocObject,
    NewObject,
    NewObjectV,
    NewObjectA,

    GetObjectClass,
    IsInstanceOf,

    GetMethodID,

    CallObjectMethod,
    CallObjectMethodV,
    CallObjectMethodA,
    CallBooleanMethod,
    CallBooleanMethodV,
    CallBooleanMethodA,
    CallByteMethod,
    CallByteMethodV,
    CallByteMethodA,
    CallCharMethod,
    CallCharMethodV,
    CallCharMethodA,
    CallShortMethod,
    CallShortMethodV,
    CallShortMethodA,
    CallIntMethod,
    CallIntMethodV,
    CallIntMethodA,
    CallLongMethod,
    CallLongMethodV,
    CallLongMethodA,
    CallFloatMethod,
    CallFloatMethodV,
    CallFloatMethodA,
    CallDoubleMethod,
    CallDoubleMethodV,
    CallDoubleMethodA,
    CallVoidMethod,
    CallVoidMethodV,
    CallVoidMethodA,

    CallNonvirtualObjectMethod,
    CallNonvirtualObjectMethodV,
    CallNonvirtualObjectMethodA,
    CallNonvirtualBooleanMethod,
    CallNonvirtualBooleanMethodV,
    CallNonvirtualBooleanMethodA,
    CallNonvirtualByteMethod,
    CallNonvirtualByteMethodV,
    CallNonvirtualByteMethodA,
    CallNonvirtualCharMethod,
    CallNonvirtualCharMethodV,
    CallNonvirtualCharMethodA,
    CallNonvirtualShortMethod,
    CallNonvirtualShortMethodV,
    CallNonvirtualShortMethodA,
    CallNonvirtualIntMethod,
    CallNonvirtualIntMethodV,
    CallNonvirtualIntMethodA,
    CallNonvirtualLongMethod,
    CallNonvirtualLongMethodV,
    CallNonvirtualLongMethodA,
    CallNonvirtualFloatMethod,
    CallNonvirtualFloatMethodV,
    CallNonvirtualFloatMethodA,
    CallNonvirtualDoubleMethod,
    CallNonvirtualDoubleMethodV,
    CallNonvirtualDoubleMethodA,
    CallNonvirtualVoidMethod,
    CallNonvirtualVoidMethodV,
    CallNonvirtualVoidMethodA,

    GetFieldID,

    GetObjectField,
    GetBooleanField,
    GetByteField,
    GetCharField,
    GetShortField,
    GetIntField,
    GetLongField,
    GetFloatField,
    GetDoubleField,
    SetObjectField,
    SetBooleanField,
    SetByteField,
    SetCharField,
    SetShortField,
    SetIntField,
    SetLongField,
    SetFloatField,
    SetDoubleField,

    GetStaticMethodID,

    CallStaticObjectMethod,
    CallStaticObjectMethodV,
    CallStaticObjectMethodA,
    CallStaticBooleanMethod,
    CallStaticBooleanMethodV,
    CallStaticBooleanMethodA,
    CallStaticByteMethod,
    CallStaticByteMethodV,
    CallStaticByteMethodA,
    CallStaticCharMethod,
    CallStaticCharMethodV,
    CallStaticCharMethodA,
    CallStaticShortMethod,
    CallStaticShortMethodV,
    CallStaticShortMethodA,
    CallStaticIntMethod,
    CallStaticIntMethodV,
    CallStaticIntMethodA,
    CallStaticLongMethod,
    CallStaticLongMethodV,
    CallStaticLongMethodA,
    CallStaticFloatMethod,
    CallStaticFloatMethodV,
    CallStaticFloatMethodA,
    CallStaticDoubleMethod,
    CallStaticDoubleMethodV,
    CallStaticDoubleMethodA,
    CallStaticVoidMethod,
    CallStaticVoidMethodV,
    CallStaticVoidMethodA,

    GetStaticFieldID,

    GetStaticObjectField,
    GetStaticBooleanField,
    GetStaticByteField,
    GetStaticCharField,
    GetStaticShortField,
    GetStaticIntField,
    GetStaticLongField,
    GetStaticFloatField,
    GetStaticDoubleField,

    SetStaticObjectField,
    SetStaticBooleanField,
    SetStaticByteField,
    SetStaticCharField,
    SetStaticShortField,
    SetStaticIntField,
    SetStaticLongField,
    SetStaticFloatField,
    SetStaticDoubleField,

    NewString,

    GetStringLength,
    GetStringChars,
    ReleaseStringChars,

    NewStringUTF,
    GetStringUTFLength,
    GetStringUTFChars,
    ReleaseStringUTFChars,

    GetArrayLength,

    NewObjectArray,
    GetObjectArrayElement,
    SetObjectArrayElement,

    NewBooleanArray,
    NewByteArray,
    NewCharArray,
    NewShortArray,
    NewIntArray,
    NewLongArray,
    NewFloatArray,
    NewDoubleArray,

    GetBooleanArrayElements,
    GetByteArrayElements,
    GetCharArrayElements,
    GetShortArrayElements,
    GetIntArrayElements,
    GetLongArrayElements,
    GetFloatArrayElements,
    GetDoubleArrayElements,

    ReleaseBooleanArrayElements,
    ReleaseByteArrayElements,
    ReleaseCharArrayElements,
    ReleaseShortArrayElements,
    ReleaseIntArrayElements,
    ReleaseLongArrayElements,
    ReleaseFloatArrayElements,
    ReleaseDoubleArrayElements,

    GetBooleanArrayRegion,
    GetByteArrayRegion,
    GetCharArrayRegion,
    GetShortArrayRegion,
    GetIntArrayRegion,
    GetLongArrayRegion,
    GetFloatArrayRegion,
    GetDoubleArrayRegion,
    SetBooleanArrayRegion,
    SetByteArrayRegion,
    SetCharArrayRegion,
    SetShortArrayRegion,
    SetIntArrayRegion,
    SetLongArrayRegion,
    SetFloatArrayRegion,
    SetDoubleArrayRegion,

    RegisterNatives,
    UnregisterNatives,

    MonitorEnter,
    MonitorExit,

    GetJavaVM,

    GetStringRegion,
    GetStringUTFRegion,

    GetPrimitiveArrayCritical,
    ReleasePrimitiveArrayCritical,

    GetStringCritical,
    ReleaseStringCritical,

    NewWeakGlobalRef,
    DeleteWeakGlobalRef,

    ExceptionCheck,

    NewDirectByteBuffer,
    GetDirectBufferAddress,
    GetDirectBufferCapacity,

    GetObjectRefType
  };
```

## 第五章

Invocation API允许软件开发商将Java VM加载到任意的原生应用程序中。开发商可以在不链接Java VM源代码的情况下交付支持Java的应用程序。

### 介绍

下面的代码示例演示了如何在Invocation API中使用函数。在这个例子中，C++代码创建了一个Java VM并调用名为Main.test的静态方法。

``` c
    #include <jni.h>       /* where everything is defined */
    ...
    JavaVM *jvm;       /* denotes a Java VM */
    JNIEnv *env;       /* pointer to native method interface */
    JavaVMInitArgs vm_args; /* JDK/JRE 6 VM initialization arguments */
    JavaVMOption* options = new JavaVMOption[1];
    options[0].optionString = "-Djava.class.path=/usr/lib/java";
    vm_args.version = JNI_VERSION_1_6;
    vm_args.nOptions = 1;
    vm_args.options = options;
    vm_args.ignoreUnrecognized = false;
    /* load and initialize a Java VM, return a JNI interface
     * pointer in env */
    JNI_CreateJavaVM(&jvm, (void**)&env, &vm_args);
    delete options;
    /* invoke the Main.test method using the JNI */
    jclass cls = env->FindClass("Main");
    jmethodID mid = env->GetStaticMethodID(cls, "test", "(I)V");
    env->CallStaticVoidMethod(cls, mid, 100);
    /* We are done. */
    jvm->DestroyJavaVM();
```

这个例子使用了API中的三个函数。Invocation API允许原生应用程序使用JNI接口指针访问VM特性。

### 创建VM

JNI_CreateJavaVM函数加载并初始化一个Java VM，并返回一个指向JNI接口指针的指针。调用JNI_CreateJavaVM函数的线程被作为主线程。

### 绑定VM

JNI接口指针JNIEnv仅在当前线程有效。如果另一个线程需要访问Java VM，它必须先调用AttachCurrentThread将自己绑定到VM，并获得JNI接口指针。一旦绑定到VM，原生线程就像运行在原生方法内部的普通Java线程一样工作。原生线程保持与VM绑定，直到调用DetachCurrentThread来解绑自己。

绑定线程应该有足够的堆栈空间来执行一定数量的工作。每个线程的堆栈空间分配特定于操作系统。

### 解绑VM

绑定到VM的原生线程在退出之前必须调用DetachCurrentThread来解绑自己。如果调用堆栈上有Java方法，线程不能解绑自己。

### 卸载VM

JNI_DestroyJavaVM函数卸载一个Java VM。从JDK/JRE1.1开始，只有主线程可以通过调用DestroyJavaVM卸载VM。从JDK/JRE1.2开始限制被取消，任何线程都可以调用DestroyJavaVM卸载VM。

VM等待直到当前线程成为唯一的非守护用户线程才真正卸载。用户线程包括Java线程和绑定的原生线程。存在此限制是因为Java线程或绑定的原生线程可能持有系统资源，比如锁、窗口等。VM不能自动释放这些资源。通过将当前线程限制为卸载VM时唯一运行的线程，程序员将承担释放任意线程占用系统资源的责任。

### 库和版本管理

从JDK/JRE 1.1开始，一旦原生库被加载，它在所有类加载器中都是可见的。因此不同类加载器中的两个类可能链接同一个原生方法。这导致两个问题:

* 一个类可能错误地链接到在不同类加载器中被同名类载入的原生库
* 原生方法可以很容易的混合来自不同类加载器的类。这打破了类加载器提供的名称空间隔离，并导致类型安全问题

从JDK/JRE 1.2开始，每个类加载器管理自己的一组原生库。同一个JNI原生库不能被载入到多个类加载器中。这样做会导致抛出UnsatisfiedLinkError。比如，System.loadLibrary在将一个原生库载入两个类加载时会抛出UnsatisfiedLinkError。新方法的好处是:

* 基于类加载器的名称空间隔离被保留在原生库中。原生库不能很容易的混合来自不同类加载器的类
* 此外，当对应的类加载器被垃圾回收时，原生库可以被卸载






