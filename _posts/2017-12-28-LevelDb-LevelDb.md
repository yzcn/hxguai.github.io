---
layout: post
title: c-ares dns 异步请求库简明教程
date:   2017-05-01 12:00:00
categories: [c, c++]
description: c c++ clang clang++ c-ares
keywords: clang/clang++, c-ares
---

## 简介

---
layout: post
title:  leveldb 学习记录
date:   2017-12-28 22:00:11
category: "leveldb"
---

# LevelDb 学习实践  

## 概述

&emsp;&emsp;LevelDb 是 google 开源的 key/value 存储系统，它的 committer 阵容相当强大，基本上是 bigtable 的原班人马，包括像 Jeff Dean 和 Sanjay Ghemawat 这样的大牛，它的代码合设计非常具有借鉴意义，是一种典型的 LSM Tree的 KV 引擎的实现，从它的数据结构来看，基本就是 sstable 的开源实现，而且针对各种平台作了 port，目前被用在 chrome 等项目中。

&emsp;&emsp; LevelDb是能够处理十亿级别规模Key-Value型数据持久性存储的C++ 程序库，就是说他不提供像 mysql等数据库一样的 C/S 模式服务，它只是一个C++库，提供一些接口对文件进行读写。

### LevelDb有如下一些特点：
* 首先，LevelDb是一个持久化存储的KV系统，和Redis这种内存型的KV系统不同，LevelDb不会像Redis一样狂吃内存，而是将大部分数据存储到磁盘上。

* 其次，LevleDb在存储数据时，是根据记录的key值有序存储的，就是说相邻的key值在存储文件中是依次顺序存储的，而应用可以自定义key大小比较函数，LevleDb会按照用户定义的比较函数依序存储这些记录。

* 再次，像大多数KV系统一样，LevelDb的操作接口很简单，基本操作包括写记录，读记录以及删除记录。也支持针对多条操作的原子批量操作。

* 另外，LevelDb支持数据快照（snapshot）功能，使得读取操作不受写操作影响，可以在读操作过程中始终看到一致的数据。

除此外，LevelDb还支持数据压缩等操作，这对于减小存储空间以及增快IO效率都有直接的帮助。

## LevelDB 原理
&emsp;&emsp; LevelDb本质上是一套存储系统以及在这套存储系统上提供的一些操作接口。为了便于理解整个系统及其处理流程，我们可以从两个不同的角度来看待LevleDb：静态角度和动态角度。从静态角度，可以假想整个系统正在运行过程中（不断插入删除读取数据），此时我们给LevelDb照相，从照片可以看到之前系统的数据在内存和磁盘中是如何分布的，处于什么状态等；从动态的角度，主要是了解系统是如何写入一条记录，读出一条记录，删除一条记录的，同时也包括除了这些接口操作外的内部操作比如compaction，系统运行时崩溃后如何恢复系统等等方面。

&emsp;&emsp; 本节所讲的整体架构主要从静态角度来描述，之后接下来的几节内容会详述静态结构涉及到的文件或者内存数据结构，LevelDb日知录后半部分主要介绍动态视角下的LevelDb，就是说整个系统是怎么运转起来的。

&emsp;&emsp; LevelDb作为存储系统，数据记录的存储介质包括内存以及磁盘文件，如果像上面说的，当LevelDb运行了一段时间，此时我们给LevelDb进行透视拍照，那么您会看到如下一番景象：  
![structual](/images/posts/leveldb/leveldb_structural.png)

