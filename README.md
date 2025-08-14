```python
import re

def is_valid_email(email):
    if not isinstance(email, str):
        return False
    email_regex = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(email_regex, email, re.IGNORECASE) is not None

# 测试
print(is_valid_email("insn.uat2@gmail.com"))  # True
