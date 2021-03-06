如果元空间过小，会引起项目启动时频繁的full gc（因为代码量较大，有较多的类信息要加载）。元空间如果不设置的话，是会动态调整大小的。

![[Pasted image 20220509223824.png]]

# 魔数
确定这个文件是否为一个能 被虚拟机接受的Class文件

# 主次版本号

# 常量池&运行时常量池（1.8后在元空间）
常量池(constant pool table)，用于存放编译期生成的各种字面量和符号引用。
使用```javap -v xxx.class```可以查看某个字节码文件中的常量池：
![[Pasted image 20220213213733.png]]
```java
int a=1;
int s="abc"
```
字面量就是等号右边的值，符号引用就是左边的变量名。除此之外，符号应用还包括：类和接口的全限定名、字段的名称和描述符、方法的名称和描述符。

这些常量池现在是静态信息，只有到运行时被加载到内存后，这些符号才有对应的内存地址信息，这些常量池一旦被装 入内存就变成运行时常量池，对应的符号引用在程序加载或运行时会被转变为被加载到内存区域的代码的直接引用，也 就是我们说的动态链接了。例如，compute()这个符号引用在运行时就会被转变为compute()方法具体代码在内存中的 地址，主要通过对象头里的类型指针去转换直接引用。

# 访问标识
在常量池结束之后，紧接着的2个字节代表访问标志(access_flags)，这个标志用于识别一些类或者接口 层次的访问信息，包括:这个Class是类还是接口;是否定义为public类型;是否定义为abstract类型;如果 是类的话，是否被声明为final;等等。
![[Pasted image 20220509223856.png]]

# 类索引，父类索引和接口索引集合
# 字段表
字段表存储的其实是变量(非局部变量)的修饰符 + 字段描述符索引(索引指向常量池) + 字段名称 索引(索引指向常量池)。
public final static String NUMBER="1" 。pulic final 和static 是方式的修饰符，这些都存放在class文件 的字段表中。 String是字段的描述符，存放于常量池中，Number是字段的名称，存放于常量池。
![[Pasted image 20220509223145.png]]

# 方法表
Class文件存储格式中对方法的描述与对字段的描述采用了几乎完全一致的方式，方法表的结构如同字段表一样， 依次包括访问标志(access_flags)、名称索引(name_index)、描述符索引(descriptor_index)、属性表集合 (attributes)几项。

![[Pasted image 20220509223304.png]]

java中的字段名和方法名的长度取决于u2，也就是constant_utf8_info的长度，也就是64kb
# 属性表
Class文件、字段表、方法表都可以携带自己的属性表 集合，以描述某些场景专有的信息。与Class文件中其他的数据项目要求严格的顺序、长度和内容不同，属性表集合 的限制稍微宽松一些，不再要求各个属性表具有严格顺序，并且《Java虚拟机规范》允许只要不与已有属性名重复， 任何人实现的编译器都可以向属性表中写入自己定义的属性信息。

对于每一个属性，它的名称都要从常量池中引用一个CONSTANT_Utf8_info类型的常量来表示，而属性值的结构则 是完全自定义的，只需要通过一个u4的长度属性去说明属性值所占用的位数即可。

![[Pasted image 20220509223444.png]]

## code属性
![[Pasted image 20220509223523.png]]

## LineNumberTable属性
![[Pasted image 20220509223551.png]]

