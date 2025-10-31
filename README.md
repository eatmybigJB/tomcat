```python
from botocore.exceptions import ClientError
import time

def reset_user_status(client, user_pool_id, username, max_retries=8):
    """
    把用户状态重新打回“需要重置密码”（也就是 FORCE_CHANGE_PASSWORD）
    实际上是调用 admin_reset_user_password
    """
    retries = 0
    while retries < max_retries:
        try:
            client.admin_reset_user_password(
                UserPoolId=user_pool_id,
                Username=username,
            )
            print(f"User status for {username} reset to FORCE_CHANGE_PASSWORD successfully.")
            return True

        except ClientError as e:
            # 限频重试
            if e.response['Error']['Code'] == 'TooManyRequestsException':
                retries += 1
                wait_time = 2 ** retries
                print(f"Too many requests when resetting status for {username}. Retrying in {wait_time} seconds...")
                time.sleep(wait_time)
            else:
                print(f"Error resetting status for {username}: {e}")
                # 跟你 set_user_password 里一样，直接抛出去让上层记录日志
                raise ValueError(f"Error resetting status for {username}: {e}")

    print(f"Failed to reset status for {username} after {max_retries} retries.")
    return False

def reset_password_b2b(user_infos, cognito_client, user_pool_id, permanent):
    # 清空日志
    with open(failed_password_updates_path, 'w') as file:
        file.write('')

    successful_updates = 0

    with ThreadPoolExecutor(max_workers=10) as executor:
        futures = []

        for user_info in user_infos:
            username = user_info.get('Username', '')
            if not username:
                continue

            password = generate_password_b2b_strong(length=16)
            print(password)

            # 1) 先提交“设密码”的任务
            pwd_future = executor.submit(
                set_user_password,
                cognito_client,
                user_pool_id,
                username,
                password,
                permanent,
            )

            # 2) 再提交“reset 状态”的任务，但要等上面的结果
            reset_future = executor.submit(
                _reset_after_password,      # 我下面写一个小助手函数
                cognito_client,
                user_pool_id,
                username,
                pwd_future,                 # 把上面的future传进去
            )

            # 方便后面统计和写日志，把密码也存着
            futures.append((username, password, pwd_future, reset_future))

        # 收集结果
        for username, password, pwd_future, reset_future in futures:
            try:
                pwd_ok = pwd_future.result()
                reset_ok = reset_future.result()

                if pwd_ok and reset_ok:
                    successful_updates += 1
                    with open(failed_password_updates_path, 'a') as file:
                        file.write(f"email={username}, password={password}\n")

            except Exception as e:
                err = str(e)
                print(f"Error for {username}: {err}")
                log_failed_password(username, password, err)

    with open(failed_password_updates_path, 'a') as file:
        file.write(f"\nTotal successful updates: {successful_updates}\n")

def _reset_after_password(client, user_pool_id, username, pwd_future, max_retries=8):
    ok = pwd_future.result()
    if not ok:
        return False
    return reset_user_status(client, user_pool_id, username, max_retries=max_retries)
