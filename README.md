```python
def get_tmp_path(filename: str) -> str:
    # Lambda 环境下固定用 /tmp
    if os.environ.get("AWS_LAMBDA_FUNCTION_NAME"):
        tmpdir = "/tmp"
    else:
        # 本地环境用系统临时目录（Windows 会是 C:\Users\<you>\AppData\Local\Temp）
        tmpdir = tempfile.gettempdir()

    p = Path(tmpdir) / filename
    p.parent.mkdir(parents=True, exist_ok=True)  # 确保目录存在
    return str(p)
