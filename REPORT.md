# Отчёт по лабораторной работе

## Часть 1. Стенд и подключение по паролю

### 1. Создание учётной записи студента
С помощью команды `adduser` была создана учётная запись студента `sde-1`.

### 2. Настройка проброса порта

В VirtualBox было настроено перенаправление портов:

-   Адрес хоста: `127.0.0.1`
-   Порт хоста: `2221`
-   Порт гостя: `22`

Это позволило подключаться к SSH-серверу виртуальной машины с основного
компьютера через локальный адрес.

### 3. Настройка SSH

В конфигурационный файл `/etc/ssh/sshd_config` были внесены следующие
параметры:

-   `PermitRootLogin yes` --- разрешён вход пользователя root по SSH.
-   `MaxAuthTries 2` --- установлено ограничение в две попытки
    аутентификации.
-   `LoginGraceTime 30` --- время на прохождение аутентификации
    составляет 30 секунд.
Для проверки настроек использовалась команда `sshd -T`.

Полученный результат:

``` text
PermitRootLogin yes
MaxAuthTries 2
LoginGraceTime 30
```
После проверки конфигурации служба SSH была перезапущена.

### 4. Подключение к виртуальной машине
С основного компьютера были установлены SSH-соединения с виртуальной
машиной. 

Для подключения использовалась команда:
``` bash
dashastrepetkova@MacBook-Air-Dara-2 ~ % ssh -p 2221 sde-1@127.0.0.1
sde-1@127.0.0.1's password: 
Linux debian-c 6.12.107+deb13-arm64 #1 SMP Debian 6.12.107-1 (2026-08-29) aarch64

The programs included with the Debian GNU/Linux system are free software;
```
При первом подключении был принят и сохранён отпечаток ключа
SSH-сервера.
### 5. Просмотр подключённых пользователей и SSH-соединений

Для определения подключённых пользователей была выполнена команда `who`.

Результат:

``` text
root  sshd pts/1   2026-11-22 16:38(10.0.2.2)
sde-1  sshd pts/0   2026-11-22 16:34(10.0.2.2)
root   seat0        2026-11-22 16:30
root   tty1        2026-11-22 16:30
```
Для просмотра сетевых соединений была выполнена команда `ss -tnp`:
``` text
State   Recv-Q   Send-Q    Local Address:Port   Peer Address:Port
ESTAB    0         0           10.0.2.15:22       10.0.2.2:50194
ESTAB    0         0           10.0.2.15:22       10.0.2.2:50036
```
Обнаружены два установленных TCP-соединения (`ESTAB`). Они направлены на
порт `22` SSH-сервера виртуальной машины с адреса `10.0.2.2`.

### 6. Передача файлов с помощью SCP

Для проверки передачи файлов между основным компьютером и виртуальной
машиной использовалась команда `scp`.

Копирование файла с виртуальной машины на основной компьютер:
```bash
scp -P 2221 sde-1@127.0.0.1:/etc/hostname ~/hostname.txt
password: 
hostname                                      100%    9     9.1KB/s   00:00   
```
Копирование файла с основного компьютера на виртуальную машину:
```bash
scp -P 2221 ~/test.txt sde-1@127.0.0.1:~/
password:
test.txt                                      100%    9    11.8KB/s   00:00
```

## Часть 2. Управление пользователями и группами

### 1. Создание скрипта для добавления пользователей

Создан скрипт31:
```bash
#!/bin/bash
if [ "$#" -ne 2 ]; then
echo "Использование: ./script31 количество начальный_номер"
exit 1
fi
if [ "$EUID" -ne 0 ]; then
echo "Ошибка: запустите скрипт от root"
exit 1
fi
if ! [[ "1" =~ ^[0-9]+$ && "2" =~ ^[0-9]+$ ]] || [ "$1" -eq 0 ]; then
echo "Ошибка: введите положительные числа"
exit 1
fi
count=$1
start=$2
for ((i=start; i<start+count; i++)); do
user="u$i"
password="DerParol$i"
if id "$user" &>/dev/null; then
echo "Пользователь $user уже существует"
else
useradd -m "$user"
echo "$user:$password" | chpasswd
echo "Создан пользователь $user"
fi
done
```
`script31` принимает два параметра: 
- количество пользователей
- начальный номер.

Скрипт проверяет права запуска, количество переданных параметров и наличие пользователя в
системе. Если пользователь уже существует, его создание пропускается. В противном
случае создаётся новая учётная запись с именем `uN` и устанавливается
пароль `DerParolN`.

Для создания пяти пользователей, начиная с первого номера, был выполнен
запуск:

``` bash
./script31 5 1
```
В результате были созданы учётные записи `u1`, `u2`, `u3`, `u4` и `u5`.
### 2. Создание группы и добавление пользователей
Была создана группа `project`, после чего в неё были добавлены
пользователи `u1` и `u2`.
```bash
groupadd project
usermod -aG project u1
usermod -aG project u2
id u1
id u2
```
Результат:
```text
uid=1002(u1) gid=1002(u1) группы=1002(u1),1007(project)
uid=1003(u2) gid=1003(u2) группы=1003(u2),1007(project)
```
Оба пользователя входят в группу `project` с GID `1007`.

