# Redis add String CAS API Support

According to our business needs, I introduced the lock-free programming CAS data manipulation into Redis 3.2.6 for Redis String storage type.

Specifically, the API includes:
- getcas 
- setcas 
- delcas

### PLAN LIST:
- 对String类型的KV支持CAS操作 `DONE`
- 兼容RDB持久化 `DONE`
- 兼容AOF持久化 `DONE`
- 支持主从复制 `DONE`
- 兼容集群模式 `DONE`

### USAGE：
#### getcas
```sh
>getcas key
return version and value
```
#### setcas
```sh
>setcas key value version
if key is not exist, version must = 0; otherwise setcas will be failed
if key is exist, version must = cas version in key storage, otherwise setcas will be failed
```
#### delcas
```sh
>delcas key
return version and value
```

## 测试
### 测试日记3 2017.01.23

>开发进度：CAS支持主从复制
>
>测试目的：主上的CAS操作可以正确地显示到从上

#### 测试详情：
开启主从服务器

- 主setcas一个kv多次，从上getcas看是否一致 `PASS`
- delcas一个kv，从上getcas看是否一致 `PASS`


### 测试日记2 2017.01.23

>开发进度：CAS支持AOF持久化
>
>测试目的：AOF数据恢复后cas格式数据一切正常


#### 测试详情：
开启Redis的AOF功能：

- setcas一个kv多次，重启server后查看这个kv是否和重启前一致，且是否可以正常CAS操作 `PASS`
- delcas一个kv，重启server后查看这个kv是否被删除了 `PASS`
- setcas一个kv多次，delcas一个kv，重启server后查看这个kv是否被删除了 `PASS`

- setcas一个kv多次，执行AOF重写，重启server后查看这个kv是否和重启前一致，且是否可以正常CAS操作 `PASS`
- setcas一个kv多次，delcas一个kv，执行AOF重写，重启server后查看这个kv是否被删除了 `PASS`

### 测试日记 2017.01.23

>开发进度：修复:RDB恢复数据后无法GETCAS、SETCAS、DELCAS
>
>测试目的：RDB数据恢复后依然可以CAS操作


#### 测试详情：
- setcas一个整数，然后RDB保存，重启server后查看这个kv是否可以正常CAS操作 `PASS`
- setcas一个短字符串，然后RDB保存，重启server后查看这个kv是否可以正常CAS操作 `PASS`
- setcas一个长字符串，然后RDB保存，重启server后查看这个kv是否可以正常CAS操作 `PASS`

### 测试日记 2017.01.22

>开发进度：基本实现，单机 + 不考虑集群、不考虑RDB\AOF、不考虑复制、不考虑订阅发布
>
>测试目的：基本使用

#### 测试详情：
- getcas 非string类型的key `PASS`
- getcas string类型的普通key `PASS`
- getcas stringcas的key `PASS`
- getcas 不存在的key `PASS`

- setcas 不存在的key，version = 0 `PASS`
- setcas 不存在的key，version != 0，是正数、负数、非数字 `PASS`
- setcas 不存在的key，value为数字，version = 0 `PASS`
- setcas 不存在的key，value为<=44字节的字符串，version = 0 `PASS`
- setcas 不存在的key，value为>44字节的字符串，version = 0 `PASS`
- setcas 存在的key，version = 0 `PASS`
- setcas 存在的key，但是key不是string类型 `PASS`
- setcas 存在的key，key是string类型，但是不是RAW类型 `PASS`
- setcas 存在的key，key是string类型、是RAW类型，但不是CAS格式  `PASS`
- setcas 存在的key，key是string类型、是RAW类型、是CAS格式，重置为数字（版本对不上、对上）`PASS`
- setcas 存在的key，key是string类型、是RAW类型、是CAS格式，重置为<=44字节的字符串（版本对不上、对上）`PASS`
- setcas 存在的key，key是string类型、是RAW类型、是CAS格式，重置为>44字节的字符串（版本对不上、对上）`PASS`


- delcas 非string类型的key `PASS`
- delcas string类型的普通key `PASS`
- delcas stringcas的key，但版本传入的不是正常正数 `PASS`
- delcas stringcas的key，版本是正常正数，但不一致 `PASS`
- delcas stringcas的key，版本是正常正数，且一致 `PASS`
- delcas 不存在的key `PASS`


特殊值：恰好有CAS格式但是原意不是CAS的key `PASS`

### BUG
- ~~回复消息好像某种情况下TM的重复回复了。。。 `FIXED`~~
- ~~RDB恢复过来以后CAS API失效 `FIXED`~~ （原因：我实现的setcas会强制将value编码为RAW，而后添加version；当value+version最终长度<=44字节时，数据从RDB恢复过来后value编码变成了EMBSTR；而我实现的GETCAS、SETCAS、DELCAS都是基于RAW编码的，所以无法操作了）
- ~~AOF加载完成后，cas数据的版本号不对，且cas数据里有了不止一个版本 `FIXED`~~（原因：AOF文件中的setcas key value version与当时执行这条命令不一样，value已经是执行命令成功后的value了！即`value+版本`，故直接set key value即可，`version`参数没用了）

### PROBLEM
- ~~delcas在出错情况下是应该返回错误原因，or 直接返回0~~，最终选择了直接返回0
- ~~del、get、set都可以直接操作CAS格式的kv，是否要限制一下~~，算了
