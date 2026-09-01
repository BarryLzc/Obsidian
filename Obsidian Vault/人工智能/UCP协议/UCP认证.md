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