### 3. Просмотр UID, GID и групп пользователей
Для одновременного получения UID, GID и списка групп двух пользователей
была использована команда:
``` bash
id u1 u2
```
В результате отображаются идентификаторы пользователей, основные группы
и дополнительные группы:

``` text
uid=1002(u1) gid=1002(u1) группы=1002(u1),1007(project)
uid=1003(u2) gid=1003(u2) группы=1003(u2),1007(project)
```

### 4. Просмотр записей пользователей и групп

Для просмотра записей пользователей и групп использовалась команда
`getent`.

Записи из базы пользователей:
``` bash
getent passwd u1
getent passwd u2
getent group project
```
Результат:
```text
u1:x:1002:1002::/home/u1:/bin/sh
u2:x:1003:1003::/home/u2:/bin/sh
project:x:1007:u1,u2
```
Поля записи `/etc/passwd`:
- Имя пользователя.
- Пароль (символ `x` означает, что хеш пароля хранится в `/etc/shadow`).
- UID --- идентификатор пользователя.
- GID --- идентификатор основной группы.
- Дополнительная информация о пользователе.
- Домашний каталог.
- Оболочка пользователя.

В последней записи отображается созданная группа `project` с GID `1007`,
в которую входят пользователи `u1` и `u2`.

### 5. Вхождение под пользователем u3
Входим как пользователь u3:
```bash
su - u3
Пароль: 
whoami
id
exit
```
Результат:
```text
u3
uid=1004(u3) gid=1004(u3) группы=1004(u3)
```
Видим, что пользователь u3 не входи в группу project. 

Далле добавляем пользователь в группу:
```bash
usermod -aG project u3
id u3
```
Результат:
```text
uid=1004(u3) gid=1004(u3) группы=1004(u3),1007(project)
```
## Часть 3. Управление правами доступа и механизмом владения



## Часть 3. Управление правами доступа и механизмом владения
### 1. Настройка прав доступа к каталогам
Был создан каталог `/DATA` и настроены права доступа для пользователей и группы `project`. По условию участники группы должны иметь возможность создавать, изменять и удалять файлы, остальные пользователи - только читать.
```text
18:05:09 root ~ #mkdir /DATA
18:08:25 root ~ #chown root:project /DATA 
18:08:48 root ~ #chmod 2775 /DATA
18:09:06 root ~ #ls -ld /DATA
drwxrwsr-x 2 root project 4096 ноя 22 18:08 /DATA
```
четырёхзначный режим прав:
- первый знак задаёт специальный бит setgid
- остальные три — обычные права владельца, группы и остальных пользователей.

Владелец каталога - `root`, группа - `project`. Владельцу и
участникам группы предоставлены права на чтение, запись и выполнение,
остальным пользователям - чтение и выполнение.

Мини проверка:
```text
18:10:29 root ~ #su - u4
$ touch /DATA/test_u4
touch: невозможно выполнить touch для '/DATA/test_u4': Отказано в доступе
```
```text
18:09:26 root ~ #su - u1
$ touch /DATA/test_u1
$ ls -l /DATA
итого 0
-rw-rw-r-- 1 u1 project 0 ноя 22 18:10 test_u1
```

### 2. Создание каталога с общим доступом
Был создан каталог `/DATA/sec1`. Для него настроен специальный бит
Sticky Bit, позволяющий всем пользователям создавать файлы, но удалять и
переименовывать только собственные файлы.
```text
18:12:16 root ~ #mkdir /DATA/sec1
18:13:56 root ~ #chmod 1777 /DATA/sec1
18:14:55 root ~ #ls -ld /DATA/sec1
drwxrwsrwt 2 root project 4096 ноя 22 18:13 /DATA/sec1
```
Как читается 1777:
- 1 — включает sticky bit (бит «липкой» защиты удаления). Он не меняет права на создание файлов, а ограничивает удаление и переименование чужих записей внутри каталога.
- 7 для владельца — rwx: полный доступ: чтение, создание/изменение/удаление, вход в каталог.
- 7 для группы — rwx: такой же полный доступ.
- 7 для остальных — rwx: любой пользователь может читать список файлов, заходить в каталог и создавать там свои файлы.

Проверка:
```text
18:18:03 root ~ #su - u1
$ touch /DATA/sec1/file_u1
$ exit 
18:21:21 root ~ #su - u3
$ touch /DATA/sec1/file_u3
$ rm /DATA/sec1/file_u1
rm: невозможно удалить '/DATA/sec1/file_u1': Операция не позволена
$ rm /DATA/sec1/file_u3
$ exit
```
### 4. Настройка SUID для редактора nano
В каталог `/DATA/sec3` была скопирована программа `nano`. Для неё был установлен бит SUID, чтобы при запуске программа использовала эффективный UID владельца файла.
```text
/usr/bin/nano
21:33:12 root ~ #mkdir /DATA/sec3
21:34:23 root ~ #cp /usr/bin/nano /DATA/sec3/nano
21:34:36 root ~ #chown u1:u1 /DATA/sec3/nano
21:35:07 root ~ #chmod 4755 /DATA/sec3/nano
21:35:33 root ~ #ls -l /DATA/sec3/nano
-rwsr-xr-x 1 u1 u1 332608 ноя 22 21:34 /DATA/sec3/nano
21:36:07 root ~ #su - u3
$ /DATA/sec3/nano /DATA/sec2/test.txt
$ exit
21:54:50 root ~ #cat /DATA/sec2/test.txt
Hello
Test
LOLLOL
```
- цифра 4 включает бит setuid. Для исполняемого файла это значит, что когда любой пользователь его запускает, программа временно работает с правами владельца этого файла, а не с правами того, кто её запустил.
- 7 для владельца: читать, изменять и запускать файл.
- 5 для группы: читать и запускать, но не изменять.
- 5 для остальных: тоже читать и запускать, но не изменять

