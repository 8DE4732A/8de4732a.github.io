---
title: invoke dynamic
date: 2022-06-24 09:00:00
tags: jvm
categories: jvm
---
## 指令

```sql
invokedynamic
indexbyte1
indexbyte2
0
0
```

indexbyte1<<8|indexbyte2指向当前类的常量池，结构为**`CONSTANT_InvokeDynamic_info`**

**`CONSTANT_InvokeDynamic_info:`**

```sql
CONSTANT_InvokeDynamic_info {
    u1 tag;
    u2 bootstrap_method_attr_index;
    u2 name_and_type_index;
}
```

tag→18

bootstrap_method_attr_index→指向BootstrapMethods

name_and_type_index→指向常量池，必须是一个`CONSTANT_NameAndType_info`

`CONSTANT_NameAndType_info`:

```sql
CONSTANT_NameAndType_info {
    u1 tag;
    u2 name_index;
    u2 descriptor_index;
}
```

tag→16

name_index→指向常量池，必须是一个CONSTANT_Utf8_info

descriptor_index→指向常量池，必须是一个CONSTANT_Utf8_info

`BootstrapMethods:`

```sql
BootstrapMethods_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 num_bootstrap_methods;
    {   u2 bootstrap_method_ref;
        u2 num_bootstrap_arguments;
        u2 bootstrap_arguments[num_bootstrap_arguments];
    } bootstrap_methods[num_bootstrap_methods];
}
```

bootstrap_method_ref→指向常量池，必须是一个`CONSTANT_MethodHandle_info`

bootstrap_arguments→类型只能是`CONSTANT_String_info`, `CONSTANT_Class_info`, `CONSTANT_Integer_info`, `CONSTANT_Long_info`
, `CONSTANT_Float_info`, `CONSTANT_Double_info`, `CONSTANT_MethodHandle_info`,  `CONSTANT_MethodType_info`

`CONSTANT_MethodHandle_info`：

```sql
CONSTANT_MethodHandle_info {
    u1 tag;
    u1 reference_kind;
    u2 reference_index;
}
```

## 执行流程

bootstrap方法的前3个参数是固定的：

1. java.lang.invoke.MethodHandles.Lookup
2. String
3. java.lang.invoke.MethodType

一开始操作数栈中的数据如下

- `java.lang.invoke.MethodHandle` bootstrap方法
- `java.lang.invoke.MethodHandles.Lookup` 调用的类
- `String` callSite中的方法名称？
- `java.lang.invoke.MethodType` 获取对象的方法？
- …剩下的参数

然后使用invokevirtual

bootstrap返回一个callSite, 其中绑定了一个methodHandler

再次将方法及参数压栈，

调用invokevirtual得到需要的对象

问题：是使用的当前方法的操作数栈吗？

### 实例

```sql
public class CallSiteTest {
    public static Predicate<String> one() {
        Set<String> closure = new HashSet<>();
        return closure::add;
    }

    public static void main(String[] args) {
        Predicate<String> one = one();
        System.out.println(one.test("ssss"));
        System.out.println(one.test("ssss"));
    }
}
```

java -p -c -v -l

```java
public static java.util.function.Predicate<java.lang.String> one();
    descriptor: ()Ljava/util/function/Predicate;
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=0
         0: new           #2                  // class java/util/HashSet
         3: dup
         4: invokespecial #3                  // Method java/util/HashSet."<init>":()V
         7: astore_0
         8: aload_0
         9: dup
        10: invokevirtual #4                  // Method java/lang/Object.getClass:()Ljava/lang/Class;
        13: pop
        14: invokedynamic #5,  0              // InvokeDynamic #0:test:(Ljava/util/Set;)Ljava/util/function/Predicate;
        19: areturn
      LineNumberTable:
        line 9: 0
        line 10: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            8      12     0 closure   Ljava/util/Set;
      LocalVariableTypeTable:
        Start  Length  Slot  Name   Signature
            8      12     0 closure   Ljava/util/Set<Ljava/lang/String;>;
    Signature: #27                          // ()Ljava/util/function/Predicate<Ljava/lang/String;>;

BootstrapMethods:
  0: #40 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #41 (Ljava/lang/Object;)Z
      #42 invokeinterface java/util/Set.add:(Ljava/lang/Object;)Z
      #43 (Ljava/lang/String;)Z
```

debug得到参数:

![metafactory](/assets/metafactory.png)

目前jvm是通过asm构建一个内部类去实现动态调用的

`InnerClassLambdaMetafactory#spinInnerClass`

根据实现代码可以给jvm传一个参数Djdk.internal.lambda.dumpProxyClasses=E:/来获取生成的class

java -p

```java
final class com.zhongan.send.message.CallSiteTest$$Lambda$1 implements java.util.function.Predicate {
  private final java.util.Set arg$1;
  private com.zhongan.send.message.CallSiteTest$$Lambda$1(java.util.Set);
  private static java.util.function.Predicate get$Lambda(java.util.Set);
  public boolean test(java.lang.Object);
}
```

最终构造出的callSite是：

![callsite](/assets/callsite.png)

就是该生成类的  get$Lambda 方法得到需要的Predicate对象，该方法是private的，只有其外部类可以访问。