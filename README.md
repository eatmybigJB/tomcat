```python
def upload(self, local_file: str, chunk_size: int = 1024 * 1024):
    """
    上传本地文件到 remote_path（分块写，默认 1MiB，避免超过协商的最大写入大小）
    """
    file_obj = None
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
    关闭连接（按 文件->树->会话->连接 的顺序；容错处理）
    """
    # 文件已在 upload() 里关闭，这里只断开通道
    for obj, method in (
        (getattr(self, "tree", None), "disconnect"),
        (getattr(self, "session", None), "disconnect"),
        (getattr(self, "conn", None), "disconnect"),
    ):
        if obj is not None:
            try:
                getattr(obj, method)()
            except Exception:
                pass