## Часть 4. Аутентификация по ключам SSH
### 1. Создание пары SSH-ключей
На основном компьютере была создана пара ключей SSH с использованием
алгоритма Ed25519. Для закрытого ключа была задана парольная фраза.
```text
dashastrepetkova@MacBook-Air-Dara-2 ~ % ssh-keygen -t ed25519
Generating public/private ed25519 key pair.
Enter file in which to save the key (/Users/dashastrepetkova/.ssh/id_ed25519)
```
Закрытый и открытый ключи сохраняются в каталоге /Users/dashastrepetkova/.ssh/id_ed25519. 
### 2. Передача открытого ключа на виртуальную машину

```text
dashastrepetkova@MacBook-Air-Dara-2 ~ % ssh-copy-id -p 2221 sde-1@127.0.0.1
Number of key(s) added:        1
```
Открытый ключ был добавлен в файл,  Это позволило настроить вход по ключу без
ввода пароля учётной записи.
```text
dashastrepetkova@MacBook-Air-Dara-2 ~ % ssh -p 2221 sde-1@127.0.0.1
Enter passphrase for key '/Users/dashastrepetkova/.ssh/id_ed25519':
```

### 3. Проверка подключения по ключу
В новом окне с основного компьютера было установлено SSH-соединение с виртуальной
машиной под пользователем `sde-1`.
```text
dashastrepetkova@MacBook-Air-Dara-2 ~ % ssh -p 2221 sde-1@127.0.0.1
Enter passphrase for key '/Users/dashastrepetkova/.ssh/id_ed25519':
```
При подключении была введена парольная фраза закрытого ключа. После
успешной аутентификации открылся терминал виртуальной машины Debian.

### 4. Настройка параметров SSH-сервера
В конфигурационный файл SSH-сервера были внесены следующие параметры:
```text
17:29:46 root ~ #sed -i 's/^PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
17:31:27 root ~ #cat >> /etc/ssh/sshd_config <<'EOF' PubkeyAuthentication yes AuthorizedKeysFile .ssh/authorized_keys PasswordAuthentication no EOF
17:32:05 root ~ #sshd -T | grep -E 'permitrootlogin|pubkeyauthentication|authorizedkeysfile|passwordauthentication'
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
authorizedkeysfile .ssh/authorized_keys
```
`cat >> /etc/ssh/sshd_config <<'EOF' PubkeyAuthentication yes AuthorizedKeysFile .ssh/authorized_keys PasswordAuthentication no EOF` Эта команда добавляет в конец /etc/ssh/sshd_config три настройки SSH-сервера: разрешает вход по SSH-ключам, указывает файл с разрешёнными ключами и запрещает вход по паролю.
- PubkeyAuthentication yes — включает аутентификацию по открытому SSH-ключу. Это значение и так обычно включено по умолчанию.
- AuthorizedKeysFile .ssh/authorized_keys — указывает, что публичные ключи пользователя сервер ищет в файле ~/.ssh/authorized_keys; путь без / считается относительным к домашнему каталогу пользователя.
- PasswordAuthentication no — запрещает вход по паролю. После перезапуска SSH подключиться можно будет только с корректным приватным ключом.
- cat >> /etc/ssh/sshd_config открывает файл на дозапись: существующие настройки не удаляются, а новый текст добавляется в самый конец.
- <<'EOF' запускает here-document: всё, что вы введёте до строки с ровно EOF, будет записано в файл. Кавычки вокруг EOF важны: они предотвращают раскрытие переменных оболочки, например $HOME, внутри добавляемого текста.

### 5. Проверка ограничений аутентификации
Также были выполнены проверки запрета входа по паролю и под
пользователем `root`. 
```text
dashastrepetkova@MacBook-Air-Dara-2 ~ % ssh -p 2221 root@127.0.0.1
root@127.0.0.1: Permission denied (publickey).
```
Проверка входа без использования аутентификации по открытому ключу:
```text
dashastrepetkova@MacBook-Air-Dara-2 ~ % ssh -o PubkeyAuthentication=no -p 2221 sde-1@127.0.0.1
sde-1@127.0.0.1: Permission denied (publickey).
```
### 6. Передача файла с помощью SCP

