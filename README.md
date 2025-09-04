```python
# === 写出 ===
# 将 NaN 替换成指定字符串，比如 "N/A"
diff = diff.fillna("N/A")

diff.to_csv(output_file, index=False, encoding="utf-8")
print(f"已保存到结果表：{output_file}")
