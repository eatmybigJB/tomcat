```python
import pandas as pd
import os

script_dir = os.path.dirname(os.path.realpath(__file__))

# === 文件路径 ===
file1 = os.path.join(script_dir, 'corepass_active_email.csv')
file2 = os.path.join(script_dir, 'users-b2c-daily-full.csv')
output = os.path.join(script_dir, 'corepass_active_email-login.csv')

# === 读取文件 ===
df1 = pd.read_csv(file1, dtype=str)
df2 = pd.read_csv(file2, dtype=str)

# === 清洗空值 ===
df1["Client Email"] = df1["Client Email"].fillna("").str.strip()
df2["email"] = df2["email"].fillna("").str.strip().str.lower()
df2["UserStatus"] = df2["UserStatus"].fillna("").str.strip().str.upper()

# === 生成邮箱到状态的映射 ===
status_map = dict(zip(df2["email"], df2["UserStatus"]))

# === 根据 Client Email 判断 login ===
def get_login(email):
    if not email or email.upper() == "NULL":
        return ""
    email_lower = email.lower()
    if email_lower not in status_map:
        return "not existed"
    status = status_map[email_lower]
    if status in {"CONFIRMED", "RESET_REQUIRED"}:
        return "yes"
    return "no"

df1["login"] = df1["Client Email"].apply(get_login)

# === 输出结果 ===
df1.to_csv(output, index=False, encoding="utf-8-sig")
print("✅ Done. Saved to", output)

