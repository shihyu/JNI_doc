##JNI函数手册
=====================
JNI所有函数可以在jni.h里找到原型，它们大多定义在JNINativeInterface和JNIInvokeInterface结构里；
前者的指针就是JNIEnv类型， 后者的指针是JavaVM类型。

```C
typedef const struct JNINativeInterface* JNIEnv;
typedef const struct JNIInvokeInterface* JavaVM;

struct JNINativeInterface {
    void*       reserved0;
    void*       reserved1;
    void*       reserved2;
    void*       reserved3;

    jint        (*GetVersion)(JNIEnv *);

    jclass      (*DefineClass)(JNIEnv*, const char*, jobject, const jbyte*,
                        jsize);
    jclass      (*FindClass)(JNIEnv*, const char*);
    ......
};

truct JNIInvokeInterface {
    void*       reserved0;
    void*       reserved1;
    void*       reserved2;

    jint        (*DestroyJavaVM)(JavaVM*);
    jint        (*AttachCurrentThread)(JavaVM*, JNIEnv**, void*);
    jint        (*DetachCurrentThread)(JavaVM*);
    jint        (*GetEnv)(JavaVM*, void**, jint);
    jint        (*AttachCurrentThreadAsDaemon)(JavaVM*, JNIEnv**, void*);
};
```
以下对这些函数进行简单分类，并介绍。
##版本信息
```java
/**
 * 获取JNI版本号
 *
 * @param env JNI interface指针
 * @return 返回一个十六进制整数，其中高16位表示主版本号，低16位标识表示次版本号，
 *  如：1.2, GetVersion()返回0x00010002, 1.4, GetVersion() returns 0x00010004.
 */
jint GetVersion(JNIEnv *env);
```
**后面再出现 JNIEnv *env 这样的参数不再注释**

###类操作
```java
    /**
     * 从原始类数据的缓冲区中加载类。
     *
     * @param loader 分派给所定义的类的类加载器
     * @param buf 包含 .class 文件数据的缓冲区 
     * @param buflen 缓冲区长度
     * @return 返回Java类对象。如果出错则返回NULL。
     *
     * @throw ClassFormatError  如果类数据指定的类无效
     *      ClassCircularityError  如果类或接口是自身的超类或超接口
     *      OutOfMemoryError  如果系统内存不足
     */
    jclass DefineClass (JNIEnv *env, jobject loader, const jbyte *buf , jsize bufLen); 
    
    /**
     * 该函数用于加载Java类。它将在CLASSPATH 环境变量所指定的目录和zip文件里搜索指定的类名。
     *
     * @param name  类全名 = (包名+‘/’+类名).replace('.', '/');
     * @return 类对象全名; 如果找不到该类，则返回 NULL。
     * @throw ClassFormatError      如果类数据指定的类无效
     *      ClassCircularityError   如果类或接口是自身的超类或超接口
     *      NoClassDefFoundError    如果找不到所请求的类或接口的定义
     *      OutOfMemoryError        如果系统内存不足
     */
    jclass FindClass(JNIEnv *env, const char *name);
    
    /**
     * 通过对象获取这个类。该函数比较简单，唯一注意的是对象不能为NULL，否则获取的class肯定返回也为NULL。
     *
     * @param obj Java 类对象实例
     */    
    jclass GetObjectClass (JNIEnv *env, jobject obj); 
    
    /**
     * 获取父类或者说超类 
     *
     * @param clazz Java类对象
     * @return 如果clazz代表一般类而非Object类，则该函数返回由clazz所指定的类的超类。 如果clazz 
           指向Object类或代表某个接口，则该函数返回NULL。
     */  
    jclass GetSuperclass (JNIEnv *env, jclass clazz); 
    
    /**
     * 确定 clazz1 的对象是否可安全地强制转换为clazz2
     *
     * @param clazz1 源类对象
     * @param clazz2 目标类对象
     * @return 以下三种情况返回JNI_TRUE, 否则返回JNI_FALSE
     *      1.第一及第二个类参数引用同一个 Java 类
     *      2.第一个类是第二个类的子类
     *      3.第二个类是第一个类的某个接口
     */    
    jboolean IsAssignableFrom (JNIEnv *env, jclass clazz1,  jclass clazz2); 
```

