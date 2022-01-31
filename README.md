## Otus Homework 3

- [x] уменьшить том под / до 8G
- [x] выделить том под /home
- [ ] выделить том под /var (/var - сделать в mirror)
- [ ] для /home - сделать том для снэпшотов
- [ ] прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)
- [ ] * сгенерировать файлы в /home/
- [ ] * снять снэпшот
- [ ] * удалить часть файлов
- [ ] * восстановиться со снэпшота (залоггировать работу можно утилитой script, скриншотами и т.п.)
- [ ] * на нашей куче дисков попробовать поставить btrfs/zfs, c кешем и снэпшотами,разметить здесь каталог /opt


### Shrink
для начала скажу что первое задание было шуткой) ну да ладно заморочился и решил сделать такое дело (переносил oc в живую на сервере бывало много раз, 
не привык к переносу на виртуалках непонятных в которые параметры initrd взяты только для загрузки на одном единственном варианте железа)
```
sudo -i
```
установим lsof для удобства получения данных о занятых файлах процессами
```
yum install -y lsof
```
проверяем какие файлы находятся в процессе удаления и затем  перезапускаем зависимые службы
```
lsof / | grep ' DEL \|delete'
```
```
tuned      972    root  DEL    REG  253,0          100664481 /tmp/ffi1Q1dJD
tuned      972    root    7u   REG  253,0     4096 100664481 /tmp/ffi1Q1dJD (deleted)
```
```
systemctl stop tuned
```
проверяем файлы в сатусе запип нас интересуют любые процессы с буквами uUwW в  колонке fd
```
lsof / | grep -v ' \(mem\|txt\|rtd\|cwd\) '
```
```
systemd-u  532    root    6r   REG  253,0  7780559  67147303 /etc/udev/hwdb.bin
auditd     636    root    5w   REG  253,0   253714 100889178 /var/log/audit/audit.log
rsyslogd   973    root    4w   REG  253,0   104527 100834552 /var/log/messages
rsyslogd   973    root    6w   REG  253,0     6065 100834553 /var/log/secure
rsyslogd   973    root    7w   REG  253,0      186 100889183 /var/log/cron
rsyslogd   973    root    9w   REG  253,0      360 100834554 /var/log/maillog
dhclient  2662    root    4w   REG  253,0      842  67147311 /var/lib/NetworkManager/dhclient-5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03-eth0.leasesystemd-u  532    root    6r   REG  253,0  7780559  67147303 /etc/udev/hwdb.bin
auditd     636    root    5w   REG  253,0   253714 100889178 /var/log/audit/audit.log
rsyslogd   973    root    4w   REG  253,0   104527 100834552 /var/log/messages
rsyslogd   973    root    6w   REG  253,0     6065 100834553 /var/log/secure
rsyslogd   973    root    7w   REG  253,0      186 100889183 /var/log/cron
rsyslogd   973    root    9w   REG  253,0      360 100834554 /var/log/maillog
dhclient  2662    root    4w   REG  253,0      842  67147311 /var/lib/NetworkManager/dhclient-5fb06bd0-0bb0-7ffb-45f1-d6edd65f3e03-eth0.lease
```
останавливаем соответствующие службы (auditd не любит перезапуска через systemctl)
```
systemctl stop rsyslog
systemctl stop postfix
pkill -15 auditd
pkill -9 dhclient
```
теперь время черной магии
создаем каталог /tmproot
```
mkdir /tmproot
```
создадим lvm том sys-root размером в 8g
```
pvcreate /dev/sda3
vgcreate sys /dev/sdb
lvcreate -L 8g -n root sys
mkfs.ext4  /dev/sys/root
```
теперь как создадим каталог в который будет перемещен оригинальный root
```
mkdir /oldroot
```
монтируем /dev/sys/root в /tmproot переводим систему в режим unshared
```
mount /dev/sys/root /tmproot
mount --make-rprivate /
```
синхронизируем данные каталогов и переводим root на /dev/sys/root
```
rsync -auxHAXSv --exclude=/dev/* --exclude=/boot/ --exclude=/proc/* --exclude=/sys/* --exclude=/tmp/* --exclude=/tmproot/* /* /tmproot
pivot_root /tmproot /tmproot/oldroot
```
переключаем виртуальные фс и boot на новый mountpoint
```
for i in dev proc sys run boot; do mount --move /oldroot/$i /$i; done
```
!!!!ОБЯЗАТЕЛЬНО после этих команд проверяем можем ли мы переподключится к ssh если нет то все с начала
```
systemctl restart sshd
systemctl restart NetworkManager
```
!!!!!!!! ПРОВЕРЬ vagrant ssh
Проверяем что еще использует oldroot
```
lsof /oldroot/
```
```
long output
```
перезапускаем требуемые службы
```
systemctl restart crond
pkill -15 agetty
systemctl restart gssproxy
systemctl restart chronyd
systemctl restart polkit
systemctl restart rpcbind
systemctl restart messagebus
systemctl restart lvm2-lvmetad.service
systemctl daemon-reexec
systemctl restart systemd*
```
после перезапуска никакие файлы не должны обращаться к oldroot
```
lsof /oldroot/
```
```
null output
```

umount -l /oldroot

и наконец  генерируем initramfs (взял из предыдущей статьи) с поддержкой ext3/4 и xfs(для boot)
```
dracut --mdadmconf --fstab --add="mdraid" --filesystems "xfs ext4 ext3" --add-drivers="raid1" --force /boot/initramfs-$(uname -r).img $(uname -r) -M
```
подменим параметры груба
```
vi /etc/default/grub
```
```
GRUB_TIMEOUT=1
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.auto rd.auto=1 rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
```
сгенерируем конфиг груба
```
grub2-mkconfig -o /boot/grub2/grub.cfg
```
перезагружаемся
вывод df -h по окончанию
```
/dev/mapper/sys-root  7.8G  831M  6.6G  12% /
devtmpfs              1.9G     0  1.9G   0% /dev
tmpfs                 1.9G     0  1.9G   0% /dev/shm
tmpfs                 1.9G  8.6M  1.9G   1% /run
tmpfs                 1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/sda2            1014M   62M  953M   7% /boot
tmpfs                 380M     0  380M   0% /run/user/1000
```
удаляем старый lv и создаем новый размером 8g на wipe соглашаемся и форматируем в xfs
``` 
lvremove /dev/VolGroup00/LogVol00
lvcreate -L 8g -n LogVol00 VolGroup00
mkfs.xfs /dev/VolGroup00/LogVol00
```
теперь повторяем операции с самого начала)) и будет вам счастье
### /home
Создадим на диске sdc vg home и lv user примонтируем в /home
```
pvcreate /dev/sdc
vgcreate home /dev/sdc
lvcreate -L 1.8g -n user home
mkfs.ext4  /dev/home/user
mount /dev/home/user /home
```
поправим fstab и добавим точку монтирования в конец файла
```
vi /etc/fstab
```
```
/dev/mapper/home-user /home                  ext4     defaults        0 0
```
### /var
создадим 
