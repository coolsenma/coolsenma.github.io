# ﻿Redis数据结构-压缩列表


Redis 为了节约内存空间使用，zset 和 hash 容器对象在元素个数较少的时候，采用压缩列表 (ziplist) 进行存储。

ziplist是一个经过特殊编码的双向链表，它的设计目标就是为了提高存储效率。它能以O(1)的时间复杂度在表的两端提供`push`和`pop`操作。ziplist是将表中每一项存放在前后连续的地址空间内，一个ziplist整体占用一大块内存。它是一个表（list），但其实不是一个链表（linked list）。

## ziplist 的构成

一个 ziplist 的典型分布结构：

```
area        |<---- ziplist header ---->|<----------- entries ------------->|<-end->|

size          4 bytes  4 bytes  2 bytes    ?        ?        ?        ?     1 byte
            +---------+--------+-------+--------+--------+--------+--------+-------+
component   | zlbytes | zltail | zllen | entry1 | entry2 |  ...   | entryN | zlend |
            +---------+--------+-------+--------+--------+--------+--------+-------+
                                       ^                          ^        ^
address                                |                          |        |
                                ZIPLIST_ENTRY_HEAD                |   ZIPLIST_ENTRY_END
                                                                  |
                                                         ZIPLIST_ENTRY_TAIL
```

图中各个域的作用如下：

| 域        | 长度/类型  | 域的值                                                       |
| :-------- | :--------- | :----------------------------------------------------------- |
| `zlbytes` | `uint32_t` | 整个 ziplist 占用的内存字节数，对 ziplist 进行内存重分配，或者计算末端时使用。 |
| `zltail`  | `uint32_t` | 到达 ziplist 表尾节点的偏移量。 通过这个偏移量，可以在不遍历整个 ziplist 的前提下，弹出表尾节点。 |
| `zllen`   | `uint16_t` | ziplist 中节点的数量。 当这个值小于 `UINT16_MAX` （`65535`）时，这个值就是 ziplist 中节点的数量； 当这个值等于 `UINT16_MAX` 时，节点的数量需要遍历整个 ziplist 才能计算得出。 |
| `entryX`  | `?`        | ziplist 所保存的节点，各个节点的长度根据内容而定。           |
| `zlend`   | `uint8_t`  | `255` 的二进制值 `1111 1111` （`UINT8_MAX`） ，用于标记 ziplist 的末端。 |

## 节点的构成

一个 ziplist 可以包含多个节点,

```
area        |<------------------- entry -------------------->|

            +------------------+----------+--------+---------+
component   | pre_entry_length | encoding | length | content |
            +------------------+----------+--------+---------+
```

### `pre_entry_length` 

`pre_entry_length` 记录了前一个节点的长度，通过这个值，可以进行指针计算，从而跳转到上一个节点

根据编码方式的不同， `pre_entry_length` 域可能占用 `1` 字节或者 `5` 字节：

- `1` 字节：如果前一节点的长度小于 `254` 字节，便使用一个字节保存它的值。如： 1000 0000 
- `5` 字节：如果前一节点的长度大于等于 `254` 字节，那么将第 `1` 个字节的值设为 `254` ，然后用接下来的 `4` 个字节保存实际长度。如：11111110 00000000000000000010011101100110 

### encoding 和 length

`encoding` 和 `length` 两部分一起决定了 `content` 部分所保存的数据的类型（以及长度）。

其中， `encoding` 域的长度为两个 bit ， 它的值可以是 `00` 、 `01` 、 `10` 和 `11` ：

- `00` 、 `01` 和 `10` 表示 `content` 部分保存着字符数组。
- `11` 表示 `content` 部分保存着整数。

以 `00` 、 `01` 和 `10` 开头的字符数组的编码方式如下：

| 编码                                         | 编码长度 | content 部分保存的值                 |
| :------------------------------------------- | :------- | :----------------------------------- |
| `00bbbbbb`                                   | 1 byte   | 长度小于等于 63 字节的字符数组。     |
| `01bbbbbb xxxxxxxx`                          | 2 byte   | 长度小于等于 16383 字节的字符数组。  |
| `10____ aaaaaaaa bbbbbbbb cccccccc dddddddd` | 5 byte   | 长度小于等于 4294967295 的字符数组。 |

`11` 开头的整数编码如下：

| 编码       | 编码长度 | content 部分保存的值                    |
| :--------- | :------- | :-------------------------------------- |
| `11000000` | 1 byte   | `int16_t` 类型的整数                    |
| `11010000` | 1 byte   | `int32_t` 类型的整数                    |
| `11100000` | 1 byte   | `int64_t` 类型的整数                    |
| `11110000` | 1 byte   | 24 bit 有符号整数                       |
| `11111110` | 1 byte   | 8 bit 有符号整数                        |
| `1111xxxx` | 1 byte   | 4 bit 无符号整数，介于 `0` 至 `12` 之间 |

### content

`content` 部分保存着节点的内容

以下是一个保存着字符数组 `hello world` 的节点的例子：

```
area      |<---------------------- entry ----------------------->|

size        ?                  2 bit      6 bit    11 byte
          +------------------+----------+--------+---------------+
component | pre_entry_length | encoding | length | content       |
          |                  |          |        |               |
value     | ?                |    00    | 001011 | hello world   |
          +------------------+----------+--------+---------------+
```

`encoding` 域的值 `00` 表示节点保存着一个长度小于等于 63 字节的字符数组， `length` 域给出了这个字符数组的准确长度 —— `11` 字节（的二进制 `001011`）， `content` 则保存着字符数组值 `hello world` 本身（为了方便表示， `content` 部分使用字符而不是二进制表示）

## 为什么要用 ziplist

ziplist 由此有性能和内存空间的优势：

* ziplist 比 hashtable 更节省内存
* redis 考虑到 如果数据紧凑的 ziplist 能够放入 CPU 缓存（hashtable 很难，因为它是非线性的），那么查 找算法甚至会比 hashtable 要快！

## 参考

[Redis设计与实现](