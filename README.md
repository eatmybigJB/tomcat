```python
import pandas as pd

# 修改这里的文件路径
main_csv = "main.csv"       # 主 CSV (包含 Username 列)
ref_csv = "ref.csv"         # 参考 CSV (包含 username 和 phone_number 等列)
output_csv = "main_with_new_phone.csv"  # 输出新文件

# 读取两个文件，强制字符串类型，避免手机号被转为数字
df_main = pd.read_csv(main_csv, dtype=str, keep_default_na=False)
df_ref = pd.read_csv(ref_csv, dtype=str, keep_default_na=False)

# 统一用户名大小写，避免大小写不一致导致匹配不上
df_main["Username_norm"] = df_main["Username"].str.strip().str.lower()
df_ref["username_norm"] = df_ref["username"].str.strip().str.lower()

# 如果参考 CSV 里有多条相同 username，只保留第一条
df_ref = df_ref.drop_duplicates(subset=["username_norm"], keep="first")

# 合并
merged = df_main.merge(
    df_ref[["username_norm", "phone_number"]],
    left_on="Username_norm",
    right_on="username_norm",
    how="left"
)

# 新列 new_phone
merged["new_phone"] = merged["phone_number"].fillna("")

# 删除辅助列
merged = merged.drop(columns=["Username_norm", "username_norm", "phone_number"])

# 保存到新文件
merged.to_csv(output_csv, index=False, encoding="utf-8-sig")

print(f"✅ 已生成 {output_csv}")

