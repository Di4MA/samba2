# Установка компонентов
```bash
sudo dnf upgrade --refresh
sudo dnf search samba* # Просто, чтобы судорожно не вспоминать, что конкретно ставить. Выводим список всех пакетов с названием, начинающимся на samaba и вспоминаем, что мы там хотели поставить                       

sudo dnf install samba
sudo dnf install samba-devel
sudo dnf install samba-tools
# Если мы ну очень тупы или сильно волнуемся, то просто ставим вообще всё, что начинается на samba вот этой командой:
sudo dnf install samba*
```
# Работа с пользователями и группами
## Создание групп
```bash
sudo groupadd other
sudo groupadd department
# Вспомогательная группа, которая нужна для общего файла
# В неё заносим всех пользователей, которых создаём далее
sudo groupadd share 
```
## Создание пользователей в группе
```bash
sudo useradd other-user1 -G other,share
sudo useradd other-user2 -G other,share
sudo useradd department-user1 -G department,share
sudo useradd department-user2 -G department,share

# Проверка, что пользователи находятся в группе
groups  other-user1
groups  other-user2
groups department-user1
groups department-user2

# Этой командой добавляем пользователя, с которого работаем в системе во все указанные группы
sudo usermod -aG department,share,other user
```
## Редактирование пароля пользователей в Linux
Вообще, желательно сразу как создали пользователя редактировать ему пароль, но лучше сделать это отдельным блоком команд, чтобы, если комиссия спросит, то удобно отобразить это в `history`
```bash
sudo passwd other-user1
sudo passwd other-user2
sudo passwd department-user1
sudo passwd department-user2
```
## Редактирование пароля для пользователей в БД SMB
В samba нужно отдельно задавать пароли для доступа пользователям. Можно сделать пароли такими же как при создании, можно отдельно: как хотите
```bash
sudo smbpasswd -a other-user1
sudo smbpasswd -a other-user2
sudo smbpasswd -a department-user1
sudo smbpasswd -a department-user2
```
# Работа с каталогами
## Создание каталогов
Просто создаём каталоги, к которым должны иметь доступ наши пользователи
```bash
sudo mkdir -p /srv/share/{dir1,dir2,dir3}
```
## Редактирование атрибутов доступа у созданных каталогов
```bash
# Что означают циферки
# владелец - группа_владельца - все_остальные_пользователи
sudo chmod 775 /srv/share/dir1
sudo chmod 775 /srv/share/dir2
sudo chmod 775 /srv/share/dir3
```
Сделали каталоги доступными для записи всем пользователям, которые входят в группу создателя. Так как создали каталоги под учёткой пользователя `user`, который сам во все каталоги группы добавлен, то получается дали доступ на уровне системы всем пользователям
## Смена владельцев каталогов
```bash
# Явно указали, что владельцем каталогов является пользователь user
sudo chown -R user: /srv/share/{dir1,dir2,dir3}

# По-умолчанию Linux считает, что все группы, в которых состоит владелец, используются для определения доступа. Явно указали группу владельца каталога, которая используются для определения права доступа
sudo chown -R :share /srv/share/dir1
sudo chown -R :other /srv/share/dir2
sudo chown -R :department /srv/share/dir3
```
## Маркирование каталогов через менеджер безопасности
### Сложный вариант

Хрень, которую нужно писать из-за ИБ. При желании можно просто вырубить SELinux:
В файле `/etc/selinux/config` замените текст `SELINUX=enforcing` на `SELINUX=permissive`и сделать `reboot`. Если хочется без ребута: `sudo setenforce 0`
```bash
# добавляем контексты безопасности для определённых директорий
sudo semanage fcontext --add --type "samba_share_t" /srv/share/
sudo semanage fcontext --add --type "samba_share_t" /srv/share/dir1
sudo semanage fcontext --add --type "samba_share_t" /srv/share/dir2
sudo semanage fcontext --add --type "samba_share_t" /srv/share/dir3

# Перезагружаем контекст безопасности
sudo restorecon -R /srv/share/

# контекст безопасности типа samba_share_t для всех директорий внутри /srv/share/
sudo semanage fcontext -a -t samba_share_t "/srv/share/(/.*)?"

# Просмотр всех логических переключателей
getsebool -a
# Включение логических переключателей (вообще не знаю что это но пусть будет)
sudo setsebool -P samba_share_fusefs on
```
### Простой вариант
```bash
sudo setenforce 0
```
Если перезапустили сервак, то командочку нужно повторить
# Редактирование конфига samba
Открываем файл `/etc/samba/smb.conf` комментируем секцию \[homes] и добавляем секцию \[dostup]. 
Итого в файле должно быть вот так:
```bash
#[homes]
#       comment = Home Directories
#       valid users = %S, %D%w%S
#       browseable = No
#       read only = No
#       inherit acls = Yes

[dostup]
        comment =  "Папка общая" # Комментарий чисто для нас
        path = /srv/share # Путь до каталога с папками
        valid users = @share # Указываем, что доступ для папок есть у всех пользователей в группе Share
        #force group = share-user
        create mask = 0775 # Маска доступа, которая задаётся для создаваемых файлов
        directory mask = 0775 # Маска доступа, которая задаётся для создавамемых директорий в этой папке
        guest ok = no # Запрет доступа  пользователям вне группы share 
        writable = yes # Разрешение на запись в директорию
        browseable = yes # Если установить `no`, пользователь не увидит папку, но сможет получить к ней доступ, зная имя. Usless для нас так как нужно увидеть папку в проводнике
        write list = user # Пользователь, который точно получает доступ на запись (ХЗ зачем, пусть будет, что системный пользователь точно сможет записывать сюда что-нибудь)
```