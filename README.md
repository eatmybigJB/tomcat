```python
# 0) 先在 Security Group 放通新端口（示例用 2222）

# 1) 同时监听 22 和 2222（双栈），只改 socket 覆盖
sudo mkdir -p /etc/systemd/system/ssh.socket.d
sudo tee /etc/systemd/system/ssh.socket.d/port-override.conf >/dev/null <<'EOF'
[Socket]
# 先清空默认，再分别加 v4/v6 的 22 与 2222（过渡期保留 22）
ListenStream=
ListenStream=0.0.0.0:22
ListenStream=[::]:22
ListenStream=0.0.0.0:2222
ListenStream=[::]:2222
BindIPv6Only=both
FreeBind=yes
EOF

# 2) 只重启 socket（不要重启 ssh 服务）
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
systemctl status ssh.socket --no-pager
ss -lntp | egrep ':22|:2222' || true

# 3) 开新终端测试新端口能登录
# ssh -p 2222 ubuntu@<IP>

# 4) 测试成功后，移除 22，只保留 2222
sudo tee /etc/systemd/system/ssh.socket.d/port-override.conf >/dev/null <<'EOF'
[Socket]
ListenStream=
ListenStream=0.0.0.0:2222
ListenStream=[::]:2222
BindIPv6Only=both
FreeBind=yes
EOF

sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
ss -lntp | grep ':2222' || true

# 5) 最后把 SG 里的 22 删掉
