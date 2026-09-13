---
title: "Linux IPC&Shell编程笔记"
date: 2025-10-01T05:28:36+08:00
draft: false
---

## 1. Linux多进程
0. 管道 + 信号 + 信号量 + 共享内存 + 消息队列 
1. 注意，无名容器只有父子进程之间，exec后子不受父进程完全控制。
2. 守护进程：setsid(0,0); + dtablesize->close(fd[x]) + chdir("/tmp")
3. 匿名管道 + 匿名信号量都是无名容器
4. 有名mkfifo+node或ftok(path,id)生成唯一标识符，多进程适用
5. signal/sigaction为SET/GET，为信号响应重写方法
6. kill(pid,sig), raise(self,sig), pause()
7. 信号量：ftok+semget / semctrl(SetDel) / semops(PV)完备操作，主要是ctl
8. 父进程，子进程
9. 进程组（前台/后台），前台对应一个终端
10. 会话 （用户的登入作为一个会话）
11. 僵尸进程，孤儿进程 
12. open(fd,O_CREAT|O_WRONLY|O_APPEND,"0600"),close,write,read (sys/fcntl.h) 
13. wait,waitid,waitpid 
13. 双端RW pipe(int[2]) , dup2(int[1],STDOUT_FILENAME) 
15. popen(cmdstring,"r"/"w")，管道定向 
16. execvp,execlp,execle
17. 不要在临界区调用sleep,send/recv类函数返回值一般>0有效，-1无效
18. IPC常用0600|IPC_CREAT|IPC_EXCLE，pthread_attr追加解耦父子回收
19. 注意socklen_t的变量和指针,sizeof()求值，指针可能需要追加临时变量
20. fcntl(user_sockfd, F_SETFL, fcntl(user_sockfd, F_GETFL, 0)|O_NONBLOCK)


## 2. 实用Shell脚本
```bash
file=/dir1/dir2/dir3/my.file.txt
可以用${ }分别替换得到不同的值：

1. 分割
${file#*/}：删掉第一个 / 及其左边的字符串：dir1/dir2/dir3/my.file.txt
${file##*/}：删掉最后一个 /  及其左边的字符串：my.file.txt
${file%/*}：删掉最后一个  /  及其右边的字符串：/dir1/dir2/dir3
${file%%/*}：删掉第一个 /  及其右边的字符串：(空值)
#是去掉左边（键盘上#在$ 的左边）
%是去掉右边（键盘上%在$ 的右边）
单一符号是最小匹配；两个符号是最大匹配

2. 切片
${file:0:5}：提取最左边的 5 个字节：/dir1
${file:5:5}：提取第 5 个字节右边的连续5个字节：/dir2

3. 替换
${file/dir/path}：将第一个dir 替换为path：/path1/dir2/dir3/my.file.txt
${file//dir/path}：将全部dir 替换为 path：/path1/path2/path3/my.file.txt
```
