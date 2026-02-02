 1. 解压文件：tar -xvf directory.tar
 2. 压缩文件：tar -czvf directory.tar directory
 3. **实时查看内核日志**。eBPF 挂载失败、校验报错、内核崩溃全看这里：sudo dmesg -Tw
 4. 检查内核是否被锁死（[none] 才表示完全自由）: cat /sys/kernel/security/lockdown