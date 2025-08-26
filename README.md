```python
import smbprotocol
from smbprotocol.connection import Connection
from smbprotocol.session import Session
from smbprotocol.tree import TreeConnect
from smbprotocol.open import Open

# ========== 修改这里 ==========
server_ip = "192.168.1.100"   # 你的 Windows/Samba IP
share_name = "share"          # 共享名，比如 Windows 上共享文件夹的名字
username = "your_user"        # 用户名
password = "your_pass"        # 密码
local_data = b"Hello Samba via smbprotocol!"  # 测试写入内容
remote_file = "test.txt"      # 上传后的文件名
# ==============================

# 初始化 smbprotocol
smbprotocol.ClientConfig(username=username, password=password)

# 建立连接
conn = Connection(uuid="12345678-1234-5678-1234-567812345678",
                  server=server_ip, port=445)
conn.connect()

# 会话
session = Session(conn, username=username, password=password)
session.connect()

# 连接共享
tree = TreeConnect(session, fr"\\{server_ip}\{share_name}")
tree.connect()

# 打开/创建远程文件
file = Open(tree, remote_file,
            desired_access=0x0012019F,  # 读/写权限
            share_access=0x00000007,
            create_disposition=1,       # 覆盖或新建
            create_options=0x00000020)
file.create()

# 写数据
file.write(local_data, 0)

file.close()
tree.disconnect()
session.disconnect()
conn.disconnect()

print("✅ 文件上传成功！")

