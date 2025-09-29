```python
import uuid
from smbprotocol.connection import Connection, ConnectionCache
from smbprotocol.session import Session
from smbprotocol.tree import TreeConnect


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

        # ==== 关键新增：清理全局连接缓存，避免复用 ====
        ConnectionCache.clear()

        # 建立连接
        self.conn = Connection(uuid.uuid4(), server, 445)
        self.conn.connect()

        # 会话
        self.session = Session(self.conn, username=username, password=password)
        self.session.connect()

        # 进入共享
        self.tree = TreeConnect(self.session, fr"\\{server}\{share}")
        self.tree.connect()
