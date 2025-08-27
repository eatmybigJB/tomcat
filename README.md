```python
import pyzipper
from pathlib import Path

def zip_with_password(input_file: str, zip_path: str, password: str) -> str:
    in_path = Path(input_file); z_path = Path(zip_path)
    z_path.parent.mkdir(parents=True, exist_ok=True)
    with pyzipper.AESZipFile(z_path, 'w',
                             compression=pyzipper.ZIP_DEFLATED,
                             encryption=pyzipper.WZ_AES) as zf:
        zf.setpassword(password.encode('utf-8'))
        # 可选：zf.setencryption(pyzipper.WZ_AES, nbits=256)
        zf.write(in_path, arcname=in_path.name)
    return str(z_path)

