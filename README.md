## otus-pam

### Запретить всем пользователям, кроме группы admin, логин в выходные (суббота и воскресенье), без учета праздников

Создаём двух юзеров и меняем им пароли
```
[root@pam ~]# useradd otusadm && useradd otus
[root@pam ~]# echo "Otus2023" | sudo passwd --stdin otusadm
Changing password for user otusadm.
passwd: all authentication tokens updated successfully.
[root@pam ~]# echo "Otus2023" | sudo passwd --stdin otus   
Changing password for user otus.
passwd: all authentication tokens updated successfully.
```
Создаем группу admin и добавляем туда юзеров root, vagrant и otusadm
```
[root@pam ~]# groupadd -f admin
[root@pam ~]# usermod otusadm -a -G admin && usermod root -a -G admin && usermod vagrant -a -G admin
[root@pam ~]# cat /etc/group
admin:x:1003:otusadm,root,vagrant
```
Проверяем, можем ли мы подключиться к этой машине пользователем otus
```
[root@pam ~]# ssh otus@192.168.57.10
The authenticity of host '192.168.57.10 (192.168.57.10)' can't be established.
ECDSA key fingerprint is SHA256:faUshInweOpBCidWQdQLVOXcvZj0k5xXJhmAO1Mne1A.
ECDSA key fingerprint is MD5:2a:f8:e3:01:ef:f0:a3:28:46:06:01:5c:5f:7b:cf:1c.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '192.168.57.10' (ECDSA) to the list of known hosts.
otus@192.168.57.10's password: 
[otus@pam ~]$
```
Затем пишем небольшой скрипт
```
#!/bin/bash
if [ $(date +%a) = "Sat" ] || [ $(date +%a) = "Sun" ]; then
 if getent group admin | grep -qw "$PAM_USER"; then
        exit 0
      else
        exit 1
    fi
  else
    exit 0
fi
```
И настраиваем PAM
```
#%PAM-1.0
auth       substack     password-auth
auth       include      postlogin
# Used with polkit to reauthorize users in remote sessions
account    required     pam_exec.so  /usr/local/bin/login.sh 
account    required     pam_nologin.so
account    include      password-auth
password   include      password-auth
# pam_selinux.so close should be the first session rule
session    required     pam_selinux.so close
session    required     pam_loginuid.so
# pam_selinux.so open should only be followed by sessions to be executed in the user context
session    required     pam_selinux.so open env_params
session    required     pam_namespace.so
session    optional     pam_keyinit.so force revoke
session    include      password-auth
session    include      postlogin
# Used with polkit to reauthorize users in remote sessions
```
Проверка. Я задал дату 3 июня 2023, суббота
```
[root@pam ~]# date
Sat Jun  3 12:40:24 UTC 2023
```
```
[root@pam ~]# ssh otusadm@192.168.57.10
otusadm@192.168.57.10's password: 
Last login: Sat Jun  3 12:33:29 2023 from 192.168.57.10

[otusadm@pam ~]$ exit
logout
Connection to 192.168.57.10 closed.
[root@pam ~]# ssh otus@192.168.57.10
otus@192.168.57.10's password: 
/usr/local/bin/login.sh failed: exit code 1
Authentication failed.
```
Как видно, залогиниться в субботу можно только пользователю из группы admin, обычнй пользователь будет "отшит" системой до понедельника
