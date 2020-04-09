## ASM介绍

Java类型描述符：

Java方法描述符：



ASM 􏰁供了三个基于 **ClassVisitor** API 的核心组件，用于生成和变化类:

􏰀 **ClassReader**类分析以字节数组形式给出的已编译类，并针对在其**accept**方法参数 中传送的 **ClassVisitor** 实例，调用相应的 **visitXxx** 方法。这个类可以看作一个事 件产生器。

􏰀 **ClassWriter** 类是 **ClassVisitor** 抽象类的一个子类，它直接以二进制形式生成编 译后的类。它会生成一个字节数组形式的输出，其中包含了已编译类，可以用 **toByteArray** 方法来􏰁取。这个类可以看作一个事件使用器。

􏰀 **ClassVisitor**类将它收到的所有方法调用都委托给另一个**ClassVisitor**类。这个 类可以看作一个事件筛选器。



##### **ClassVisitor:**

ClassVisitor 类的方法必须按以下顺序调用:

```
visit 
visitSource?   // ？代表最多一个
visitOuterClass? 
( visitAnnotation | visitAttribute )*    // |符号代表任意顺序，*代表任意多个
( visitInnerClass | visitField | visitMethod )*
visitEnd
```

这意味着必须首先调用 visit，然后是对 visitSource 的最多一个调用，接下来是对 visitOuterClass 的最多一个调用，然后是可按任意顺序对 visitAnnotation 和 visitAttribute 的任意多个访问，接下来是可按任意顺序对 visitInnerClass、 visitField 和 visitMethod 的任意多个调用，最后以一个 visitEnd 调用结束。



##### MethodVisitor:

MethodVisitor类的方法必须按以下顺序调用，标识符含义可以参考上面ClassVisitor。

```
visitAnnotationDefault?
(visitAnnotation | visitParameterAnnotation | visitAttribute)*
visitCode
(visitTryCatchBlock | visitLabel | visitFrame | visitXxxInsn | visitLocalVariable | visitLineNumber)*
visitMaxs?
visitEnd 
```

这就意味着，对于非抽象方法，如果存在注释和属性的话，必须首先访问它们，然后是该方 法的字节代码。对于这些方法，其代码必须按顺序访问，位于对 visitCode 的调用（有且仅有 一个调用）与对 visitMaxs 的调用（有且仅有一个调用）之间。

于是，visitCode 和 visitMaxs 方法可用于检测该方法的字节代码在一个事件序列中的 开始与结束。和类的情况一样，visitEnd 方法也必须在最后调用，用于检测一个方法在一个事 件序列中的结束。



示例：

```
package pkg;
public class Bean {
 private int f;
 public int getF() {
 return this.f;
 }
 public void setF(int f) {
 this.f = f;
 }
} 
```

上例getF()方法的字节代码可以用一下方法生成：

```
mv.visitCode() // 启动字节代码的生成过程
mv.visitVarInsn(Opcodes.ALOAD, 0)	// “this”是实例方法的隐藏参数，所以先装载参数”this“
mv.visitFieldInsn(Opcodes.GETFIELD, "pkg/Bean", "f", "I")	// 访问成员变量f
mv.visitInsn(Opcodes.IRETURN) // 方法return标记，返回int型数据，必须要有，如果是void，则方法返回值类型是void，则使用Opcodes.RETURN
mv.visitMaxs(0, 0) // 指定操作数栈最大深度和局部变量表最大空间
mv.visitEnd() // 结束方法访问
```

其中，visitMaxs两个参数的含义可以看下JVM虚拟机规范中栈帧这部分内容。



如果将 ClassVisitor 和 MethodVisitor 类合并，生成完整的类流程如下：

