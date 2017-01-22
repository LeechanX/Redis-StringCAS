# Redis add String CAS API Support

According to our business needs, I introduced the lock-free programming CAS data manipulation into Redis 3.2.6 for Redis String storage type.

Specifically, the API includes:
- getcas 
- setcas 
- delcas

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
