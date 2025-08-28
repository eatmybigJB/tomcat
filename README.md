```python
import base64

# 原始数据
data = "Hello, Pig!"

# 第一次 base64 编码
encoded1 = base64.b64encode(data.encode())
print("第一次编码:", encoded1)

# 第二次 base64 编码（对第一次的结果再编码）
encoded2 = base64.b64encode(encoded1)
print("第二次编码:", encoded2)

# 第一次解码
decoded1 = base64.b64decode(encoded2)
print("第一次解码:", decoded1)

# 第二次解码（得到原始数据）
decoded2 = base64.b64decode(decoded1).decode()
print("第二次解码:", decoded2)
