```python
import json
import requests
import jwt
from jwt.algorithms import RSAAlgorithm

pip install pyjwt cryptography requests



COGNITO_REGION = "ap-east-1"
USER_POOL_ID = "ap-east-1_LNf5zPST"

JWKS_URL = f"https://cognito-idp.{COGNITO_REGION}.amazonaws.com/{USER_POOL_ID}/.well-known/jwks.json"


def get_public_key(token):
    """从 token header 的 kid 找到 JWKS 中对应公钥"""
    headers = jwt.get_unverified_header(token)
    kid = headers["kid"]

    jwks = requests.get(JWKS_URL).json()

    for jwk in jwks["keys"]:
        if jwk["kid"] == kid:
            return RSAAlgorithm.from_jwk(json.dumps(jwk))

    raise Exception("No matching JWK found")


def verify_signature_only(token):
    """只验证 token 是否被篡改，不校验 iss、aud、exp 等"""
    public_key = get_public_key(token)

    try:
        # options={"verify_*": False} 禁止所有其他校验，只验证签名
        payload = jwt.decode(
            token,
            public_key,
            algorithms=["RS256"],
            options={
                "verify_signature": True,  # ✔ 只验证签名
                "verify_exp": False,       # ❌ 不验证过期
                "verify_aud": False,       # ❌ 不验证 audience
                "verify_iss": False,       # ❌ 不验证 issuer
            }
        )
        return True, payload
    except jwt.InvalidSignatureError:
        return False, "Signature INVALID (token may be tampered)"
    except Exception as e:
        return False, str(e)


# ---- 测试 ----
if __name__ == "__main__":
    token = input("Enter access token: ").strip()
    ok, info = verify_signature_only(token)

    if ok:
        print("\n✅ Signature is VALID (token NOT tampered)")
        print(json.dumps(info, indent=4))
    else:
        print("\n❌ Signature INVALID")
        print("Reason:", info)
