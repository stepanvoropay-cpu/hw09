## Задание 1

Создаем нового пользователя "pig" и задаем срок действия его пароля:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo useradd -m -s /bin/zsh pig
[sudo] password for deathpod: 
                                                                                                                                                   
┌──(deathpod㉿kali)-[~]
└─$ sudo passwd pig                
New password: 
Retype new password: 
passwd: password updated successfully
                                                                                                                                                   
┌──(deathpod㉿kali)-[~]
└─$ sudo chage -M 45 pig

┌──(deathpod㉿kali)-[~]
└─$ sudo chage -l pig
Last password change                                    : May 09, 2026
Password expires                                        : Jun 23, 2026
Password inactive                                       : never
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 45
Number of days of warning before password expires       : 7```

Настроим количество попыток входа и время блокировки:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo nano /etc/pam.d/common-auth 

auth    required    pam_faillock.so    preauth    silent    deny=3    unlock_time=120

auth    [success=1 default=ignore]      pam_unix.so nullok

auth    [default=die]    pam_faillock.so    authfail    deny=3    unlock_time=120

auth    sufficient    pam_faillock.so    authsucc

┌──(deathpod㉿kali)-[~]
└─$ sudo nano /etc/pam.d/common-account 

account required                        pam_permit.so

account required pam_faillock.so```

## Задание 2

Создаем группу и добавляем в нее пользователей:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo groupadd galera               
                                                                                                                                                   
┌──(deathpod㉿kali)-[~]
└─$ sudo usermod -aG galera dot
                                                                                                                                                   
┌──(deathpod㉿kali)-[~]
└─$ sudo usermod -aG galera pig```

Создаем общую директорию, назначаем владельца и настраиваем доступ:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo mkdir /home/galera_int           
                                                                                                                                                   
┌──(deathpod㉿kali)-[~]
└─$ sudo chown root:galera /home/galera_int 
                                                                                                                                                   
┌──(deathpod㉿kali)-[~]
└─$ sudo chmod 770 /home/galera_int

┌──(root㉿kali)-[/home/deathpod]
└─# ls -lah /home/galera_int 
total 8.0K
drwxrwx--- 2 root galera 4.0K May  9 11:57 .
drwxr-xr-x 6 root root   4.0K May  9 11:57 ..```

Настраиваем автоматическое сохранение принадлежности:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo chmod g+s /home/galera_int

┌──(root㉿kali)-[/home/deathpod]
└─# ls -lah /home/galera_int
total 8.0K
drwxrws--- 2 root galera 4.0K May  9 12:03 .
drwxr-xr-x 6 root root   4.0K May  9 11:57 ..
-rw-rw-r-- 1 pig  galera    0 May  9 12:03 123.txt```

## Задание 3

Задаем конфиг о запрете чтения, записи в указанную директорию, сохранив остальной функционал:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo nano /etc/apparmor.d/usr.bin.myapp

/usr/bin/myapp {
        deny /etc/secret.conf r,
        deny /tmp/restricted/** w,
        /** mr,
        /usr/bin/myapp mr,
        /usr/bin/bash ix,
}```

Загружаем профиль и проверяем работает ли он:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo apparmor_parser -r /etc/apparmor.d/usr.bin.myapp

┌──(deathpod㉿kali)-[~]
└─$ sudo aa-status                     
apparmor module is loaded.
2 profiles are loaded.
2 profiles are in enforce mode.
   /usr/bin/myapp
   docker-default
0 profiles are in complain mode.
0 profiles are in prompt mode.
0 profiles are in kill mode.
0 profiles are in unconfined mode.
0 processes have profiles defined.
0 processes are in enforce mode.
0 processes are in complain mode.
0 processes are in prompt mode.
0 processes are in kill mode.
0 processes are unconfined but have a profile defined.
0 processes are in mixed mode.```

Проверяем на запись:

```
┌──(deathpod㉿kali)-[/etc/apparmor.d]
└─$ touch 123.txt           
touch: cannot touch '123.txt': Permission denied```

## Задание 4

Зададим условный ip-адрес, который будет блокироваться:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo iptables -A INPUT -s 192.168.50.25 -j DROP```

