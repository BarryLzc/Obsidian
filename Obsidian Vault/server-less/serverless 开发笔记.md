pre-condition：创建aws账号、创建IAM user
1. 安装serverless框架：npm i serverless@3.39.0 -g
2. 配置aws凭证：serverless config credentials --provider aws --key <> --secret <> --profile serverlessUser
3. serverless create --template aws-nodejs --path myServerlessProject / serverless create --template aws-go --path myServerlessProject
4. cd myServerlessProject
5. open VScode
6. 配置profile in provider：profile: serverlessUser
7. sls deploy
8. debug:[[serverless-debug]]

Reference：
https://www.serverless.com/blog/framework-example-golang-lambda-support/