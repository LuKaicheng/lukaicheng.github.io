---
title: Java序列化机制总结
date: 2017-06-24 08:42:31
tags: Serialization
---

## 序列化缘由

在实际的程序运行过程中，总会有这样的场景：程序可以将本次运行的某些对象或者状态保存下来，以供下次启动时使用；在一个联网环境里，不同节点之间的系统需要进行通信，发送者要将某些对象发送到远程的接收者从而达到沟通协调的目的。序列化正是缘于这些场景而出现的一种技术概念，为了能够序列化Java平台的对象，JDK从很早的版本开始就提供了一种序列化机制，它可以将对象编码成字节流(*序列化*)，也可以反向从字节流编码中重新构建对象(*反序列化*)，以用于支持RMI和JavaBean。

<!-- more -->

## 序列化实现

想要一个类的实例能被序列化，只要让类实现`java.io.Serializable`或者`java.io.Externalizable`接口即可。其中，`java.io.Serializable`仅仅只是一个标记接口，系统通过判断是否是此接口的子类来确定对象能否被序列化，而整个序列化过程对于应用开发者来说完全是透明的。`java.io.Externalizable`不同于前者，开发者必须通过实现`writeExternal`和`readExternal`这两个方法，来决定如何保存和恢复对象的内容，另外需要注意的是`Externalizable`子类要有一个**公共的无参构造器**，否则会导致反序列化失败。

下面的例子展示了一个`Person`类，其继承`Serializable`接口，表示能被序列化：

```java
public class Person implements Serializable {

    private static final long serialVersionUID = 1L;

    private String name;
    private String birth;
    private int age;

    public Person(String name, String birth, int age) {
        this.name = name;
        this.birth = birth;
        this.age = age;
    }

    @Override
    public String toString() {
        return String.format("My name is %s, My birthDay is %s,My age is %d", name, birth, age);
    }

}
```

当实际进行序列化操作时，可以创建`ObjectOutputStream`，调用相应的**writeXXX**方法就可以将对象序列化成字节流。当需要进行反序列化操作时，可以创建`ObjectInputStream`，调用相应的**readXXX**方法就可以从字节流中还原对象信息。下面的例子展示了如何将对象序列化到文件，并从文件反序列化成对象：

```java
public class SerialiazationTest {

    public static void main(String[] args) throws Exception {
        String file = "D:/extern.out";
        try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(file))) {
            oos.writeObject(new Person("Lucifer", "1986-01-01", 30));
            oos.flush();
        }
        try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream(file))) {
            Person p = (Person) ois.readObject();
            System.out.println(p);
        }

    }

}
```

### 序列化字段修饰

默认情况下，除了用**transient**或**static**声明的字段，本类中所声明的其他字段都会被序列化。在实际序列化过程中，会创建实例类对应的`ObjectStreamClass`，它会探测实例类中哪些字段需要进行序列化，具体细节可参考`ObjectStreamClass.getDefaultSerialFields`方法：

```java
    private static ObjectStreamField[] getDefaultSerialFields(Class<?> cl) {
        Field[] clFields = cl.getDeclaredFields();
        ArrayList<ObjectStreamField> list = new ArrayList<>();
        int mask = Modifier.STATIC | Modifier.TRANSIENT;

        for (int i = 0; i < clFields.length; i++) {
            if ((clFields[i].getModifiers() & mask) == 0) {
                list.add(new ObjectStreamField(clFields[i], false, true));
            }
        }
        int size = list.size();
        return (size == 0) ? NO_FIELDS :
            list.toArray(new ObjectStreamField[size]);
    }
```

### 序列化字段声明

除了使用修饰符来控制字段是否需要被序列化，还有一种方式可以显式的声明需要进行序列化的字段，并且会**覆盖默认使用修饰符的方式**，那就是声明特殊的`serialPersistentFields`字段。

```java
    private static final ObjectStreamField[] serialPersistentFields = {
            new ObjectStreamField("name", String.class),
            new ObjectStreamField("birth", String.class) };
```

需要注意的是`serialPersistentFields`字段的修饰符必须包含`private` `static` `final`，而且类型必须是`ObjectStreamField[]`。使用这种方式也可以使得类中的字段在后续版本中可以发生变化，个人理解为这个字段声明了在类版本演化过程中需要维持兼容性的一个边界。想要了解更详细的情况，可以查看`ObjectStream.getDeclaredSerialFields`方法：

