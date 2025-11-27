```python
from cryptography.hazmat.primitives import serialization

def public_key_to_pem(public_key):
    pem = public_key.public_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PublicFormat.SubjectPublicKeyInfo
    )
    return pem.decode()

public_key = get_public_key(token)
print(public_key_to_pem(public_key))
