```python
def is_valid_email(email: str) -> bool:
    """
    验证邮箱格式是否合法
    规则参考 RFC 5322 常用子集
    """
    if not isinstance(email, str):
        return False
    
    # 允许的本地部分字符：字母、数字、点、下划线、加号、减号、百分号
    # 点不能出现在开头/结尾，也不能连续多个点
    email_regex = (
        r"^(?![.])"                      # 不能以点开头
        r"[A-Za-z0-9._%+-]+"             # 本地部分
        r"(?<![.])"                      # 不能以点结尾
        r"@"                             # @ 符号
        r"(?:[A-Za-z0-9-]+\.)+"          # 域名的每一部分
        r"[A-Za-z]{2,}$"                 # 顶级域名至少2个字母
    )
    
    return re.match(email_regex, email) is not None
