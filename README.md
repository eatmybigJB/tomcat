```python
def close(self):
    """
    关闭连接（按 文件->树->会话->连接 的顺序；强制关闭）
    """
    for obj, method in (
        (getattr(self, "tree", None), "disconnect"),
        (getattr(self, "session", None), "disconnect"),
        (getattr(self, "conn", None), "disconnect"),
    ):
        if obj is not None:
            try:
                if method == "disconnect" and hasattr(obj, "disconnect"):
                    obj.disconnect(True)  # 强制断开
                else:
                    getattr(obj, method)()
            except Exception:
                pass
