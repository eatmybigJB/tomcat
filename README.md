```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from botocore.exceptions import ClientError
import time

def enable_user_mfa(users_info_gen, cognito_client, user_pool_id: str, max_workers: int = 10):
    """
    批量为用户启用 SMS MFA（多线程调用 admin_set_user_mfa_preference）。
    users_info_gen 是生成器，元素为 dict，至少包含 'Username' 或 'sub' 字段。
    """

    scanned = queued = success = skipped_missing = failed = 0
    futures = []

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        for info in users_info_gen:
            username = info.get("Username") or info.get("sub")

            # 无 username 跳过
            if not username:
                skipped_missing += 1
                continue

            scanned += 1
            # 提交任务
            fut = executor.submit(
                admin_set_user_mfa_preference,
                cognito_client,
                user_pool_id,
                username
            )
            futures.append((username, fut))
            queued += 1

        # 等待结果
        for username, fut in futures:
            try:
                ok = fut.result()
                if ok:
                    success += 1
                else:
                    failed += 1
            except ClientError as e:
                failed += 1
                print(f"[ERR] {username} failed: {e.response['Error'].get('Message')}")
            except Exception as e:
                failed += 1
                print(f"[ERR] {username} failed: {e}")

    print(f"""
[Summary]
  scanned={scanned}
  queued={queued}
  success={success}
  skipped_missing={skipped_missing}
  failed={failed}
""")
