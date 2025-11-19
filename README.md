```python
#!/usr/bin/env python3
import pandas as pd
import re

# ======= 改成你自己的文件路径 =======
OLD_CSV = "old.csv"      # 图一：包含 UserStatus,preferred_username,Username,custom:svoc_id,phone_number
NEW_CSV = "new.csv"      # 图二：包含 _id,matchingId,emailAddressNumber,countryCode,mobilePhoneNumber
OUTPUT_CSV = "merged_output.csv"
# ==================================


# 去除不可见字符并strip
def clean_str(s: str) -> str:
    if s is None:
        return ""
    s = str(s)
    for ch in ["\u200b", "\u200c", "\u200d", "\ufeff"]:
        s = s.replace(ch, "")
    return s.strip()


# 邮箱处理
EMAIL_RE = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")

def normalize_email(email: str) -> str:
    return clean_str(email).lower()

def is_valid_email(email: str) -> bool:
    email = clean_str(email)
    if not email:
        return False
    return bool(EMAIL_RE.match(email))


# 电话处理：要求 +65 后面 8 位数字
PHONE_RE = re.compile(r"^\+65\d{8}$")

def normalize_phone(phone: str) -> str:
    phone = clean_str(phone)
    # 去掉空格和中划线等，但保留+
    phone = phone.replace(" ", "").replace("-", "")
    return phone

def is_valid_phone(phone: str) -> bool:
    phone = normalize_phone(phone)
    return bool(PHONE_RE.match(phone))


def main():
    # 所有列都当字符串读，避免自动转数字
    old_df = pd.read_csv(OLD_CSV, dtype=str, keep_default_na=False)
    new_df = pd.read_csv(NEW_CSV, dtype=str, keep_default_na=False)

    # 清理关键列
    for col in ["UserStatus", "preferred_username", "Username", "custom:svoc_id", "phone_number"]:
        if col in old_df.columns:
            old_df[col] = old_df[col].map(clean_str)
        else:
            # 如果列名有打错，这里直接抛异常方便你发现
            raise KeyError(f"旧文件缺少列: {col}")

    for col in ["matchingId", "emailAddressNumber", "countryCode", "mobilePhoneNumber"]:
        if col in new_df.columns:
            new_df[col] = new_df[col].map(clean_str)
        else:
            raise KeyError(f"新文件缺少列: {col}")

    # 同一个 matchingId 多行的话，只保留最后一行
    new_df = new_df.drop_duplicates(subset=["matchingId"], keep="last")

    # 按 custom:svoc_id / matchingId 做左连接
    merged = old_df.merge(
        new_df,
        how="left",
        left_on="custom:svoc_id",
        right_on="matchingId",
        suffixes=("", "_new"),
    )

    tags = []

    for idx, row in merged.iterrows():
        email_updated = False
        phone_updated = False

        # ==== 邮箱处理 ====
        old_username = row.get("Username", "")
        old_preferred = row.get("preferred_username", "")
        # 旧邮箱用于比较：Username 优先，其次 preferred_username
        old_email_for_compare = normalize_email(old_username or old_preferred)

        new_email_raw = row.get("emailAddressNumber", "")
        new_email_norm = normalize_email(new_email_raw)

        if is_valid_email(new_email_raw) and new_email_norm:
            # 邮箱不同（大小写不敏感）
            if old_email_for_compare != new_email_norm:
                # 覆盖两个邮箱列，统一写成小写
                merged.at[idx, "Username"] = new_email_norm
                merged.at[idx, "preferred_username"] = new_email_norm
                email_updated = True

        # ==== 电话处理 ====
        old_phone_raw = row.get("phone_number", "")
        old_phone_norm = normalize_phone(old_phone_raw)

        cc = row.get("countryCode", "")
        mp = row.get("mobilePhoneNumber", "")
        cc = clean_str(cc)
        mp = clean_str(mp)

        new_phone_full = ""
        if cc and mp:
            new_phone_full = f"+{cc}{mp}"

        if new_phone_full and is_valid_phone(new_phone_full):
            new_phone_norm = normalize_phone(new_phone_full)
            if old_phone_norm != new_phone_norm:
                merged.at[idx, "phone_number"] = new_phone_full
                phone_updated = True

        # ==== tag 逻辑 ====
        if email_updated and phone_updated:
            tag = "overwrite_all"
        elif email_updated:
            tag = "overwrite_email"
        elif phone_updated:
            tag = "overwrite_phone"
        else:
            tag = "original"

        tags.append(tag)

    merged["tag"] = tags

    # 按你要求的列顺序输出
    out_cols = [
        "UserStatus",
        "preferred_username",
        "Username",
        "custom:svoc_id",
        "phone_number",
        "tag",
    ]
    output_df = merged[out_cols]
    output_df.to_csv(OUTPUT_CSV, index=False)

    print(f"处理完成，已输出到: {OUTPUT_CSV}")


if __name__ == "__main__":
    main()
