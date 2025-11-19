```python
#!/usr/bin/env python3
import pandas as pd
import os
import re

script_dir = os.path.dirname(os.path.realpath(__file__))

# ======= 改成你自己的文件路径 =======
# 图一：包含 UserStatus,preferred_username,sub,Username,email,phone_number,phone_number_verified,custom:svoc_id,...
OLD_CSV = os.path.join(script_dir, "users-b2c-daily.csv")

# 图二：包含 _id,matchingId,emailAddressNumber,countryCode,mobilePhoneNumber
NEW_CSV = os.path.join(script_dir, "CEP_SG_SvocGoldenRecords.csv")

# 图三：包含 current svocId,before svocId
SVOC_MAP_CSV = os.path.join(script_dir, "svocid_mapping.csv")

# 输出
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


# ================= 邮箱相关 =================
EMAIL_RE = re.compile(r"^[^@\s]+@[^@\s]+\.[^@\s]+$")


def normalize_email(email: str) -> str:
    return clean_str(email).lower()


def is_valid_email(email: str) -> bool:
    email = clean_str(email)
    if not email:
        return False
    return bool(EMAIL_RE.match(email))


# ================= 电话相关 =================
# 要求：+65 开头，后面 8 位数字
PHONE_RE = re.compile(r"^\+65\d{8}$")


def normalize_phone(phone: str) -> str:
    phone = clean_str(phone)
    # 去掉空格和中划线，但保留 +
    phone = phone.replace(" ", "").replace("-", "")
    return phone


def is_valid_phone(phone: str) -> bool:
    phone = normalize_phone(phone)
    return bool(PHONE_RE.match(phone))


def main():
    # ========= 读取三个 CSV =========
    old_df = pd.read_csv(OLD_CSV, dtype=str, keep_default_na=False)
    new_df = pd.read_csv(NEW_CSV, dtype=str, keep_default_na=False)
    svoc_df = pd.read_csv(SVOC_MAP_CSV, dtype=str, keep_default_na=False)

    # ========= 清理 OLD_CSV 关键列 =========
    required_old_cols = [
        "UserStatus",
        "preferred_username",
        "Username",
        "email",
        "phone_number",
        "custom:svoc_id",
    ]
    for col in required_old_cols:
        if col in old_df.columns:
            old_df[col] = old_df[col].map(clean_str)
        else:
            raise KeyError(f"旧文件缺少列: {col}")

    # ========= 清理 NEW_CSV 关键列 =========
    for col in ["matchingId", "emailAddressNumber", "countryCode", "mobilePhoneNumber"]:
        if col in new_df.columns:
            new_df[col] = new_df[col].map(clean_str)
        else:
            raise KeyError(f"新文件缺少列: {col}")

    # ========= 清理 SVOC_MAP_CSV 关键列 =========
    CURRENT_SVOC_COL = "current svocId"
    BEFORE_SVOC_COL = "before svocId"

    for col in [CURRENT_SVOC_COL, BEFORE_SVOC_COL]:
        if col in svoc_df.columns:
            svoc_df[col] = svoc_df[col].map(clean_str)
        else:
            raise KeyError(f"svoc 映射文件缺少列: {col}")

    # 生成 before svocId -> current svocId 映射字典
    svoc_map = dict(
        zip(
            svoc_df[BEFORE_SVOC_COL],
            svoc_df[CURRENT_SVOC_COL],
        )
    )

    # NEW_CSV 去重：同一个 matchingId 只保留最后一条
    new_df = new_df.drop_duplicates(subset=["matchingId"], keep="last")

    # ========= 旧文件和新文件按 svoc_id / matchingId 关联 =========
    merged = old_df.merge(
        new_df,
        how="left",
        left_on="custom:svoc_id",
        right_on="matchingId",
        suffixes=("", "_new"),
    )

    # ========= 过滤：只保留 svocid 在 NEW_CSV 或 SVOC_MAP_CSV 中出现的行 =========
    valid_svoc_from_new = set(new_df["matchingId"].dropna().map(clean_str))
    valid_svoc_from_map = set(svoc_df[BEFORE_SVOC_COL].dropna().map(clean_str))
    valid_all_svoc = valid_svoc_from_new.union(valid_svoc_from_map)

    merged = merged[merged["custom:svoc_id"].map(clean_str).isin(valid_all_svoc)].reset_index(drop=True)

    # ========= 主循环：邮箱、电话、svocid 替换 + tag =========
    tags = []

    for idx, row in merged.iterrows():
        email_updated = False
        phone_updated = False
        svoc_updated = False

        # ---------- 邮箱逻辑 ----------
        old_email_col = row.get("email", "")
        old_username = row.get("Username", "")
        old_preferred = row.get("preferred_username", "")

        # 旧邮箱对比优先用 email 列，其次 Username / preferred_username
        old_email_for_compare = normalize_email(
            old_email_col or old_username or old_preferred
        )

        new_email_raw = row.get("emailAddressNumber", "")
        new_email_norm = normalize_email(new_email_raw)

        if is_valid_email(new_email_raw) and new_email_norm:
            if old_email_for_compare != new_email_norm:
                # 覆盖 email / Username / preferred_username
                merged.at[idx, "email"] = new_email_norm
                merged.at[idx, "Username"] = new_email_norm
                merged.at[idx, "preferred_username"] = new_email_norm
                email_updated = True

        # ---------- 电话逻辑 ----------
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

        # ---------- svocId 映射逻辑 ----------
        old_svoc = row.get("custom:svoc_id", "")
        old_svoc_clean = clean_str(old_svoc)

        if old_svoc_clean in svoc_map:
            current_svoc = clean_str(svoc_map[old_svoc_clean])
            if current_svoc and current_svoc != old_svoc_clean:
                merged.at[idx, "custom:svoc_id"] = current_svoc
                svoc_updated = True

        # ---------- tag 逻辑（细分 7 种情况） ----------
        if not (email_updated or phone_updated or svoc_updated):
            tag = "original"
        elif phone_updated and not email_updated and not svoc_updated:
            tag = "overwrite_phone"
        elif email_updated and not phone_updated and not svoc_updated:
            tag = "overwrite_email"
        elif svoc_updated and not phone_updated and not email_updated:
            tag = "overwrite_svocid"
        elif phone_updated and email_updated and not svoc_updated:
            tag = "overwrite_phone_email"
        elif phone_updated and svoc_updated and not email_updated:
            tag = "overwrite_phone_svocid"
        elif svoc_updated and email_updated and not phone_updated:
            tag = "overwrite_svocid_email"
        else:  # 三个都改了
            tag = "overwrite_all"

        tags.append(tag)

    merged["tag"] = tags

    # ========= 输出 CSV =========
    out_cols = [
        "UserStatus",
        "preferred_username",
        "Username",
        "email",
        "phone_number",
        "custom:svoc_id",
        "tag",
    ]
    output_df = merged[out_cols]
    output_df.to_csv(OUTPUT_CSV, index=False)

    print(f"处理完成，已输出到: {OUTPUT_CSV}")


if __name__ == "__main__":
    main()
