## Log文件格式
所有的写操作都必须先成功的**append**到操作日志中，然后再更新内存**memtable**。

Leveldb把日志文件切分成了大小为32KB的连续block块，block由连续的log record（一个record可能跨几个块）组成，log record的格式为：
![](../../img/Pasted%20image%2020220909201210.png)
每一个块由头部与内容两部分组成。
头部由4字节校验，2字节的长度与1字节的类型构成，即每一个块的开始7字节属于头部。
头部中的类型字段有如下4种：
```c++
kFullType = 1,说明该log record包含一个完整的user record；
kFirstType = 2,说明是user record的第一条log record
kMiddleType = 3,说明是user record中间的log record
kLastType = 4,说明是user record最后的一条log record
```

kFullType表示一条记录完整地写到了一个块上。当一条记录跨几个块时，kFirstType表示该条记录的第一部分，kMiddleType表示该条记录的中间部分（可能有多个该类型），kLastType表示该条记录的最后一部分。
![](../../img/Pasted%20image%2020220909201532.png)
![](../../img/Pasted%20image%2020220909201542.png)


考虑到如下序列的**user records**： 
A: length 1000 
B: length 97270 
C: length 8000

-   A作为FULL类型的record存储在第一个block中；
-   B将被拆分成3条log record，分别存储在第1、2、3个block中，这时block3还剩6byte，将被填充为0；
-   C将作为FULL类型的record存储在block 4中。

由于**一条logRecord长度最短为7**，如果一个block的剩余空间<=6byte，那么将**被填充为空字符串**，另外长度为7的log record是**不包括任何用户数据的。**

## 写Log
![](../../img/Pasted%20image%2020220909202117.png)

Writer类中的crc，是指在调用AddRecode时，计算的CRC的值。
AddRecode传入的是一个Slice参数。


## 读log
Log文件的读取代码主要位于db/log_reader.h和db/log_reader.cc两个文件。头文件db/log_reader.h中包含一个公共方法ReadRecord：
```C++
 bool ReadRecord(Slice* record, std::string* scratch);
```

ReadRecord方法读取到的记录会保存在record参数中。一个记录可能会跨越几个块，因此ReadRecord方法中包括了一个scratch参数，以该参数作为临时存储，保存或者追加读取到的记录，直到完整读取一条记录之后赋值给record参数并返回。如果成功读取到完整的记录则返回true，否则返回false。
