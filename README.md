```python
import copy
from concurrent.futures import ThreadPoolExecutor, as_completed

# 这个是你已有的 context manager
# from xxx import b2b_exception_handler

# ====== 单个用户的处理函数 ======
def _process_single_user_reset_password(
    user_info,
    cognito_client,
    user_pool_id,
    permanent,
    s3_client,
    base_log_template,
    action,
):
    """
    处理一个用户：
    1. 生成密码
    2. set_user_password
    3. _reset_after_password
    4. 整个过程放在 b2b_exception_handler 里，写一条 S3 log
    """

    username = user_info.get("Username", "")
    if not username:
        # 没有用户名，直接跳过
        return None

    # 生成密码
    password = generate_password_b2b_strong(length=16)

    # 每个用户都复制一份独立的 log dict，避免多线程互相覆盖
    log = copy.deepcopy(base_log_template)
    log["action"] = action
    log["username"] = username
    # 如果你不想在 S3 里看到明文密码，可以不要这一行
    log["generated_password"] = password  

    with b2b_exception_handler(s3_client, action, log):
        # 这里出现异常也会被 context manager 捕获并写入 S3
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
            pwd_ok,   # 如果你原来传的是 future，就直接传 True/False 或者删掉这个参数
        )

        # 在 log 里记录结果，让 context manager 在 __exit__ 里一起写出去
        log["pwd_ok"] = pwd_ok
        log["reset_ok"] = reset_ok
        log["status"] = "success" if (pwd_ok and reset_ok) else "failed"

        # 返回给主线程，用来统计、写本地文件
        return {
            "username": username,
            "password": password,
            "pwd_ok": pwd_ok,
            "reset_ok": reset_ok,
        }

failed_password_updates_path = "/tmp/failed_password_updates.txt"  # 你原来的路径

def reset_password_b2b(
    user_infos,
    cognito_client,
    user_pool_id,
    permanent,
    s3_client,
    base_log_template,
):
    # 清空日志文件
    with open(failed_password_updates_path, "w") as file:
        file.write("")

    successful_updates = 0
    action = "bulk_reset_password"

    with ThreadPoolExecutor(max_workers=10) as executor:
        futures = []

        for user_info in user_infos:
            username = user_info.get("Username", "")
            if not username:
                continue

            future = executor.submit(
                _process_single_user_reset_password,
                user_info,
                cognito_client,
                user_pool_id,
                permanent,
                s3_client,
                base_log_template,
                action,
            )
            futures.append(future)

        # 收集结果
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
                    # 把成功的账号和密码写到本地文件（你原来的逻辑）
                    with open(failed_password_updates_path, "a") as file:
                        file.write(f"email={username}, password={password}\n")

            except Exception as e:
                # 如果 _process_single_user_reset_password 里面没捕获，
                # 这里还是兜个底，方便你在 CloudWatch 看
                print(f"Error in future: {e!r}")

    # 统计总数
    with open(failed_password_updates_path, "a") as file:
        file.write(f"\nTotal successful updates: {successful_updates}\n")

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

    # 这里构造一个基础 log 模板，公用的字段放这里
    base_log_template = {
        "source": "bulk_reset_password_lambda",
        "request_id": context.aws_request_id if context else None,
        # 你原来想带的其他字段可以加在这里
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
