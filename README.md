# base_setup_ubuntu_server
Шпаргалка по стартовой настройке Ubuntu Server

## Обновление

Как только скачали и установили – выполним обновление пакетов и перезагрузку:

```bash
sudo apt update && \
sudo apt dist-upgrade -y && \
sudo apt autoclean -y && \
sudo apt clean -y && \
sudo apt autoremove -y && \
sudo reboot
```

Если необходимо - обновляем дистрибутив до актуальной версии:

`sudo do-release-upgrade`

---

Ubuntu предоставляет инструмент unattended-upgrades для автоматического получения и установки исправлений безопасности и других важных обновлений для вашего сервера:

`sudo apt install unattended-upgrades`

Конфигурацию можно отредактировать с помощью nano: `sudo nano /etc/apt/apt.conf.d/50unattended-upgrades`

Если вы внесли изменения в конфигурацию, перезагрузите сервис unattended-upgrades, чтобы они вступили в силу:

`sudo systemctl reload unattended-upgrades.service`

---

Если не делали при установке, отключаем вход под root  пользователем и заводим пользователя **faralex-dev** с ограниченными правами, из под которого будем работать

```bash
useradd -m faralex-dev  && \
passwd faralex-dev

usermod -aG sudo faralex-dev && \
sudo chsh -s /bin/bash faralex-dev
```

Если не делали при установке, указываем имя хоста

```bash
sudo hostnamectl set-hostname faralex-dev-server
```

Пробуем подключиться под созданным пользователем. Если все работает отключаем вход от имени root:

```bash
sudo passwd -l root
```

Установить необходимые локали:

```bash
sudo dpkg-reconfigure locales
```

Установить агента proxmox (если это виртуальная машина proxmox)

```bash
sudo apt-get install qemu-guest-agent -y
sudo systemctl start qemu-guest-agent
sudo systemctl enable qemu-guest-agent
```

## Устанавливаем время на сервере

Спросить который час

```bash
date "+%H:%M:%S   %d/%m/%y"
```

Настраиваем свой часовой пояс (вызвать меню выбора или установить сразу)

```bash
# sudo dpkg-reconfigure tzdata
sudo timedatectl set-timezone Europe/Moscow
```

Проверяем и смотрим, запущен-ли сервис синхронизации времени. В Ubuntu демон systemd timesync:

```bash
timedatectl status
sudo systemctl status systemd-timesyncd
```

Файл конфигурации для systemd-timesyncd — это /etc/systemd/timesyncd.conf
```bash
sudo nano /etc/systemd/timesyncd.conf
		NTP=ntp.msk-ix.ru
		NTP=ntp1.vniiftri.ru
```

Запустить и сделать systemd-timesyncd активным можно так:

```bash
sudo systemctl enable systemd-timesyncd.service
sudo systemctl start systemd-timesyncd.service
```

И синхронизируем аппаратные часы с системным временем:

```bash
sudo /sbin/hwclock --systohc --localtime
timedatectl
```

---

Вы не обязаны использовать инструменты systemd для реализации NTP. Можно использовать старую версию ntpd

Для установки ntpd из терминала введите:

```bash
sudo apt install ntp -y
```

Отредактируйте `/etc/ntp.conf` для добавления/удаления серверов.

В крупных отечественных организациях идёт рекомендация на использование публичных сервисов от известной компании MSK-IX. Это компания - крупнейшая в стране точка обмена трафиком со своими дата-центрами и каналами связи. 

Ещё один от ВНИИФТРИ - государственный научный центр РФ. Крупнейший метрологический центр международного уровня.
```bash
sudo nano /etc/ntpsec/ntp.conf
		server ntp.msk-ix.ru
		server ntp1.vniiftri.ru
		pool 0.ubuntu.pool.ntp.org
		pool 1.ubuntu.pool.ntp.org
		pool 2.ubuntu.pool.ntp.org
		pool 3.ubuntu.pool.ntp.org
```

После изменений конфигурационного файла вам надо перезапустить ntpd:

```bash
sudo service ntp restart
```


## **SSH**

```bash
sudo nano /etc/ssh/sshd_config
```

Правим строчки

```bash
PermitRootLogin no
PermitEmptyPasswords no
PubkeyAuthentication yes
```

Перезапускаем SSH

```bash
sudo systemctl restart sshd
```

### Настройка входа по сертификату в SSH

Генерировать ключи будем алгоритмом ed25519 

```bash
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519 -C "faralex-dev@example.com"
```
```
-o : указывает сохранить приватный ключ с использованием нового OpenSSH формата вместо PEM. На самом деле, эта опция по умолчанию включена при генерации ключа Ed25519.
-a : указывает количество циклов KDF (Key Derivation Function, функции формирования ключа). Высокие значения приведут к замедлению проверки парольной фразы, увеличивая её устойчивость к перебору при краже приватного ключа.
-t : указывает тип создаваемого ключа, в нашем случае это Ed25519.
-f : указывает имя файла для сохранения ключа. Если вы хотите, чтобы ключ подхватывался автоматически вашим SSH-агентом, то он должен быть сохранён в дефолтной директории  .ssh домашней директории вашего пользователя.
-C : указывает комментарий. Он несёт исключительно информационный смысл и может содержать что угодно. Обычно он заполняется данными <login>@<hostname> того, кто сгенерировал ключ.
```

