1. 下载.out文件：
	wget -O trace.out http://localhost:6060/debug/pprof/heap?seconds=5
  
2. go tool pprof trace.out
  
3. Web/Top command打开分析图