&emsp;&emsp; LevelDb 使用了[LSM](http://blog.csdn.net/v_july_v/article/details/7526689)（Log Structured Merge Tree)算法，修改、添加、删除都是先对 Log 进行操作，这样可以保证系统崩溃时也可以从 Log 恢复，对数据的修改、添加、删除都是以追加的方式操作，对连续存储的追加操作极大的提高了数据处理速度。当应用写入一条 Key:Value 记录的时候，LevelDb 会先往 log 文件里写入，成功后将记录插进 Memtable 中，这样基本就算完成了写入操作，因为一次写入操作只涉及一次磁盘顺序写和一次内存写入，所以这是为何说LevelDb写入速度极快的主要原因。

&emsp;&emsp; 当Memtable插入的数据占用内存到了一个界限后，需要将内存的记录导出到外存文件中，LevleDb会生成新的Log文件和Memtable，原先的Memtable就成为Immutable Memtable，顾名思义，就是说这个Memtable的内容是不可更改的，只能读不能写入或者删除。新到来的数据被记入新的Log文件和Memtable，LevelDb后台调度会将Immutable Memtable的数据导出到磁盘，形成一个新的SSTable文件。SSTable就是由内存中的数据不断导出并进行Compaction操作后形成的，而且SSTable的所有文件是一种层级结构，第一层为Level 0，第二层为Level 1，依次类推，层级逐渐增高，这也是为何称之为LevelDb的原因。

&emsp;&emsp; SSTable中的文件是Key有序的，就是说在文件中小key记录排在大Key记录之前，各个Level的SSTable都是如此，但是这里需要注意的一点是：Level 0的SSTable文件（后缀为.sst）和其它Level的文件相比有特殊性：这个层级内的.sst文件，两个文件可能存在key重叠，比如有两个level 0的sst文件，文件A和文件B，文件A的key范围是：{bar, car}，文件B的Key范围是{blue,samecity}，那么很可能两个文件都存在key=”blood”的记录。对于其它Level的SSTable文件来说，则不会出现同一层级内.sst文件的key重叠现象，就是说Level L中任意两个.sst文件，那么可以保证它们的key值是不会重叠的。这点需要特别注意，后面您会看到很多操作的差异都是由于这个原因造成的。

&emsp;&emsp; SSTable中的某个文件属于特定层级，而且其存储的记录是key有序的，那么必然有文件中的最小key和最大key，这是非常重要的信息，LevelDb应该记下这些信息。Manifest就是干这个的，它记载了SSTable各个文件的管理信息，比如属于哪个Level，文件名称叫啥，最小key和最大key各自是多少。

&emsp;&emsp; Current文件是干什么的呢？这个文件的内容只有一个信息，就是记载当前的manifest文件名。因为在LevleDb的运行过程中，随着Compaction的进行，SSTable文件会发生变化，会有新的文件产生，老的文件被废弃，Manifest也会跟着反映这种变化，此时往往会新生成Manifest文件来记载这种变化，而Current则用来指出哪个Manifest文件才是我们关心的那个Manifest文件。

&emsp;&emsp; LevelDb对于一个log文件，会把它切割成以32K为单位的物理Block，每次读取的单位以一个Block作为基本读取单位，下图展示的log文件由3个Block构成，所以从物理布局来讲，一个log文件就是由连续的32K大小Block构成的。
### Memtable
&emsp;&emsp;  所有KV数据都是存储在Memtable，Immutable Memtable和SSTable中的，Immutable Memtable从结构上讲和Memtable是完全一样的，区别仅仅在于其是只读的，不允许写入操作，而Memtable则是允许写入和读取的。当Memtable写入的数据占用内存到达指定数量，则自动转换为Immutable Memtable，等待Dump到磁盘中，系统会自动生成新的Memtable供写操作写入新数据，理解了Memtable，那么Immutable Memtable自然不在话下。 

&emsp;&emsp; LevelDb的MemTable提供了将KV数据写入，删除以及读取KV记录的操作接口，但是事实上Memtable并不存在真正的删除操作,删除某个Key的Value在Memtable内是作为插入一条记录实施的，但是会打上一个Key的删除标记，真正的删除操作是Lazy的，会在以后的Compaction过程中去掉这个KV。需要注意的是，LevelDb的Memtable中KV对是根据Key大小有序存储的，在系统插入新的KV时，LevelDb要把这个KV插到合适的位置上以保持这种Key有序性。其实，LevelDb的Memtable类只是一个接口类，真正的操作是通过背后的SkipList来做的，包括插入操作和读取操作等，所以Memtable的核心数据结构是一个[SkipList](http://www.cnblogs.com/xuqiang/archive/2011/05/22/2053516.html)。SkipList不仅是维护有序数据的一个简单实现，而且相比较平衡树来说，在插入数据的时候可以避免频繁的树节点调整操作，所以写入效率是很高的，LevelDb整体而言是个高写入系统，SkipList在其中应该也起到了很重要的作用。Redis为了加快插入操作，也使用了SkipList来作为内部实现数据结构。

### 读取记录

![get_item](/images/posts/leveldb/get_item.png)

&emsp;&emsp; LevelDb首先会去查看内存中的Memtable，如果Memtable中包含key及其对应的value，则返回value值即可；如果在Memtable没有读到key，则接下来到同样处于内存中的Immutable Memtable中去读取，类似地，如果读到就返回，若是没有读到,那么只能万般无奈下从磁盘中的大量SSTable文件中查找。因为SSTable数量较多，而且分成多个Level，所以在SSTable中读数据是相当蜿蜒曲折的一段旅程。总的读取原则是这样的：首先从属于level 0的文件中查找，如果找到则返回对应的value值，如果没有找到那么到level 1中的文件中去找，如此循环往复，直到在某层SSTable文件中找到这个key对应的value为止（或者查到最高level，查找失败，说明整个系统中不存在这个Key)。

&emsp;&emsp; 那么为什么是从Memtable到Immutable Memtable，再从Immutable Memtable到文件，而文件中为何是从低level到高level这么一个查询路径呢？道理何在？之所以选择这么个查询路径，是因为从信息的更新时间来说，很明显Memtable存储的是最新鲜的KV对；Immutable Memtable中存储的KV数据对的新鲜程度次之；而所有SSTable文件中的KV数据新鲜程度一定不如内存中的Memtable和Immutable Memtable的。对于SSTable文件来说，如果同时在level L和Level L+1找到同一个key，level L的信息一定比level L+1的要新。也就是说，上面列出的查找路径就是按照数据新鲜程度排列出来的，越新鲜的越先查找。

### 数据修改

&emsp;&emsp; 本节介绍levelDb的记录更新操作，即插入一条KV记录或者删除一条KV记录。levelDb的更新操作速度是非常快的，源于其内部机制决定了这种更新操作的简单性。

![insert](/images/posts/leveldb/insert_item.png)

&emsp;&emsp; 对于一个插入操作 Put(Key,Value) 来说，完成插入操作包含两个具体步骤：首先是将这条KV记录以顺序写的方式追加到之前介绍过的 log 文件末尾，因为尽管这是一个磁盘读写操作，但是文件的顺序追加写入效率是很高的，所以并不会导致写入速度的降低；第二个步骤是:如果写入 log 文件成功，那么将这条 KV 记录插入内存中的 Memtable 中，前面介绍过，Memtable 只是一层封装，其内部其实是一个 Key 有序的 SkipList 列表，插入一条新记录的过程也很简单，即先查找合适的插入位置，然后修改相应的链接指针将新记录插入即可。完成这一步，写入记录就算完成了，所以一个插入记录操作涉及一次磁盘文件追加写和内存SkipList 插入操作，这是为何 levelDb 写入速度如此高效的根本原因。

&emsp;&emsp; 从上面的介绍过程中也可以看出：log文件内是key无序的，而Memtable中是key有序的。那么如果是删除一条KV记录呢？对于levelDb来说，并不存在立即删除的操作，而是与插入操作相同的，区别是，插入操作插入的是Key:Value 值，而删除操作插入的是“Key:删除标记”，并不真正去删除记录，而是后台Compaction的时候才去做真正的删除操作。

&emsp;&emsp; 同样对于一条数据的修改也并不会真的去找到记录进行修改，修改一条记录时只是简单的进行一次写操作，LevelDb 中 Memtable的数据一定比 Immutable Memtable 更新， 而 Immutable Memtable 中的数据一定比 sst 文件中的新， 因此查询时如果在 Memtable 中找到了记录，那么这条记录一定是最新的，查询中止；否则若在 Immutable Memtable 中找到则这条记录是最新的查询中止。在数据归并时遇到相同记录只保留最新记录，完成一次真实的删除。

&emsp;&emsp; SSTable文件很多，如何快速地找到key对应的value值？在LevelDb中，level 0一直都爱搞特殊化，在level 0和其它level中查找某个key的过程是不一样的。因为level 0下的不同文件可能key的范围有重叠，某个要查询的key有可能多个文件都包含，这样的话LevelDb的策略是先找出level 0中哪些文件包含这个key（manifest文件中记载了level和对应的文件及文件里key的范围信息，LevelDb在内存中保留这种映射表）， 之后按照文件的新鲜程度排序，新的文件排在前面，之后依次查找，读出key对应的value。而如果是非level 0的话，因为这个level的文件之间key是不重叠的，所以只从一个文件就可以找到key对应的value。

&emsp;&emsp; 最后一个问题,如果给定一个要查询的key和某个key range包含这个key的SSTable文件，那么levelDb是如何进行具体查找过程的呢？levelDb一般会先在内存中的Cache中查找是否包含这个文件的缓存记录，如果包含，则从缓存中读取；如果不包含，则打开SSTable文件，同时将这个文件的索引部分加载到内存中并放入Cache中。 这样Cache里面就有了这个SSTable的缓存项，但是只有索引部分在内存中，之后levelDb根据索引可以定位到哪个内容Block会包含这条key，从文件中读出这个Block的内容，在根据记录一一比较，如果找到则返回结果，如果没有找到，那么说明这个level的SSTable文件并不包含这个key，所以到下一级别的SSTable中去查找。

### 数据压缩

&emsp;&emsp; 前文有述，对于LevelDb来说，写入记录操作很简单，删除记录仅仅写入一个删除标记就算完事，但是读取记录比较复杂，需要在内存以及各个层级文件中依照新鲜程度依次查找，代价很高。为了加快读取速度，levelDb采取了compaction的方式来对已有的记录进行整理压缩，通过这种方式，来删除掉一些不再有效的KV数据，减小数据规模，减少文件数量等。

&emsp;&emsp; levelDb的compaction机制和过程与Bigtable所讲述的是基本一致的，Bigtable中讲到三种类型的compaction: minor ，major和full。所谓minor Compaction，就是把memtable中的数据导出到SSTable文件中；major compaction就是合并不同层级的SSTable文件，而full compaction就是将所有SSTable进行合并。

![compaction](/images/posts/leveldb/compaction.png)

&emsp;&emsp; LevelDb 包含其中两种，minor 和 major。Minor compaction 的目的是当内存中的 memtable 大小到了一定值时，将内容保存到磁盘文件中,而 major compaction 是某个 level 下的 SSTable 文件数目超过一定设置值后，levelDb 会从这个 level 的 SSTable 中选择一个文件（level>0），将其和高一层级的 level+1 的 SSTable 文件合并。

&emsp;&emsp;  当memtable数量到了一定程度会转换为immutable memtable，此时不能往其中写入记录，只能从中读取KV内容。之前介绍过，immutable memtable其实是一个多层级队列SkipList，其中的记录是根据key有序排列的。所以这个minor compaction实现起来也很简单，就是按照immutable memtable中记录由小到大遍历，并依次写入一个level 0 的新建SSTable文件中，写完后建立文件的index 数据，这样就完成了一次minor compaction。从图中也可以看出，对于被删除的记录，在minor compaction过程中并不真正删除这个记录，原因也很简单，这里只知道要删掉key记录，但是这个KV数据在哪里?那需要复杂的查找，所以在minor compaction的时候并不做删除，只是将这个key作为一个记录写入文件中，至于真正的删除操作，在以后更高层级的compaction中会去做。

&emsp;&emsp; 我们知道在大于 0 的层级中，每个 SSTable 文件内的 Key 都是由小到大有序存储的，而且不同文件之间的 key 范围（文件内最小 key 和最大 key 之间）不会有任何重叠。Level 0 的 SSTable 文件有些特殊，尽管每个文件也是根据 Key 由小到大排列，但是因为 level 0 的文件是通过 minor compaction 直接生成的，所以任意两个 level 0 下的两个 sstable 文件可能再key范围上有重叠。所以在做 major compaction 的时候，对于大于 level 0 的层级，选择其中一个文件就行，但是对于 level 0 来说，指定某个文件后，本 level 中很可能有其他 SSTable 文件的 key 范围和这个文件有重叠，这种情况下，要找出所有有重叠的文件和 level 1 的文件进行合并，即 level 0 在进行文件选择的时候，可能会有多个文件参与 major compaction。

&emsp;&emsp; levelDb在选定某个level进行compaction后，还要选择是具体哪个文件要进行compaction，levelDb在这里有个小技巧， 就是说轮流来，比如这次是文件A进行compaction，那么下次就是在key range上紧挨着文件A的文件B进行compaction，这样每个文件都会有机会轮流和高层的level 文件进行合并。如果选好了level L的文件A和level L+1层的文件进行合并，那么问题又来了，应该选择level L+1哪些文件进行合并？levelDb选择L+1层中和文件A在key range上有重叠的所有文件来和文件A进行合并。也就是说，选定了level L的文件A,之后在level L+1中找到了所有需要合并的文件B,C,D…..等等。剩下的问题就是具体是如何进行major 合并的？就是说给定了一系列文件，每个文件内部是key有序的，如何对这些文件进行合并，使得新生成的文件仍然Key有序，同时抛掉哪些不再有价值的KV 数据。

![compaction_1](/images/posts/leveldb/compaction_1.png)

&emsp;&emsp; Major compaction的过程如下：对多个文件采用多路归并排序的方式，依次找出其中最小的Key记录，也就是对多个文件中的所有记录重新进行排序。之后采取一定的标准判断这个Key是否还需要保存，如果判断没有保存价值，那么直接抛掉，如果觉得还需要继续保存，那么就将其写入level L+1层中新生成的一个SSTable文件中。就这样对KV数据一一处理，形成了一系列新的L+1层数据文件，之前的L层文件和L+1层参与compaction 的文件数据此时已经没有意义了，所以全部删除。这样就完成了L层和L+1层文件记录的合并过程。

&emsp;&emsp; 那么在major compaction过程中，判断一个KV记录是否抛弃的标准是什么呢？其中一个标准是:对于某个key来说，如果在小于L层中存在这个Key，那么这个KV在major compaction过程中可以抛掉。因为我们前面分析过，对于层级低于L的文件中如果存在同一Key的记录，那么说明对于Key来说，有更新鲜的Value存在，那么过去的Value就等于没有意义了，所以可以删除。

## 源码

### 下载  
git clone https://github.com/google/leveldb.git

### 编译

cd leveldb & make all
编译完成之后在当前目录多了两个目录：out-shared 和 out-static
在 out-static 目录下有我们需要的 libleveldb.a

### 测试 

在当前目录新建文件夹 test  
touch test; cd test  
touch test.cpp  

test.cpp:  

	#include <assert.h>
	#include <string.h>
	#include <iostream>
	#include "leveldb/db.h"
	
	int main(){
	        leveldb::DB* db;
	        leveldb::Options options;
	        options.create_if_missing = true;
	        leveldb::Status status = leveldb::DB::Open(options,"/tmp/testdb", &db);
	        assert(status.ok());
	
	        std::string k1 = "name";
	        std::string v1 = "jim";
	
	        status = db->Put(leveldb::WriteOptions(), k1, v1);
	        assert(status.ok());
	
	        status = db->Get(leveldb::ReadOptions(), k1, &v1);
	        assert(status.ok());
	        std::cout<<"k1:"<<k1<<"; v1:"<<v1<<std::endl;
	        
	        std::string k2 = "age";
	        std::string v2 = "20";
	
	        status = db->Put(leveldb::WriteOptions(), k2, v2);
	        assert(status.ok());
	        status = db->Get(leveldb::ReadOptions(), k2, &v2);
	        assert(status.ok());
	        std::cout<<"k2:"<<k2<<"; v2:"<<v2<<std::endl;
	
	        status = db->Delete(leveldb::WriteOptions(), k2);
	        assert(status.ok());
	        std::cout<<"Delete k2.."<<std::endl;
	        status = db->Get(leveldb::ReadOptions(),k2, &v2);
	        if(!status.ok())
	            std::cerr<<"k2:"<<k2<<"; "<<status.ToString()<<std::endl;
	        else
	            std::cout<<"k2:"<<k2<<"; v2:"<<v2<<std::endl;
	
	        delete db;
	        return 0;
	}

把 libleveldb.a 拷贝到当前目录  
cp ../out-static/libleveldb.a ./  
把 leveldb/include 目录添加到 PATH :  
cd ..; export PATH=$PATH:$(pwd)/include; cd test  

编译：  
g++ -o test test.cpp libleveldb.a -lpthread -I../include  
运行：  
	
	➜  test git:(master) ✗ ./test
	k1:name; v1:jim
	k2:age; v2:20
	Delete k2..
	k2:age; NotFound:

## 测试
&emsp;&emsp; 对 LevelDb 接口做简单封装后进行测试，在一台如下配置虚拟机上进行写测试，写入5000w条记录，key 字段长度最多 12 位，value 字段长度最多92位，写入速度约为15w/s，其中包含了的io等待时间，对写磁盘块进行调优后应该能有更快速度。

CPU: intel 3.4 G  
Mem: 512 G  
Disk: 50 G  

代码如下：  
LDB.h  

	#ifndef LDB_CLASS_H_
	#define LDB_CLASS_H_
	#include <assert.h>
	#include "leveldb/db.h"
	#include "leveldb/write_batch.h"
	#include "leveldb/options.h"
	#include <iostream>
	
	
	using namespace leveldb;
	
	typedef leveldb::Status LDBSTATUS;
	typedef leveldb::DB* LDBPTR;
	typedef leveldb::Options LDBOPT;
	typedef leveldb::ReadOptions LDBROPT;
	typedef leveldb::WriteOptions LDBWOPT;
	
	class Batch
	{
	private:
		DB* ldbptr;
		leveldb::WriteBatch batch;
	public:
		Batch(DB* ptr):ldbptr(ptr){};
		~Batch(){};
		void Write(const std::string &k, const std::string &v);
		void Delete(const std::string &k);
		LDBSTATUS DoBatch(const LDBWOPT &opt = WriteOptions());
		void Clear(){batch.Clear();}
	};
	
	class LDB
	{
	private:
		DB* m_ldb;
	public:
		LDB():m_ldb(NULL){};
		~LDB(){Close();};
		LDBSTATUS Open(std::string dbPath, const LDBOPT &opt = LDBOPT());
		LDBSTATUS Read(std::string k, std::string *v, const LDBROPT &opt =  ReadOptions());
		LDBSTATUS Write(std::string k, std::string v, const LDBWOPT &opt = WriteOptions());
		LDBSTATUS Delete(std::string k, const LDBWOPT &opt = WriteOptions());
		void Close();
		DB * GetDb(){return m_ldb;}
	};
	
	#endif 

LDB.cpp  

	#include "LDB.h"
	
	LDBSTATUS LDB::Open(std::string dbPath, const LDBOPT &opt)
	{
		if(!m_ldb)
		    return m_ldb->Open(opt, dbPath, &m_ldb); 
		else
		{
			LDBSTATUS status;
			return status;
		}
	}
	
	LDBSTATUS LDB::Read(std::string k, std::string *v, const LDBROPT &opt)
	{
		return m_ldb->Get(opt, k, v); 
	}
	
	LDBSTATUS LDB::Write(std::string k, std::string v, const LDBWOPT &opt)
	{
		return m_ldb->Put(opt, k, v);
	}
	
	LDBSTATUS LDB::Delete(std::string k, const LDBWOPT &opt)
	{
		return m_ldb->Delete(opt, k);
	}
	
	void LDB::Close()
	{
		if(m_ldb)
		{
			delete m_ldb;
			m_ldb = NULL;
		}
	}
	
	void Batch::Write(const std::string &k, const std::string &v)
	{
		batch.Put(k, v);
	}
	
	void Batch::Delete(const std::string &k)
	{
		batch.Delete(k);
	}
	
	LDBSTATUS Batch::DoBatch(const LDBWOPT &opt)
	{
		return ldbptr->Write(opt, &batch);
	}


test.cpp  

	#include "LDB.h"
	#include <time.h>
	#include <iostream>
	#include <sstream>
	#include <unistd.h>
	
	template<class T>
	void to_string(std::string & result,const T& t)
	{
		std::ostringstream oss;
		oss<<t;
		result=oss.str();
	}
	
	int main()
	{
		LDBOPT opts;
		opts.write_buffer_size *= 4; 
		opts.create_if_missing = true;
		LDB ldb;
		LDBSTATUS ldbStatus = ldb.Open("/tmp/tldb", opts);
		if(!ldbStatus.ok())
		{
			std::cout << "open ldb fail" << std::endl;
			exit(1);
		}
	        
		std::string key, k1, k2, k3;
		std::string v1;
		unsigned long long seq;
		srand((unsigned)time(NULL));
	        k1 = "19223892374|2311";
		v1 = "bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb";
	
		/*
		*/
		ldbStatus = ldb.Write(k1, v1);
		assert(ldbStatus.ok());
	
		Batch batch(ldb.GetDb());
	
		for(seq = 0; seq < 50000000; seq++)
		{
			to_string(k1, seq);
			to_string(k2, rand()%10000);
			k1 = k1 + "|" + k2;
			std::string v1 = "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa";
			batch.Write(k1, v1);
			if(seq%5000 == 0)
			{
				ldbStatus = batch.DoBatch();
				assert(ldbStatus.ok());
				batch.Clear();
				std::cout << "write 5000 items, seq:" << seq << std::endl;
			}
		}
		if(seq%100 != 0)
		{
			ldbStatus = batch.DoBatch();
			assert(ldbStatus.ok());
		}
	
		std::cout << "insert data done" << std::endl;
	
		system("date");
	        k1 = "19223892374|2311";
		ldbStatus = ldb.Read(k1, &v1);
		assert(ldbStatus.ok());
	
		std::cout << "K1:" << k1 << std::endl;
		std::cout << "V1:" << v1 << std::endl;
		system("date");
	
	
		ldbStatus = ldb.Delete(k1);
		assert(ldbStatus.ok());
	
		/*
		system("date");
		std::cout << "begin traverse" << std::endl;
		leveldb::Iterator* it = ldb.GetDb()->NewIterator(LDBROPT());
		for(it->SeekToFirst(); it->Valid(); it->Next())
		{
			//std::cout << it->key().ToString() << ":" << it->value().ToString() << std::endl;
		}
		assert(it->status().ok());
		delete it;
		std::cout << "finish traverse" << std::endl;
		system("date");
	
		it = ldb.GetDb()->NewIterator(LDBROPT());
		it->SeekToLast();
		k1 = it->key().ToString();
		it->SeekToFirst();
		std::cout << "K1:" << k1 << std::endl;
		std::cout << "V1:" << v1 << std::endl;
		*/
	
		ldbStatus = ldb.Read(k1, &v1);
		if(!ldbStatus.ok())
			std::cout << "get error:" << ldbStatus.ToString() << std::endl;
		else
			std::cout << "get ok" << std::endl;
		system("date");
	
		ldb.Close();
	}



## 参考
* https://www.cnblogs.com/haippy/archive/2011/12/04/2276064.html
* http://blog.csdn.net/doc_sgl/article/details/52727475