```java
    private static ObjectStreamField[] getDeclaredSerialFields(Class<?> cl)
        throws InvalidClassException
    {
        ObjectStreamField[] serialPersistentFields = null;
        try {
            Field f = cl.getDeclaredField("serialPersistentFields");
            int mask = Modifier.PRIVATE | Modifier.STATIC | Modifier.FINAL;
            if ((f.getModifiers() & mask) == mask) {
                f.setAccessible(true);
                serialPersistentFields = (ObjectStreamField[]) f.get(null);
            }
        } catch (Exception ex) {
        }
        if (serialPersistentFields == null) {
            return null;
        } else if (serialPersistentFields.length == 0) {
            return NO_FIELDS;
        }

        ObjectStreamField[] boundFields =
            new ObjectStreamField[serialPersistentFields.length];
        Set<String> fieldNames = new HashSet<>(serialPersistentFields.length);

        for (int i = 0; i < serialPersistentFields.length; i++) {
            ObjectStreamField spf = serialPersistentFields[i];

            String fname = spf.getName();
            if (fieldNames.contains(fname)) {
                throw new InvalidClassException(
                    "multiple serializable fields named " + fname);
            }
            fieldNames.add(fname);

            try {
                Field f = cl.getDeclaredField(fname);
                if ((f.getType() == spf.getType()) &&
                    ((f.getModifiers() & Modifier.STATIC) == 0))
                {
                    boundFields[i] =
                        new ObjectStreamField(f, spf.isUnshared(), true);
                }
            } catch (NoSuchFieldException ex) {
            }
            if (boundFields[i] == null) {
                boundFields[i] = new ObjectStreamField(
                    fname, spf.getType(), spf.isUnshared());
            }
        }
        return boundFields;
    }
```

## 自定义序列化行为

除了使用修饰符或者特殊字段的方式之外，Java序列化机制还提供了`writeObject`和`readObject`这两个特殊的方法，可以让我们覆盖默认的序列化方式，实现自定义序列化行为，下面是完整的方法签名：

```java
//自定义序列化行为
private void writeObject(ObjectOutputStream stream) throws IOException;
//自定义反序列化行为
private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException;
```

需要自定义序列化行为的`Serializable`子类可以通过实现这两个方法来自定义整个序列化过程。以`LinkedList`为例，其内部就定义实现了这两个方法，之所以如此，这是由于其内部实现目前采用了双端链表的形式，假如采用默认的序列化方式，那么不仅会镜像链表中的所有项，整个链表的拓扑结构也会被镜像，这不但会导致不必要的空间消耗，也会使得类永远被束缚其内部表示法，因此`Linked`实现了自定义的序列化方式，仅仅序列化数据项的数量以及实际数据项，而拓扑结构是在反序列化过程中重新组织：

```java
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        // Write out any hidden serialization magic
        s.defaultWriteObject();

        // Write out size
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (Node<E> x = first; x != null; x = x.next)
            s.writeObject(x.item);
    }

    @SuppressWarnings("unchecked")
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        // Read in any hidden serialization magic
        s.defaultReadObject();

        // Read in size
        int size = s.readInt();

        // Read in all elements in the proper order.
        for (int i = 0; i < size; i++)
            linkLast((E)s.readObject());
    }
```

这里有两点需要注意：首先，即使你需要实现自定义的序列化过程，也请在序列化时优先调用`defaultWriteObject`方法以及在反序列化时优先调用`defaultReadObject`方法，这可以使类保持更大的灵活性，即使以后类增加了非**transient**字段，也能保持向前和向后兼容性；其次，在反序列化过程中，对于类的约束关系也需要进行保证，这是因为反序列化其实也是一种对象创建机制，是一种**隐藏的构造器**，否则类的约束关系就容易遭到破坏，从而可能遭受非法访问。

## 序列化版本号

