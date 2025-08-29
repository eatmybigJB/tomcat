```python
from datetime import datetime, timedelta, timezone

def generate_filename(prefix: str = "Cognito") -> str:
    # 设置时区 UTC+8（北京时间）
    tz = timezone(timedelta(hours=8))
    date_str = datetime.now(tz).strftime("%d%b%Y")  # 29Aug2025
    return f"{prefix}_{date_str}"

