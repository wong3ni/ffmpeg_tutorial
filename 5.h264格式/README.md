# H264格式解析

**注意：本文只介绍编程中需要用到的部分，关于更深的原理，本文不会涉及。**

## 基本结构

h264基本结构如下，由`Start Code`和`NALU`组成。

<img src="README.assets/image-20210911154728614.png" alt="image-20210911154728614" style="zoom:50%;" />

其中`Start Code`用来分割`NALU`，值为`0x000001`(3字节)或者`0x00000001`(4字节)。网上很多地方没有说清楚`Start Code`什么时候是3字节，什么时候是4字节，大部分都是一笔带过。要弄清楚就需要知道`slice`这个东西，有时候一帧数据并不会全部包含在`NALU`，而是可能会被分割成多个`slice`(具体是啥我也不清楚)，放在多个`NALU`中，当完整的帧被分割成多个`slice`时，包含这些`slice`的`NALU`就使用3字节，其它情况都用4字节。简单来说当遇到4字节的起始码就意味着是一个新的数据开始，而当遇到3字节的起始码就意味着这个`NALU`的数据是被分割的。

`NALU`里面包含了编码的数据，它的结构如下：

<img src="README.assets/image-20210911160538605.png" alt="image-20210911160538605" style="zoom:50%;" />

`Header`占一个字节，为`NALU`头部，`Palylaod`为携带的数据，具体是什么暂不考虑。

`Header`各个字段如下：

```text
| forbidden_zero_bit | nal_ref_idc | nal_unit_type |
|--------------------+-------------+---------------|
|        1 bit       |    2 bit    |    5 bit      |
```

第一个为禁止位，置为1表示这个`NALU`无效。

第二个表示优先级，数值越大越重要。

第三个表示这个`NALU`的类型，如`pps`或`sps`等。

## 参考文献

https://blog.csdn.net/charleslei/article/details/52770173

https://zhuanlan.zhihu.com/p/71928833
