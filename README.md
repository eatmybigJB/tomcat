```python
# OpenSSH 私钥格式（默认）
ssh-keygen -t rsa -b 4096 -C "nginx" -f ./nginx

# 如果你明确需要 PEM（BEGIN RSA PRIVATE KEY 那种）
ssh-keygen -t rsa -b 4096 -m PEM -C "nginx" -f ./nginx

