```python
from uuid import uuid4
from smbprotocol.connection import Connection
from smbprotocol.session import Session
from smbprotocol.tree import TreeConnect
from smbprotocol.open import Open, CreateDisposition, CreateOptions, FilePipePrinterAccessMask, ShareAccess

# 连接
conn = Connection(str(uuid4()), server, 445)
conn.connect()

session = Session(conn, username=username, password=password)
session.connect()

tree = TreeConnect(session, fr"\\{server}\{share}")
tree.connect()

# ✅ 构造 Open 对象，只给 tree 和 path
file = Open(tree, remote_path)

# ✅ 在 create() 里指定访问方式、共享模式等
file.create(
    desired_access=FilePipePrinterAccessMask.GENERIC_WRITE,
    share_access=ShareAccess.FILE_SHARE_READ,
    create_disposition=CreateDisposition.FILE_OVERWRITE_IF,
    create_options=CreateOptions.FILE_NON_DIRECTORY_FILE
)

file.write(data, 0)
file.close()

tree.disconnect()
session.disconnect()
conn.disconnect()

print("✅ 上传成功")
