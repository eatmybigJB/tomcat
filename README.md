```python
import string
import secrets

def generate_strong_password(length=16):
    # 定义字符集：大写、小写、数字、特殊字符
    upper = string.ascii_uppercase
    lower = string.ascii_lowercase
    digits = string.digits
    symbols = "!@#$%^&*()-_=+[]{}|;:,.<>?/"

    # 确保至少包含每种类型一个字符
    all_chars = upper + lower + digits + symbols
    password = [
        secrets.choice(upper),
        secrets.choice(lower),
        secrets.choice(digits),
        secrets.choice(symbols),
    ]

    # 剩余字符随机补足
    password += [secrets.choice(all_chars) for _ in range(length - 4)]

    # 打乱顺序，防止前4个位置固定类型
    secrets.SystemRandom().shuffle(password)

    return ''.join(password)

# 示例
print(generate_strong_password())
