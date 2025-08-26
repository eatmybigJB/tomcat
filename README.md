```python
import uuid
from smbprotocol.connection import Connection
from smbprotocol.session import Session
from smbprotocol.tree import TreeConnect
from smbprotocol.open import (
    Open,
    CreateDisposition,
    CreateOptions,
    FilePipePrinterAccessMask,
    ShareAccess,
    ImpersonationLevel,
    FileAttributes,
)


class SmbUploader:
    def __init__(self, server, share, username, password, remote_path):
        """
        初始化上传器
        """
        self.server = server
        self.share = share
        self.username = username
        self.password = password
        self.remote_path = remote_path

        # 建立连接
        self.conn = Connection(str(uuid.uuid4()), server, 445)
        self.conn.connect()

        # 会话
        self.session = Session(self.conn, username=username, password=password)
        self.session.connect()

        # 进入共享
        self.tree = TreeConnect(self.session, fr"\\{server}\{share}")
        self.tree.connect()

    def upload(self, local_file: str):
        """
        上传本地文件到 remote_path
        """
        with open(local_file, "rb") as f:
            data = f.read()

        file = Open(self.tree, self.remote_path)
        file.create(
            desired_access=FilePipePrinterAccessMask.GENERIC_WRITE,
            share_access=ShareAccess.FILE_SHARE_READ,
            create_disposition=CreateDisposition.FILE_OVERWRITE_IF,
            create_options=CreateOptions.FILE_NON_DIRECTORY_FILE,
            impersonation_level=ImpersonationLevel.Impersonation,
            file_attributes=FileAttributes.FILE_ATTRIBUTE_NORMAL,
        )

        file.write(data, 0)
        file.close()
        print(f"✅ 文件 {local_file} 上传成功 -> {self.remote_path}")

    def close(self):
        """
        关闭连接
        """
        self.tree.disconnect()
        self.session.disconnect()
        self.conn.disconnect()


# ========== 使用示例 ==========
if __name__ == "__main__":
    # 假设你已经从 config.toml 读出来这些变量
    server = "sg-nas-01.sg.hsbc"
    share = "solanio_data"
    username = r"HBAP\45120987"
    password = "135798okokok!"
    remote_path = "Cognito/test.txt"

    uploader = SmbUploader(server, share, username, password, remote_path)
    uploader.upload("local_test.txt")  # 把当前目录的 local_test.txt 上传
    uploader.close()