Зададим отслеживание попыток подключения:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --set --name SSH_LIMIT```

Зададим блокировку при превышении количества попыток за минуту:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -m recent --update --seconds 60 --hitcount 4 --name SSH_LIMIT -j DROP```

Зададим разрешение нормальных SSH-подключений, не заблокированных ранее:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW -j ACCEPT```

Зададим разрешение уже установленных соединений или моединение оборвется:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT```

Сохраним изменения, чтоб остались после перезагрузки:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo netfilter-persistent save
run-parts: executing /usr/share/netfilter-persistent/plugins.d/15-ip4tables save
run-parts: executing /usr/share/netfilter-persistent/plugins.d/25-ip6tables save```

## Задание 5

Установим аудит сервис:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo apt install -y auditd audispd-plugins```

Запустим сервис и включим автозагрузку:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo systemctl enable auditd              
Created symlink '/etc/systemd/system/multi-user.target.wants/auditd.service' → '/usr/lib/systemd/system/auditd.service'.
                                                                                                                                                   
┌──(deathpod㉿kali)-[~]
└─$ sudo systemctl start auditd```

Добавим правила отслеживания:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo nano /etc/audit/rules.d/audit.rules

# Increase the buffers to survive stress events.

-w /etc/passwd -p rwxa -k critical_passwd
-w /etc/shadow -p rwxa -k critical_shadow
-w /etc/sudoers -p rwxa -k critical_sudoers
-w /etc/group -p rwxa -k critical_group```

Применим и проверим загруженные правила:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo augenrules --load

┌──(deathpod㉿kali)-[~]
└─$ sudo auditctl -l      
-w /etc/passwd -p rwxa -k critical_passwd
-w /etc/shadow -p rwxa -k critical_shadow
-w /etc/sudoers -p rwxa -k critical_sudoers
-w /etc/group -p rwxa -k critical_group```

Настроим отправку сообщений в системный журнал:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo nano /etc/audit/plugins.d/syslog.conf

┌──(deathpod㉿kali)-[~]
└─$ sudo cat /etc/audit/plugins.d/syslog.conf

active = yes
direction = out
path = /sbin/audisp-syslog
type = always 
args = LOG_INFO
format = string```

Перезапустим сервисы:

```
┌──(deathpod㉿kali)-[~]
└─$ systemctl restart auditd

┌──(deathpod㉿kali)-[~]
└─$ systemctl restart rsyslog```

Просмотрим события из audit.log конкретного пользователя, где comm="cat" - команда, auid - исходный пользователь, uid/euid - эффективный UID, name="/etc/passwd" - файл, к которому обратились, success=yes - разрешение доступа:

```
┌──(deathpod㉿kali)-[~]
└─$ sudo ausearch -k critical_passwd -ua pig

time->Sat May  9 17:53:22 2026
type=PROCTITLE msg=audit(1778363602.083:1250): proctitle=636174002F6574632F706173737764
type=PATH msg=audit(1778363602.083:1250): item=0 name="/etc/passwd" inode=133470 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=0 cap_fe=0 cap_fver=0 cap_frootid=0
type=CWD msg=audit(1778363602.083:1250): cwd="/home/deathpod"
type=SYSCALL msg=audit(1778363602.083:1250): arch=c000003e syscall=257 success=yes exit=3 a0=ffffffffffffff9c a1=7ffcafaa520d a2=0 a3=0 items=1 ppid=46628 pid=48229 auid=1000 uid=1002 gid=1002 euid=1002 suid=1002 fsuid=1002 egid=1002 sgid=1002 fsgid=1002 tty=pts1 ses=6 comm="cat" exe="/usr/bin/cat" subj=unconfined key="critical_passwd"```

Просмотр в "syslog":

```
┌──(deathpod㉿kali)-[~]
└─$ sudo grep "audit" /var/log/syslog | grep "passwd" | grep "pig"```

