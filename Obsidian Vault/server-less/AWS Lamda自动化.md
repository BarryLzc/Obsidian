1. 概述
	AWS Lambda 是目前 FaaS 中运用比较广泛的一个服务。用户只需要实现业务逻辑，将代码上传到 AWS 之后，AWS 会负责处理接下来的所有事情：调度，伸缩，高可用，日志等等。而这些只有在方法被调用的时候才会计费，可以真正的做到按需运行，按毫秒计费。（https://xuanwo.io/2018/01/14/automation-based-on-aws-lambda/）
2. 该服务的优点：
	- 除了偶尔请求 timeout 之外，服务很稳定，上线之后不用费心维护
	- 自动集成的 CloudWatch 日志挺好用，调试很方便
3. 该服务的缺点：
	- 调试的过程比较麻烦，不能接入外部的 Git 服务，只能用 AWS 自己的那个
	- 上线的脚本多了之后维护起来很麻烦，没有一个统一管理的方案
	- 强依赖 AWS 自己的服务，日后迁移要大改脚本
4. AWS serverless架构图
	- Serverless Web Application Architecture with AWS 1
	![[Screen Shot 2024-09-22 at 17.27.26.png]]
	- Serverless Web Application Architecture with AWS 2
	![[Screen Shot 2024-09-22 at 17.27.32 1.png]]



reference:
1. https://www.researchgate.net/profile/Manoj-Kumar-318/publication/331729868_Serverless_Architectures_Review_Future_Trend_and_the_Solutions_to_Open_Problems/links/5c984797a6fdccd460384edb/Serverless-Architectures-Review-Future-Trend-and-the-Solutions-to-Open-Problems.pdf