Для проверки передачи файла с виртуальной машины на основной компьютер
была использована команда `scp` с аутентификацией по SSH-ключу.
```text
dashastrepetkova@MacBook-Air-Dara-2 ~ % scp -P 2221 sde-1@127.0.0.1:/home/sde-1/script31 ~/script31
Enter passphrase for key '/Users/dashastrepetkova/.ssh/id_ed25519': 
script31                                      100%  703   764.5KB/s   00:00    
dashastrepetkova@MacBook-Air-Dara-2 ~ % ls -l ~/script31
-rwxr-xr-x  1 dashastrepetkova  staff  703 Nov 23 17:51 /Users/dashastrepetkova/script31
```
Сравнение хэша:
Хэш на компьютере:
```text
f535b89e387b5faf25c0e78090191a063ccaebd7d8892d11a265038e9646ec57  /Users/dashastrepetkova/script31
```
Хэш на виртуальной машине:
```text
7:47:26 root ~ #sha256sum /root/script31
f535b89e387b5faf25c0e78090191a063ccaebd7d8892d11a265038e9646ec57  /root/script31
```
## Часть 5. su, sudo, процессы и фоновый запуск

### 1. Настройка ограничений sudo для пользователя u1
sde-1 теперь входит в группу sudo, может выполнять команды через sudo, а su и su - переключают вас на root. Разница между ними — в том, насколько полностью меняется окружение.
```text
18:02:02 root ~ #usermod -aG sudo sde-1
dashastrepetkova@MacBook-Air-Dara-2 ~ %  ssh -p 2221 sde-1@127.0.0.1
sde-1@debian-c:~$ groups
sde-1 sudo users
```
Перезапускаем в терминале:
```text
sde-1@debian-c:~$ sudo -l
[sudo] password for sde-1: 
sde-1@debian-c:~$ sudo id
uid=0(root) gid=0(root) groups=0(root)
sde-1@debian-c:~$ su
Password: 
18:09:06 root /home/sde-1 #pwd /home/sde-1
18:09:13 root /home/sde-1 #echo $HOME
/root
18:09:25 root /home/sde-1 #env | head
SHELL=/bin/bash...
sde-1@debian-c:~$ su -
Password: 
18:10:05 root ~ #pwd
/root
18:10:10 root ~ #echo $HOME
/root
18:10:23 root ~ #env | head
SHELL=/bin/bash...
```
- `sde-1 sudo users` - означает, что пользователь sde-1 состоит в дополнительных группах sudo и users. Поэтому sudo -l прошёл
- команда `id` была выполнена от root: UID 0 — это root.
Команда su по умолчанию не меняет текущий каталог, но устанавливает HOME=/root, потому что вы стали root.
- sudo команда для разовой административной команды
- 
Правило было размещено в отдельном файле `/etc/sudoers.d/u1` с
использованием `visudo -f`.
Содержимое правила:
``` text
u1 ALL=(root) /usr/bin/passwd, !/usr/bin/passwd root
```
### 2. Проверка ограничений sudo
Попытка изменить пароль пользователя `u2`:
``` bash
sudo passwd u2 
```
Результат:
```text
Новый пароль: 
Повторите ввод нового пароля: 
passwd: пароль успешно обновлён
$ 
u1 смог изменить пароль u2
```
Попытка изменить пароль root:
``` bash
sudo passwd root
```
Результат:
``` text
Извините, пользователю u1 не разрешено выполнять «/usr/bin/passwd root» как root на debian-c.
```
## 3.Установка htop
```bash
18:35:22 root ~ #apt install htop -y 
18:35:47 root ~ #htop
```
Появляется интерактивный монитор процессов он постоянно обновляет список процессов поэтому цифры и строки меняются нажимаем на ф6, справа появляется окно и выбираем PERCRNT_MEM нажимаем Интер. нажимаем на ф5 и включается древовидный режим выходим   
 
### 4. Анализ процессов SSH-сессии
Для просмотра процессов использовалась команда:

``` bash
18:56:13 root ~ #pstree -sp 1124
systemd(1)───sshd-session(1116)───sshd-session(1123)───bash(1124)
18:58:38 root ~ #ps -o pid,ppid,user,stat,cmd -p 1,1116,1123,1124
    PID    PPID USER     STAT CMD
      1       0 root     Ss   /sbin/init
   1116       1 root     Ss   sshd-session: sde-1 [priv]
   1123    1116 sde-1    S    sshd-session: sde-1@pts/1
   1124    1123 sde-1    Ss+  -bash
18:58:58 root ~ #pstree -sp 1124
systemd(1)───sshd-session(1116)───sshd-session(1123)───bash(1124)
```
 - PID 1, systemd — инициализатор системы; от него запущен SSH-сервер.
 - PID 1116, sshd-session: sde-1 [priv] — привилегированный монитор этого SSH-соединения. Он работает от root и выполняет операции, которым нужны повышенные права: проверку учётной записи, аутентификацию, создание PTY и т. п.
 - PID 1123, sshd-session: sde-1@pts/1 — непривилегированный процесс сессии от имени sde-1. Он обрабатывает SSH-протокол и подключён к псевдотерминалу pts/1.
 - PID 1124, -bash — ваша интерактивная оболочка Bash, запущенная после успешного входа. Дефис в -bash означает login shell.
### 4. Завершение процессов с помощью сигналов
Для проверки управления процессами использовались сигналы:

