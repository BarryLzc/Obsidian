https://ucp.dev/documentation/core-concepts/#authentication-mechanisms
# 架构图

```
                 UCP Authentication
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
    API Key            OAuth              mTLS
       │                 │                 │
       ↓                 ↓                 ↓
  Shared Secret      Access Token      Client Cert
       │                 │                 │
       └───────┬─────────┴─────────┬───────┘
               │                   │
         Pre-established       Pre-established
               │                   │
               └─────────┬─────────┘
                         │
                         X
                    不能 Permissionless


                  HTTP Message Signature
                           │
                           ↓
                    Private Key
                           │
                           ↓
                    Sign HTTP Message
                           │
                           ↓
                 UCP-Agent / Profile
                           │
                           ↓
                     Public Key
                           │
                           ↓
                       Verify
                           │
                           ↓
                   Permissionless ✅
```

# UCP支持的四种[[传统认证机制对比]]
- [[HTTP Message Signatures(RFC 9421)]]
- OAuth2.0
- API Keys
- mTLS

## [[HTTP Message Signatures(RFC 9421)]]
### 1. 公钥（Public Key）的存储与发布

UCP 不依赖传统的 CA 机构或中心化密钥库，公钥是**公开存储在各自节点的 Profile 配置文件中**。

- **存储格式：** 遵循 **RFC 7517 (JWK / JSON Web Key)** 标准，支持 `ES256` (P-256) 或 `ES384`。
    
- **商家 (Business)：** 公开存储在根域名的 **`/.well-known/ucp`** 路径下的 JSON Profile 文件中。其中的 `signing_keys` (或 `keys[]`) 字段包含了公钥数组。
    
- **平台/客户端 (Platform / AI Agent)：** 存储在其配置的 Profile URL 对应文档中。平台在发送 HTTP 请求时，会在请求头中注入 `UCP-Agent` 标头（例如 `UCP-Agent: [https://agent.example.com/profile.json](https://agent.example.com/profile.json)`），指向包含其 JWK 公钥的 JSON 文件。

#### 验签方如何获取公钥？

1. **获取 URL：** 验签方收到请求后，从 `UCP-Agent` 请求头（验证对方 Agent）或直接读取 `/.well-known/ucp`（验证商家）中取得 Profile 地址。
    
2. **提取公钥：** 从 HTTP 请求头的 `Signature-Input` 拿到 `keyid` (即 `kid`)。
    
3. **匹配验签：** 抓取 Profile 并匹配 `signing_keys` 中 `kid` 一致的公钥，完成 RFC 9421 验签。

### 2. 私钥（Private Key）的存储

私钥是绝对不能公开或泄露给对方的，由签名发起方（无论是 Merchant 后端还是 AI Agent 服务）**保存在各自内部的安全性设施中**：

- **应用/微服务后端：**
    
    - 存储在服务端的加密环境变量（Environment Variables）中。
        
    - 存储在专用的云端密钥管理服务（KMS / Vault）中（如 AWS KMS、HashiCorp Vault、GCP Secret Manager）。系统在启动或签名时调用 KMS/Vault 的 Crypto API 生成签名，甚至私钥本身不出 KMS 硬件防护边界（HSM）。
- **客户端 / 边缘 Agent 节点：**
    
    - 移动端 App / Desktop 节点：存放在系统级的安全芯片/硬件安全模块中（如 iOS `Secure Enclave`、Android `Android KeyStore`）。
        
    - Node.js / Serverless 环境：存放在安全的运行时 App Secrets 管理服务中。

### 总结

- **公钥：** **“随用随抓”**，存放在标准的 HTTP 静态 Profile (`/.well-known/ucp` 或 `UCP-Agent` URL) 里的 `signing_keys` JWK 列表中，任何人均可公开读取。
- **私钥：** **“各自保管”**，严格保存在签名方自己的 KMS、安全硬件 (HSM) 或保密环境变量中，绝不可暴露在网络中。