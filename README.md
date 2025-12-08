```python
Amazon Cognito 可以对托管登录的 .well-known 以您的 RP ID 作为路径的关联文件作出一致响应。

这里的 “关联文件路径” 指的是 FIDO2 / WebAuthn 标准要求 RP（Relying Party）提供的一个 .well-known 配置文件，路径固定为：

✅ https://<你的域名>/.well-known/webauthn.json

或（视规范不同，也可能是以下之一）：

/.well-known/webauthn

/.well-known/webauthn/<rpId>

/.well-known/apple-app-site-association（iOS passkey 关联）

/.well-known/assetlinks.json（Android passkey 关联）
