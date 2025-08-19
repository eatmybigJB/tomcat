```python
import pandas as pd
import os

script_dir = os.path.dirname(os.path.realpath(__file__))
file_a = os.path.abspath(os.path.join(script_dir, "b2c_new_output_cognito_ct.csv"))
file_b = os.path.abspath(os.path.join(script_dir, "new_list.csv"))

# 读取csv
a = pd.read_csv(file_a)
b = pd.read_csv(file_b)

# 统一列名为小写
a.columns = a.columns.str.lower()
b.columns = b.columns.str.lower()

# 构造小写的匹配键，忽略大小写和前后空格
a["_key"] = a["preferred_username"].astype(str).str.strip().str.lower()
b["_key"] = b["username"].astype(str).str.strip().str.lower()

# 合并（用小写键匹配）
result = b.merge(a[["_key", "custom:nric"]], on="_key", how="left")

# 输出结果只保留原始username和nric
result = result[["username", "custom:nric"]].rename(columns={"custom:nric": "nric"})

file_result = os.path.abspath(os.path.join(script_dir, "result.csv"))

# 保存结果
result.to_csv(file_result, index=False)

print("已生成 result.csv，内容如下：")
print(result.head())
