```python
import os
import pandas as pd

# 修改这里的文件路径
script_dir = os.path.dirname(os.path.realpath(__file__))
main_csv = os.path.abspath(os.path.join(script_dir, "a.csv"))          # 主 CSV
ref_csv = os.path.abspath(os.path.join(script_dir, "user-b2c.csv"))   # 参考 CSV
output_csv = os.path.abspath(os.path.join(script_dir, "a-final.csv")) # 输出文件

# 读取两个文件，强制字符串类型，避免手机号被转为数字
df_main = pd.read_csv(main_csv, dtype=str, keep_default_na=False)
df_ref = pd.read_csv(ref_csv, dtype=str, keep_default_na=False)

# 统一用户名大小写，避免大小写不一致导致匹配不上
df_main["Username_norm"] = df_main["Username"].str.strip().str.lower()
df_ref["username_norm"] = df_ref["Usernames"].str.strip().str.lower()

# 如果参考 CSV 里有多条相同 Usernames，只保留第一条
df_ref = df_ref.drop_duplicates(subset=["username_norm"], keep="first")

# 合并
merged = df_main.merge(
    df_ref[["username_norm", "phones"]],
    left_on="Username_norm",
    right_on="username_norm",
    how="left"
)

# 新列 new_phone
merged["new_phone"] = merged["phones"].fillna("")

# 删除辅助列
merged = merged.drop(columns=["Username_norm", "username_norm", "phones"])

# 保存到新文件
merged.to_csv(output_csv, index=False, encoding="utf-8-sig")

print(f"✅ 已生成 {output_csv}")
