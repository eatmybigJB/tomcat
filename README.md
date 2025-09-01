```python
import pandas as pd
import os

# 获取脚本路径
script_dir = os.path.dirname(os.path.realpath(__file__))

# 文件路径
file_a = os.path.abspath(os.path.join(script_dir, "b2c_new_output_cognito_ct.csv"))
file_b = os.path.abspath(os.path.join(script_dir, "new_list.csv"))
output_file = os.path.abspath(os.path.join(script_dir, "svocid_diff.csv"))

# 读取 CSV
df_a = pd.read_csv(file_a, dtype=str)
df_b = pd.read_csv(file_b, dtype=str)

# 只保留 nric 和 svocid 字段
df_a_sub = df_a[['nric', 'svocid']].dropna()
df_b_sub = df_b[['nric', 'svocid']].dropna()

# 合并，比较 svocid
merged = pd.merge(df_a_sub, df_b_sub, on="nric", suffixes=('_a', '_b'))

# 找出 svocid 不一致的记录
diff = merged[merged['svocid_a'] != merged['svocid_b']][['nric', 'svocid_a']]

# 保存到新的 CSV
diff.to_csv(output_file, index=False)

print(f"已保存结果到: {output_file}")

