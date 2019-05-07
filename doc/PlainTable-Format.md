# 介绍

PlainTable是一个RocksDB SST文件格式，针对低延迟纯内存或者非常低延迟的媒介进行了优化。

优势：

- 一个内存中的索引会被建立，使用二分法+哈希的方法替代直接二分查找。
- 绕过快缓存，来避免块拷贝和LRU缓存机制带来的浪费
- 查询的时候避免任何内存拷贝（mmap）。

限制：
- 文件大小需要小于31bits整形。
- 数据压缩不支持
- delta编码不支持
- Iterator.Prev()不支持
- 无前缀Seek不支持
- 加载表比构建索引慢
- 只支持mmap模式

我们有计划消减部分限制。

# 使用

你可以调用 table.h 中的两个工厂方法，NewPlainTableFactory或者NewTotalOrderPlainTableFactory，来生成一个平表的表工厂，他们通过Options.table_factory或者ColumnFamilyOptions.table_factory传递参数。如果你使用前者，你还需要指定一个前缀提取器。

例子：

```cpp

options.table_factory.reset(NewPlainTableFactory());
options.prefix_extractor.reset(NewFixedPrefixTransform(8));

```

或者

```cpp
options.table_factory.reset(NewTotalOrderPlainTableFactory());

```

参考[include/rocksdb/table.h](https://github.com/facebook/rocksdb/blob/master/include/rocksdb/table.h)中关于这两个方法的注释和参数说明。

NewPlainTableFactory创建的表工厂生成的平表会使用基于key前缀的哈希索引。这是PlainTable优化的方向。

NewTotalOrderPlainTableFactory不需要提供前缀提取器，而是使用一个完全的二分索引。这个函数主要用于保证PlainTable的功能完整。我们没有对这个场景做高度查询性能优化

```
### File Format
#### Basic
    <beginning_of_file>
      [data row1]
      [data row1]
      [data row1]
      ...
      [data rowN]
      [Property Block]
      [Footer]                               (定长;从 file_size - sizeof(Footer) 开始)
    <end_of_file>
```

Property Block的格式和脚注都跟[基于块的表格式]()相同。

参考下面了解数据行的格式。

属性块中的两个属性用于读取数据：

- data_size： 文件中数据部分的结束
- fixed_key_len：如果所有key都有相同的长度，就会记录key的长度。否则为0

# 行格式

每个数据行都以下列格式编码：

```
    <beginning of a row>
      编码的key
      数值长度: varint32
      数值字节流
    <end of a row>
```

key编码方式参见下方

# key编码方式

有两种key编码方式：kPlain和kPrefix，在创建平表工厂的时候声明。

## Plain编码

如果key是定长，那么key就直接存储。

如果key不是定长，key会是变长的，以下面方式编码：

```
[length of key: varint32] + user key + internal bytes
```

参考下面内部字节编码了解internal bytes

## 前缀编码

kPrefix编码类型是一个特别的delta编码，如果一行有跟前面一行一样的前缀（根据用户提供的前缀提取器决定），我们可以避免重复key的前缀部分。

在这种情况，有三种编码方式（记得所有的key都是排序好，这样有相同前缀的key就在一起）

- 一个前缀的第一个key，完整key需要被保留
- 一个前缀的第二个key，保存前缀长度，和除前缀以外的部分（或者说，尾缀）即可。
- 一个前缀的第三个或者之后的key，只需要写尾缀部分即可。

我们定义了三个标志来区分完整的key，前缀，尾缀。对于所有三种类型，我们需要一个大小。他们用一下格式编码：

第一个byte的8个bits
```
+----+----+----+----+----+----+----+----+
|  Type   |            Size             |
+----+----+----+----+----+----+----+----+
```

签名两个bit表示他是一个完整的key（00），或者前缀（01）或者尾缀（02）。最后六个bit用于存储大小。如果大小的bit不是全部为1，意味着这个就是key的大小，否则，一个varint32会在这个比特后面被写入。这个varint值+ 0x3F (6个bit全1的值) 就是这个key的大小。这样，比较短的key只要一个比特就可以了。

这里是三种类型的格式：

(1)完整key

```
+----------------------+---------------+----------------+
| Full Key Flag + Size | Full User Key | Internal Bytes |
+----------------------+---------------+----------------+
```

(2)前缀的第二个key

```
+--------------------+--------------------+------------+----------------+
| Prefix Flag + Size | Suffix Flag + Size | Key Suffix | Internal Bytes |
+--------------------+--------------------+------------+----------------+
```

(3) 前缀的第三个或者更后面的key
```
+--------------------+------------+----------------+
| Suffix Flag + Size | Key Suffix | Internal Bytes |
+--------------------+------------+----------------+
```

参考[Internal Bytes]() 以了解Internal Bytes的更多细节

使用这种格式，不用知道前缀，行key只能使用完整key的文件偏移进行查找。所以如果一个前缀有太多key，平表的构筑器可能会决定重写完整的key，及时这个不是该前缀的第一个key，以减轻seek的压力。

这里是一个例子，我们有下面的key（前缀和尾缀通过空格分隔）

```
AAAA AAAB
AAAA AAABA
AAAA AAAC
AAABB AA
AAAC AAAB
```

会被加密成下面的格式

```
FK 8 AAAAAAAB
PF 4 SF 5 AAABA
SF 4 AAAC
FK 7 AAABBAA
FK 8 AAACAAAB
```
这里FK表示完整key flag，PF表示前缀flag，SF表示尾缀Flag

### 内部字节编码(internal bytes encoding)

不管是Plain还是前缀编码类型，内部key的内部字节码都以同一种方式编码。在RocksDB，一个key的内部字节包括一个行类型（值，delete，merge，等）以及一个序列ID。通常，一个key以这样的格式排布：

```
+----------- ...... -------------+----+---+---+---+---+---+---+---+
|       user key                 |type|      sequence ID          |
+----------- ..... --------------+----+---+---+---+---+---+---+---+
```

这里type使用1byte，序列号使用7个byte。

在平表格式，这也是一个key被优化的一个方法。更进一步，我们有一个优化用来保存一个通用例子的一些额外的字节：序列ID为0的值类型。在RocksDB，我们有一个优化，如果确认该key没有更早的值，把序列ID填充为0（或者说，level风格压缩的最后一层的第一个key，或者universal风格的最后一个文件），这样能带来更好的压缩（compress）或者编码结果。在PlainTable，我么使用一个bite"0x80"，而不是8个比特来存储内部字节。

```
 +----------- ...... -------------+----+
 |       user key                 |0x80|
 +----------- ..... --------------+----+
```

# 内存索引格式

## 基本思想

内存索引构建的时候是尽可能压缩打包的。在最高层，索引是一个哈希表，每个桶都是一个文件内的偏移或者一个二分搜索索引。二分搜索在以下两个情况时出现：

- 哈希冲突：两个或者更多前缀被哈希到一个桶里
- 一个前缀有太多key：需要加速该前缀内的搜索

## 格式

索引有两部分内存组成：一个哈希桶用的数组，以及一个带有二分搜索索引的缓冲区（下面称之为“二分搜索缓冲区”，以及“文件”特指SST文件）

根据前缀（使用Options.prefix_extractor提取）的哈希结果，key被哈希到桶。

```
+--------------+------------------------------------------------------+
| Flag (1 bit) | Offset to binary search buffer or file (31 bits)     +
+--------------+------------------------------------------------------+
```

如果Flag为0，并且offset字段等于文件数据的结尾，这意味着null，这个桶没有数据；如果offset比较小，这意味着这个桶只有一个前缀，从该文件的offset位置开始。如果Flag为1，这意味着offset是一个二分搜索缓冲区。该偏移处的格式如下所示：

从二分搜索缓冲区的偏移起始处开始，一个二分搜索索引根据下列方式编码：

```
<begin>
  number_of_records:  varint32
  record 1 file offset:  fixedint32
  record 2 file offset:  fixedint32
  ....
  record N file offset:  fixedint32
<end>
```

这里N = 记录数量。偏移为递增排序。

之所以使用31bit偏移使用1bit来决定是不是二分搜索，只是为了是的索引更加紧凑。

## 一个索引的例子

我们假设这是一个文件的内容：

```
+----------------------------+ <== offset_0003_0000 = 0
| row (key: "0003 0000")     |
+----------------------------+ <== offset_0005_0000
| row (key: "0005 0000")     |
+----------------------------+
| row (key: "0005 0001")     |
+----------------------------+
| row (key: "0005 0002")     |
+----------------------------+
|                            |
|    ....                    |
|                            |
+----------------------------+
| row (key: "0005 000F")     |
+----------------------------+ <== offset_0005_0010
| row (key: "0005 0010")     |
+----------------------------+
|                            |
|    ....                    |
|                            |
+----------------------------+
| row (key: "0005 001F")     |
+----------------------------+ <== offset_0005_0020
| row (key: "0005 0020")     |
+----------------------------+
| row (key: "0005 0021")     |
+----------------------------+
| row (key: "0005 0022")     |
+----------------------------+ <== offset_0007_0000
| row (key: "0007 0000")     |
+----------------------------+
| row (key: "0007 0001")     |
+----------------------------+ <== offset_0008_0000
| row (key: "0008 0000")     |
+----------------------------+
| row (key: "0008 0001")     |
+----------------------------+
| row (key: "0008 0002")     |
+----------------------------+
|                            |
|    ....                    |
|                            |
+----------------------------+
| row (key: "0008 000F")     |
+----------------------------+ <== offset_0008_0010
| row (key: "0008 0010")     |
+----------------------------+ <== offset_end_data
|                            |
| property block and footer  |
|                            |
+----------------------------+
```

假设在这个例子里，我们使用2个字节的固定长度前缀，在每个前缀中，行总是增加1。

现在我们给这个文件构建索引。通过扫描这个文件，我们知道有4个不同的前缀（"0003", "0005", "0007" 和 "0008"），并且假设我们选择使用5个哈希桶，根据哈希函数，前缀被哈希到记个桶：

```
bucket 0: 0005
bucket 1: 空
bucket 2: 0007
bucket 3: 0003 0008
bucket 4: 空
```

bucket2不需要二分搜索，因为那里只有一个前缀（“0007”），并且他只有两行（<16行）

Bucket 0 需要二分搜索，因为前缀0005有多于16个行。

bucket 3 需要二分搜索，因为他有多于一个前缀。

我们需要为bucket 0 和3分配二分搜索索引。这里是结果：

```
+---------------------+ <== bs_offset_bucket_0
+  2 (in varint32)    |
+---------------------+----------------+
+  offset_0005_0000 (in fixedint32)    |
+--------------------------------------+
+  offset_0005_0010 (in fixedint32)    |
+---------------------+----------------+ <== bs_offset_bucket_3
+  3 (in varint32)    |
+---------------------+----------------+
+  offset_0003_0000 (in fixedint32)    |
+--------------------------------------+
+  offset_0008_0000 (in fixedint32)    |
+--------------------------------------+
+  offset_0008_0010 (in fixedint32)    |
+--------------------------------------+
```

然后这里是哈希桶的数据：

```
+---+---------------------------------------+
| 1 |    bs_offset_bucket_0 (31 bits)       |  <=== bucket 0
+---+---------------------------------------+
| 0 |    offset_end_data    (31 bits)       |  <=== bucket 1
+---+---------------------------------------+
| 0 |    offset_0007_0000   (31 bits)       |  <=== bucket 2
+---+---------------------------------------+
| 1 |    bs_offset_bucket_3 (31 bits)       |  <=== bucket 3
+---+---------------------------------------+
| 0 |    offset_end_data    (31 bits)       |  <=== bucket 4
+---+---------------------------------------+
```

## 索引查找

为了找到一个key，首先通过Options.prefix_extractor计算key的前缀，然后找到该前缀的桶。否则，如果桶没有记录在里面（Flag为0且偏移指向文件数据末尾），那么key就找不到。

如果Flag=0，这意味着只有一个前缀在桶里并且该前缀没有很多key，所以偏移字段的指针指向索引的文件偏移。我们只需要在那里做现行搜搜就可以了。

如果Flag=1，那里有一个二分搜索树。二分搜索缓冲区可以从偏移字段指向的地方取出。二分搜索之后，通过线性搜索查找二分搜索中找到的偏移量。

## 构建索引

当构建索引的时候，扫描文件。对于每个key，计算他的前缀，从第一个开始，对于每个前缀的第（16n+1）行(n=0,1,2....)，记住（前缀的哈希值，偏移）信息。16是在二分搜索之后需要检查线性搜索的行的最大数量。通过增加这个数字，我们可以减少内存消耗，但是需要付出更多线性搜索的时间。减少这个值则相反。基于前缀的数量，决定一个合适的桶的大小。根据桶的大小，分配需要用来填充索引的额外的桶和二分搜索缓冲区。

## Bloom过滤器

可以配置一个用于查询的作用在前缀上的bloom过滤器。用户可以配置每个前缀使用多少bit。当做查询的时候（Seek()或者Get()），bloom过滤器会在查找索引前检查并过滤不存在的前缀。

# 未来的计划

- 可能考虑把这个索引当成SST文件的一部分来实现
- 增加一个选项来移除大小的限制，这会需要额外的索引的内存。
- 可能建立额外的稀疏索引来实现通用迭代搜索的。
