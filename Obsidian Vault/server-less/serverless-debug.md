1. **plugins**: serverless.yml文件plugins配置
	  - serverless-offline
2. **运行offline创建**：
	- npm install serverless-offline --save-dev
	- npx serverless offline
3. **创建 GoLand 调试配置**：
	- 安装aws-toolkit插件
4. **运行 Lambda 模拟器**：
	- 根目录创建template.yml：写入handler函数配置
5. 注：检查aws链接配置
	- ~/.aws/config
