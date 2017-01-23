# Redis add String CAS API Support

According to our business needs, I introduced the lock-free programming CAS data manipulation into Redis 3.2.6 for Redis String storage type.

Specifically, the API includes:
- getcas 
- setcas 
- delcas

###FEATURE:
- 对String类型的KV支持CAS操作
- 兼容RDB持久化；

###USAGE：
####getcas
```sh
>getcas key
return version and value
```
####setcas
```sh
>setcas key value version
if key is not exist, version must = 0; otherwise setcas will be failed
if key is exist, version must = cas version in key storage, otherwise setcas will be failed
```
####delcas
```sh
>delcas key
return version and value
```
###测试日记 2017.01.23

>开发进度：修复:RDB恢复数据后无法GETCAS、SETCAS、DELCAS
>
>测试目的：RDB数据恢复后依然可以CAS操作


####测试详情：
- setcas一个整数，然后RDB保存，重启server后查看这个kv是否可以正常CAS操作 `PASS`
- setcas一个短字符串，然后RDB保存，重启server后查看这个kv是否可以正常CAS操作 `PASS`
- setcas一个长字符串，然后RDB保存，重启server后查看这个kv是否可以正常CAS操作 `PASS`

###测试日记 2017.01.22

>开发进度：基本实现，单机 + 不考虑集群、不考虑RDB\AOF、不考虑复制、不考虑订阅发布
>
>测试目的：基本使用

####测试详情：
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

####BUG：

- 回复消息好像某种情况下TM的重复回复了。。。 `FIXED`
- del、get、set都可以直接操作CAS格式的kv，不知道算不算BUG `IGNORED`
- RDB恢复过来以后CAS API失效 `TODO`
- 原因：我实现的setcas会强制将value编码为RAW，而后添加version；当value+version最终长度<=44字节时，数据从RDB恢复过来后value编码变成了EMBSTR；而我实现的GETCAS、SETCAS、DELCAS都是基于RAW编码的，所以无法操作了
