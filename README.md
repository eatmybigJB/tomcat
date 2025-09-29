```python
# OpenSSH 私钥格式（默认）
ssh-keygen -t rsa -b 4096 -C "nginx" -f ./nginx

# 如果你明确需要 PEM（BEGIN RSA PRIVATE KEY 那种）
ssh-keygen -t rsa -b 4096 -m PEM -C "nginx" -f ./nginx

# 覆盖 ssh.socket（方法A）
sudo mkdir -p /etc/systemd/system/ssh.socket.d
sudo tee /etc/systemd/system/ssh.socket.d/port-override.conf >/dev/null <<'EOF'
[Socket]
# 先清空默认 ListenStream=22，再分别绑定 v4/v6 的 6443
ListenStream=
ListenStream=0.0.0.0:6443
ListenStream=[::]:6443
# 默认文件里是 ipv6-only，这里改为 both，保证 v4 能连
BindIPv6Only=both
# 保留 FreeBind=yes（可选，但安全）
FreeBind=yes
EOF

sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
sudo systemctl restart ssh   # 可选
