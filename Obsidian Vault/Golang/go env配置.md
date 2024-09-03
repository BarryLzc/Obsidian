1. GOPROXY='https://goproxy.io,direct' 代表够官方获取依赖的方式。GOPROXY=https://goproxy.inshopline.com,direct 公司获取go依赖代理的方法
2. GOSUMDB="sum.golang.org" 这个环境变量用于确定用于下载Go模块和相关元数据的可信源。GOSUMDB='gosum.inshopline.com+d8f83899+AUX3heidcb2phi+3OTnmPfq/l42tBIojQ7nA1PLLbEtU'，用于获取公司Go模块和相关元数据的可信源。


写入方式：go env -w <%s>