- `SIGTERM` - запрос на штатное завершение процесса. Позволяет процессу корректно завершить работу и освободить ресурсы.
- `SIGKILL` - принудительное завершение процесса без возможности обработать сигнал.
- `SIGSTOP` - приостановка процесса.
- `SIGCONT` - продолжение приостановленного процесса.
Сначала для завершения процесса рекомендуется использовать `SIGTERM`, а
`SIGKILL` применять, если процесс не завершился самостоятельно.
Запускаем в первом окне sleep600, заходим во второе окно:
```text
sde-1@debian-c:~$ pgrep -a sleep
956 sleep 600
sde-1@debian-c:~$ ps -o pid,ppid,user,stat,cmd -p 956
    PID    PPID USER     STAT CMD
    956     876 sde-1    S+   sleep 600
sde-1@debian-c:~$ kill -SIGTERM 956
sde-1@debian-c:~$
```
PID — это номер самого процесса, PPID — номер его родительского процесса:
- PID - 956
- PPID - 876

### 5. Создание скрипта script32
Был создан скрипт `script32`, который в бесконечном цикле раз в две
секунды записывает в файл `~/script32.log` текущую дату и время, PID и
PPID процесса.
Содержимое скрипта:

``` bash
#!/bin/bash
while true; do
echo "(date '+%F %T') PID=$ PPID=$PPID" >> ~/script32.log
sleep 2
done
```
### 6. Запуск скрипта в фоновом режиме
Для запуска скрипта в фоне использовалась команда:

``` bash
sde-1@debian-c:~$ ~/script32 &
[1] 1107
sde-1@debian-c:~$ jobs -l
[1]+  1107 Запущен          ~/script32 &
sde-1@debian-c:~$ exit
```

После запуска был получен PID процесса `1107`. Затем SSH-сессия была
закрыта командой `exit`. 

### 7. Проверка работы nohup

Для повторного эксперимента скрипт был запущен с помощью `nohup`:

``` bash
nohup bash script32 &
```

После запуска был получен PID `1191`. Команда `nohup` сообщила, что ввод
игнорируется, а вывод добавляется в файл `nohup.out`.
```text
sde-1@debian-c:~$ jobs -l
[1]+  1191 Запущен          nohup ~/script32 > ~/nohup.log 2>&1 &
```

После закрытия SSH-сессии и повторного подключения процесс продолжал
работать.
Проверка:

``` bash
sde-1@debian-c:~$ ps -o pid,ppid,user,stat,cmd -p 1191
```

Результат:

``` text
PID    PPID USER     STAT CMD
   1191       1 sde-1    S    /bin/bash /home/sde-1/script32
```
PPID процесса равен `1`, то есть после завершения исходной оболочки процесс был передан системному процессу `systemd`.

При обычном запуске через `&` процесс завершился после закрытия SSH-сессии, а при запуске через `nohup` продолжил работу и запись данных в файл `script32.log`.
## Часть 6. Аудит безопасности в скрипте инвентаризации

### 1. Доработка скрипта script_passport_v3.sh

На основе предыдущей версии `script_passport_v2.sh` был создан и
доработан скрипт `script_passport_v3.sh`. Существующая структура сбора
данных, обработки и логирования была сохранена.

Содержимое скрипта:

