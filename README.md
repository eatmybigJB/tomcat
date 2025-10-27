```python
def admin_set_user_mfa_preference(client, user_pool_id: str, username: str, max_retries: int = 8) -> bool:
    retries = 0
    while retries < max_retries:
        try:
            client.admin_set_user_mfa_preference(
                UserPoolId=user_pool_id,
                Username=username,
                SMSMfaSettings={
                    "Enabled": True,
                    "PreferredMfa": True
                },
                SoftwareTokenMfaSettings={
                    "Enabled": False,
                    "PreferredMfa": False
                }
            )
            print(f"[OK] set MFA active (SMS only) for user={username}")
            return True

        except ClientError as e:
            code = e.response.get("Error", {}).get("Code")

            if code == "TooManyRequestsException":
                retries += 1
                wait_time = 2 ** retries  # 指数退避
                print(f"[429] {username} throttled. retry in {wait_time}s (#{retries})")
                time.sleep(wait_time)
                continue

            # 其他错误让上层处理
            print(f"[Error] failed to set MFA for {username}: {code}")
            raise

    raise RuntimeError(f"exceeded max_retries when setting MFA for {username}")
