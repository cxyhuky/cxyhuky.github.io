---
title: Methods Common to All Objects 
draft: false
authors: [huyi]
date: 2024-09-25
slug: effectivejava-chapter3
description: Effective-Java阅读笔记
    以下所言，或来自读书笔记，或来自书籍的勾画，或只是回忆...作为参考。
categories:
  - 技术书籍
  - 学习笔记

---

effective-java阅读笔记，第三章——对象的通用方法 <!-- more -->

# Chapter 3. Methods Common to All Objects 

## Item10. 覆盖equals方法时应遵守的约定

> 本条目讨论是否覆盖equals时应该注意的点，即什么情况下不需要覆盖equals，若覆盖equals需要遵守的约定；以及覆盖不合理所导致的后果。

---

`java.lang.Object`类提供了`equals`的默认实现（比较对象引用是否相等），**类的每个实例只等于它自己**。

绝大部分情况下，值类都是需要覆盖equals实现，如果不需要覆盖equals，类应该符合以下几种条件：

- **类的每个实例都是唯一的**，对于像Thread这类表示活动实体类也是如此，Object的默认实现就满足此要求
- 对于**不需要提供【逻辑相等】的类**
- **超类已经实现equals，并且对于子类完全适用**
- 类是私有的，并且你**确保该equals方法永远不会被调用**



如果需要类需要实现**逻辑相等**这类概念时（e.g. `Map`中作为key存在时），则需要覆盖其`equals`方法。以下是覆盖equals时遵守的**通用约定**：

- **反身性**：x.equals(x)必须返回true
- **对称性**：x.equals(y)必须在y.equals(x)返回true时返回true
  - e.g.` java.sql.TimeStamp`继承自`java.util.Date`，如果在同一个集合中使用时间戳和日期对象，或者以其他方式混合使用时间戳和日期对象，那么时间戳的 equals 实现确实违反了对称性
- **传递性**：x.equals(y)，y.equals(z)，则x.equals(z)必须满足
  - 除非你愿意放弃面向对象的抽象优点，否则**无法**继承一个可实例化的类并添加新属性时，并同时保留equals约定（传递性）
  - 可以考虑使用抽象类的子类添加一个值组件或者不采用继承转而采用组合的方式实现
- **一致性**：x.equals(y)的多次调用结果必须一致
  - 无论一个类是否不可变，都**不要编写依赖于不可靠资源的equals方法**，如果违反则很难满足一致性。e.g. `java.net.URL#equals()`方法依赖于与URL相关联的主机的IP地址的比较
- **x.equals(null)必须false**
  - `instanceof`自带判定null的能力，即 obj=null, obj instanceof Object 一定是false



综上所述，构建高质量equals方法的秘诀（按照顺序）：

1. 使用`==`检查参数是否是该对象的引用

2. 使用instanceof检查参数类型是否正确

3. 将参数转为正确类型，并对类中的每个【重要】字段检查对应的字段值是否匹配

   > 注：重要字段可以理解为类继承结构中，使用最多，最通用的字段。对于float或double等字段，使用`Float.compare(float, float)`方法比较


不比较不属于对象逻辑状态的字段，例如同步操作的锁字段，优先比较那些更可能不同、比较成本更低的字段。

写完问自己三个问题：它具备对称性吗？具备传递性吗？具备一致性吗？并编写UT来检查



---

## Item11. hashCode总是同equals一起覆盖

> 讨论hashCode()方法的一般约定，为什么hashCode要同equals一起覆盖？有什么比较好的hash值生成方法？hash生成中要避免的点有哪些？

`java.lang.Object#hashCode`中明确指出：使用hashCode方法是为了便于使用哈希表（类实例如果有使用到hash表，则需要使用此方法生成hashCode）

由于在hash表中，总是结合使用equals和hashCode方法来判断对象是否相等，所以，hashCode方法的一般约定总是与equals有关，具体如下：

1. 同一对象多次调用`hashCode()`，**必须**始终返回**相同**的hash值（前提是用于equals比较的信息未修改）
2. 若两个对象通过equals比较相等(true)，则它们的hash值必须一致
3. 若两个对象通过equals比较不相等，则不强制要求hash值必须一致；但程序员必须意识到，对**不相等的对象产生不同的hash值可以提供hash表的性能**

