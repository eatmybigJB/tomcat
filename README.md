```python
aws cognito-idp update-user-pool-client \
  --user-pool-id xxxxxx \
  --client-id xxxxxx \
  --supported-identity-providers 'COGNITO' \
  --allowed-o-auth-flows 'code' \
  --allowed-o-auth-scopes 'openid' 'email' \
  --allowed-o-auth-flows-user-pool-client \
  --explicit-auth-flows 'ALLOW_USER_SRP_AUTH' \
  --token-validity-units AccessToken=minutes \
  --access-token-validity 60 \
  --enable-token-customization

aws cognito-idp describe-user-pool-client \
    --user-pool-id xxxx \
    --client-id xxxxx


