```python
c.admin_set_user_mfa_preference(
    UserPoolId=POOL,
    Username=USER,
    SMSMfaSettings={'Enabled': True, 'PreferredMfa': True},
    SoftwareTokenMfaSettings={'Enabled': False, 'PreferredMfa': False}
)
