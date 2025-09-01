```python
# 合并
merged = pd.merge(df_a_sub, df_b_sub, on="nric", how="inner")

# 找出 svocid 不一致的记录，同时保留 A 和 B 的 svocid
diff = merged.loc[merged["svocid_a"] != merged["svocid_b"], ["nric", "svocid_a", "svocid_b"]]

# 去重（如果需要）
diff = diff.drop_duplicates()

# 保存结果
diff.to_csv(output_file, index=False, encoding="utf-8")
print(f"已保存结果到: {output_file}")