``` bash
#!/bin/bash
# ===== Functions  ==== 
check_privileges() {
if [[ $EUID -eq 0 ]]; then
SUDO=""
PRIVILEGE_MODE="root"
elif id -nG | grep -qw sudo && sudo -v; then
SUDO="sudo"
PRIVILEGE_MODE="sudo"
else
echo "mistake:try from root sudo." >&2
exit 1
fi
}
check_tool() {
if ! command -v "$1" >/dev/null 2>&1; then
echo " utilita $1 not found. install pocket $2"
return 1
fi
}
write_section() {
{
echo
echo "===== $1 ====="
cat "$2"
} >> "$REPORT_FILE"
}
log_run() {
echo "[$(date '+%F %T')] zapusk scripta, format: v3" >> "$LOG_FILE"
}
usage() {
echo "use: $0 [-h] [output_file]"
echo "default output: PASSPORT/inventory.txt"
exit 0
}
cleanup() {
rm -f "$TMP_DIR"/*.tmp
}
# ===== Initialization =====
if [[ "$1" == "-h" ]]; then
usage
fi
REPORT_FILE="${1:-PASSPORT/inventory.txt}"
LOG_FILE="PASSPORT/log/report.log"
TMP_DIR="PASSPORT/tmp"
check_privileges
mkdir -p "$(dirname "$REPORT_FILE")" PASSPORT/log "$TMP_DIR"
trap cleanup EXIT
trap 'exit 130' INT
trap 'exit 143' TERM
log_run 
mkdir -p PASSPORT/log PASSPORT/tmp
date > PASSPORT/tmp/date.tmp 
hostname > PASSPORT/tmp/hostname.tmp 
whoami > PASSPORT/tmp/user.tmp 
who > PASSPORT/tmp/who.tmp 
pwd > PASSPORT/tmp/pwd.tmp 
ls > PASSPORT/tmp/ls.tmp
uname -a > PASSPORT/tmp/kernel.tmp
cat /etc/os-release > PASSPORT/tmp/os.tmp 
lscpu > PASSPORT/tmp/lscpu.tmp 
free -h > PASSPORT/tmp/memory.tmp 
lspci > PASSPORT/tmp/lspci.tmp 
# Users information
{
echo "Name | UID | Groups | Last login | Last password change"
getent passwd | while IFS=: read -r name pass uid gid gecos home shell; do
if [[ "$uid" -eq 0 || ( "$uid" -ge 1000 && "$uid" -ne 65534 ) ]]; then
groups=$(id -nG "$name" 2>/dev/null)
if command -v lastlog2 >/dev/null 2>&1; then
last_login=$(lastlog2 -u "$name" 2>/dev/null | tail -n 1)
elif command -v lslogins >/dev/null 2>&1; then
last_login=$(lslogins -o USER,LAST-LOGIN 2>/dev/null | awk -v u="$name" '$1 == u {print $2}')
else
last_login="No data"
fi
[[ -z "$last_login" ]] && last_login="No data"
password_date=$(LC_ALL=C $SUDO chage -l "$name" 2>/dev/null | grep "Last password change" | cut -d: -f2-)
[[ -z "$password_date" ]] && password_date="No data"
echo "$name | $uid | $groups | $last_login | $password_date"
fi
done
} > "$TMP_DIR/users.tmp"
#SSH configuration
{
echo "Parameter | Effective value | Configuration"
if check_tool sshd openssh-server; then
$SUDO sshd -T > "$TMP_DIR/sshd_effective.tmp" 2>/dev/null
for param in permitrootlogin port logingracetime maxauthtries passwordauthentication pubkeyauthentication; do
value=$(awk -v p="$param" '$1 == p {print $2}' "$TMP_DIR/sshd_effective.tmp")
if grep -Eiq "^[[:space:]]*$param[[:space:]]+" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/*.conf 2>/dev/null; then
source="Explicitly configured"
else
source="Default"
fi
echo "$param | ${value:-No data} | $source"
done
else
echo "sshd utility is not available"
fi
} > "$TMP_DIR/sshd.tmp"
#Top 5 processes by memory
{
ps -eo pid,ppid,user,%mem,comm --sort=-%mem | head -n 6
} > "$TMP_DIR/top_processes.tmp"
#Suspicious accounts
{
echo "Users with empty passwords:"
$SUDO awk -F: '$2 == "" {print $1}' /etc/shadow
echo
echo "Users with UID 0 except root:"
getent passwd | awk -F: '$3 == 0 && $1 != "root" {print $1}'
} > "$TMP_DIR/suspicious.tmp"
#SUID files
{
echo "SUID files in /usr/bin and /usr/sbin:"
find /usr/bin /usr/sbin -type f -perm -4000 -print 2>/dev/null
} > "$TMP_DIR/suid.tmp"
{ 
echo "===== Host information =====" 
echo "Date:" 
cat PASSPORT/tmp/date.tmp 
echo "Hostname:" 
cat PASSPORT/tmp/hostname.tmp
echo "===== User information =====" 
echo "Current user:" 
cat PASSPORT/tmp/user.tmp 
echo "Users in system:" 
cat PASSPORT/tmp/who.tmp
echo "===== Current directory =====" 
cat PASSPORT/tmp/pwd.tmp 
echo "===== Files in current directory =====" 
cat PASSPORT/tmp/ls.tmp
echo "===== Kernel and operating system =====" 
echo "Kernel:" 
uname -s 
echo "Kernel version:" 
uname -r 
echo "OS:" 
grep '^PRETTY_NAME=' PASSPORT/tmp/os.tmp | cut -d '"' -f 2
echo "===== CPU model =====" 
grep -E '^(Model name|Model|Vendor ID):' PASSPORT/tmp/lscpu.tmp
echo "===== CPU frequency ====="
grep -iE 'mhz|ghz|frequency|cpu max|cpu min' PASSPORT/tmp/lscpu.tmp
echo "===== Logical CPU cores =====" 
grep -E '^(CPU\(s\)|Core\(s\) per socket|Socket\(s\):|Thread\(s\) per core):' PASSPORT/tmp/lscpu.tmp
echo "===== Cache sizes ====="
grep -E '^(L1d cache|L1i cache|L2 cache|L3 cache):' PASSPORT/tmp/lscpu.tmp
echo "===== Physical memory =====" 
cat PASSPORT/tmp/memory.tmp
echo "===== PCI devices =====" 
cat PASSPORT/tmp/lspci.tmp
} > "$REPORT_FILE"
write_section "Users" "$TMP_DIR/users.tmp"
write_section "SSH settings" "$TMP_DIR/sshd.tmp"
write_section "Top 5 processes" "$TMP_DIR/top_processes.tmp"
write_section "Suspicious accounts" "$TMP_DIR/suspicious.tmp"
write_section "SUID files" "$TMP_DIR/suid.tmp"
```
Результат:
```text
===== Host information =====
Date:
Чт 26 ноя 2026 14:02:04 MSK
Hostname:
debian-c
===== User information =====
Current user:
root
Users in system:
sde-1    seat0        2026-11-26 14:01
sde-1    tty1         2026-11-26 14:01
===== Current directory =====
/home/sde-1
===== Files in current directory =====
nohup.log
PASSPORT
REPORT_FILE
script31
script32
script32.log
script_passport_v3.bak
script_passport_v3.sh
test.txt
===== Kernel and operating system =====
Kernel:
Linux
Kernel version:
6.12.107+deb13-arm64
OS:
Debian GNU/Linux 13 (trixie)
===== CPU model =====
Vendor ID:                               Apple
Model name:                              -
Model:                                   0
===== CPU frequency =====
===== Logical CPU cores =====
CPU(s):                                  2
Thread(s) per core:                      1
===== Cache sizes =====
===== Physical memory =====
               total        used        free      shared  buff/cache   available
Mem:           961Mi       188Mi       752Mi       508Ki        87Mi       773Mi
Swap:          939Mi          0B       939Mi
===== PCI devices =====
00:00.0 VGA compatible controller: VMware SVGA II Adapter (Fusion)
00:01.0 System peripheral: InnoTek Systemberatung GmbH VirtualBox Guest Service
00:02.0 Audio device: Intel Corporation 82801FB/FBM/FR/FW/FRW (ICH6 Family) High Definition Audio Controller (rev 01)
00:03.0 SCSI storage controller: Red Hat, Inc. Virtio 1.0 SCSI (rev 01)
00:04.0 PCI bridge: Intel Corporation 82801 Mobile PCI Bridge (rev f2)
00:06.0 USB controller: Intel Corporation 7 Series/C210 Series Chipset Family USB xHCI Host Controller
00:08.0 Ethernet controller: Intel Corporation 82540EM Gigabit Ethernet Controller (rev 02)

===== Users =====
Name | UID | Groups | Last login | Last password change
root | 0 | root | ноя22/17:15 |  Sep 16, 2026
myuser | 1000 | myuser cdrom floppy audio dip video plugdev users netdev | окт27/00:34 |  Sep 16, 2026
sde-1 | 1001 | sde-1 sudo users | ноя23/20:23 |  Nov 22, 2026
u1 | 1002 | u1 project | No data |  Nov 22, 2026
u2 | 1003 | u2 project | No data |  Nov 23, 2026
u3 | 1004 | u3 project | No data |  Nov 22, 2026
u4 | 1005 | u4 sec2read | No data |  Nov 22, 2026
u5 | 1006 | u5 sec2read | No data |  Nov 22, 2026

===== SSH settings =====
Parameter | Effective value | Configuration
permitrootlogin | yes | Explicitly configured
port | 22 | Default
logingracetime | 30 | Explicitly configured
maxauthtries | 2 | Explicitly configured
passwordauthentication | no | Explicitly configured
pubkeyauthentication | yes | Explicitly configured

===== Top 5 processes =====
    PID    PPID USER     %MEM COMMAND
    256       1 root      1.5 systemd-journal
      1       0 root      1.4 systemd
    736       1 sde-1     1.2 systemd
    305       1 root      0.9 systemd-udevd
    584       1 root      0.8 systemd-logind

===== Suspicious accounts =====
Users with empty passwords:

Users with UID 0 except root:

===== SUID files =====
SUID files in /usr/bin and /usr/sbin:
/usr/bin/su
/usr/bin/umount
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/mount
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/sudo
/usr/bin/chfn
```
## Вопросы и задания

