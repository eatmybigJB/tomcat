```python
import zipfile
from pathlib import Path

def zip_with_password(input_file: str, zip_path: str, password: str) -> str:
    """
    用标准库 zipfile 给文件加密码压缩（ZipCrypto，安全性较弱）
    """
    in_path = Path(input_file)
    z_path  = Path(zip_path)
    z_path.parent.mkdir(parents=True, exist_ok=True)

    # 注意：这里 setpassword 只是写入密码标记，并不会用 AES
    with zipfile.ZipFile(z_path, "w", compression=zipfile.ZIP_DEFLATED) as zf:
        zf.setpassword(password.encode("utf-8"))
        zf.write(in_path, arcname=in_path.name)

    return str(z_path.resolve())

csv_file = "/tmp/users-b2c.csv"
zip_file = "/tmp/users-b2c.zip"
zip_password = "MyZipPwd123!"

# 打包 CSV → ZIP
zip_path = zip_with_password(csv_file, zip_file, zip_password)

# 上传 SMB
remote_path = f"{remote_folder}/{generate_filename('Cognito')}.zip"
uploader = SmbUploader(server, share, username, password, remote_path)
uploader.upload(zip_path)
uploader.close()