###异常操作
```java
    /**
     * 抛出 java.lang.Throwable 对象
     *
     * @param obj java.lang.Throwable 对象
     * @return 成功时返回 0，失败时返回负数
     * @throw java.lang.Throwable 对象 obj
     */
    jint  Throw(JNIEnv *env, jthrowable obj);
    
    /**
     * 利用指定类的消息（由 message 指定）构造异常对象并抛出该异常
     *
     * @param clazz java.lang.Throwable 的子类
     * @param message 用于构造java.lang.Throwable对象的消息
     * @return 成功时返回 0，失败时返回负数
     * @throw 新构造的 java.lang.Throwable 对象
     */    
    jint ThrowNew (JNIEnv *env ,  jclass clazz,  const char *message);

    /**
     * 确定某个异常是否正被抛出。在本地代码调用ExceptionClear()或Java代码处理该异常前，异常将始终保持
     *  抛出状态。
     *
     * @return 返回正被抛出的异常对象，如果当前无异常被抛出，则返回NULL
     */    
    jthrowable ExceptionOccurred (JNIEnv *env);

    /**
     * 将异常及堆栈的回溯输出到标准输出（例如 stderr）。该例程可便利调试操作。
     */    
    void ExceptionDescribe (JNIEnv *env);
    
    /*
     * 清除当前抛出的任何异常。如果当前无异常，则不产生任何效果。
     */
    void ExceptionClear (JNIEnv *env); 

    /*
     * 抛出致命错误并且不希望虚拟机进行修复。该函数无返回值
     * @param msg 错误消息
     */    
    void FatalError (JNIEnv *env, const char *msg);  
```

###全局及局部引用
```java
    /**
     * 创建 obj 参数所引用对象的新全局引用, 创建的引用要通过调用DeleteGlobalRef() 来显式撤消
     *
     * @param obj 全局或局部引用
     * @return 返回全局引用，如果系统内存不足则返回 NULL
     */
    object NewGlobalRef (JNIEnv *env, jobject obj); 
    
    /**
     * 删除 globalRef 所指向的全局引用
     *
     * @param globalRef 全局引用
     */    
    void DeleteGlobalRef (JNIEnv *env, jobject globalRef); 
    
    /**
     * 创建 obj 参数所引用对象的局部引用, 创建的引用要通过调用DeleteLocalRef()来显式删除
     *
     * @param obj 全局或局部引用
     * @return 返回局部引用，如果系统内存不足则返回 NULL
     */    
    jobject NewLocalRef(JNIEnv *env, jobject ref);
    
    /**
     * 删除 localRef所指向的局部引用
     *
     * @param localRef局部引用
     */    
    void  DeleteLocalRef (JNIEnv *env, jobject localRef); 
    
    /**
     * 用obj创建新的弱全局引用，
     * @param obj 全局或局部因哟娜
     * @return 返回弱全局引用，弱obj指向null，或者内存不足时返回NULL，同时抛出异常
     */    
    jweak NewWeakGlobalRef(JNIEnv *env, jobject obj);
    
    /**
     * 删除弱全局引用
     */    
    void DeleteWeakGlobalRef(JNIEnv *env, jweak obj);
```

###对象操作
```java
    /**
     * 分配新Java对象而不调用该对象的任何构造函数,返回该对象的引用；clazz 参数务必不要引用数组类。
     *
     * @param clazz  Java 类对象
     * @return 返回 Java对象；如果无法构造该对象，则返回NULL
     * @throw InstantiationException：如果该类为一个接口或抽象类
     *      OutOfMemoryError：如果系统内存不足
     */
    jobject AllocObject (JNIEnv *env, jclass clazz);
    /**
     * 构造新Java对象。方法methodId指向应调用的构造函数方法。注意：该ID特指该类class的构造函数ID，必须通过调用 
     * GetMethodID()获得，且调用时的方法名必须为 <init>，而返回类型必须为 void (V)，clazz参数务必不要引用数组类。
     *
     * @return 返回Java对象，如果无法构造该对象，则返回NULL
     * @throw InstantiationException  如果该类为接口或抽象类
     *       OutOfMemoryError   如果系统内存不足
     */    
    jobject NewObject (JNIEnv *env ,  jclass clazz,  jmethodID methodID, ...);   //参数附加在函数后面
    jobject NewObjectA (JNIEnv *env , jclassclazz,  jmethodID methodID, jvalue *args);    //参数以指针形式附加 
    jobjec tNewObjectV (JNIEnv *env , jclassclazz,  jmethodID methodID, va_list args);    //参数以"链表"形式附加

    /**
     * 返回对象的类
     *
     * @param obj  Java对象（不能为 NULL）
     * @return Java 类对象
     */    
    jclass GetObjectClass (JNIEnv *env, jobject obj);  
    
    /**
     * 测试对象是否为某个类的实例
     *
     * @param obj Java对象
     * @param clazz Java类对象
     * @return 如果可将obj强制转换为clazz，则返回JNI_TRUE。否则返回JNI_FALSE。NULL对象可强制转换为任何类
     */    
    jboolean IsInstanceOf (JNIEnv *env, jobject obj, jclass clazz);

    /**
     * 测试两个引用是否引用同一Java对象
     *
     * @param ref1 java对象
     * @param ref2 java对象
     * @return 如果ref1和ref2引用同一Java对象或均为NULL，则返回JNI_TRUE。否则返回JNI_FALSE
     */    
    jbooleanIsSameObject (JNIEnv *env, jobject ref1, jobject ref2);  
```