Вы можете найти свежесозданный приватный ключ в файле  `~/.ssh/id_ed25519` и публичный ключ в файле `~/.ssh/id_ed25519.pub` Всегда помните, что для аутентификации на удалённых хостах следует прописывать именно публичный ключ.

Публичный ключ переносим на машину откуда будем подключаться к серверу в каталог .ssh профиля пользователя. В Windows сюда `C:\Users\faralex-dev\.ssh\authorized_keys`  В один файл authorized_keys можно добавить несколько открытых ключей.

На сервере можно авторизацию по паролям отключить `PasswordAuthentication no`

Можно указать путь к закрытому ключу, который нужно использовать для SSH аутентификации: 
```bash
ssh faralex-dev@192.168.1.1 -i "C:\Users\faralex-dev\.ssh\id_ed25519
```

## Установка Fail2ban

Задача - ограничение попыток входа по ssh. Анализируя журналы, fail2ban обнаруживает повторяющиеся неудачные попытки аутентификации и автоматически устанавливает правила брандмауэра для отбрасывания трафика, исходящего с IP-адреса злоумышленника.

```bash
sudo apt install fail2ban -y
```


Правим конфиг-файл

```bash
sudo nano /etc/fail2ban/jail.conf
```

Раскомментируем строки:

```bash
ignoreip = 127.0.0.1/8
```

```bash
destemail = root@localhost //куда будет отправлено сообщение
sendername = Fail2Ban // имя отправителя
mta = mail // сервис при помощи которого будет отправлен email
```

```bash
enabled =true
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

```bash
sudo systemctl status fail2ban
```

В большинстве случаев конфигурации по умолчанию должно быть достаточно. Тем не менее, полезно понимать, что это за значения по умолчанию и как их можно изменить в соответствии с вашими потребностями.

В стандартной конфигурации fail2ban защитит SSH-сервер и заблокирует злоумышленника на 10 минут после 5 неудачных попыток входа в систему в течение 10 минут. Файл конфигурации по умолчанию можно найти в /etc/fail2ban/jail.conf. Файл хорошо документирован и в основном не требует пояснений. Имейте в виду, что вам не следует вносить какие-либо изменения в этот файл, так как он может быть перезаписан во время обновления fail2ban.

Чтобы установить пользовательские настройки и в дальнейшем при обновлении они не были изменены, надо файлы .conf переименовать в .local. Fail2ban будет автоматически предпочитать local файл относительно conf файлов

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo systemctl restart fail2ban
```

## **Сетевая безопасность**

```bash
sudo nano /etc/sysctl.conf
```

Раскомментируем строки

```bash
# prevent some spoofing attacks
net.ipv4.conf.default.rp_filter=1
net.ipv4.conf.all.rp_filter=1

# Uncomment the next line to enable TCP/IP SYN cookies
net.ipv4.tcp_syncookies=1

# Do not accept ICMP redirects (prevent MITM attacks)
net.ipv4.conf.all.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0

#====== Do not accept IP source route packets (we are not a router) ======
#net.ipv4.conf.all.accept_source_route = 0
#net.ipv6.conf.all.accept_source_route = 0

# Do not send ICMP redirects (we are not a router)
net.ipv4.conf.all.send_redirects = 0

# Log Martian Packets
net.ipv4.conf.all.log_martians = 1
```

Перезапускаем сервис

```bash
sudo sysctl -p
```

## Установка и настройка брандмауэра UFW (по желанию, конечно)

UFW — это простой интерфейс управления брандмауэром, скрывающий сложность низкоуровневых технологий фильтрации пакетов.

Если вдруг uwf не установлен - установим:

```bash
sudo apt update
sudo apt install ufw -y
```

Политика брандмауэра по умолчанию лежит по пути **/etc/default/ufw**

