```python
import pandas as pd

# === 文件路径 ===
file1 = "file1.csv"
file2 = "file2.csv"
output = "output.csv"

# === 读取文件 ===
df1 = pd.read_csv(file1, dtype=str)
df2 = pd.read_csv(file2, dtype=str)

# === 清洗空值 ===
df1["Client Email"] = df1["Client Email"].fillna("").str.strip()
df2["email"] = df2["email"].fillna("").str.strip().str.lower()
df2["user_status"] = df2["user_status"].fillna("").str.strip().str.upper()

# === 生成邮箱到状态的集合 ===
valid_status = {"CONFIRMED", "RESET REQUIRED"}
valid_email_set = set(df2.loc[df2["user_status"].isin(valid_status), "email"])

# === 根据 Client Email 判断 login ===
def get_login(email):
    if not email or email.upper() == "NULL":
        return ""
    return "yes" if email.lower() in valid_email_set else "no"

df1["login"] = df1["Client Email"].apply(get_login)

# === 输出结果 ===
df1.to_csv(output, index=False, encoding="utf-8-sig")
print("✅ Done. Saved to", output)
