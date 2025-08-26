```python
from smbprotocol.connection import Connection
from smbprotocol.session import Session
from smbprotocol.tree import TreeConnect
from smbprotocol.open import Open

server   = "sg-nas-01.sg.hsbc"
share    = "solanio_data"
username = r"HSBC\your_user"
password = "your_password"
remote_path = "Cognito/test.txt"
data = b"hello via smbprotocol"

# 建立 TCP 连接
conn = Connection(uuid="12345678-1234-5678-1234-567812345678",
                  server=server, port=445)
conn.connect()

# 认证会话（这里直接给账号密码，不需要 ClientConfig）
session = Session(conn, username=username, password=password)
session.connect()

# 进入共享
tree = TreeConnect(session, fr"\\{server}\{share}")
tree.connect()

# 打开/创建文件并写入
file = Open(tree, remote_path,
            desired_access=0x0012019F,
            share_access=0x00000007,
            create_disposition=1,     # 覆盖或新建
            create_options=0x00000020)
file.create()
file.write(data, 0)
file.close()

tree.disconnect()
session.disconnect()
conn.disconnect()
print("✅ 上传成功")
