```python
from datetime import datetime

def generate_filename(prefix: str = "Cognito") -> str:
    # 格式：日+三位英文月份缩写+四位年份
    date_str = datetime.now().strftime("%d%b%Y")  # 25Aug2025
    return f"{prefix}_{date_str}"

# 示例
print(generate_filename())          # Cognito_26Aug2025
print(generate_filename("Users"))   # Users_26Aug2025

