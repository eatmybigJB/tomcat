```python
from uuid import uuid4
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



# 建立连接
conn = Connection(str(uuid4()), server, 445)
conn.connect()

session = Session(conn, username=username, password=password)
session.connect()

tree = TreeConnect(session, fr"\\{server}\{share}")
tree.connect()

# 构造 Open 对象
file = Open(tree, remote_path)

# create() 里传入所有必需参数
file.create(
    desired_access=FilePipePrinterAccessMask.GENERIC_WRITE,
    share_access=ShareAccess.FILE_SHARE_READ,
    create_disposition=CreateDisposition.FILE_OVERWRITE_IF,
    create_options=CreateOptions.FILE_NON_DIRECTORY_FILE,
    impersonation_level=ImpersonationLevel.Impersonation,
    file_attributes=FileAttributes.FILE_ATTRIBUTE_NORMAL,
)

# 写入数据
file.write(data, 0)
file.close()

tree.disconnect()
session.disconnect()
conn.disconnect()

print("✅ 上传成功")
