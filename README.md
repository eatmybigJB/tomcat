```python
import base64

def multi_base64_encode(data: str, n: int) -> str:
    """对字符串进行 n 次 base64 编码，返回字符串"""
    encoded = data.encode("utf-8")
    for _ in range(n):
        encoded = base64.b64encode(encoded)
    return encoded.decode("utf-8")

def multi_base64_decode(data: str, n: int) -> str:
    """对字符串进行 n 次 base64 解码，返回字符串"""
    decoded = data.encode("utf-8")
    for _ in range(n):
        decoded = base64.b64decode(decoded)
    return decoded.decode("utf-8")

# 示例
text = "Hello, Pig!"
encoded_twice = multi_base64_encode(text, 2)
print("两次编码:", encoded_twice)

decoded_twice = multi_base64_decode(encoded_twice, 2)
print("两次解码:", decoded_twice)
