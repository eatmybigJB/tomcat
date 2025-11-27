```python
import base64
import json

def base64url_decode(input_str):
    """Decode base64url without padding."""
    padding = '=' * (4 - (len(input_str) % 4))
    return base64.urlsafe_b64decode(input_str + padding)

def split_jwt(token):
    """Split JWT into header, payload, signature and decode the JSON parts."""
    parts = token.split('.')
    if len(parts) != 3:
        raise ValueError("Invalid JWT: should contain 3 parts")

    header_b64, payload_b64, signature_b64 = parts

    header = json.loads(base64url_decode(header_b64))
    payload = json.loads(base64url_decode(payload_b64))
    signature = signature_b64  # signature 不需要 JSON decode

    return header, payload, signature


# ---------------------------
#  Example usage:
# ---------------------------

access_token = input("Enter access token: ").strip()

header, payload, signature = split_jwt(access_token)

print("===== HEADER =====")
print(json.dumps(header, indent=4))

print("\n===== PAYLOAD =====")
print(json.dumps(payload, indent=4))

print("\n===== SIGNATURE (base64url) =====")
print(signature)