```
ClassVisitor cv = ...;
cv.visit(...);
MethodVisitor mv1 = cv.visitMethod(..., "m1", ...);
mv1.visitCode();
mv1.visitInsn(...);
...
mv1.visitMaxs(...);
mv1.visitEnd();
MethodVisitor mv2 = cv.visitMethod(..., "m2", ...);
mv2.visitCode();
mv2.visitInsn(...);
...
mv2.visitMaxs(...);
mv2.visitEnd();
cv.visitEnd(); 
```



下面以创建类FPSMonitor，实例方法start()为例，回顾下ClassVisitor和MethodVisitor用法：

```
public static void createFile(String output) {
        println("createFile")
        // 1.构建ClassWriter，自动计算max_stack和max_local
        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_MAXS | ClassWriter.COMPUTE_FRAMES)

        // 2.创建类FPSMonitor
        ClassVisitor cv = new ClassVisitor(Opcodes.ASM5, cw) {}
        String className = Constants.CLASS_FPS_MONITOR
        cv.visit(50, Opcodes.ACC_PUBLIC, className, null, "java/lang/Object", null)

        // 3.创建方法start()
        String methodName = Constants.METHOD_START
        MethodVisitor mv = cv.visitMethod(Opcodes.ACC_PUBLIC | Opcodes.ACC_STATIC, methodName, "()V", null, null)
        mv.visitCode()
        mv.visitMaxs(0, 0)
        mv.visitInsn(Opcodes.RETURN)
        mv.visitEnd()
        cv.visitEnd()

				// 4.创建类文件并写入数据
        File outputFile = writeFile(output, className + SdkConstants.DOT_CLASS, cw.toByteArray())
        println("createFile success, file:$outputFile.path")
    }

    static File writeFile(String dir, String fileName, byte[] data) {
        File clzFile = new File(dir, fileName)
        clzFile.getParentFile().mkdir()
        new FileOutputStream(clzFile).write(data)
        return clzFile
    }
```





##### ClassWriter 

为一个方法计算栈映射帧并不是非常容易：必须计算所有帧，找出与 跳转目标相对应的帧，或者跳在无条件跳转之后的帧，最后压缩剩余帧。与此类似，为一个方法 计算局部变量与操作数栈部分的大小要容易一些，但依然算不上非常容易。 

幸好 ASM 能为我们完成这一计算。在创建 ClassWriter 时，可以指定必须自动计算哪些 内容： 

*  在使用 new ClassWriter(0)时，不会自动计算任何东西。必须自行计算帧、局部变 量与操作数栈的大小。 
*  在使用 new ClassWriter(ClassWriter.COMPUTE_MAXS)时，将为你计算局部变量 与操作数栈部分的大小。还是必须调用 visitMaxs，但可以使用任何参数：它们将被 忽略并重新计算。使用这一选项时，仍然必须自行计算这些帧。  
* 在 new ClassWriter(ClassWriter.COMPUTE_FRAMES)时，一切都是自动计算。 不再需要调用 visitFrame，但仍然必须调用 visitMaxs（参数将被忽略并重新计 算）。 

这些选项的使用很方便，但有一个代价：COMPUTE_MAXS 选项使 ClassWriter 的速度降 低 10%，而使用 COMPUTE_FRAMES 选项则使其降低一半。这必须与我们自行计算时所耗费的时 间进行比较：在特定情况下，经常会存在一些比 ASM 所用算法更容易、更快速的计算方法，但 ASM 使用的算法必须能够处理所有情况。 



##### 生成类介绍

为生成一个类，惟一必需的组件是 **ClassWriter** 组件。以下面例子说明：

```
package pkg;
public interface Comparable extends Mesurable {
  int LESS = -1;
  int EQUAL = 0;
  int GREATER = 1;
  int compareTo(Object o);
}
```

可以只用ClassVisitor完成创建上述类的任务：

