```python
import pyzipper
from pathlib import Path

def zip_with_password(input_file: str, zip_path: str, password: str) -> str:
    """
    用 AES 加密把 input_file 打成 zip，返回 zip 绝对路径
    """
    in_path = Path(input_file)
    z_path  = Path(zip_path)
    z_path.parent.mkdir(parents=True, exist_ok=True)

    with pyzipper.AESZipFile(z_path, 'w',
                             compression=pyzipper.ZIP_DEFLATED,
                             encryption=pyzipper.WZ_AES) as zf:
        zf.setpassword(password.encode('utf-8'))
        # 也可以：zf.setencryption(pyzipper.WZ_AES, nbits=256)
        zf.write(in_path, arcname=in_path.name)

    return str(z_path.resolve())

# ⭐ 新增：压缩并加密
zip_password = os.getenv("ZIP_PASSWORD", "ChangeMe!")  # 建议放到 Lambda 环境变量
zip_path = os.path.splitext(export_users_csv_path_b2c)[0] + ".zip"
zip_path = zip_with_password(export_users_csv_path_b2c, zip_path, zip_password)

# ⭐ 上传 ZIP 到 SMB（把 remote_path 改成 .zip）
remote_path = f"{remote_folder}/{generate_filename('Cognito')}.zip"
