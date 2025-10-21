```python
import pandas as pd

# 输入文件（按需改路径）
main_path   = "main.csv"    # 含 nric, username, phone, user_status
emails_path = "emails.csv"  # 含 nric, email

# 输出文件
out_match    = "matched.csv"
out_unmatch  = "unmatched.csv"

# 读入（全部按字符串）
df_main   = pd.read_csv(main_path, dtype=str).rename(columns=str.lower)
df_email  = pd.read_csv(emails_path, dtype=str).rename(columns=str.lower)

# 合并：按 nric 带入 email
merged = df_main.merge(df_email[['nric','email']], on='nric', how='left')

# 条件：
# 1) user_status == "forcechanged password"
cond_status = merged['user_status'] == 'forcechanged password'

# 2) phone 形如 "+65" 后恰好 8 位数字（只能这个格式）
cond_phone = merged['phone'].str.match(r'^\+65\d{8}$', na=False)

# 3) username == email（直接字符串相等）
cond_user = merged['username'] == merged['email']

mask = cond_status & cond_phone & cond_user

# 导出
merged[mask].to_csv(out_match, index=False)
merged[~mask].to_csv(out_unmatch, index=False)

print(f"符合条件: {mask.sum()} 行 -> {out_match}")
print(f"不符合条件: {(~mask).sum()} 行 -> {out_unmatch}")
