```python
    sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak

    sudo vim /etc/ssh/sshd_config
    Port 2404

    sudo vim /etc/systemd/system/sockets.target.wants/ssh.socket
    [Socket]
    ListenStream=
    ListenStream=2404
    Accept=no
    FreeBind=yes
    sudo systemctl daemon-reload
    sudo systemctl restart sshd
