```python
merged = pd.merge(df_a_sub, df_b_sub, on="nric", how="inner")

# 找出 svocid 不一致的记录，保留 nric 和 B 文件的 svocid
diff = (
    merged.loc[merged["svocid_a"] != merged["svocid_b"], ["nric", "svocid_b"]]
          .rename(columns={"svocid_b": "svocid"})
)

# 保存结果
diff.to_csv(output_file, index=False, encoding="utf-8")
print(f"已保存结果到: {output_file}")

# 去重（完全相同的行只保留一条）
diff = diff.drop_duplicates()

# 保存结果
diff.to_csv(output_file, index=False, encoding="utf-8")
print(f"已保存结果到: {output_file}")