```
ClassWriter cw = new ClassWriter(0);

cw.visit(V1_5, ACC_PUBLIC + ACC_ABSTRACT + ACC_INTERFACE, "pkg/Comparable", null, "java/lang/Object", new String[] { "pkg/Mesurable" });

cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "LESS", "I", null, new Integer(-1)).visitEnd();

cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "EQUAL", "I", null, new Integer(0)).visitEnd();

cw.visitField(ACC_PUBLIC + ACC_FINAL + ACC_STATIC, "GREATER", "I", null, new Integer(1)).visitEnd(); 

cw.visitMethod(ACC_PUBLIC + ACC_ABSTRACT, "compareTo", "(Ljava/lang/Object;)I", null, null).visitEnd(); 

cw.visitEnd();

byte[] b = cw.toByteArray();
```

对 **visit** 方法的调用定义了类的标头。**V1_5** 参数是一个常数，与所有其他 ASM 常量一样， 在ASM **Opcodes**接口中定义。它指明了类的版本——Java 1.5。**ACC_****XXX**常量是与Java修饰 符对应的标志。这里规定这个类是一个接口，而且它是 **public** 和 **abstract** 的(因为它不能 被实例化)。下一个参数以内部形式规定了类的名字(见 2.1.2 节)。回忆一下，已编译类不包含 **Package** 和 **Import** 部分，因此，所有类名都必须是完全限定的。下一个参数对应于泛型(见 4.1 节)。在我们的例子中，这个参数是 **null**，因为这个接口并没有由类型变量进行参数化。第 五个参数是内部形式的超类(接口类隐式继承自 **Object**)。最后一个参数是一个数组，其中是 被扩展的接口，这些接口由其内部名指定。

**visitMethod** 方法返回 **MethodVisitor**(见图 3.4)，可用 于定义该方法的注释和属性，最重要的是这个方法的代码。这里，由于没有注释，而且这个方法 是抽象的，所以我们立即调用所返回的 **MethodVisitor** 的 **visitEnd** 方法。

对 **visitEnd** 的最后一个调用是为了通知 **cw**:这个类已经结束，对 **toByteArray** 的调用 用于以字节数组的形式􏰁取它。



参考文章：

