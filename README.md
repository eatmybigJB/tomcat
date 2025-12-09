```python
下面是一段 可直接运行的代码，实现了：

启动 Cognito Hosted UI 登录

iOS Safari 自动弹 Passkey

获取 Authorization Code

换取 Token（可选）

import Foundation
import AuthenticationServices

class CognitoAuthService: NSObject {

    private var authSession: ASWebAuthenticationSession?
    
    // 你的 Cognito 自定义域名 + OAuth2 授权端点
    private let clientId = "YOUR_CLIENT_ID"
    private let domain = "https://auth.inspb2b-dev.insurance.hsbc.com.sg"
    private let redirectUri = "myapp://callback/"
    private let scope = "openid+email+profile"
    
    func signInWithCognito() {
        let state = UUID().uuidString
        let nonce = UUID().uuidString
        
        let authURLString = """
        \(domain)/oauth2/authorize?response_type=code\
        &client_id=\(clientId)\
        &redirect_uri=\(redirectUri)\
        &scope=\(scope)\
        &state=\(state)\
        &nonce=\(nonce)
        """
        
        guard let authURL = URL(string: authURLString) else { return }

        authSession = ASWebAuthenticationSession(url: authURL, callbackURLScheme: "myapp") { callbackURL, error in
            
            if let error = error {
                print("Authentication failed: \(error)")
                return
            }
            
            guard let callbackURL = callbackURL else { return }
            self.handleCallback(url: callbackURL)
        }
        
        authSession?.presentationContextProvider = self
        authSession?.prefersEphemeralWebBrowserSession = false
        
        authSession?.start()
    }
    
    private func handleCallback(url: URL) {
        guard let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
              let queryItems = components.queryItems else { return }
        
        if let code = queryItems.first(where: { $0.name == "code" })?.value {
            print("Authorization code: \(code)")
            // 接下来发送 POST 请求换 Token
        }
    }
}

extension CognitoAuthService: ASWebAuthenticationPresentationContextProviding {
    func presentationAnchor(for session: ASWebAuthenticationSession) -> ASPresentationAnchor {
        return UIApplication.shared.windows.first!
    }
}
