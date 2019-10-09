# OTUS ДЗ 14 Резервное копирование  (Centos 7)
-----------------------------------------------------------------------
### Домашнее задание

    Настроить стенд Vagrant с двумя виртуальными машинами server и client.

    Настроить политику бэкапа директории /etc с клиента:
    1) Полный бэкап - раз в день
    2) Инкрементальный - каждые 10 минут
    3) Дифференциальный - каждые 30 минут

    Запустить систему на два часа. Для сдачи ДЗ приложить list jobs, list files jobid=<id>
    и сами конфиги bacula-*

    * Настроить доп. Опции - сжатие, шифрование, дедупликация


В домашнем задании будет рассмотрены две системы резервного копирования для Linux - Borg Backup и Bacula.
Для этого домашнего задания будет поднято две виртуальные машины - сервер бэкапов и клиент. В приложенном вагрант файле поднимаются две преднастроенных ВМ. 
Для поднятия ВМ надо скачать вагрант файл и выполнить в директории со скачанным файлом команду ```vagrant up```

1. Borg Backup
- Пишем vagrantfile 
- Первое что мы делаем это устанавливаем Borg Backup на обе ВМ, но сначала устанавливаем epel репозиторий ```yum install epel-release -y; yum install borgbackup -y```
- Далее создаем пользователя borg на обоих машинах ```useradd -m borg```
- Так как borg будет делать резервные копии на сервер бэкапов, то нам необходимо включить авторизацию по ssh ключам, для этого необходимо сформировать ssh ключ на клиенте командой ```ssh-keygen``` под пользователем borg (на все вопросы нажимаем enter)
- Далее создаем на клиенте в ```/home/borg/.ssh/``` папку keys, куда перемещаем из ```/home/borg/.ssh/``` наши созданные ключи
- Далее на клиенте в ```/home/borg/.ssh/``` создаем файл с названием ```config``` и следующим содержимым:
```
Host borg-server
IdentityFile ~/.ssh/keys/id_rsa
```
Здесь указывается сервер к которому будем подключаться и путь до закрытого ключа на клиенте
Т.е. на сервере у нас будет открытый ключ, а на клиенте закрытый ключ
- Далее необходимо скопировать открытый ключ из файла (с клиента) ```id_rsa.pub``` в файл ```/home/borg/.ssh/authorized_keys``` на сервер бэкапов и привести файл к такому виду (так как ВМ у меня тестовые то привожу вывод файла, если у вас продакшен, о ваших ключах никто не должен знать):
```
'command="/usr/local/bin/borg serve" ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCkh81l3BvHNITaN/EbwaCtMjBqxP+e7qVey6kAeTR+SgZaxIwuVdSKl/LbEBM2PRIEk4swuo4WtRNTPGYjsBjtAJV6Njodb8qs+G0YNVTFbBSzQ0UUhU30jLCANsR+fpm14Bvg1FmI6swyhtpSwCJdSX1//9gfvm8LC0F0AU4u2JWvO7iggAdrPLOc8LThZcADJc7+yERfTwbFoHY6jVahDFtLIClIcCIrA8P66/WGCjviMTdTpz3A2FdEMcIwNsBFkTjwQjOXjllKNPIguNR0ejbnzAeUEIMUAB4ptSiVayYPPm8Py2rzj6gb08I8tfuMXBY/8M2vTFo4af4H29Cx borg@borg-client'

ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCkh81l3BvHNITaN/EbwaCtMjBqxP+e7qVey6kAeTR+SgZaxIwuVdSKl/LbEBM2PRIEk4swuo4WtRNTPGYjsBjtAJV6Njodb8qs+G0YNVTFbBSzQ0UUhU30jLCANsR+fpm14Bvg1FmI6swyhtpSwCJdSX1//9gfvm8LC0F0AU4u2JWvO7iggAdrPLOc8LThZcADJc7+yERfTwbFoHY6jVahDFtLIClIcCIrA8P66/WGCjviMTdTpz3A2FdEMcIwNsBFkTjwQjOXjllKNPIguNR0ejbnzAeUEIMUAB4ptSiVayYPPm8Py2rzj6gb08I8tfuMXBY/8M2vTFo4af4H29Cx borg@borg-client
```
Это два одинаковых публичных ключа, первый дает возможность программе Borg Backup подключаться по ssh по ключу, второй дает возможность подключиться администратору через обычный ssh сеанс

ВАЖНО!
Необходимо чтобы и на сервере и на клиенте в директориях ```/home/borg/.ssh``` для папок были права 700, а для файлов 600. Задается утилитой ```chmod```. Если не правильные права то работать не будет.

Далее необходимо в ```/etc/hosts``` на клиенте ввести сопоставление адреса и имени сервера, для клиента:
```
192.168.10.22 borg-server
```
На сервере:
```
192.168.10.23 borg-client
```
Протестировать работоспособность можно введя под пользователем borg (на клиенте), команду ```ssh borg@borg-server```, все должно работать.

