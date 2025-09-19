```python
import pandas as pd
import os
import re

# 修改这里的文件路径
script_dir = os.path.dirname(os.path.realpath(__file__))
main_csv   = os.path.abspath(os.path.join(script_dir, "b2c_new_output_cognito_ct.csv"))
ref_csv    = os.path.abspath(os.path.join(script_dir, "users-b2c-daily.csv"))
output_csv = os.path.abspath(os.path.join(script_dir, "users-b2c-daily-migrated.csv"))

# 读取两个文件，禁止字符串转数值，避免手机号等被转为数字
df_main = pd.read_csv(main_csv, dtype=str, keep_default_na=False)
df_ref  = pd.read_csv(ref_csv,  dtype=str, keep_default_na=False)

# 统一用户名大小写，避免大小写不一致导致匹配不上
df_main["Username_norm"]  = df_main["Username"].str.strip().str.lower()
df_ref["username_norm"]   = df_ref["Usernames"].str.strip().str.lower()

# 如果参考 CSV 里有多条同一 username，只保留第一条
df_ref = df_ref.drop_duplicates(subset=["username_norm"], keep="first")

# 合并
merged = df_main.merge(
    df_ref[["username_norm", "phone_numbers", "preferred_username", "UserStatus"]],
    left_on="Username_norm",
    right_on="username_norm",
    how="left"
)

# 新列 new_phone
merged["new_phone"]          = merged["phone_numbers"].fillna("")
merged["preferred_username"] = merged["preferred_username_y"].fillna("")
merged["Username"]           = merged["Username_norm"].fillna("")

# 删除辅助列
merged = merged.drop(columns=[
    "Username_norm", "username_norm", "phone_numbers",
    "preferred_username_x", "preferred_username_y", "email",
    "birthdate", "custom:svoc_id"
], errors="ignore")

final_columns = [
    "custom:nric",
    "Username",
    "preferred_username",
    "UserStatus",
    "phone_number",
    "new_phone",
    "tag"
]

# 按顺序重新排列
merged = merged[final_columns]

# 保存到新文件
merged.to_csv(output_csv, index=False, encoding="utf-8-sig")
print(f"✅ 已生成 {output_csv}")

# ========== 新增功能：找出 ref_csv 中那些不在 output_csv 里的 username ==========
# 使用与上面完全一致的规范化规则进行对比
# 1) output 侧用户名集合（规范化）
output_usernames_norm = merged["Username"].astype(str).str.strip().str.lower().unique()

# 2) ref 侧：筛出不在 output 集合中的行
mask_missing = ~df_ref["username_norm"].isin(output_usernames_norm)
missing_in_output = df_ref.loc[mask_missing].copy()

# 3) 保存时去掉辅助列（保持和原 ref_csv 更接近）
missing_in_output = missing_in_output.drop(columns=["username_norm"], errors="ignore")

missing_csv = os.path.abspath(os.path.join(script_dir, "ref_not_in_output.csv"))
missing_in_output.to_csv(missing_csv, index=False, encoding="utf-8-sig")
print(f"🔎 已导出 ref 中但不在 output 的用户名清单：{missing_csv}")
