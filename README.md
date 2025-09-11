```python
aws cognito-idp delete-user-pool --user-pool-id <USER_POOL_ID> --region <REGION>
aws cognito-idp update-user-pool \
  --user-pool-id <USER_POOL_ID> \
  --deletion-protection INACTIVE \
  --region <REGION>
