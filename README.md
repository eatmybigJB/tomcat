```python
# OpenSSH 私钥格式（默认）
ssh-keygen -t rsa -b 4096 -C "nginx" -f ./nginx

# 如果你明确需要 PEM（BEGIN RSA PRIVATE KEY 那种）
ssh-keygen -t rsa -b 4096 -m PEM -C "nginx" -f ./nginx

sudo mkdir -p /etc/systemd/system/ssh.socket.d
printf "[Socket]\nListenStream=\nListenStream=33\n" | sudo tee /etc/systemd/system/ssh.socket.d/port-override.conf

# 重新加载并切换到 33
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket

# 可选：确认端口
ss -lntp | grep -E ':(22|33)\b' || true
systemctl status ssh.socket --no-pager
