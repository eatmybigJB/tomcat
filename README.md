```python
#!/usr/bin/env python3
import pandas as pd
import os
import re

script_dir = os.path.dirname(os.path.realpath(__file__))

# ======= 改成你自己的文件路径（按你刚才的要求） =======
OLD_CSV = os.path.join(script_dir, "users-b2c-daily-full3.csv")
NEW_CSV = os.path.join(script_dir, "CEP_SG.SvocGoldenRecords.csv")      # 图二
SVOC_MAP_CSV = os.path.join(script_dir, "CEP_SG.change_svocId.csv")     # 图三
OUTPUT_CSV = os.path.join(script_dir, "users-b2c-daily-replace.csv")
# ======================================================


def clean_str(s: str) -> str:
    """去除常见不可见字符和 BOM，并 strip。"""
    if s is None:
        return ""
    s = str(s)
    for ch in ["\u200b", "\u200c", "\u200d", "\ufeff", "\u2060",
               "\u00a0", "\u202c", "\u202d", "\u202e", "\u200e", "\u200f"]:
        s = s.replace(ch, "")
    # 全角加号转半角
    s = s.replace("＋", "+")
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
    return f"+{cc}{mp}"


def is_valid_phone_sg(phone: str) -> bool:
    """新加坡电话规则：+65 + 8位数字（连+号共 11 个字符）。"""
    if not phone:
        return False
    return bool(re.match(r"^\+65\d{8}$", phone))


def is_valid_phone_general(country_code: str, phone: str) -> bool:
    """
    非新加坡电话：
    只要 countryCode 和 mobilePhoneNumber 都有值就算合法。
    """
    return bool(clean_str(country_code)) and bool(clean_str(phone))


# ==================== 主流程 ====================

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
        if col not in old_df.columns:
            raise KeyError(f"旧文件缺少列: {col}")
        old_df[col] = old_df[col].map(clean_str)

    # ========= 清理 NEW_CSV 关键列 =========
    for col in ["matchingId", "emailAddressNumber", "countryCode", "mobilePhoneNumber"]:
        if col not in new_df.columns:
            raise KeyError(f"新文件缺少列: {col}")
        new_df[col] = new_df[col].map(clean_str)

    # ========= 清理 SVOC_MAP_CSV 关键列 =========
    CURRENT_SVOC_COL = "current svocId"
    BEFORE_SVOC_COL = "before svocId"

    for col in [CURRENT_SVOC_COL, BEFORE_SVOC_COL]:
        if col not in svoc_df.columns:
            raise KeyError(f"svoc 映射文件缺少列: {col}")
        svoc_df[col] = svoc_df[col].map(clean_str)

    # before svocId -> current svocId 映射
    svoc_map = dict(zip(svoc_df[BEFORE_SVOC_COL], svoc_df[CURRENT_SVOC_COL]))

    # NEW_CSV 去重：同一个 matchingId 只保留最后一条
    new_df = new_df.drop_duplicates(subset=["matchingId"], keep="last")

    # ========= 只保留有用的 svocid 行 =========
    valid_svoc_from_new = set(new_df["matchingId"].dropna().map(clean_str))
    valid_svoc_from_map = set(svoc_map.keys())  # before svocId
    valid_all_svoc = valid_svoc_from_new.union(valid_svoc_from_map)

    base_df = old_df[
        old_df["custom:svoc_id"].map(lambda x: clean_str(x) in valid_all_svoc)
    ].copy()

    # ========= 计算 lookup_svocid：用来去 NEW_CSV 查数据 =========
    def resolve_lookup_svocid(old_svoc):
        x = clean_str(old_svoc)
        return svoc_map.get(x, x)

    base_df["lookup_svocid"] = base_df["custom:svoc_id"].map(resolve_lookup_svocid)

    # ========= 用 lookup_svocid 和 NEW_CSV 关联，拿最新 email / phone =========
    merged = base_df.merge(
        new_df,
        how="left",
        left_on="lookup_svocid",
        right_on="matchingId",
        suffixes=("", "_new"),
    )

    # ========= 主循环：邮箱、电话、svocid 替换 + tag =========
    tags = []

    for idx, row in merged.iterrows():
        email_updated = False
        phone_updated = False
        svoc_updated = False

        # ---------- 邮箱逻辑 ----------
        old_email = row.get("email", "")
        new_email_raw = row.get("emailAddressNumber", "")
        new_email_norm = normalize_email(new_email_raw)

        # 邮箱是否无效或缺失（用于 original 细分）
        email_invalid_or_missing = False
        if not new_email_raw or not is_valid_email(new_email_raw):
            email_invalid_or_missing = True

        if is_valid_email(new_email_raw):
            if normalize_email(old_email) != new_email_norm:
                # 只更新 email 列，不动 Username / preferred_username
                merged.at[idx, "email"] = new_email_norm
                email_updated = True

        # ---------- 电话逻辑 ----------
        old_phone_norm = normalize_phone(row.get("phone_number", ""))

        # 保留 raw_cc 用于判断是否存在“特殊符号”
        raw_cc = row.get("countryCode", "")
        cc = clean_str(raw_cc)
        mp = clean_str(row.get("mobilePhoneNumber", ""))

        new_phone_full = build_phone(cc, mp)

        # 电话是否无效或缺失（用于 original 细分）
        raw_cc_stripped = str(raw_cc).strip()
        has_weird_char_in_cc = bool(raw_cc_stripped) and (not raw_cc_stripped.isdigit())

        if has_weird_char_in_cc:
            # 只要 countryCode 中出现非数字字符（例如前面有不可见符号），整个电话视为非法
            phone_invalid_or_missing = True
        else:
            if cc == "65":
                phone_invalid_or_missing = not is_valid_phone_sg(new_phone_full)
            else:
                phone_invalid_or_missing = not is_valid_phone_general(cc, mp)

        # 通过合法性检查才允许覆盖
        if not phone_invalid_or_missing:
            if old_phone_norm != normalize_phone(new_phone_full):
                merged.at[idx, "phone_number"] = new_phone_full
                phone_updated = True

        # ---------- svocId 映射逻辑 ----------
        old_svoc = clean_str(row.get("custom:svoc_id", ""))
        lookup_svoc = clean_str(row.get("lookup_svocid", ""))

        if lookup_svoc and lookup_svoc != old_svoc:
            merged.at[idx, "custom:svoc_id"] = lookup_svoc
            svoc_updated = True

        # ---------- tag 逻辑（细分 original + 各种 overwrite） ----------
        if not (email_updated or phone_updated or svoc_updated):
            # 原来会是 orignal 的行，这里根据邮箱/电话情况细分
            if email_invalid_or_missing and phone_invalid_or_missing:
                tag = "orignal_ignore_phone_email"
            elif email_invalid_or_missing:
                tag = "orignal_ignore_email"
            elif phone_invalid_or_missing:
                tag = "orignal_ignore_phone"
            else:
                tag = "orignal"
        elif email_updated and not phone_updated and not svoc_updated:
            tag = "overwrite_email"
        elif phone_updated and not email_updated and not svoc_updated:
            tag = "overwrite_phone"
        elif svoc_updated and not phone_updated and not email_updated:
            tag = "overwrite_svocid"
        elif phone_updated and email_updated and not svoc_updated:
            tag = "overwrite_phone_email"
        elif phone_updated and svoc_updated and not email_updated:
            tag = "overwrite_phone_svocid"
        elif svoc_updated and email_updated and not phone_updated:
            tag = "overwrite_svocid_email"
        else:
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

