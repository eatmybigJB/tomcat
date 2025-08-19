```python
import pandas as pd

# 读取两个csv
a = pd.read_csv("a.csv")
b = pd.read_csv("b.csv")

# 合并，根据username匹配
result = b.merge(a[['username', 'nric']], on="username", how="left")

# 保存结果
result.to_csv("result.csv", index=False)

print("已生成 result.csv，内容如下：")
print(result.head())
