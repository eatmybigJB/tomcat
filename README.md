```python
#cloud-config

# 1) 基础设置（可选）
timezone: "UTC"
preserve_hostname: false
hostname: "ubuntu-ec2"
manage_etc_hosts: true

# 2) 创建可 sudo 用户（示例：devops）
# 方案 A：使用明文密码（简单，但不安全；仅示范）
# 方案 B：使用加密哈希（推荐，见下方“可选：使用哈希密码”）
users:
  - name: devops
    groups: [sudo, adm]
    shell: /bin/bash
    lock_passwd: false
    passwd: "$6$Hts8q....$vAq8QqY...HnQm4wYyWnJ2U1"


# chpasswd: 配置是否强制过期
chpasswd:
  expire: false

# 3) 启用 SSH 密码登录（Ubuntu cloud-init 级别）
ssh_pwauth: true

# 4) 通过 drop-in 文件显式开启 PasswordAuthentication，并禁用 root 登录
write_files:
  - path: /etc/ssh/sshd_config.d/99-cloudinit-password.conf
    owner: root:root
    permissions: '0644'
    content: |
      # Managed by cloud-init: enable password auth, disable root login
      PasswordAuthentication yes
      KbdInteractiveAuthentication no
      ChallengeResponseAuthentication no
      UsePAM yes
      PermitRootLogin no

# 5) 首次启动时的命令：重载/重启 sshd 以应用配置
runcmd:
  - [ systemctl, daemon-reload ]
  - [ systemctl, reload, ssh ]      # 若 reload 不生效，可改成 restart
  # - [ systemctl, restart, ssh ]

# 6) 可选：更新包（首启会更久）
#package_update: true
#package_upgrade: false
