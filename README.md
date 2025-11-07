```python
def _process_single_user_reset_password(
    user_info,
    cognito_client,
    user_pool_id,
    permanent,
    s3_client,
    base_log_template,
):
    username = user_info.get("Username", "")
    if not username:
        return None

    password = generate_password_b2b_strong(length=16)
    log = copy.deepcopy(base_log_template)
    log["username"] = username
    log["action"] = "reset_user_password"
    log["generated_password"] = password

    with b2b_exception_handler(s3_client, "reset_user_password", log):
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
