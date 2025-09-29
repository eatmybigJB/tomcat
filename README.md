```python
# -*- coding: utf-8 -*-
import uuid
from typing import Optional

from smbprotocol.connection import Connection
from smbprotocol.session import Session
from smbprotocol.tree import TreeConnect
from smbprotocol.open import (
    Open,
    CreateDisposition,
    CreateOptions,
    ShareAccess,
    ImpersonationLevel,
    FileAttributes,
    FilePipePrinterAccessMask,
)


def _clear_smb_cache_best_effort() -> None:
    """
    兼容不同 smbprotocol 版本的“清理连接缓存”：
    - 新版本：ConnectionCache.clear()
    - 旧版本：尝试清理模块中的私有缓存容器
    - 都不行：静默忽略（交给 disconnect(True) 兜底）
    """
    # 1) 新版：ConnectionCache
    try:
        from smbprotocol.connection import ConnectionCache  # type: ignore
        try:
            ConnectionCache.clear()  # type: ignore
            return
        except Exception:
            pass
    except Exception:
        pass

    # 2) 旧版：尝试私有缓存
    try:
        import smbprotocol.connection as _conn_mod  # type: ignore
        for name in ("_CONNECTION_CACHE", "_connection_cache", "_CONNECTIONS", "_connections"):
            cache = getattr(_conn_mod, name, None)
            if cache is not None and hasattr(cache, "clear"):
                try:
                    cache.clear()
                    return
                except Exception:
                    pass
    except Exception:
        pass
    # best-effort，不抛异常


class SmbUploader:
    def __init__(
        self,
        server: str,
        share: str,
        username: str,
        password: str,
        remote_path: str,
        *,
        port: int = 445,
        timeout: float = 10.0,
    ):
        """
        初始化上传器 —— 保留你的原始写法，只是在建立连接前加了清缓存，
        并在失败时做强制断开，避免后续复用旧会话。
        """
        self.server = server
        self.share = share
        self.username = username
        self.password = password
        self.remote_path = remote_path

        self.conn: Optional[Connection] = None
        self.session: Optional[Session] = None
        self.tree: Optional[TreeConnect] = None

        # ==== 关键新增：建立新连接前清理缓存，避免复用 ====
        _clear_smb_cache_best_effort()

        # 建立连接
        self.conn = Connection(uuid.uuid4(), server, port)
        try:
            self.conn.connect(timeout=timeout)
        except Exception:
            try:
                self.conn.disconnect(True)
            except Exception:
                pass
            raise

        # 会话
        self.session = Session(self.conn, username=username, password=password)
        try:
            self.session.connect()
        except Exception:
            try:
                self.session.disconnect(True)
            except Exception:
                pass
            try:
                self.conn.disconnect(True)
            except Exception:
                pass
            raise

        # 进入共享
        self.tree = TreeConnect(self.session, fr"\\{server}\{share}")
        try:
            self.tree.connect()
        except Exception:
            try:
                self.tree.disconnect(True)
            except Exception:
                pass
            try:
                self.session.disconnect(True)
            except Exception:
                pass
            try:
                self.conn.disconnect(True)
            except Exception:
                pass
            raise

    # ====== 按照你现有的“分块上传”逻辑（默认 1MiB 一块） ======
    def upload(self, local_file: str, chunk_size: int = 1024 * 1024):
        """
        上传本地文件到 remote_path（分块写，默认 1MiB，避免超过协商的最大写入大小）
        """
        file_obj: Optional[Open] = None
        try:
            # 打开远端文件（覆盖或新建）
            file_obj = Open(self.tree, self.remote_path)
            file_obj.create(
                desired_access=FilePipePrinterAccessMask.GENERIC_WRITE,
                share_access=ShareAccess.FILE_SHARE_READ,
                create_disposition=CreateDisposition.FILE_OVERWRITE_IF,
                create_options=CreateOptions.FILE_NON_DIRECTORY_FILE,
                impersonation_level=ImpersonationLevel.Impersonation,
                file_attributes=FileAttributes.FILE_ATTRIBUTE_NORMAL,
            )

            # 本地分块读取并写入远端
            offset = 0
            with open(local_file, "rb") as f:
                while True:
                    chunk = f.read(chunk_size)
                    if not chunk:
                        break
                    file_obj.write(chunk, offset)
                    offset += len(chunk)

            print(f"✅ 文件 {local_file} 分块上传成功 -> {self.remote_path}")

        finally:
            # 先关远端文件句柄；忽略重复关闭等异常
            if file_obj is not None:
                try:
                    file_obj.close()
                except Exception:
                    pass

    def close(self):
        """
        关闭连接（按 文件->树->会话->连接 的顺序；使用强制断开，避免底层复用）
        """
        # 文件在 upload() 里已经关闭，这里只断开通道
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