Задаем базовые правила:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw limit ssh/tcp
```

Включаем фаервол

```bash
sudo ufw enable
```

Проверить статус UFW

```bash
sudo ufw status verbose
```

## **Настраиваем сообщения на почту о попытке входа на сервер под учеткой root**

```bash
sudo nano ~/.profile .
```

Вставляем в конец строку

```bash
echo 'ALERT - Root Shell Access on:' `date` `who` | mail -s "Alert: Root Access from`who | awk '{print $6}'`"  faralex-dev@faralex-dev.ru
```

### **Установка postfix**

```bash
sudo apt install mailutils -y
```

При установке выбираем опцию «Internet site» в меню

Если требуется повторно вызвать меню конфигурации

```bash
sudo dpkg-reconfigure postfix
```

Если нужна только отправка сообщений, меняем в конфиге

```bash
sudo nano /etc/postfix/main.cf
```

строку

```bash
inet_interfaces = all
```

на

```bash
inet_interfaces = loopback-only
```

Затем перегружаем сервис

```bash
sudo systemctl restart postfix
```

Проверяем работоспособность

```bash
echo 'Hello!' | mail -s 'Greeteings' faralex-dev@faralex-dev.ru
```

Ошибки логируются в файле /var/log/mail.err

### **Настройка SSH MOTD (сообщение при логине)**

```bash
sudo nano /etc/motd
```

Пишем приветственное сообщение для хакера. Например:

“This system is restricted to authorized access only. 
All activities on this system are recorded and logged. 
Unauthorized access will be fully investigated and reported 
to the appropriate law enforcement agencies.”

## **History**

Исправим недостатки стандартных настроек хранения истории команд. Для этого нужно отредактировать файл **.bashrc**, который находится в том же каталоге, что и файл с историей. Добавляем в него следующие строки:

```bash
nano  ~/.bashrc

export HISTSIZE=10000
export HISTTIMEFORMAT="%h %d %H:%M:%S "
PROMPT_COMMAND='history -a'
export HISTIGNORE="ls:ll:history:w:htop"
shopt -s histappend
shopt -s cdspell
export HISTCONTROL="ignoredups"
export HISTIGNORE="&:ls:[bf]g:exit"
```

Первый параметр увеличивает размер файла до 10000 строк. Можно сделать и больше, хотя обычно хватает такого размера. Второй параметр указывает, что необходимо сохранять дату и время выполнения команды. Третья строка вынуждает сразу же после выполнения команды сохранять ее в историю. В последней строке мы создаем список исключений для тех команд, запись которых в историю не требуется. Я привел пример самого простого списка. Можете дополнить его на свое усмотрение.

Для применения изменений необходимо разлогиниться и подключиться заново или выполнить команду:

```bash
source ~/.bashrc
```

## ServerAdmin HOwTO

Получился готовый how-to, который можно использовать при настройке:

1. Первым делом обновляю репозитории и устанавливаю все обновления. Очень часто у хостеров типовые шаблоны несвежие. Лучше сразу же всё обновить.

2️. Проверяю сетевые настройки. В основном это нужно, чтобы понять, как они управляются. У разных хостеров и в разных системах могут быть большие отличия. Иногда меняю DNS серверы. В РФ чаще всего ставлю DNS от Яндекса. Меняю, если нужно, hostname.

3️. Устанавливаю привычные утилиты и программы: htop, iftop, screen, mc, net-tools, bind9-dnsutils (bind-utils).

4️. Проверяю настройки времени, часовых поясов, автообновления. Настраиваю, если что-то не сделано. Обязательно проверяю, что обновление времени работает. Некоторые хостеры блокируют ntp порты в том числе и на выход.

5️. Меняю настройки службы SSH. В основном это смена порта с 22 на любой другой, либо разрешение/запрет аутентификации под root или по паролю. Либо всё вместе, либо по отдельности, в зависимости от потребностей.

6️. Увеличиваю глубину хранения history терминала, настраиваю мгновенную запись команды в историю, а не после выхода из сеанса. Также добавляю сохранение времени выполнения команд. [https://disnetern.ru/configure-bash-history/](https://disnetern.ru/configure-bash-history/)

7️. Если за севером постоянного наблюдения и обслуживания не будет, то ставлю пакеты и настраиваю автоматическую установку обновления безопасности. Подключаю swap, если его нет.

8️. Делаю настройку системной почты для root. Либо просто алиас с нужным ящиком добавляю, либо делаю полноценную настройку отправки через внешний smtp сервер. [https://disnetern.ru/configure-bash-history/](https://disnetern.ru/configure-bash-history/)

9️. В завершении настраиваю Firewall, либо отключаю, если не нужен.

После всех настроек обязательно перезагружаю сервер и убеждаюсь, что он нормально стартует с заданными настройками, что все службы запущены (ntpd, sshd, подключается swap и т.д.). В основном это нужно, чтобы проверить настройки Firewall и сети, если они менялись.

---

В крупных отечественных организациях идёт рекомендация на использование публичных сервисов от известной компании MSK-IX. Это компания - крупнейшая в стране точка обмена трафиком со своими дата-центрами и каналами связи. Думаю, это разумно и обоснованно. Вот их NTP сервер (https://kb.msk-ix.ru/public/ntp-server/):

ntp.msk-ix.ru

Я бы к нему в список добавил ещё один от ВНИИФТРИ:

ntp1.vniiftri.ru

Вот список DNS серверов (https://kb.msk-ix.ru/public/dns-server/):

dns.msk-ix.ru  62.76.76.62
dns2.msk-ix.ru 62.76.62.76

