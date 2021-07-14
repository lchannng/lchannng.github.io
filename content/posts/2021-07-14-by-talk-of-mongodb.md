---
title: MongoDB杂谈
date: 2021-07-15 21:06:43+0800
tags:
    - MongoDB
---

## 1 序

前一段时间疫情爆发，周末宅家里闲来无事，写点小玩具，期间使用并稍微深入学习了一下 MongoDB 。

本文主要记录一下在学习、使用 MongoDB 过程中遇到的一些问题和学到的一些姿势。

## 2 Objectid

### 2.1 数据结构

在 MongoDB 中，集合中每个文档都需要一个 唯一的 _id 字段作为主键，如果插入的文档没有 _id 字段， MongoDB 会自动生成一个 ObjectId 作为 _id。

ObjectId是一个12字节 BSON 类型数据，最初的数据格式如下：

> - a 4-byte value representing the seconds since the Unix epoch (which will not run out of seconds until the year 2106)
> - a 3-byte machine identifier (usually derived from the MAC address),
> - a 2-byte process id, and
> - a 3-byte counter, starting with a random value.

```
timestamp(big endian)  machine hash    pid        counter
|<----------------->|<------------>|<------->|<------------>|
[----|----|----|----|----|----|----|----|----|----|----|----]
0                   4                   8                   12
```

在 MongoDB 最新版本中，ObjectId 的格式有所变化：

>- a 4-byte *timestamp value*, representing the ObjectId's creation, measured in seconds since the Unix epoch
>- a 5-byte *random value*
>- a 3-byte *incrementing counter*, initialized to a random value

```
timestamp(big endian)   process unique data      counter
|<----------------->|<---------------------->|<------------>|
[----|----|----|----|----|----|----|----|----|----|----|----]
0                   4                   8                   12
```

如上所示，主要变化在中间5个字节的，由原来的 3字节机器名称hash + 2字节进程id改为 5字节随机值，官方driver里面的描述是 system/process unique data。

