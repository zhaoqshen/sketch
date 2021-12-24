# 对象头

对象头在32位系统上占用8bytes，64位系统上占用16bytes。

![img](https://images0.cnblogs.com/i/288950/201405/281942192132905.png)

![img](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-10-14/281942312133838.png)

 

# 实例数据

原生类型(primitive type)的内存占用如下：

| Primitive Type | Memory Required(bytes) |
| -------------- | ---------------------- |
| boolean        | 1                      |
| byte           | 1                      |
| short          | 2                      |
| char           | 2                      |
| int            | 4                      |
| float          | 4                      |
| long           | 8                      |
| double         | 8                      |

reference类型在32位系统上每个占用4bytes, 在64位系统上每个占用8bytes。

 

# 对齐填充

HotSpot的对齐方式为8字节对齐：

> （对象头 + 实例数据 + padding） % 8等于0且0 <= padding < 8

 

# 指针压缩

对象占用的内存大小收到VM参数**UseCompressedOops**的影响。

## 1）对对象头的影响

开启（-XX:+UseCompressedOops）对象头大小为12bytes（64位机器）。

```
static class A {
        int a;
    }
```

A对象占用内存情况：

关闭指针压缩： 16+4=20不是8的倍数，所以+padding/4=24

![img](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-10-14/281948383224150.png)

开启指针压缩： 12+4=16已经是8的倍数了，不需要再padding。

![img](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-10-14/281948446666546.png)

 

## 1） 对reference类型的影响

64位机器上reference类型占用8个字节，开启指针压缩后占用4个字节。

```
static class B2 {
        int b2a;
        Integer b2b;
}
```

B2对象占用内存情况：

关闭指针压缩： 16+4+8=28不是8的倍数，所以+padding/4=32

![img](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-10-14/281950335883037.png)

开启指针压缩： 12+4+4=20不是8的倍数，所以+padding/4=24

![img](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-10-14/281950405253616.png)

 

# 数组对象

64位机器上，数组对象的对象头占用24个字节，启用压缩之后占用16个字节。之所以比普通对象占用内存多是因为需要额外的空间存储数组的长度。

先考虑下new Integer[0]占用的内存大小，长度为0，即是对象头的大小：

未开启压缩：24bytes

![img](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-10-14/281951361666018.png)

开启压缩后：16bytes

![img](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-10-14/281951534639534.png)

接着计算new Integer[1]，new Integer[2]，new Integer[3]和new Integer[4]就很容易了：

未开启压缩：

![img](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-10-14/281952456196066.png)

开启压缩：

![img](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-10-14/281952563533487.png)

拿new Integer[3]来具体解释下：

未开启压缩：24（对象头）+8*3=48，不需要padding；

开启压缩：16（对象头）+3*4=28，+padding/4=32，其他依次类推。

自定义类的数组也是一样的，比如：

```
static class B3 {
        int a;
        Integer b;
    }
```

new B3[3]占用的内存大小：

未开启压缩：48

开启压缩后：32

 

# **复合对象**

计算复合对象占用内存的大小其实就是运用上面几条规则，只是麻烦点。

## 1）对象本身的大小

直接计算当前对象占用空间大小，包括当前类及超类的基本类型实例字段大小、引用类型实例字段引用大小、实例基本类型数组总占用空间、实例引用类型数组引用本身占用空间大小; 但是不包括超类继承下来的和当前类声明的实例引用字段的对象本身的大小、实例引用数组引用的对象本身的大小。

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

```
static class B {
        int a;
        int b;
    }
static class C {
        int ba;
        B[] as = new B[3];

        C() {
            for (int i = 0; i < as.length; i++) {
                as[i] = new B();
            }
        }
    }
```

[![复制代码](https://common.cnblogs.com/images/copycode.gif)](javascript:void(0);)

未开启压缩：16（对象头）+4（ba）+8（as引用的大小）+padding/4=32

开启压缩：12+4+4+padding/4=24

 

## 2)当前对象占用的空间总大小

递归计算当前对象占用空间总大小，包括当前类和超类的实例字段大小以及实例字段引用对象大小。

递归计算复合对象占用的内存的时候需要注意的是：对齐填充是以每个对象为单位进行的，看下面这个图就很容易明白。

![img](https://wsx666-1302523054.cos.ap-nanjing.myqcloud.com/image/2021-10-14/281956463229130-20211014115358860.png)

 

 

 



 

现在我们来手动计算下C对象占用的全部内存是多少，主要是三部分构成：C对象本身的大小+数组对象的大小+B对象的大小。

未开启压缩：

(16 + 4 + 8+4(padding)) + (24+ 8*3) +(16+8)*3 = 152bytes

开启压缩：

(12 + 4 + 4 +4(padding)) + (16 + 4*3 +4(数组对象padding)) + (12+8+4（B对象padding）)*3= 128bytes

 