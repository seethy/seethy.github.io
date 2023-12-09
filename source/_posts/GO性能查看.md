---
title: GO性能查看
date: 2023-12-09 16:20:20
tags:
- pprof
- Graphviz
- goroutine
categories:
- ["编程语言","GO"]
---
### 查看CPU：
curl http://100.100.254.75:8080/debug/pprof/profile?seconds=20 -o  pprof.out

go tool pprof -http :2234 ./pprof.out (需要安装Graphviz)

### 查看内存：
curl http://100.100.223.81:8080/debug/pprof/heap?seconds=10 -o heap.out

go tool pprof -http :2234 ./heap.out (需要安装Graphviz)

go tool pprof -inuse_space 100.96.92.176:8080/debug/pprof/heap   （交互命令）
```
[root@bd01 ~]# go tool pprof -inuse_space 100.100.223.70:8080/debug/pprof/heap 
Fetching profile over HTTP from http://100.100.223.70:8080/debug/pprof/heap
Saved profile in /root/pprof/pprof.zteagent.alloc_objects.alloc_space.inuse_objects.inuse_space.009.pb.gz
File: xxagent
Type: inuse_space
Time: Dec 9, 2023 at 3:56pm (CST)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top 10
Showing nodes accounting for 10818.04kB, 50.12% of 21582.56kB total
Showing top 10 nodes out of 231
      flat  flat%   sum%        cum   cum%
 1805.17kB  8.36%  8.36%  2354.01kB 10.91%  compress/flate.NewWriter
 1577.92kB  7.31% 15.68%  1577.92kB  7.31%  github.com/denisenkom/go-mssqldb/internal/cp.init
 1571.92kB  7.28% 22.96%  1571.92kB  7.28%  regexp/syntax.(*compiler).inst
 1184.27kB  5.49% 28.45%  1184.27kB  5.49%  zte.com.cn/ztekit/server/signals.(*Handler).Loop
 1033.28kB  4.79% 33.23%  1033.28kB  4.79%  runtime.procresize
 1030.24kB  4.77% 38.01%  1030.24kB  4.77%  reflect.unsafe_NewArray
 1024.28kB  4.75% 42.75%  1536.30kB  7.12%  github.com/benthosdev/benthos/v4/internal/docs.FieldSpecs.YAMLToMap
 548.84kB  2.54% 45.30%   548.84kB  2.54%  compress/flate.(*compressor).initDeflate (inline)
 522.06kB  2.42% 47.71%   522.06kB  2.42%  github.com/aws/aws-sdk-go-v2/service/s3/internal/endpoints.init
 520.04kB  2.41% 50.12%   520.04kB  2.41%  github.com/benthosdev/benthos/v4/public/service.newStream
(pprof) exit
[root@bd01 ~]#
```
### 查看协程：
 curl http://100.100.223.81:8080/debug/pprof/goroutine  -o goroutine.out

### 查看进程的线程数：
 cat /proc/进程ID/status|grep Threads
