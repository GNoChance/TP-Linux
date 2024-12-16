# TP Linux

## Installation de Nginx

```
root@debian:~# sudo apt install nginx
```
## Configuration d'un pare-feu

```
root@debian:~# sudo apt install ufw 
```
```
root@debian:~# sudo ufw allow 22
```
```
root@debian:~# sudo ufw reload -y
```
```
root@debian:~# sudo ufw restart -y
```
```
root@debian:~# sudo ufw enable
```

## Configuration d'un user avec les permissions admin 

```
root@debian:~# sudo useradd gno
root@debian:~# sudo passwd gno
```
```
root@debian:~# usermod -aG sudo gno
```
```
root@debian:~# sudo nano /etc/ssh/sshd_config
```
```
Port 22
PermitRootLogin no
PasswordAuthentication yes
```
```
root@debian:~# sudo systemctl restart sshd
```
## Configuration du fail2ban

```
root@debian:~# sudo dnf install epel-release -y
```
```
root@debian:~# sudo dnf install fail2ban -y
```
```
root@debian:~# sudo nano /etc/fail2ban/jail.local
```
```
[sshd]
enabled = true
port = 22
logpath = /var/log/auth.log
maxretry = 3
bantime = 600
```
```
root@debian:~# sudo systemctl restart fail2ban
```
```
root@debian:~# sudo systemctl enable fail2ban
```
```
root@debian:~# sudo systemctl start fail2ban
```
```
root@debian:~# sudo systemctl status fail2ban
```
