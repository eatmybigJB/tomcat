```python
import copy
from concurrent.futures import ThreadPoolExecutor, as_completed
import boto3

# 引入你的上下文管理器
from b2b_exception_handler import b2b_exception_handler
from common_function.s3_action import s3_record  # 你已有的函数

# ===================== 单个用户逻辑 =====================
def _process_single_user_reset_password(
    user_info,
    cognito_client,
    user_pool_id,
    permanent,
    s3_client,
    base_log_template,
):
    """
    每个用户：
      - 生成密码
      - set_user_password
      - reset_after_password
      - 自动记录成功/失败日志到 S3
    """
    username = user_info.get("Username", "")
    if not username:
        return None

    password = generate_password_b2b_strong(length=16)

    # 每个线程独立的日志字典
    log = copy.deepcopy(base_log_template)
    log["username"] = username
    log["action"] = "reset_user_password"
    log["generated_password"] = password

    with b2b_exception_handler(s3_client, "reset_user_password", log):
        # 执行 Cognito 操作
        pwd_ok = set_user_password(
            cognito_client,
            user_pool_id,
            username,
            password,
            permanent,
        )

        reset_ok = _reset_after_password(
            cognito_client,
            user_pool_id,
            username,
            pwd_ok,
        )

        log["pwd_ok"] = pwd_ok
        log["reset_ok"] = reset_ok
        log["result"] = "success" if (pwd_ok and reset_ok) else "failed"

        return {
            "username": username,
            "password": password,
            "pwd_ok": pwd_ok,
            "reset_ok": reset_ok,
        }


# ===================== 批量重置函数 =====================
def reset_password_b2b(
    user_infos,
    cognito_client,
    user_pool_id,
    permanent,
    s3_client,
    base_log_template,
):
    failed_password_updates_path = "/tmp/failed_password_updates.txt"

    with open(failed_password_updates_path, "w") as file:
        file.write("")

    successful_updates = 0

    with ThreadPoolExecutor(max_workers=10) as executor:
        futures = [
            executor.submit(
                _process_single_user_reset_password,
                user_info,
                cognito_client,
                user_pool_id,
                permanent,
                s3_client,
                base_log_template,
            )
            for user_info in user_infos
            if user_info.get("Username")
        ]

        for future in as_completed(futures):
            try:
                result = future.result()
                if not result:
                    continue

                username = result["username"]
                password = result["password"]
                pwd_ok = result["pwd_ok"]
                reset_ok = result["reset_ok"]

                if pwd_ok and reset_ok:
                    successful_updates += 1
                    with open(failed_password_updates_path, "a") as file:
                        file.write(f"email={username}, password={password}\n")

            except Exception as e:
                print(f"[ThreadError] {e}")

    with open(failed_password_updates_path, "a") as file:
        file.write(f"\nTotal successful updates: {successful_updates}\n")


# ===================== Lambda 入口函数 =====================
def lambda_handler(event, context):
    permanent = True

    attributes_for_reset_password = [
        "Username",
        "custom:password_update_date",
        "UserStatus",
    ]

    users_reset_password = get_cognito_users_attributes_pwd_expired(
        cognito_client,
        user_pool_id,
        attributes_for_reset_password,
    )

    # 创建 S3 客户端
    s3_client = boto3.client("s3")

    # 基础日志模板
    base_log_template = {
        "source": "bulk_reset_password_lambda",
        "request_id": context.aws_request_id if context else None,
        "trigger_event": event,
    }

    reset_password_b2b(
        users_reset_password,
        cognito_client,
        user_pool_id,
        permanent,
        s3_client,
        base_log_template,
    )

    return {"statusCode": 200}
