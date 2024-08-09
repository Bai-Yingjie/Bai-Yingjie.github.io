- [ssh relay](#ssh-relay)
  - [on home server](#on-home-server)
  - [login to relay server](#login-to-relay-server)
  - [from any PC](#from-any-pc)
  - [ssh keep alive](#ssh-keep-alive)
  - [make it persistent](#make-it-persistent)

# ssh relay
you want to access your home server behind a firewall or nat from outside.
you must have a **public server(relay server)** 122.235.139.194(sh_cvmx shanghaicavm!), which both home server and outside PC can access.

## on home server
`homeserver~$ ssh -fN -R 10022:localhost:22 sh_cvmx@122.235.139.194`
10022 is an unused port on relay server

## login to relay server
```shell
ssh sh_cvmx@122.235.139.194
shanghaicavm!
```
check 10022 port, shoud be like this:
```shell
relayserver~$ sudo netstat -nap | grep 10022
tcp      0    0 127.0.0.1:10022          0.0.0.0:*               LISTEN      8493/sshd    
```

## from any PC
login to relay server
```shell
ssh sh_cvmx@122.235.139.194
shanghaicavm!
```
Then on relay server, login to home server:
```
relayserver~$ ssh -p 10022 cvm@localhost
cvmx@sh
```

## ssh keep alive
```shell
# vim /etc/ssh/ssh_config
Host *
    ServerAliveInterval 60
    ServerAliveCountMax 2
```
`sudo service ssh restart`

## make it persistent
1. passwordless ssh login, on home server
```shell
ssh-keygen -t rsa -P ''
cat ~/.ssh/id_rsa.pub | ssh root@112.74.210.173 "cat >> ~/.ssh/authorized_keys"
```
注: 有的时候还是不能免密登录, 此时要该服务端的权限 chmod 600 authorized_keys

2. `yum install autossh`
3. `homeserver~$ autossh -M 10985 -fN -o "PubkeyAuthentication=yes" -o "StrictHostKeyChecking=false" -o "PasswordAuthentication=no" -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -R 112.74.210.173:11985:localhost:22 root@112.74.210.173`
4. `ssh root@112.74.210.173 Cavium@sh`
5. `ssh -p 11985 cvm@localhost cvmx@sh`