理想情况下，一个散列算法应该在所有int值上均匀合理分布所有不相等实例集合。实现该理想很困难，但幸运的是实现一个类似的并不太难，方法如下：

1. 声明一个result的int变量，并将其初始化为对象中第一个重要字段的散列码c

2. 对象中剩余的重要字段f，执行以下操作

   1. 为字段计算一个整数散列码c

      1. 若字段为基本类型，计算`Type.hashCode(f)`，其中`type`是与f类型对应的包装类
      2. 若字段为对象引用，调用其hashCode，若字段值为null，则使用0
      3. 若字段是一个数组，则将其每个重要元素都视为一个单独的字段，循环的计算每个元素的hash值，若数组中没有重要元素，则使用常量（最好不是0），如果所有元素都重要，则使用`Arrays.hashCode`

   2. 将步骤1中计算的散列码c合并到result变量

      ```java
      //31 是一个常用的质数，它能在哈希值的计算中产生较好的分布效果，避免哈希冲突
      result = 31 * result + c; 
      ```

3. 返回result变量

4. 完成hashCode方法的编写后，尝试相同的实例是否具有相同的hash值，并编写UT来验证



hash值的生成还要考虑以下几点：

1. **不要试图从散列码计算中排除重要字段**，以提高性能；这**会导致不好的hash分布，影响散列表的性能**
2. 若一个类是**不可变**的，并且**散列码的计算成本很高**，可以考虑在对象中**缓存散列码**

```java
private int hashCode; // Automatically initialized to 0
@Override
public int hashCode() {
    int result = hashCode;
    //no cache
    if (result == 0) {
        result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        hashCode = result; // set cache
    }
    
    return result;
}
```

3. **不要为hashCode返回的值提供详细的规范，这样客户端就不能理所应当的依赖它，这也为你以后可能的更改留有余地。**Java库中很多类，e.g. String和Integer，都将hashCode返回的确切值指定为实例值，这不是一个好主意，它阻碍了未来版本中提高散列算法的能力



## Item12. 始终覆盖toString方法

> 本条目讨论覆盖toString方法的好处是什么，为什么一定要覆盖

toString的通用约定：返回的字符串应该符合**"简洁但信息丰富，易于阅读"**。

覆盖toString方法的好处及一些覆盖的建议：

- **更易于使用**(被打印时更好的展示)，使用该类的系统也**更易于调试**(日志错误消息中更好的展示)
- toString返回的信息中应该**包含最适合该类的表示形式**，e.g. Phone类应该返回"xxxx-xxxx-xxxx"这种格式的内容，若类的对象很大且包含不利于展示的内容，toString应该返回一个**摘要信息**
- 决定**是否对toString返回的内容指定<font color="red">格式</font>**
  - 优点：作为类的**标准的、明确的**表示，**可以作为输入或输出**，还可以提供一个匹配的静态工厂(valueOf)或构造器，来达到对象及其字符串标识之间来回的切换，e.g. java中BigInteger和大多数包装类就是这样实现的。
  - 缺点：一旦指定则必须**终生使用**它，若你的类被广泛使用，则后续修改toString的格式将会破坏引用该类的代码和数据。若不选择格式，增加了在后续版本中添加信息或改进格式的灵活性
- 指定/不指定格式，你都**应该为toString提供详细的文档注释**
- **提供对toString返回值中包含的信息的程序性访问（即getter方法）**，否则引用者不得不通过解析toString来获取他们的值，这种方式会让系统脆弱
- **在任何抽象类中编写toString方法**，并让子类共享公共的字符串表示形式



## Item13. 明智的覆盖clone

若想调用Object.clone()方法来实现克隆，则类必须实现Cloneable接口，否则将抛出CloneNotSupportedException异常。

如果不是通过Object.clone()方法实现克隆的，则不需要实现Cloneable接口。如下：

```java
class Base {
    @Override
    protected Object clone() throw CloneNotSupportedException {
        //这种情况下会抛出CloneNotSupportedException，因为调用的是Object.clone()方法
        //return super.clone(); 
        return new Base(); //这种方式可不实现Cloneable接口
    }
}
```

