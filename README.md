```python
import pandas as pd
import os

script_dir = os.path.dirname(os.path.realpath(__file__))
file_a = os.path.abspath(os.path.join(script_dir, "b2c_new_output_cognito_ct.csv"))
file_b = os.path.abspath(os.path.join(script_dir, "new_list.csv"))

# 读取csv
a = pd.read_csv(file_a)
b = pd.read_csv(file_b)

# 统一列名为小写，避免大小写不一致的问题
a.columns = a.columns.str.lower()
b.columns = b.columns.str.lower()

# 合并，根据username匹配
result = b.merge(a[['username', 'custom:nric']], on="username", how="left")

# 保存结果
result.to_csv("result.csv", index=False)

print("已生成 result.csv，内容如下：")
print(result.head())
