---
layout: post
title:  "Apache Arrow Memory Management"
date:   2019-04-06
categories: jekyll update
---
* 目录
{:toc}
# Apache Arrow简介
Apache Arrow是为内存中的数据设计的跨语言开发平台。主要特点是定义了列存数据在物理内存中的布局。除此之外，Arrow还定义了元数据规范，数据类型，进程间通信以及内存管理等模块并提供了相应的代码实现。

本文主要关注Apache Arrow的物理内存布局以及内存管理设计与实现。

# 物理内存布局
Arrow中基本存储单元，从OS层面来看，是一片连续的内存区域，该区域内保存了一些字节信息；从开发者的角度来看，是类似数组的结构，主要是保存了一组数据。

和一些高级语言中的数组主要区别是
- 一些高级语言中的数组是基于面向对象的分配
- Arrow中则是利用了连续的内存地址实现对一组数据的读写

## 组成部分
- validity buffer,用于描述一组数据的空值情况；当数组中的元素没有空值时，可以省略
- offset buffer，用于描述一组数据中每个元素的偏移量；当数组中的元素类型为定长时，可以省略
- value buffer，用于保存实际的数据

## 示例
### 数组类型
![avatar](https://raw.githubusercontent.com/wangbo/wangbo.github.io/master/_posts/image/p1.png)

- 查找一组数据中的第n个元素
    - value array起始偏移量 = offset array[n - 1]
    - value array终止偏移量 = value array起始偏移量 + offset array[n] - 1

### 结构体
![avatar](https://raw.githubusercontent.com/wangbo/wangbo.github.io/master/_posts/image/p1.png)

- 结构体的每个字段都被作为一个数组存储到一片连续的内存区域，以此可以实现更加复杂的数据结构；但是嵌套层级越多，相应的查找数据的时间也会线性增加。

## 对齐与填充
由于Arrow是基于内存地址的内存分配，因此连续内存区域的大小推荐都是8的倍数，推荐值为64字节。主要是面向硬件层次的优化，例如CPU缓存友好，可以更有效的利用SIMD寄存器特性。

## Java实现
### 描述连续内存
Java的堆内的内存分配，主要是基于对象。如果想实现基于内存地址的分配，需要依赖相对底层的接口，Arrow中是基于netty封装的接口实现，而netty又是基于Java的UNSAFE接口，UNSAFE提供了分配内存地址以及基于地址和偏移量的数据访问接口。

### 数据类型的存储
在Arrow的Java实现中，实际写入内存区域的数据类型主要是整型和字节数组，因此其他类型例如浮点和字符串，都需要通过一些方式转成整型再进行存储。例如浮点需要转成bit的格式(Double.doubleToRawLongBits)，字符串需要转成字节数组。

# 内存管理
Arrow基于树的模型实现内存的管理。以下分析均基于Java版的实现

## 关键组件与对应接口
- 直接内存的分配与读写
    - UnsafeDirectLittleEndian，比较底层的接口，通常不会暴露给用户
- 应用层的内存分配接口
    -  Allocator，用户可以直接使用的接口，通过该接口完成内存的分配操作
- 内存限制与用量记录
    - Accountant，该接口为Allocator的父类，在每次分配内存时，记录每个Allcator的用量
- 连续内存区域的归属和移交
    - AllocationManager，维护了一片内存区域(UnsafeDirectLittleEndian)，该内存区域可以分配给多个Allocator，AllocationManager也维护了Allocator和从该Allocator分配出去的内存区域的关系
    - BufferLedger，负责创建和维护一组ArrowBuf。由于一个BufferLedger对应一个Allocator，因此也维护了Allocator和内存区域的映射关系
- 内存访问
    - ArrowBuf，基于UnsafeDirectLittleEndian的封装
    - 一个UDLE可以划分给多个ArrowBuf

### Allocator视角
<pre>
+ RootAllocator
|-+ ChildAllocator 1
| | - ChildAllocator 1.1
| ` ...
|
|-+ ChildAllocator 2
|-+ ChildAllocator 3
| |
| |-+ BufferLedger 1 ==> AllocationManager 1 ==> UDLE
| | `- ArrowBuf 1
| `-+ BufferLedger 2 ==> AllocationManager 2 ==> UDLE
| 	`- ArrowBuf 2
|
|-+ BufferLedger 3 ==> AllocationManager 1 ==> UDLE
| ` - ArrowBuf 3
|-+ BufferLedger 4 ==> AllocationManager 2 ==> UDLE
  | - ArrowBuf 4
  | - ArrowBuf 5
  ` - ArrowBuf 6
</pre>

## 内存分配流程
1. 首先创建一个RootAllocator，通常一个JVM一个。可分配的内存大小在创建Allocator时指定。
2. 基于RootAllocator，创建ChildAllocator。
3. 创建一个list，该list保存了Allocator的引用。
4. 向list中写入元素
    - 向Allocator申请一片固定大小的内存
        - 当前Allocator的内存余量无法满足需求的量时，就向父级Allcator申请。如果最终都无法满足内存申请的需求，会向应用程序抛出异常；否则就返回成功。
    - 完成内存的分配之后，就可以开始向已申请的固定大小的内存区域写入数据。
    - 每次写入时需要判断已申请的固定内存剩余大小是否足够写入新的数据如果剩余大小不足的话，需要发起新的内存分配请求

### 内存分配流程中的一些细节
- data buffer，validity buffer和offset buffer都需要计入内存的使用量
- 每次向Allocator申请的内存大小计算方式为，固定元素个数*类型的宽度，然后需要转成8的倍数
- 写入数据时的内存用量计算
    - 对于定长类型，计算元素个数即可
    - 对于变长类型，例如字符串，需要转成字节数组，数组的长度就代表写入数据的大小
- 关于连续内存区域的扩容操作
    - 每次扩容容量都需要转成原来的2倍
    - 且存在内存中的数据拷贝操作

## 内存所有权的共享与转移
一个Allocator可以拥有一个或多个连续的内存区域的所有权，为了应对现实中相对复杂的场景，内存的所有权还需要能够完成共享和转移。例如算子之间的数据交换。

### 举个例子
ChildAllocator 1的ArrowBuf 1，移交给ChildAllocator 2

#### 逻辑视角
<pre>
移交前
+ RootAllocator
|-+ ChildAllocator 1
| |
| |-+ BufferLedger 1 ==> AllocationManager 1 ==> UDLE
|   `- ArrowBuf 1
|
|-+ ChildAllocator 2
</pre>

<pre>
移交后
+ RootAllocator
|-+ ChildAllocator 1
|
|-+ ChildAllocator 2
| |
| |-+ BufferLedger 2 ==> AllocationManager 1 ==> UDLE
|   `- ArrowBuf 2
</pre>

#### 代码视角
移交过程类似于就是ChildAllocator 2向AllocationManager 1申请了一片内存区域，但是和正常的分配流程存在两点不同
1. 不会分配新的内存地址，只是完成了内存地址的转移
2. 分配过程不会因为ChildAllocator 2的内存使用超出限制而失败。在Arrow看来，内存归属移交的优先级高于内存限制，经常确认当前Allocator的用量是否超出限制应该是应用代码的责任。

# 总结
- 堆外分配
    - 对于比较轻量级的查询，走堆内更加合适，可以避免数据序列化以及内存分配的开销
    - 对于比较重量级（非批处理）的查询，走堆外可以减轻GC压力
- 基于内存地址和偏移量的分配方式，使得开发者可以控制数据在内存中的对齐和填充方式，从而可以充分利用硬件的特性；相对应的，内存分配的实现逻辑也会更加复杂。
- 内存大小测量的方式比较简单，测量成本也比较低。
- 内存管理上，由于内存在Allocator之间的转移的情况下，需要应用程序频繁确认内存使用是否超限，因此在判断上存在一定的滞后性。
- Arrow目前的内存控制还只是单节点的实现，如果要应用于分布式的查询场景，还需要额外的改造。

# 参考文献
[Apache Arrow官方文档](https://arrow.apache.org/)

[Apache Arrow源码](https://github.com/apache/arrow)

[堆外内存分配](https://github.com/apache/arrow)