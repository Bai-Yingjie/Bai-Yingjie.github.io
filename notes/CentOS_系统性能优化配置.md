- [System Level performance tuning](#system-level-performance-tuning)
  - [CPU frequency](#cpu-frequency)
  - [limits](#limits)
  - [disable firewall](#disable-firewall)
  - [disable selinux](#disable-selinux)
  - [disable auditd](#disable-auditd)
  - [disable swap](#disable-swap)

# System Level performance tuning
## CPU frequency
Where performance "score" is the key index for benchmarks/applications tests, make sure CPU frequency is set to "performance", which is usually set to "conservative" by default.
```
sudo cpupower frequency-set -g performance
```
to check the current settings and CPU frequency:
```
sudo cpupower frequency-info
sudo cpupower -c all frequency-info | grep "current CPU frequency"
```

## limits
add below to `/etc/security/limits.conf` and reboot
```
* soft noproc 65535
* hard noproc 65535
* soft nofile 655350
* hard nofile 655350
* soft memlock 60397977
* hard memlock 60397977
```

## disable firewall
```
sudo systemctl stop firewalld.service
sudo systemctl disable firewalld.service
```

## disable selinux
```
sestatus
setenforce 0
sudo vi /etc/sysconfig/selinux
    SELINUX=disabled
```

## disable auditd
```
sudo systemctl stop auditd.service
sudo systemctl disable auditd.service
```

## disable swap
```
sudo swapoff
```