```python
try:
    uploader = SmbUploader(server, share, username, password, remote_path)
    uploader.upload(file_path)
    print("✅ 上传完成")
except Exception as e:
    print(f"❌ 上传失败: {e}")
finally:
    if uploader:
        uploader.close()
