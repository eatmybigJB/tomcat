```python
#!/usr/bin/env python3
import pandas as pd
import os
import re

script_dir = os.path.dirname(os.path.realpath(__file__))

# ======= 改成你自己的文件路径 =======
OLD_CSV = os.path.join(script_dir, "users-b2c-daily.csv")
NEW_CSV = os.path.join(script_dir, "CEP_SG_SvocGoldenRecords.csv")
SVOC_MAP_CSV = os.path.join(script_dir, "svocid_mapping.csv")
OUTPUT_CSV = os.path.join(script_dir, "users-b2c-daily-replace.csv")
# ==================================


def clean_str(s: str) -> str:
    """去除不可见字符并 strip。"""
    if s is None:
        return ""
    s = str(s)
    for ch in ["\u200b", "\u200c", "\u200d", "\ufeff"]:
        s = s.replace(ch, "")
    return s.strip()


# ==================== 邮箱相关 ====================

EMAIL_RE = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")


def normalize_email(email: str) -> str:
    return clean_str(email).lower()


def is_valid_email(email: str) -> bool:
    email = clean_str(email)
    if not email:
        return False
    return bool(EMAIL_RE.match(email))


# ==================== 电话相关 ====================

def normalize_phone(phone: str) -> str:
    phone = clean_str(phone)
    phone = phone.replace(" ", "").replace("-", "")
    return phone


def build_phone(country_code: str, phone: str) -> str:
    """拼接国际号码 +<countryCode><mobilePhoneNumber>。"""
    cc = clean_str(country_code)
    mp = clean_str(phone)
    if not cc or not mp:
        return ""
    retur