当然不管最终采用默认的抑或自定义的序列化行为，显式声明`serialVersionUID ` 都是一种较好的编程习惯。首先，如果不提供显式的`serialVersionUID` ，那么需要在运行时通过一个高开销的计算过程([Stream Unique Identifiers](https://docs.oracle.com/javase/7/docs/platform/serialization/spec/class.html#4100))产生一个`serialVersionUID`，这会导致一些性能损失。其次，`serialVersionUID `计算会综合考虑类、接口、方法以及字段，随着类的演进，这可能导致旧版本和新版本计算出来的`serialVersionUID`不一致，从而导致现有类与原有类的不兼容(*即使两者实际上可以兼容*)。基于以上考虑，显式声明`serialVersionUID`，可以让我们明确是否需要为类创建新版本。

### 不可兼容变更

当类演进过程中，碰到如下情况，可以考虑为不兼容，需要创建新版本：

- 删除字段
- 提升或降低类层级
- 将一个非**transient**字段变成**transient**或者将一个非**static**字段变成**static**
- 改变一个基本类型字段的原有声明类型
- 修改`writeObject`或`readObject`方法使得与原有版本在处理默认字段时方式不同
- 将类从实现`Serializable`接口 变成实现`Externalizable`接口或者相反情况
- 将类从非枚举类型变成枚举类型或者相反情况
- 移除`Serializable`接口或`Externalizable`接口
- 为类增加`writeReplace`或者`readResolve`方法

### 可兼容变更

当类演进过程中，碰到如下情况，可以考虑为兼容情况：

- 增加字段
- 增加类
- 删除类
- 增加`writeObject`和`readObject`方法
- 移除`writeObject`和`readObject`方法
- 增加`java.io.Serializable`接口
- 改变字段的访问权限
- 将一个字段从**static**变成非**static**或者从**transient**变成非**transient**

## 序列化代理模式

由于序列化机制会利用普通构造器之外的机制来创建对象，增加了出错和出现安全问题的可能性，于是JDK设计人员提出了所谓的序列化代理模式，这种模式大概的思路是为可序列化的类设计一个私有的静态嵌套类，其被称作序列化代理，它有一个单独的构造器来接收外围类，从而复制外围类的数据，这个复制过程不需要一致性检查和保护性拷贝。接着利用两个特殊方法`writeReplace`和`readResolve`来实现外围类实例和序列化代理类实例之间的互相转换，从而最终序列化目的。

两个方法的完整签名以及简要说明如下：

```java
//类通过实现此方法，可以使得其实例在序列化写入流时替换成此方法返回的对象
ANY-ACCESS-MODIFIER Object writeReplace() throws ObjectStreamException;
//类通过实现此方法，可以使得反序列化读取到其实例之后，并在返回调用者之前，替换成此方法的返回对象
ANY-ACCESS-MODIFIER Object readResolve() throws ObjectStreamException;
```

`EnumSet`内部就实现了序列化代理，它定义了静态内部类`SerializationProxy`并实现了`writeReplace`方法。当对`EnumSet`实例进行序列化时，会调用`writeReplace`方法，这会返回一个`SerializationProxy`实例并替换`EnumSet`实例写入字节流；当反序列化时，读取出来的首先是`SerializationProxy`实例，由于该实例的类定义了`readResolve`方法，因此在返回给调用者之前，此方法会被调用，从而利用已有的工厂方法重建回`EnumSet`实例。另外需要注意的是`EnumSet`定义`readObject`方法是为了防止可能的攻击。以下是`EnumSet`类的部分源码：

```java
public abstract class EnumSet<E extends Enum<E>> extends AbstractSet<E>
    implements Cloneable, java.io.Serializable
{

    final Class<E> elementType;

    final Enum[] universe;
  
    //省略部分代码
   	public static <E extends Enum<E>> EnumSet<E> noneOf(Class<E> elementType) {
        Enum[] universe = getUniverse(elementType);
        if (universe == null)
            throw new ClassCastException(elementType + " not an enum");

        if (universe.length <= 64)
            return new RegularEnumSet<>(elementType, universe);
        else
            return new JumboEnumSet<>(elementType, universe);
    }
  	//省略部分代码

    private static class SerializationProxy <E extends Enum<E>>
        implements java.io.Serializable
    {

        private final Class<E> elementType;

        private final Enum[] elements;

        SerializationProxy(EnumSet<E> set) {
            elementType = set.elementType;
            elements = set.toArray(ZERO_LENGTH_ENUM_ARRAY);
        }

        private Object readResolve() {
            EnumSet<E> result = EnumSet.noneOf(elementType);
            for (Enum e : elements)
                result.add((E)e);
            return result;
        }

        private static final long serialVersionUID = 362491234563181265L;
    }

    Object writeReplace() {
        return new SerializationProxy<>(this);
    }

    private void readObject(java.io.ObjectInputStream stream)
        throws java.io.InvalidObjectException {
        throw new java.io.InvalidObjectException("Proxy required");
    }
}
```

## 参考

[Java Object Serialization Specification](https://docs.oracle.com/javase/7/docs/platform/serialization/spec/serialTOC.html)

[Effective Java中文版(第2版)](https://book.douban.com/subject/3360807/)