启动失败

```mysql
ERROR: child process failed, exited with error number 1

报错原因：日志读取失败
解决办法：查看配置文件中日志路径与目录中的是否一致

ERROR: child process failed, exited with error number 100
报错原因：重复启动
```





多实例使用场景

多数选举是否适用所有高可用场景 

仲裁节点可以有几个

