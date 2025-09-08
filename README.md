```python
aws cognito-idp update-user-pool \
  --region <region> \
  --user-pool-id <user-pool-id> \
  --key-configuration KeyType=CUSTOMER_MANAGED_KEY,KmsKeyArn=arn:aws:kms:<region>:<account-id>:key/<key-id-uuid>
