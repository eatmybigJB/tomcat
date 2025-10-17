```python
# 筛选 new_phone 以 +6565 开头的行
filtered = output[output['new_phone'].astype(str).str.match(r'^\+6565')]

# 保存到新文件
filtered_output_csv = "filtered_6565.csv"
filtered.to_csv(filtered_output_csv, index=False, encoding="utf-8-sig")

print(f"✅ 已生成 {filtered_output_csv}，共 {len(filtered)} 行匹配 +6565 开头的号码。")

