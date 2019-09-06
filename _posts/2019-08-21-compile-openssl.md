---
title: 在Linux下编译OPENSSL1.1.1c
date: 2019-08-21 18:54:22
categories: 计算机技术
tags:
 - openssl
---

openssl在linux下编译很简单

# 运行配置
解压源码后，进入源文件目录，运行配置
```
$./config
```

# make编译
直接执行make
```
$ make
```

等一会

# 在源码当前目录下会发现
```
libcrypto.a libcrypto.so libssl.a libssl.o
```
表示编译成功

# 测试
可以运行测试，看一切是否正常
```
$ make test
```

```
...
../test/recipes/90-test_threads.t .................. ok
../test/recipes/90-test_time_offset.t .............. ok
../test/recipes/90-test_tls13ccs.t ................. ok
../test/recipes/90-test_tls13encryption.t .......... ok
../test/recipes/90-test_tls13secrets.t ............. ok
../test/recipes/90-test_v3name.t ................... ok
../test/recipes/95-test_external_boringssl.t ....... skipped: No external tests in this configuration
../test/recipes/95-test_external_krb5.t ............ skipped: No external tests in this configuration
../test/recipes/95-test_external_pyca.t ............ skipped: No external tests in this configuration
../test/recipes/99-test_ecstress.t ................. ok
../test/recipes/99-test_fuzz.t ..................... ok
All tests successful.
Files=155, Tests=1453, 208 wallclock secs ( 1.95 usr  1.06 sys + 69.00 cusr 104.85 csys = 176.86 CPU)
Result: PASS
make[1]: Leaving directory '/mnt/c/private/openssl-1.1.1c'
```

一切正常 