[ASM全解析]([https://kahnsen.github.io/kahnblog/2017/11/29/ASM%E5%85%A8%E8%A7%A3%E6%9E%90/](https://kahnsen.github.io/kahnblog/2017/11/29/ASM全解析/))



## JVM虚拟机规范

这部分内容想要详细了解的话，可以看看《深入理解Java虚拟机：JVM高级特性与最佳实践》、《Java虚拟机规范》这两本书。下面内容也是从书中摘抄，主要是配合ASM对类和方法的操作指令进行理解。

### 栈帧

栈帧（StackFrame）是用于支持虚拟机进行方法调用和方法执行的数据结构，它是虚拟机运行时数据区中的虚拟机栈（VirtualMachineStack）的栈元素。

栈帧存储了方法的局部变量表、操作数栈、动态连接和方法返回地址等信息。每一个方法从调用开始至执行完成的过程，都对应着一个栈帧在虚拟机栈里面从入栈到出栈的过程。

每一个栈帧都包括了局部变量表、操作数栈、动态连接、方法返回地址和一些额外的附加信息。在编译程序代码的时候，栈帧中需要多大的局部变量表，多深的操作数栈都已经完全确定了，并且写入到方法表的Code属性之中，因此一个栈帧需要分配多少内存不会受到程序运行期变量数据的影响，而仅仅取决于具体的虚拟机实现。

一个线程中的方法调用链可能会很长，很多方法都同时处于执行状态。对于执行引擎来说，在活动线程中，只有位于栈顶的栈帧才是有效的，称为当前栈帧（CurrentStackFrame），与这个栈帧相关联的方法称为当前方法（CurrentMethod）。执行引擎运行的所有字节码指令都只针对当前栈帧进行操作，在概念模型上，典型的栈帧结构如图：

<img src="java_栈帧.png" style="zoom:80%;" />

##### 局部变量表

局部变量表主要是用来存放方法参数和方法内局部变量。

在方法执行时，虚拟机是使用局部变量表完成参数值到参数变量列表的传递过程的，如果执行的是实例方法（非static的方法），那局部变量表中第0位索引的Slot默认是用于传递方法所属对象实例的引用，在方法中可以通过关键字"this"来访问到这个隐含的参数。

##### 操作数栈

操作数栈（OperandStack）也常称为操作栈，它是一个后入先出（LastInFirstOut,LIFO）栈。同局部变量表一样，操作数栈的最大深度也在编译的时候写入到Code属性的max_stacks数据项中。操作数栈的每一个元素可以是任意的Java数据类型，包括long和double。32位数据类型所占的栈容量为1，64位数据类型所占的栈容量为2。在方法执行的任何时候，操作数栈的深度都不会超过在max_stacks数据项中设定的最大值。

当一个方法刚刚开始执行的时候，这个方法的操作数栈是空的，在方法的执行过程中，会有各种字节码指令往操作数栈中写入和提取内容，也就是出栈/入栈操作。例如，在做算术运算的时候是通过操作数栈来进行的，又或者在调用其他方法的时候是通过操作数栈来进行参数传递的。

举个例子，整数加法的字节码指令iadd在运行的时候操作数栈中最接近栈顶的两个元素已经存入了两个int型的数值，当执行这个指令时，会将这两个int值出栈并相加，然后将相加的结果入栈。

操作数栈中元素的数据类型必须与字节码指令的序列严格匹配，在编译程序代码的时候，编译器要严格保证这一点，在类校验阶段的数据流分析中还要再次验证这一点。再以上面的iadd指令为例，这个指令用于整型数加法，它在执行时，最接近栈顶的两个元素的数据类型必须为int型，不能出现一个long和一个float使用iadd命令相加的情况。

##### 动态链接

##### 方法返回地址





## 常见问题：

下面的几个问题可能对于其他同学来说并不算常见问题，只是在我开发插件过程中遇到的问题。

### groovy.lang.MissingPropertyException: No such property

自定义类中缺少对使用的类的引用时就会报错。这点比较坑，因为IDE并不会提示所有应用的类需要import，groovy又是一种动态语言，只会在运行时报错。



###  javassist.CannotCompileException: [source error] no such class

使用Javassist的ClassPool对class文件做处理时，实例化ClassPool时要用``ClassPool.getDefault()``或``ClassPool(true)``，本质上是要执行``ClassPool.appendSystemPath()``。这个方法的意思是类的搜索路径添加了platform library、 extension libraries和classpath定义环境变量中的类。

```
   /**
     * Appends the system search path to the end of the
     * search path.  The system search path
     * usually includes the platform library, extension
     * libraries, and the search path specified by the
     * <code>-classpath</code> option or the <code>CLASSPATH</code>
     * environment variable.
     *
     * @return the appended class path.
     */
    public ClassPath appendSystemPath() {
        return source.appendSystemPath();
    }
```



### 处理Android相关类

因为Android SDK中的类并不是Javassist默认支持的类库，需要特殊处理。处理方式就是通过ClassPool手动添加SDK jar文件路径。

```
String androidJar = "${android.sdkDirectory.absolutePath}/platforms/${android.compileSdkVersion}/android.jar"
ClassPool.getDefault().insertClassPath(androidJar)
```







编译后的文件路径：

app/build/intermediates/javac/debug/compileDebugJavaWithJavac/...







## 参考

[Developing Custom Gradle Plugins](https://docs.gradle.org/current/userguide/custom_plugins.html#custom_plugins)

[Using Gradle Plugins](https://docs.gradle.org/current/userguide/plugins.html)

[Gradle build lifecycle](https://docs.gradle.org/4.3/userguide/build_lifecycle.html)

[深入理解Transform](https://juejin.im/post/5cbffc7af265da03a97aed41#heading-6)