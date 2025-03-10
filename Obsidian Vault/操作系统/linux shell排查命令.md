### **1. 系统资源排查**

#### **CPU使用情况**
top                         # 实时查看CPU、内存使用情况
htop                       # 更美观的top（需安装）
mpstat -P ALL 1   # 按核查看CPU使用情况（需安装sysstat）
uptime                   # 查看系统负载

#### **内存使用情况**
free -h                         # 查看内存使用情况 
vmstat 1 10                 # 统计CPU、内存、I/O情况 
cat /proc/meminfo     # 查看详细内存信息


#### **磁盘使用情况**
df -h                  # 查看磁盘分区使用情况
du -sh /path     # 查看目录大小
iostat -dx 1 5   # 查看磁盘I/O情况（需安装sysstat）
lsblk                 # 查看磁盘分区结构


---

### **2. 进程与线程**

ps aux                         # 查看所有进程
ps -ef | grep xxx        # 查询某个进程
pstree -p                    # 以树形展示进程
pidstat -p PID 1         # 监控某个进程的资源占用（需安装sysstat）
strace -p PID             # 跟踪进程系统调用
lsof -p PID                 # 查看进程打开的文件和端口


---

### **3. 网络排查**

#### **查看网络连接**
netstat -tulnp    # 查看监听端口
ss -tulnp         # 更快的netstat替代品
lsof -i :80       # 查看占用某端口的进程

#### **网络连通性**
ping 8.8.8.8        # 检测网络连通性
traceroute 8.8.8.8  # 跟踪数据包路径（需安装）
mtr 8.8.8.8        # 综合ping和traceroute的工具


#### **抓包分析**

tcpdump -i eth0 port 80  # 抓取80端口数据包（需安装）

---

### **4. 文件系统相关**

ls -lh                            # 查看文件详细信息
stat filename              # 查看文件详细信息（包括修改时间）
find / -name xxx        # 搜索文件
grep "error" logfile   # 在日志中搜索错误


---

### **5. 日志分析**

dmesg | tail -50             # 查看最近50条内核日志
journalctl -xe                 # 查看系统日志（适用于systemd）
tail -f /var/log/syslog     # 实时查看系统日志
tail -f /var/log/messages  # 查看CentOS系统日志


---

### **6. 性能监控**

sar -u 1 5            # 监控CPU使用情况（需安装sysstat）
iostat -dx 1 5     # 监控磁盘I/O情况
iotop                  # 实时查看哪个进程占用最多磁盘（需安装）
