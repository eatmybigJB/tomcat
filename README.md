```python
# 输出文件
out_match   = os.path.abspath(os.path.join(script_dir, "users-b2c-daily-matched.csv"))
out_unmatch = os.path.abspath(os.path.join(script_dir, "users-b2c-daily-unmatched.csv"))

# 读入 AXA（全部按字符串；只把列名转小写，方便对齐 custom:nric / email）
df_ref_axa = pd.read_csv(ref_csv_axa, dtype=str).rename(columns=str.lower)

# 合并：按 nric 带入 email（merged 已存在，列：custom:nric / Username / UserStatus / phone_current ...）
merged_new = merged.merge(df_ref_axa[['custom:nric', 'email']], on='custom:nric', how='left')

# 条件：
# 1) user_status == "FORCE_CHANGE_PASSWORD"
cond_status = merged_new['UserStatus'] == 'FORCE_CHANGE_PASSWORD'

# 2) phone 形如 "+65" 后恰好 8 位数字（只能这种格式）
cond_phone = merged_new['phone_current'].str.match(r'^\+65\d{8}$', na=False)

# 3) username == email（大小写不敏感更稳；若要严格区分大小写，去掉 .str.lower()）
cond_user = merged_new['Username'].str.lower() == merged_new['email'].str.lower()

mask = cond_status & cond_phone & cond_user

# 导出
merged_new[mask].to_csv(out_match, index=False)
merged_new[~mask].to_csv(out_unmatch, index=False)

print(f"符合条件: {mask.sum()} 行 -> {out_match}")
print(f"不符合条件: {(~mask).sum()} 行 -> {out_unmatch}")