这个改动的原因可以在 MongoDB 的[特性设计文档](https://github.com/mongodb/specifications/blob/master/source/objectid.rst#random-value)中找到:

>**Random Value:** Originally, this field consisted of the Machine ID and Process ID fields. There were numerous divergences between drivers due to implementation choices, and the Machine ID field traditionally used the MD5 hashing algorithm which can't be used on FIPS compliant machines. In order to allow for a similar behaviour among all drivers **and** the MongoDB Server, these two fields have been collated together into a single 5-byte random value, unique to a machine and process.

大概的原因是因为 MD5 算法不允许在需要遵循 [FIPS 140-2](https://en.wikipedia.org/wiki/FIPS_140-2) (美国政府安全标准)的机器上使用，为了统一各种语言的 driver 和平台下的实现和表现，采用了5字节系统唯一的随机数值作为一个机器的标识码。

除此之外，原来 3 byte + 2 byte的形式在有些使用场景中容易获得同样的数值，举个栗子。

在容器中部署 Mongo 的实例，系统编排启动、重启的一批容器，自动启动Mongo的时候，由于容器使用同样的镜像，执行同样的步骤，很大概率会出现相同pid的情况，而且机器名称也很可能是一样的。

### 2.2 driver实现

MongoDB driver在首次启动/首次生成 ObjectID 的时候，会初始化一次机器识别码和随机一个couter初始值，但是不同driver中间的实现五花八门。

```c
/*C版本 https://github.com/mongodb/mongo-c-driver/blob/454a01422ee61d2add82f054aa3799750b7947e2/src/libbson/src/bson/bson-context.c#L268
* C版本将时间、进程id、主机名称异或得到一个随机种子，再随机5字节机器码和counter初始值。
*/
static void
_bson_context_init_random (bson_context_t *context, bool init_sequence)
{
   int64_t rand_bytes;
   struct timeval tv;
   unsigned int seed = 0;
   char hostname[HOST_NAME_MAX];
   char *ptr;
   int hostname_chars_left;

   /*
    * The seed consists of the following xor'd together:
    * - current time in seconds
    * - current time in milliseconds
    * - current pid
    * - current hostname
    */
   bson_gettimeofday (&tv);
   seed ^= (unsigned int) tv.tv_sec;
   seed ^= (unsigned int) tv.tv_usec;
   seed ^= (unsigned int) context->pid;

   context->gethostname (hostname);
   hostname_chars_left = strlen (hostname);
   ptr = hostname;
   while (hostname_chars_left) {
      uint32_t hostname_chunk = 0;
      uint32_t to_copy = hostname_chars_left > 4 ? 4 : hostname_chars_left;

      memcpy (&hostname_chunk, ptr, to_copy);
      seed ^= (unsigned int) hostname_chunk;
      hostname_chars_left -= to_copy;
      ptr += to_copy;
   }

#ifndef BSON_HAVE_RAND_R
   srand (seed);
#endif

   /* Generate a seed for the random starting position of our increment
    * bytes and the five byte random number. */
   if (init_sequence) {
      /* We mask off the last nibble so that the last digit of the OID will
       * start at zero. Just to be nice. */
      context->seq32 = _get_rand (&seed) & 0x007FFFF0;
   }

   rand_bytes = _get_rand (&seed);
   rand_bytes <<= 32;
   rand_bytes |= _get_rand (&seed);

   /* Copy five random bytes, endianness does not matter. */
   memcpy (&context->rand, (char *) &rand_bytes, sizeof (context->rand));
}
```



```go
/* golang版本 https://github.com/mongodb/mongo-go-driver/blob/master/bson/primitive/objectid.go
* go版本使用 rand.Reader 直接产生随机数，rand.Reader 是一个密码学安全的随机数生成器的公共实例
* rand.Reader在Linux下使用/dev/urandom实现，在windows下使用 RtlGenRandom 实现
*/
...
var objectIDCounter = readRandomUint32()
var processUnique = processUniqueBytes()
...
func processUniqueBytes() [5]byte {
	var b [5]byte
	_, err := io.ReadFull(rand.Reader, b[:])
	if err != nil {
		panic(fmt.Errorf("cannot initialize objectid package with crypto.rand.Reader: %v", err))
	}

	return b
}

func readRandomUint32() uint32 {
	var b [4]byte
	_, err := io.ReadFull(rand.Reader, b[:])
	if err != nil {
		panic(fmt.Errorf("cannot initialize objectid package with crypto.rand.Reader: %v", err))
	}

	return (uint32(b[0]) << 0) | (uint32(b[1]) << 8) | (uint32(b[2]) << 16) | (uint32(b[3]) << 24)
}
```



```python
# python版本 https://github.com/mongodb/mongo-python-driver/blob/master/bson/objectid.py
# python 使用 os.urandom 生成随机数, os.urandom 跟 go 的类似，使用/dev/urandom 或者 系统提供的 api
# os.urandom() method is used to generate a string of size random bytes suitable for cryptographic use or we can say this method generates a string containing random characters.
...
def _random_bytes():
    """Get the 5-byte random field of an ObjectId."""
    return os.urandom(5)


class ObjectId(object):
    """A MongoDB ObjectId.
    """

    _pid = os.getpid()

    _inc = SystemRandom().randint(0, _MAX_COUNTER_VALUE)
    _inc_lock = threading.Lock()

    __random = _random_bytes()
...
```



### 2.3 真的唯一吗

作为一个唯一ID生成方案，ObjectId 真的100%唯一吗？

从实现方案本身分析：

* 当一个进程一秒内生成超过 2^24 个 ObjectId，counter会溢出，得到重复 id
* 机器识别码重复，在一定的时空条件下，在同样的时间戳和couter下生成重复的id

* 一些年久失修的第三方driver错误的实现可能导致一定条件下产生重复id

第1点出现的前提条件达成的概率，毕竟普通服务器单进程千万级别写入/s，cpu要冒烟了。

第2点达成的条件也比较苛刻，但是出现了就真的是，走鬼，见到运了。

第3点比较容易避免，使用官方driver和较新的MongoDB即可。

总的来说，ObjectId 能保证数据库级别的唯一性（不同collection之间），至于系统级的唯一性需要一些防御性的措施做保证。



### 2.4 其他唯一ID生成方案

常见的方案由UUID、SnowFlake等，各有优劣，篇幅有限不展开细说，可查看https://segmentfault.com/a/1190000020993874



##  3 TTL索引

>TTL indexes are special single-field indexes that MongoDB can use to automatically remove documents from a collection after a certain amount of time or at a specific clock time. Data expiration is useful for certain types of information like machine generated event data, logs, and session information that only need to persist in a database for a finite amount of time.

[文档传送门](https://docs.mongodb.com/manual/core/index-ttl/)

TTL全称是(Time To Live),TTL索引能对一个单列配置过期属性来`实现对文档的自动过期删除`，我们可以在对字段创建索引时添加`expireAfterSeconds`选项将索引转换为TTL索引，该`字段需要是date类型`，在以下几种场景下即使索引设置了expireAfterSeconds属性也不会生效
\- 如果该字段不是date类型，则文档不会过期
\- 如果文档没包含索引的这个字段，则文档不会过期

```
# 创建TTL索引
db.yourdb.createIndex( { "fieldKey": 1 }, { expireAfterSeconds: 3600 } )
```

一次偶然的机会，跑了一个driver的测试用例，发现里面有个检查TTL功能的用例，把 expireAfterSeconds 设置成 1，然后开定时器等待几秒后，执行一个assert

```
# 伪代码
db.yourdb.createIndex( { "fieldKey": 1 }, { expireAfterSeconds: 1 } )
db.yourdb.insert({"fieldKey": "hehe"})
sleep(5)
assert(db.yourdb.findOne({"fieldKey": "hehe"}) == null)
```

然而assert失败了，文档还存在，打开 Mongo 客户端上去一看发现确实还在，但是过了一段时间后文档就被删除了。

似乎 MongoDB 是按一定周期去检测、删除过期文档的。

带着疑问，去扒了一下 MongoDB 的源码。

```c++
// https://github.com/mongodb/mongo/blob/master/src/mongo/db/ttl.cpp
...
class TTLMonitor : public BackgroundJob {
    void run() {
        {
            // Wait until either ttlMonitorSleepSecs passes or a shutdown is requested.
            auto deadline = Date_t::now() + Seconds(ttlMonitorSleepSecs.load());
            stdx::unique_lock<Latch> lk(_stateMutex);

            MONGO_IDLE_THREAD_BLOCK;
            _shuttingDownCV.wait_until(
                    lk, deadline.toSystemTimePoint(), [&] { return _shuttingDown; });

            if (_shuttingDown) {
                return;
            }
        }
        ...
        doTTLPass();
        ...
    }
    
    /**
     * Gets all TTL specifications for every collection and deletes expired documents.
     */
    void doTTLPass() {
        ...
    }
}
...

```

MongoDB起了一个后台线程，每间隔 `ttlMonitorSleepSecs` 这段之间检测一次过期的文档并删除，通过搜索 MongoDB 的代码和实测，这个值默认为60秒。

改变`ttlMonitorSleepSecs`

> As of today, it's not possible, but already tracked in [MongoDB JIRA](https://jira.mongodb.org/):
>
> - [`SERVER-6712`](https://jira.mongodb.org/browse/SERVER-6712): *Make TTL Collection background task period user defined (command line option)*
> - [`SERVER-8616`](https://jira.mongodb.org/browse/SERVER-8616): *Adding Tunable to TTL Collection thread*
> - [`SERVER-13937`](https://jira.mongodb.org/browse/SERVER-13937): *Allow setting a window and interval for the TTL monitor*
>
> There's also kind of a workaround - you can turn TTL monitor off and on manually:
>
> ```js
> db.adminCommand({setParameter: 1, ttlMonitorEnabled: false});
> db.adminCommand({setParameter: 1, ttlMonitorEnabled: true});
> ```
>
> **EDIT:** It turned out, that there is a `ttlMonitorSleepSecs` flag. It's mentioned for example [here](http://hassansin.github.io/working-with-mongodb-ttl-index#ttlmonitor-sleep-interval), but it's not mentioned in the official docs.
>
> ```js
> db.adminCommand({setParameter: 1, ttlMonitorSleepSecs: 60});
> ```

[原文地址](https://stackoverflow.com/a/51071023)



## 4 MongoDB Wire Protocol

[MongoDB Wire Protocol](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/) 是一个简单的基于套接字的请求-响应样式协议。Client 端通过常规的 TCP/IP 套接字与数据库服务器通信。

Client 端应使用常规的 TCP/IP 套接字连接到数据库，所有整数都使用低位字节序：即，最低有效字节在前。

协议的消息头

```
struct MsgHeader {
    int32   messageLength; // 消息的总长度，包括这个字段本身
    int32   requestID;     // 请求的标识id，同一个客户端，每次请求都生成一个唯一id
    int32   responseTo;    // 原始的requestID，回复给客户端，客户端用于处理请求回复
    int32   opCode;        // 请求的类型，不同的类型对应不同的消息体
}
```

opCode的类型：

| Opcode Name                                   | Value | Comment                                                      |
| :-------------------------------------------- | :---- | :----------------------------------------------------------- |
| `OP_MSG`                                      | 2013  | Send a message using the format introduced in MongoDB 3.6.   |
| `OP_REPLY`*Deprecated in MongoDB 5.0.*        | 1     | Reply to a client request. `responseTo` is set.              |
| `OP_UPDATE`*Deprecated in MongoDB 5.0.*       | 2001  | Update document.                                             |
| `OP_INSERT`*Deprecated in MongoDB 5.0.*       | 2002  | Insert new document.                                         |
| `RESERVED`                                    | 2003  | Formerly used for OP_GET_BY_OID.                             |
| `OP_QUERY`*Deprecated in MongoDB 5.0.*        | 2004  | Query a collection.                                          |
| `OP_GET_MORE`*Deprecated in MongoDB 5.0.*     | 2005  | Get more data from a query. See Cursors.                     |
| `OP_DELETE`*Deprecated in MongoDB 5.0.*       | 2006  | Delete documents.                                            |
| `OP_KILL_CURSORS`*Deprecated in MongoDB 5.0.* | 2007  | Notify database that the client has finished with the cursor. |
| `OP_COMPRESSED`                               | 2012  | Wraps other opcodes using compression                        |

在较新版本的MongoDB中，很多请求类型都已经废弃，取而代之的是 [OP_MSG ](https://docs.mongodb.com/manual/reference/mongodb-wire-protocol/#std-label-wire-op-msg)([特性设计文档](https://github.com/mongodb/specifications/blob/master/source/message/OP_MSG.rst))

> `OP_MSG` is a bi-directional wire protocol opcode introduced in MongoDB 3.6 with the goal of replacing most existing opcodes, merging their use into one extendable opcode.

按文档的描述，`OP_MSG` 是一个更具拓展性的协议格式，用来取代现有的一些 opcodes，例如 OP_DELETE, OP_QUERY等。

```
OP_MSG {
    MsgHeader header;          // standard message header
    uint32 flagBits;           // message flags
    Sections[] sections;       // data sections
    optional<uint32> checksum; // optional CRC-32C checksum
}
```

篇幅有限，这里不详细展开，后续有空再开新坑扒一扒，这里先立个flag。

也是偶然的一次机会，跑了一次 driver 的测试用例，发现某些请求会出现奇奇怪怪的现象，比如请求没回复，排除了一下是MongoDB版本和driver版本对不上，新版本 MongoDB 废弃了某些协议，因此去扒了一下。



## 5 附录
MongoDB官方文档： https://docs.mongodb.com/manual/

MongoDB特性文档： https://github.com/mongodb/specifications

分布式唯一ID的几种生成方案： https://segmentfault.com/a/1190000020993874

stackoverflow：
- https://stackoverflow.com/questions/4677237/possibility-of-duplicate-mongo-objectids-being-generated-in-two-different-colle
-  https://stackoverflow.com/questions/62118154/why-mongodb-java-driver-uses-random-bytes-instead-of-machine-id-in-objectid