- Далее нам надо инициализировать с клиента репозитории Borg Backup командой ```borg init -e none borg@borg-server:MyBorgRepo```, там будут храниться файлы резервных копий
- Далее мы можем командой ```borg create --stats --list borg@borg-server:MyBorgRepo::"MyFirstBackup-{now:%Y-%m-%d_%H:%M:%S}" /etc``` создать свою первую резервную копию
Здесь указаны такие ключи как ```--stats``` это означает выводить статистику, ```--list``` выводит в процессе листинг файлов которые резервируются, в самом конце указываем директорию, которую бэкапим
- Можем с клиента посмотреть наши бэкапы командой ```borg list borg@borg-server:MyBorgRepo```:
```
MyFirstBackup-2019-10-01_23:03:32    Tue, 2019-10-01 23:03:33 [0f0bf875a2eec9674d7dcb1d5303e69bc6a863edc99cebda3479349d0ea45e26]
MyFirstBackup-2019-10-02_00:25:28    Wed, 2019-10-02 00:25:28 [2d5feb4f1eb414c282191ebc0b7cc5e1345456fef508a6506599ed9c750ade62]
```
- Можем посмотреть расширенную информацию командой ```borg info borg@borg-server:MyBorgRepo```:
```
Repository ID: efa0dc2b20e0a79f47158329ad58c06c99532209977ff251b9b9a04c3c4d15bf
Location: ssh://borg@borg-server/./MyBorgRepo
Encrypted: No
Cache: /home/borg/.cache/borg/efa0dc2b20e0a79f47158329ad58c06c99532209977ff251b9b9a04c3c4d15bf
Security dir: /home/borg/.config/borg/security/efa0dc2b20e0a79f47158329ad58c06c99532209977ff251b9b9a04c3c4d15bf
------------------------------------------------------------------------------
                       Original size      Compressed size    Deduplicated size
All archives:               20.99 GB             87.97 MB              5.72 MB

                       Unique chunks         Total chunks
Chunk index:                     412                 2916
```
- Можем посмотреть какие файлы есть в конкретной резервной копии командой ```borg list borg@borg-server:MyBorgRepo::MyFirstBackup-2019-10-01_23:03:32```

Отличительной особенностью Borg Backup является то что:
а) Здесь нет понятия дифференциальная и инкрементальная резервная копия, резервная копия всегда Full
б) Кроме этого, особенностью этого решения является, то что Borg Backup очень хорошо сжимает и дедублицирует данные. В листинге выше мы можем видеть, данные по сжатию и дедубликации.
в) Borg Backup поддерживает шифрование резервных копий с помощью ключа ```-e```, однако в этом примере мы не шифруем резервные копии.
г) При восстановлении резервной копии необходимо учитывать под каким пользователем восстаналиваются файлы, так как владелец и группа назначаются того пользователя, под каким происходит процесс восстановления

- Достать файл из резервной копии можно слудующей командой ```borg extract borg@borg-server:MyBorgRepo::MyFirstBackup-2019-10-01_23:03:32 etc/hostname```
- Извлечь весь архив ```borg extract borg@borg-server:MyBorgRepo::MyFirstBackup-2019-10-01_23:03:32```

- Для того чтобы очищать старые бэкапы можно воспользоваться командой ```borg prune --verbose --list -d 30  borg@borg-server:MyBorgRepo``` или указать через префикс, если у вас в одном репозитории несколько разных наименований резервных копий ```borg prune --verbose --list -d 30  borg@borg-server:MyBorgRepo --prefix MyFirstBackup```, т.е. для разных можно задать разный период очистки:
```
[borg@borg-client ~]$ borg prune --verbose --list -d 30  borg@borg-server:MyBorgRepo --prefix MyFirstBackup
Keeping archive: MyFirstBackup-2019-10-02_00:25:28    Wed, 2019-10-02 00:25:28 [2d5feb4f1eb414c282191ebc0b7cc5e1345456fef508a6506599ed9c750ade62]
Keeping archive: MyFirstBackup-2019-10-01_23:03:32    Tue, 2019-10-01 23:03:33 [0f0bf875a2eec9674d7dcb1d5303e69bc6a863edc99cebda3479349d0ea45e26]
```
Также можно воспользоваться командой для указания сколько времени хранить архивы ```borg prune --verbose --keep-within 2d  borg@borg-server:MyBorgRepo```

Еще можно например сохранить ежеминутные бэкапы в течении 100 минут командой ```borg prune -v --list --dry-run --keep-minutely=100  borg@borg-server:MyBorgRepo```:
```
Keeping archive: MyFirstBackup3-2019-10-10_02:28:48   Thu, 2019-10-10 02:28:49 [7b64bf7a8680ce478cc99428260b3cc4c36de10fccbd5b9fad7a0cb44d14f416]
Keeping archive: MyFirstBackup4-2019-10-10_02:24:58   Thu, 2019-10-10 02:24:59 [e5d54912ded6f0718d66a5da5c56edfdde3ea2eea22a55c24201880b7c348fb7]
Would prune:     MyFirstBackup4-2019-10-10_02:24:56   Thu, 2019-10-10 02:24:57 [2a09b5275e1a958fa660b0d86efa51919a10ef87948ed6f3da3e777f75cc669b]
Would prune:     MyFirstBackup4-2019-10-10_02:24:54   Thu, 2019-10-10 02:24:54 [17fd0c947b12f556647103f5419ddb5e7f994341ce2ee703917f24ae711af2c5]
Keeping archive: MyFirstBackup4-2019-10-10_02:18:25   Thu, 2019-10-10 02:18:26 [8262cda01b6d9ce979922bdc4f6581511da2430bb5f24beda2a20d0191c29372]
```
Эта команда с ключом ```--dry-run``` не удалит архивы, а покажет что удалит команда без этого ключа, т.е. это как бы эмуляция удаления.

Далее мы можем автоматизировать выполнение резервных копий, написав обычный bash скрипт, куда добавляем наши задания и политики удаления и можно добавить bash скрипт в cron. Также можно воспользоваться сервисами systemd, т.е. написать службу, которая также с определенной периодичностью будет запускать задания. 

[Пример настройки через systemd]

[Пример настройки через systemd]:https://blog.andrewkeech.com/posts/170719_borg.html


