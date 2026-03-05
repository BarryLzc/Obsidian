 1. 解压文件：tar -xvf directory.tar
 2. 压缩文件：tar -czvf directory.tar directory
 3. **实时查看内核日志**。eBPF 挂载失败、校验报错、内核崩溃全看这里：sudo dmesg -Tw
 4. 检查内核是否被锁死（[none] 才表示完全自由）: cat /sys/kernel/security/lockdown
 5. 传输文件：**scp** [文件在local的具体位置][hostname]@[net IP]:[目标机器路径]
    **scp** /Users/lp211200146/Desktop/README.md lzc@172.16.114.12:/home/lzc/
 6. 查看本机ip：ifconfig
 7. 查看应用程序：which / whereis
 8. 搜索历史终端指令：Control + R