###字符串操作
```java
    /**
     * 利用Unicode字符数组构造新的java.lang.String对象
     *
     * @param 指向Unicode字符串的指针
     * @param Unicode字符串的长度
     * @return Java 字符串对象。如果无法构造该字符串，则为NULL.
     * @throw OutOfMemoryError：如果系统内存不足
     */
    jstring  NewString (JNIEnv *env, const jchar *unicodeChars, jsize len);

    /**
     * 返回Java字符串的长度（Unicode 字符数）
     *
     * @param string Java 字符串对象
     * @return ava 字符串的长度
     * @throw
    jsize  GetStringLength (JNIEnv *env, jstring string);

    /**
     * 返回指向字符串的Unicode字符数组的指针。该指针在调用 ReleaseStringchars() 前一直有效。          
     * 如果 isCopy 非空，则在复制完成后将 *isCopy 设为 JNI_TRUE。如果没有复制，则设为JNI_FALSE
     *
     * @param string Java 字符串对象
     * @param isCopy 指向布尔值的指针
     * @return 指向 Unicode 字符串的指针，如果操作失败，则返回NULL
     */
    const jchar * GetStringChars(JNIEnv*env, jstring string, jboolean *isCopy); 

    /**
     * 通知本地代码不要再访问 chars。参数chars 是一个指针，可通过 GetStringChars() 从 string 获得
     *
     * @param chars 指向 Unicode 字符串的指针
     */    
    void ReleaseStringChars(JNIEnv *env, jstring string, const jchar *chars); 

    /**
     * 利用UTF-8字符数组构造新java.lang.String对象
     *
     * @param bytes 指向UTF-8字符串的指针
     * @return Java 字符串对象。如果无法构造该字符串，则为NULL
     * @throw OutOfMemoryError 如果系统内存不足
     */    
    jstring  NewStringUTF (JNIEnv *env, const char *bytes);

    /**
     * 以字节为单位返回字符串的 UTF-8 长度
     *
     * @param string Java字符串对象
     * @return  返回字符串的长度
     */    
    jsize  GetStringUTFLength (JNIEnv *env, jstring string);

    /**
     * 返回指向字符串的UTF-8字符数组的指针。该数组在被ReleaseStringUTFChars()释放前将一直有效。    
     * 如果isCopy不是 NULL，*isCopy 在复制完成后即被设为JNI_TRUE。如果未复制，则设为 JNI_FALSE。
     *
     * @param string Java 字符串对象
     * @param isCopy 指向布尔值的指针
     * @return 指向 UTF-8 字符串的指针。如果操作失败，则为 NULL
     */    
    const char* GetStringUTFChars (JNIEnv*env, jstring string, jboolean *isCopy);
    
    /**
     * 通知本地代码不要再访问 utf。utf 参数是一个指针，可利用 GetStringUTFChars() 获得
     *
     * @param string Java字符串对象
     * @param utf 指向UTF-8字符串的指针
     */    
    void  ReleaseStringUTFChars (JNIEnv *env, jstring string,  const char *utf); 
```
###数组操作
```java
    /**
     *
     *
     * @param
     * @param
     * @param
     * @param
     * @return
     * @throw
     */
```
###访问对象的属性和方法
###注册本地方法
