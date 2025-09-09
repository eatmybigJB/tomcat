```python
# 你现有的收集函数里，把需要的属性一并取出来（建议加入 phone_number_verified，可减少一次 admin_get_user）：
attributes_for_phone_verify = ['sub', 'username', 'name', 'email', 'phone_number', 'phone_number_verified']
users = get_cognito_users_attributes(cognito_client, user_pool_id, attributes_for_phone_verify)

# 执行批量验证
verify_phone_b2b(users, cognito_client, user_pool_id, max_workers=10)

