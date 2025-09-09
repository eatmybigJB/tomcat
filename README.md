```python
# -*- coding: utf-8 -*-
import time
from concurrent.futures import ThreadPoolExecutor
from botocore.exceptions import ClientError

# 成功/失败日志文件（可按需改名/改路径）
phone_verify_success_path = "phone_verify_success.log"
phone_verify_failed_path  = "phone_verify_failed.log"


def log_failed_phone_verify(username: str, phone_number: str, error_message: str):
    """记录失败日志"""
    with open(phone_verify_failed_path, 'a', encoding='utf-8') as f:
        f.write(f"username={username}, phone_number={phone_number}, error={error_message}\n")


def _admin_mark_phone_verified(client, user_pool_id: str, username: str, max_retries: int = 8) -> bool:
    """
    仅负责把该用户标记为 phone 已验证（phone_number_verified=true）。
    带指数退避的限频重试；成功返回 True，失败抛异常（由上层捕获记录）。
    """
    retries = 0
    while retries < max_retries:
        try:
            client.admin_update_user_attributes(
                UserPoolId=user_pool_id,
                Username=username,
                UserAttributes=[{"Name": "phone_number_verified", "Value": "true"}]
            )
            print(f"[OK] mark phone verified for user={username}")
            return True

        except ClientError as e:
            code = e.response.get("Error", {}).get("Code")
            if code == "TooManyRequestsException":
                retries += 1
                wait_time = 2 ** retries  # 指数退避
                print(f"[429] {username} throttled. retry in {wait_time}s (#{retries})")
                time.sleep(wait_time)
                continue
            # 其它错误让上层处理
            raise
    raise RuntimeError(f"exceeded max_retries when marking phone verified for {username}")


# 简化版：仅当 users 里 phone_number_verified 明确为 False 才触发更新
from concurrent.futures import ThreadPoolExecutor

def verify_phone_b2b(user_infos, cognito_client, user_pool_id, max_workers: int = 10):
    """
    批量把“手机未验证”的用户标记为已验证。
    仅依赖 user_infos 中自带的 phone_number_verified 字段，不做在线查询。
    需要字段：username(或 sub)、phone_number、phone_number_verified
    """

    # 可选：清空日志
    try:
        open(phone_verify_success_path, 'w', encoding='utf-8').close()
        open(phone_verify_failed_path,  'w', encoding='utf-8').close()
    except Exception:
        pass

    scanned = queued = success = skipped_verified = skipped_missing = failed = 0
    futures = []

    def _is_true(v) -> bool:
        return str(v).strip().lower() in ("true", "1")

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        for info in user_infos:
            username = info.get("username") or info.get("sub")
            phone_number = info.get("phone_number")
            phone_verified = info.get("phone_number_verified")

            # 基本字段校验
            if not username or not phone_number:
                skipped_missing += 1
                continue

            scanned += 1

            # 已验证就跳过
            if _is_true(phone_verified):
                skipped_verified += 1
                continue

            # 未验证 -> 提交更新任务
            fut = executor.submit(_admin_mark_phone_verified, cognito_client, user_pool_id, username)
            futures.append((username, phone_number, fut))
            queued += 1

        # 汇总异步结果
        for username, phone_number, fut in futures:
            try:
                if fut.result():
                    success += 1
                    with open(phone_verify_success_path, 'a', encoding='utf-8') as f:
                        f.write(f"username={username}, phone_number={phone_number}, action=marked_verified\n")
            except Exception as e:
                failed += 1
                log_failed_phone_verify(username, phone_number, str(e))

    print(f"[SUMMARY] scanned={scanned}, queued={queued}, success={success}, "
          f"skipped_verified={skipped_verified}, skipped_missing={skipped_missing}, failed={failed}")