### 1. Зачем при создании SSH-ключа задаётся парольная фраза?

Парольная фраза защищает закрытый SSH-ключ. Если посторонний получит файл закрытого ключа, без парольной фразы он не сможет использовать его для подключения. Это дополнительная защита на случай кражи или копирования ключа.

### 2. Зачем SSH показывает отпечаток ключа сервера при первом подключении?

Отпечаток позволяет проверить подлинность SSH-сервера. Клиент просит подтвердить его, чтобы пользователь убедился, что подключается к нужной машине, а не к поддельному серверу.

При последующих подключениях клиент сравнивает сохранённый отпечаток с новым. Если он изменился, SSH предупреждает о возможной подмене сервера или изменении его ключей. До проверки причины продолжать подключение не следует.

### 3. Можно ли определить по `/etc/shadow`, что у двух пользователей одинаковые пароли?

Не всегда. В `/etc/shadow` хранятся не сами пароли, а их хеши. При использовании соли одинаковые пароли обычно дают разные хеши, поэтому простое сравнение строк не позволяет определить совпадение паролей.

### 4. Права доступа Linux

| Атрибут | Сокращённое название | Значение для файла | Значение для каталога |
|---|---|---|---|
| Чтение | `r` | Просмотр содержимого файла | Просмотр списка имён файлов |
| Запись | `w` | Изменение содержимого файла | Создание, удаление и переименование записей в каталоге |
| Выполнение | `x` | Запуск файла как программы | Переход в каталог и доступ к объектам по известному имени |
| SUID | `s` | Запуск исполняемого файла с эффективным UID владельца | Обычно не используется для каталогов в Linux |
| SGID | `s` | Запуск исполняемого файла с эффективным GID группы-владельца | Новые объекты наследуют группу каталога; подкаталоги также наследуют SGID |
| Sticky bit | `t` | Специального обычного значения для файлов не имеет | Удалять и переименовывать файлы могут их владельцы, владелец каталога и root |

