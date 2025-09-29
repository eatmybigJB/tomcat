```python
import uuid
from smbprotocol.connection import Connection
from smbprotocol.session import Session
from smbprotocol.tree import TreeConnect

def _clear_smb_cache_best_effort():
    """
    兼容不同 smbprotocol 版本的“清理连接缓存”：
    - 新版本：ConnectionCache.clear()
    - 旧版本：尝试清理模块中的私有缓存容器
    - 都不行：静默忽略（交给 disconnect(True) 兜底）
    """
    # 1) 新版本（若存在）
    try:
        from smbprotocol.connection import ConnectionCache  # type: ignore
        try:
            ConnectionCache.clear()  # type: ignore
            return
        except Exception:
            pass
    except Exception:
        pass

    # 2) 旧版本：尝试私有变量名
    try:
        import smbprotocol.connection as _conn_mod  # type: ignore
        for name in (
            "_CONNECTION_CACHE",
            "_connection_cache",
            "_CONNECTIONS",
            "_connections",
        ):
            cache = getattr(_conn_mod, name, None)
            if cache is not None and hasattr(cache, "clear"):
                try:
                    cache.clear()
                    return
                except Exception:
                    pass
    except Exception:
        pass
    # 3) best-effort，不抛异常


class SmbUploader:
    def __init__(self, server, share, username, password, remote_path):
        """
        初始化上传器（在建立连接之前尝试清理缓存，避免复用旧会话）
        """
        self.server = server
        self.share = share
        self.username = username
        self.password = password
        self.remote_path = remote_path

        # === 关键新增：建立新连接前清理缓存（兼容多版本） ===
        _clear_smb_cache_best_effort()

        # 建立连接
        self.conn = Connection(uuid.uuid4(), server, 445)
        self.conn.connect()

        # 会话
        self.session = Session(self.conn, username=username, password=password)
        self.session.connect()

        # 进入共享
        self.tree = TreeConnect(self.session, fr"\\{server}\{share}")
        self.tree.connect()

    def close(self):
        """
        关闭连接：树 -> 会话 -> 连接；使用强制断开，避免底层复用
        """
        for obj in (getattr(self, "tree", None), getattr(self, "session", None)):
            if obj is not None:
                try:
                    obj.disconnect(True)  # 强制
                except Exception:
                    pass
        if getattr(self, "conn", None) is not None:
            try:
                self.conn.disconnect(True)  # 强制
            except Exception:
                pass
