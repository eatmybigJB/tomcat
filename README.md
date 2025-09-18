```python
final_columns = [
    "Username", 
    "preferred_username", 
    "phone_numbers", 
    "UserStatus", 
    "email", 
    "birthdate", 
    "custom:svoc_id"
]

# 按顺序重新排列
merged = merged[final_columns]

# 输出到 CSV
merged.to_csv(output_csv, index=False, encoding="utf-8-sig")
