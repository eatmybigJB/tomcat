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


def _need_mark_verified_from_cache(user_info: dict) -> bool | None:
    """
    如果 user_info 里本身带有 phone_number_verified，就直接判断是否需要标记。
    - 返回 True  => 需要标记
    - 返回 False => 不需要标记
    - 返回 None  => 不知道（没带该字段），交给在线查询判定
    """
    v = user_info.get("phone_number_verified")
    if v is None:
        return None
    # 兼容 True/False 或 'true'/'false' 字符串
    v_str = str(v).strip().lower()
    return v_str not in ("true", "1")


def _get_phone_verified_online(client, user_pool_id: str, username: str) -> bool:
    """
    在线查询该用户 phone_number_verified 当前状态；返回 True/False。
    若查询不到该属性，视为 False（未验证）。
    """
    resp = client.admin_get_user(UserPoolId=user_pool_id, Username=username)
    attrs = {a["Name"]: a["Value"] for a in resp.get("UserAttributes", [])}
    return str(attrs.get("phone_number_verified", "false")).lower() == "true"


def verify_phone_b2b(user_infos: list[dict], cognito_client, user_pool_id: str,
                     max_workers: int = 10) -> None:
    """
    批量把“手机未验证”的用户更新成已验证。
    - user_infos: 需要包含 username（或 sub）、phone_number；若带 phone_number_verified 更佳（可减少在线查询）
    - cognito_client: boto3 的 cognito-idp 客户端
    - user_pool_id: 用户池 ID
    - max_workers: 线程数
    """

    # 启动前清空日志
    open(phone_verify_success_path, 'w', encoding='utf-8').close()
    open(phone_verify_failed_path,  'w', encoding='utf-8').close()

    total = 0
    queued = 0
    success = 0
    skipped = 0

    futures = []
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        for info in user_infos:
            # 兼容：优先 username，没有则用 sub 充当
            username = info.get("username") or info.get("sub") or ""
            phone_number = info.get("phone_number", "")

            # 基本字段校验
            if not username or not phone_number:
                skipped += 1
                continue

            total += 1

            # 先用缓存字段判断；没有就在线查一下
            need_mark = _need_mark_verified_from_cache(info)
            if need_mark is None:
                try:
                    already_verified = _get_phone_verified_online(cognito_client, user_pool_id, username)
                    need_mark = not already_verified
                except Exception as e:
                    # 查询都失败了，记失败并略过该用户
                    log_failed_phone_verify(username, phone_number, f"admin_get_user failed: {e}")
                    continue

            if not need_mark:
                # 已经是 verified，跳过
                skipped += 1
                continue

            # 异步提交真正的“标记为已验证”动作
            fut = executor.submit(_admin_mark_phone_verified, cognito_client, user_pool_id, username)
            futures.append((username, phone_number, fut))
            queued += 1

        # 汇总结果
        for username, phone_number, fut in futures:
            try:
                if fut.result():
                    success += 1
                    with open(phone_verify_success_path, 'a', encoding='utf-8') as f:
                        f.write(f"username={username}, phone_number={phone_number}, action=marked_verified\n")
            except Exception as e:
                msg = str(e)
                print(f"[ERR] mark phone verified failed: user={username}, err={msg}")
                log_failed_phone_verify(username, phone_number, msg)

    print(f"[SUMMARY] scanned={total}, queued_update={queued}, success={success}, skipped={skipped}")
