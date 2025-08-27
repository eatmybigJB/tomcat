```python
import pyzipper
from pathlib import Path

def zip_with_password(input_file: str, zip_path: str, password: str, compat: bool = True) -> str:
    """
    compat=True  -> ZipCrypto（弱加密），Windows 资源管理器可直接解压
    compat=False -> AES-256（强加密），需 7-Zip/WinRAR/pyzipper 解压
    """
    in_path = Path(input_file)
    z_path  = Path(zip_path)
    z_path.parent.mkdir(parents=True, exist_ok=True)

    if compat:
        # ✅ 资源管理器可直接打开（安全性弱）
        with pyzipper.ZipFile(z_path, 'w', compression=pyzipper.ZIP_DEFLATED) as zf:
            zf.setpassword(password.encode('utf-8'))
            zf.setencryption(pyzipper.ZIP_CRYPTO)  # 传统 ZipCrypto
            zf.write(in_path, arcname=in_path.name)
    else:
        # 🔒 AES-256 强加密（资源管理器不支持）
        with pyzipper.AESZipFile(
            z_path, 'w',
            compression=pyzipper.ZIP_DEFLATED,
            encryption=pyzipper.WZ_AES
        ) as zf:
            zf.setpassword(password.encode('utf-8'))
            zf.setencryption(pyzipper.WZ_AES, nbits=256)
            zf.write(in_path, arcname=in_path.name)

    return str(z_path)
zip_with_password("/tmp/users-b2c.csv", "/tmp/users-b2c.zip", "YourPwd", compat=True)