### 5. Почему для удаления файла нужны права на каталог? Что меняет Sticky Bit?

Удаление файла означает удаление записи об этом файле из каталога. Поэтому для удаления обычно нужны права записи (`w`) и выполнения (`x`) на каталог, а не право записи на сам файл.

Sticky Bit ограничивает удаление и переименование файлов в общем каталоге: пользователь может удалять или переименовывать только принадлежащие ему файлы. Также это могут делать владелец каталога и root.

### 6. Как выдать пользователю права через ACL и решить задание 3 Части 3?

Чтобы выдать конкретному пользователю все права на файл, не меняя владельца и группы, используется команда:

```bash
setfacl -m u:имя_пользователя:rwx имя_файла
```

Для каталога `/DATA/sec2` из третьей части можно настроить ACL так:

```bash
setfacl -m u:u1:rwx,u:u4:rx,u:u5:rx,o::--- /DATA/sec2
```

Чтобы права распространялись и на создаваемые внутри каталога файлы и подкаталоги, можно задать ACL по умолчанию:

```bash
setfacl -d -m u::rwx,u:u1:rwx,u:u4:r-x,u:u5:r-x,g::---,m::rwx,o::--- /DATA/sec2
```

Для уже существующих файлов права на чтение и запись нужно настроить отдельно. ACL позволяет назначать права конкретным пользователям, не добавляя их в общую группу.

### 7. Чем различаются `su`, `su -` и `sudo`?

- `su` переключает пользователя, но обычно сохраняет часть текущего окружения и рабочий каталог.
- `su -` переключает пользователя с загрузкой его полноценного login-окружения, включая домашний каталог и переменные окружения.
- `sudo` запускает указанную команду с правами другого пользователя, обычно root, не переключая всю текущую оболочку.

В Debian для повседневной работы рекомендуют `sudo`, потому что можно выдавать необходимые административные права отдельным командам и контролировать их использование, не работая постоянно под root.

### 8. Удалось ли обойти ограничение sudo из Части 5? Почему опасны подстановочные знаки в sudoers?

Обойти ограничение в ходе проверки не удалось. Попытка передать команде `passwd` несколько аргументов (`u2 root`) не позволила изменить пароль root: команда не поддерживает одновременную обработку нескольких пользователей и вывела справку.

Подстановочные знаки в правилах `sudoers` опасны тем, что могут разрешить больше вариантов аргументов, чем предполагалось. Пользователь может подобрать аргументы или способ вызова команды, который позволит выполнить нежелательное действие с повышенными привилегиями.

### 9. Чем отличаются SIGTERM и SIGKILL?

`SIGTERM` просит процесс завершить работу. Процесс может перехватить этот сигнал, выполнить необходимые действия и корректно освободить ресурсы.

`SIGKILL` принудительно завершает процесс. Его нельзя перехватить, проигнорировать или обработать. Поэтому сначала обычно используют `SIGTERM`, а `SIGKILL` — если процесс не завершился.

### 10. Что происходит с процессом, запущенным через `&`, при закрытии SSH-сессии? Как помогает `nohup`?

Запуск через `&` переводит команду в фоновый режим, но сам по себе не защищает её от завершения при закрытии SSH-сессии. При закрытии соединения оболочка может отправить процессу сигнал `SIGHUP`, после чего процесс завершается.

`nohup` игнорирует `SIGHUP`, поэтому команда может продолжить работу после выхода из SSH-сессии. После завершения родительской оболочки процесс может быть переподчинён процессу с PID 1. Поэтому его PPID меняется на `1`.

В проведённом эксперименте обычный фоновый процесс завершился после закрытия сессии, а запущенный через `nohup` продолжил работу.

### 11. Что такое зомби-процесс? Можно ли его завершить командой `kill -9`?

Зомби-процесс — это уже завершившийся процесс, запись о котором остаётся в таблице процессов, пока родительский процесс не прочитает его код завершения с помощью `wait()`.

Зомби нельзя завершить командой `kill -9`: он уже не выполняется и не обрабатывает сигналы. Чтобы убрать запись, нужно дождаться, пока родитель вызовет `wait()`, либо разобраться с родительским процессом.

### 12. Чем отличается вывод `sshd -T` от строк в `sshd_config`, найденных через `grep`?

`sshd -T` показывает эффективные параметры конфигурации SSH-сервера после обработки настроек и значений по умолчанию.

`grep` показывает только строки, явно найденные в конфигурационных файлах. Он не отображает параметры, которые не прописаны и поэтому берутся по умолчанию, а также сам по себе не вычисляет итоговую конфигурацию.

Например, в полученном отчёте команда `sshd -T` показала:

```text
port 22
```

При этом строка `Port` отсутствовала среди явно прописанных параметров. Значит, использовалось значение порта по умолчанию — `22`.

### 13. Какой риск возникает, если владелец исполняемого файла — root и на файл установлен SUID?

При запуске такого файла программа получает эффективный UID владельца, то есть root. Если в программе есть уязвимость или она позволяет выполнять произвольные команды, обычный пользователь может получить возможность выполнять действия с привилегиями суперпользователя. Это создаёт риск повышения привилегий и компрометации системы.

