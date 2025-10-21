```python
df_main = pd.read_csv(merged_path, dtype=str)
df_axa  = pd.read_csv(axa_path, dtype=str)

# --------------------------
# 标准化列名（全小写便于合并）
# --------------------------
df_main.columns = df_main.columns.str.lower()
df_axa.columns  = df_axa.columns.str.lower()

# --------------------------
# 合并 email （custom:nric 对 CUSTOM:NRIC）
# --------------------------
merged_new = df_main.merge(
    df_axa[['custom:nric', 'email']],
    on='custom:nric',
    how='left'
)

# --------------------------
# 筛选条件
# --------------------------

# 1. user_status == "FORCE_CHANGE_PASSWORD"
cond_status = merged_new['userstatus'] == 'FORCE_CHANGE_PASSWORD'

# 2. phone_current 匹配 +65 + 8位数字
cond_phone = merged_new['phone_current'].str.match(r'^\+65\d{8}$', na=False)

# 3. Username == email（忽略大小写比较更安全）
cond_user = merged_new['username'].str.lower() == merged_new['email'].str.lower()

mask = cond_status & cond_phone & cond_user

# --------------------------
# 输出
# --------------------------
merged_new[mask].to_csv(out_match, index=False)
merged_new[~mask].to_csv(out_unmatch, index=False)

print(f"符合条件: {mask.sum()} 行 -> {out_match}")
print(f"不符合条件: {(~mask).sum()} 行 -> {out_unmatch}")
