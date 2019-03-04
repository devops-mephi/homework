# Docker

## Установка docker

в виртуалке с centos7

1. установка зависимостей
```
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

2. добавление репозитория
```
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

3. установка
```
sudo yum install -y docker-ce
```

4. включение докера в автозагрузку
```
sudo systemctl enable docker.service
```

5. старт сервиса докера
```
sudo systemctl start docker.service
```

6. проверка, что всё прошло хорошо
```
sudo docker run hello-world
```

должно вывести что-то вроде
```
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
1b930d010525: Pull complete
Digest: sha256:2557e3c07ed1e38f26e389462d03ed943586f744621577a99efb77324b0fe535
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

## Работа с docker

Докер — клиент-серверное приложение. Всю работу выполняет демон dockerd, а управлять им можно с помощью консольной утилиты docker, которая посылает ему команды. Также у Docker-демона есть HTTP REST API, но в рамках данного курса мы не будем его рассматривать.

При работе с докером нужно понимать сущности, которыми он оперирует.
Первая сущность это images (образы).
Можно сказать, что это слепок файловой системы. Причем слепок многослойный, каждый следующий слой хранит только изменения относительно предыдущего слоя.
Существуют базовые образы, например: debian, ubuntu, centos и прочие. На их базе строят образы для запуска других процессов. Многослойная архитектура позволяет в данном случае легко наследоваться от других образов и избежать дублирования.

Вторая сущность это registry.
Это репозиторий для хранения образов (images). По умолчанию при установке Docker'а в него прописан registry Docker Hub. Можно поднимать свои registry и хранить образы там.

Третья сущность — контейнер.
Контейнер запускается на базе образа (image), поверх существующих в образе слоев файловой системы (которые read-only) добавляется слой read-write, который и будет изменяться при работе контейнера. Контейнер живет вокруг одного процесса и завершается вместе с этим процессом (у этого процесса pid=1). Помимо этого процесса в контейнере могут запускаться и другие процессы.

1. Давайте скачаем базовый образ для centos 7.

```
[vagrant@localhost ~]$ sudo docker pull centos:7
7: Pulling from library/centos
a02a4930cb5d: Pull complete 
Digest: sha256:184e5f35598e333bfa7de10d8fb1cebb5ee4df5bc0f970bf2b1e7c7345136426
Status: Downloaded newer image for centos:7
```

2. Посмотреть список имеющихся на машине образов можно так
```
[vagrant@localhost ~]$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello-world         latest              fce289e99eb9        2 months ago        1.84kB
centos              7                   1e1148e4cc2c        2 months ago        202MB
```

Каждый образ имеет три составляющих в имени.
REPOSITORY - имя образа в репозитории. Если вы хотите загрузить свой image в реджистри, то скорее всего его имя будет примерно следующим логин/имя_образа. Образы без префикса с логином, лежащие на Docker Hub являются официально одобренными и проверенными командой Docker. Если вы не планируете загружать свой image никуда, то имя может быть любым.
TAG — тег образа. Обычно в качестве тега указывают версию. Если не указывать тег, будет использоваться тег latest.
IMAGE ID — уникальный локально генерируемый ID образа, по которому к нему можно обращаться.

Если опустить тег при выполнении pull, то будет скачан образ с тегом latest. На данный момент последней версией centos является версия 7, логично предположить, что если опустить тег, будет скачан тот же самый образ. Давайте проверим:
```
[vagrant@localhost ~]$ sudo docker pull centos       
Using default tag: latest
latest: Pulling from library/centos
Digest: sha256:184e5f35598e333bfa7de10d8fb1cebb5ee4df5bc0f970bf2b1e7c7345136426
Status: Downloaded newer image for centos:latest
[vagrant@localhost ~]$ sudo docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello-world         latest              fce289e99eb9        2 months ago        1.84kB
centos              7                   1e1148e4cc2c        2 months ago        202MB
centos              latest              1e1148e4cc2c        2 months ago        202MB
```
Как мы видим, образы с тегом 7 и latest имеют одинаковые IMAGE ID — это один и тот же образ.
При построении сред из докер-контейнеров всегда указывайте версии используемых вами образов (да и вообще, всегда указывайте четкие версии всего, не только образов докера). Потому что в один момент может выйти новая, и что-то будет с ней несовместимо и всё сломается.

3. Теперь запустим контейнер с процессом bash
```
[vagrant@localhost ~]$ sudo docker run -ti centos bash
[root@56153529900b /]# 
```
Ух! Оцените скорость, с которой это произошло. Сейчас мы находимся в bash'е внутри centos 7. Вы можете работать как с полноценной "машиной".

Имя машины является именем контейнера:
```
[root@56153529900b /]# hostname
56153529900b
```

Процесс bash действительно имеет pid=1
```
[root@56153529900b /]# ps -aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.1  11820  1892 pts/0    Ss   13:40   0:00 bash
root        16  0.0  0.0  51740  1724 pts/0    R+   13:44   0:00 ps -aux
```

Позапускаем какие-то команды
```
[root@56153529900b /]# ls /
anaconda-post.log  bin  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
[root@56153529900b /]# date
Sun Mar  3 13:21:54 UTC 2019
[root@56153529900b /]# nano
bash: nano: command not found
```

Ой, редактора nano тут нет, давайте установим
```
[root@56153529900b /]# yum install -y nano
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors

.....


Installed:
  nano.x86_64 0:2.3.1-10.el7                                                                                                            

Complete!
```

Теперь есть, можно запустить
```
[root@56153529900b /]# nano
```

Теперь выйдем из bash
```
[root@56153529900b /]# exit
exit
[vagrant@localhost ~]$ 
```

Посмотрим на список запущенных контейнеров
```
[vagrant@localhost ~]$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```
Никаких контейнеров не запущено, тот что мы запускали, завершился вместе с bash.

4. Запустим еще один контейнер, такой же
```
[vagrant@localhost ~]$ sudo docker run -ti centos bash
[root@41f06a2d0dc2 /]# nano
bash: nano: command not found
```
Мы видим, что новый контейнер получил новое имя (41f06a2d0dc2) и что изменения, сделанные нами в пункте 3 (установка nano) никак не отразились на новом контейнере.
Итак, команда docker run создает новый контейнер и запускает его. Если указан образ, которого нет локально, делается попытка добыть его из реджистри с помощью docker pull.

Выйдем из него
```
[root@41f06a2d0dc2 /]# exit
exit
[vagrant@localhost ~]$
```

5. Увидеть все контейнеры, включая остановленные, можно так
```
[vagrant@localhost ~]$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS                        PORTS               NAMES
41f06a2d0dc2        centos              "bash"              About a minute ago   Exited (127) 14 seconds ago                       angry_blackburn
56153529900b        centos              "bash"              16 minutes ago       Exited (0) 8 minutes ago                          gifted_booth
f312847f158e        hello-world         "/hello"            39 minutes ago       Exited (0) 39 minutes ago                         inspiring_tu
```

Мы видим три остановленных контейнера (Exited). Первый мы запустили при установке, второй в пунте 3, третий в пунте 4.
Кроме CONTAINER ID докер каждому контейнеру присвоил человекочитаемые имена: inspiring_tu, gifted_booth, angry_blackburn. Это имя автогенерируемое и его можно использовать также, как и CONTAINER ID. 
Можно задавать свое имя при старте контейнера опцией --name

Попробуем запустить контейнер, в который мы устанавливали nano снова.
```
[vagrant@localhost ~]$ sudo docker start gifted_booth
gifted_booth
[vagrant@localhost ~]$ 
```
На этот раз мы не попали в bash в терминале. В пунктах 3 и 4 мы использовали опции -ti, которые создают в контейнере TTY-интерфейс и подключают текущую консоль к нему.
Но мы видим, что контейнер живет и работает
```
[vagrant@localhost ~]$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
56153529900b        centos              "bash"              23 minutes ago      Up About a minute                       gifted_booth
```

"Подсоединиться" к нему можно так:
```
[vagrant@localhost ~]$ sudo docker attach 56153529900b
[root@56153529900b /]#
```

Попробуем запустить nano
```
[root@56153529900b /]# nano
[root@56153529900b /]# 
```
Успех! Это тот самый контейнер.

Снова выйдем и завершим его
```
[root@56153529900b /]# exit
exit
[vagrant@localhost ~]$ 
```

Совсем удалить контейнер можно так:
```
[vagrant@localhost ~]$ sudo docker rm gifted_booth
gifted_booth
```

6. Теперь запустим контейнер, который будет работать как демон. Нам поможет опция -d.
```
[vagrant@localhost ~]$ sudo docker run -d --name deamon centos bash -c "while true; do echo 'I am deamon'; sleep 1; done"
f1609b2a7528a9af4ebe0bc8f5e21428a91b82d3fcdf635a79386b9690070d4f
```

Мы запустили команду, которая каждую секунду будет выводить "I am deamon". Но куда она это выводит?

Найдем наш контейнер:
```
[vagrant@localhost ~]$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
f1609b2a7528        centos              "bash -c 'while true…"   17 seconds ago      Up 15 seconds                           deamon
```
На этот раз докер использовал имя, которые мы задали при запуске (docker).

Теперь сделаем так:
```
[vagrant@localhost ~]$ sudo docker logs deamon
I am deamon
I am deamon
I am deamon
I am deamon
I am deamon
I am deamon
I am deamon
I am deamon
I am deamon
I am deamon
I am deamon
I am deamon
I am deamon
I am deamon
I am deamon
I am deamon
```

О, мы выяснили следующее: каждый раз когда процесс выводит что-то в STDOUT или STDERR, докер считает написанное логом и успешно сохраняет вне контейнера.

Можно смотреть лог в реальном времени, как "tail -f" вот так
```
[vagrant@localhost ~]$ sudo docker logs deamon -f
I am deamon
I am deamon
I am deamon
^C
```

А если добавить еще опцию -t, то к каждому сообщению будет подписан timestamp
```
[vagrant@localhost ~]$ sudo docker logs deamon -ft
2019-03-03T13:56:29.117258362Z I am deamon
2019-03-03T13:56:30.124673915Z I am deamon
2019-03-03T13:56:31.127844702Z I am deamon
2019-03-03T13:56:32.129189340Z I am deamon
2019-03-03T13:56:33.135152267Z I am deamon
^C
```

7. Еще можно сделать следующее:

```
[vagrant@localhost ~]$ sudo docker exec -ti deamon bash
```
Что мы сделали: мы запустили еще один процесс внутри контейнера и подсоединились к нему.

Посмотрим список процессов
```
[root@f1609b2a7528 /]# ps -ax
  PID TTY      STAT   TIME COMMAND
    1 ?        Ss     0:00 bash -c while true; do echo 'I am deamon'; sleep 1; done
  506 pts/0    Ss     0:00 bash
  525 ?        S      0:00 sleep 1
  526 pts/0    R+     0:00 ps -ax
```

Видим запущенный "демон"-процесс с pid=1 и наш bash, в котором мы находимся прямо сейчас.

С помощью этой команды (docker exec) можно производить какие-то обслуживающие действия, запускать скрипты внутри работающих контейнеров.

Заметьте, что завершение этого bash не приведет к остановке контейнера
```
[root@f1609b2a7528 /]# exit
exit
[vagrant@localhost ~]$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
f1609b2a7528        centos              "bash -c 'while true…"   14 minutes ago      Up 14 minutes                           deamon
```

Остановить работающий контейнер можно так
```
[vagrant@localhost ~]$ sudo docker stop deamon
deamon
```
Проверим, что он остановился
```
[vagrant@localhost ~]$ sudo docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[vagrant@localhost ~]$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                         PORTS               NAMES
f1609b2a7528        centos              "bash -c 'while true…"   15 minutes ago      Exited (137) 13 seconds ago                        deamon
```

Снова запустить вот так:
```
[vagrant@localhost ~]$ sudo docker start deamon
deamon
```

Убедимся, что он работает и исправно пишет логи
```
[vagrant@localhost ~]$ sudo docker logs deamon -tf
...
2019-03-03T14:10:31.390380022Z I am deamon
2019-03-03T14:10:32.392474637Z I am deamon
2019-03-03T14:10:33.395287694Z I am deamon
2019-03-03T14:10:34.442961371Z I am deamon
2019-03-03T14:10:35.657940767Z I am deamon
2019-03-03T14:10:36.667822682Z I am deamon
```

8. Команда docker stop посылает процессу в контейнере с pid=1 сигнал SIGTERM, ждем 10 секунд (это можно переопределить опцией -t) и затем, если процесс не завершился, убивает его уже сигналом SIGKILL. 

Сразу послать SIGKILL можно так
```
[vagrant@localhost ~]$ sudo docker kill deamon
deamon
```

9. Можно пробросить файл(ы) с хостовой машины внутрь контейнера. Делается это с помощью опции -v (volumes).

Создадим файл 
```
[vagrant@localhost ~]$ echo 'i will be inside container' > fl.txt
[vagrant@localhost ~]$
```

Запустим контейнер, пробросив этот файл
```
[vagrant@localhost ~]$ sudo docker run -ti -v /home/vagrant/fl.txt:/home/root/fl.txt centos bash
[root@f268c19a893e /]#
```

Файл действительно внутри
```
[root@f268c19a893e root]# cat fl.txt 
i will be inside container
[root@f268c19a893e root]#
```

При изменении файла на хостовой машине, изменения будут видны внутри контейнера.
Откроем еще одну консоль на хосте и изменим файл
```
[vagrant@localhost ~]$ echo 'it changed!' > fl.txt 
[vagrant@localhost ~]$
```

Изменения действительно видны:
```
[root@f268c19a893e root]# cat fl.txt 
it changed!
```

Выйдем из контейнера
```
[root@f268c19a893e root]# exit
exit
[vagrant@localhost ~]$
```

10. Специальной командой docker для удаления всех контейнеров не существует, но можно использовать вот такой "хак":
```
[vagrant@localhost ~]$ sudo docker rm $(sudo docker ps -a -q)
f268c19a893e
ce596ea9ba06
f1609b2a7528
41f06a2d0dc2
f312847f158e

[vagrant@localhost ~]$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
[vagrant@localhost ~]$ 
```

## Сборка собственных image (docker commit)

Существует два способа создать собственные docker image: docker commit и Dockerfile.
Способ через docker commit не является рекомендованным, однако мы его рассмотрим для улучшения понимания того, что происходит.

Суть способа довольно проста: нужно поднять контейнер, сделать в нем что-то и потом сохранить получившийся контейнер в виде образа.

Будем решать следующую задачу: допустим, мы не любим текстовый редактор vi и хотим, чтобы во всех контейнерах был редактор nano (он не входит в базовый образ centos).

1. Поднимаем контейнер с centos:7
```
[vagrant@localhost ~]$ sudo docker run -ti centos bash
[root@e0862eed7c42 /]#
```

2. Устанавливаем nano
```
[root@e0862eed7c42 /]# yum install -y nano
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
 * base: mirror.sale-dedic.com
 * extras: mirror.logol.ru
 * updates: dedic.sh
....

Installed:
  nano.x86_64 0:2.3.1-10.el7                                                      

Complete!
```

3. Выходим из bash, останавливая контейнер

```
[root@e0862eed7c42 /]# exit
exit
[vagrant@localhost ~]$
```

4. Находим ID контейнера

```
[vagrant@localhost ~]$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS                      PORTS               NAMES
e0862eed7c42        centos              "bash"              About a minute ago   Exited (0) 29 seconds ago                       romantic_noether
```

5. Делаем docker commit

```
[vagrant@localhost ~]$ sudo docker commit e0862eed7c42 centos7_with_nano:ver1    
sha256:8fb998ac2c98c216cdd8481f14c8e03efd8c64c9e76360155721f22528d3d3da
```

6. Видим образ в списке существующих:

```
[vagrant@localhost ~]$ sudo docker images   
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
centos7_with_nano   ver1                8fb998ac2c98        49 seconds ago      280MB
hello-world         latest              fce289e99eb9        2 months ago        1.84kB
centos              7                   1e1148e4cc2c        2 months ago        202MB
centos              latest              1e1148e4cc2c        2 months ago        202MB
```

7. Можно посмотреть историю того, как был создан образ

```
[vagrant@localhost ~]$ sudo docker history centos7_with_nano:ver1
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
8fb998ac2c98        2 minutes ago       bash                                            78MB                
1e1148e4cc2c        2 months ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B                  
<missing>           2 months ago        /bin/sh -c #(nop)  LABEL org.label-schema.sc…   0B                  
<missing>           2 months ago        /bin/sh -c #(nop) ADD file:6f877549795f4798a…   202MB
```
Обратите внимание, как непонятно отображены изменения в истории, которые мы вносили данным способо (bash). Сборка такие образом непрозрачна и впоследствии непонятно, как именно был собран тот или иной образ.

8. Поднимем контейнер на базе нового образа и убедимся, что в нем есть nano.

```
[vagrant@localhost ~]$ sudo docker run -ti --rm centos7_with_nano:ver1 bash
[root@d181bff2e851 /]# nano
[root@d181bff2e851 /]# exit
exit
[vagrant@localhost ~]$
```
Заметьте, что в этот раз мы использовали опцию --rm при создании контейнера, она означает, что по завершению контейнер будет удален (будто выполнили docker rm).

## Сборка собственных image (Dockerfile)

Теперь перейдем к рекомендованному способу. В данном случае нужно создать специальный файл Dockerfile, который будет описывать, как именно собирается образ.

1. Создадим папку и в ней Dockerfile
```
[vagrant@localhost ~]$ mkdir centos_with_nano
[vagrant@localhost ~]$ cd centos_with_nano/
[vagrant@localhost centos_with_nano]$ vi Dockerfile
```

2. Заполним файл следующим образом
```
# Version: 0.0.1

FROM centos:7

MAINTAINER Alexey Kurt <a.kurt@corp.mail.ru>

RUN yum install -y nano
```

3. Запустим сборку
```
[vagrant@localhost centos_with_nano]$ sudo docker build -t centos7_with_nano:ver2 .
Sending build context to Docker daemon  2.048kB
Step 1/3 : FROM centos:7
7: Pulling from library/centos
Digest: sha256:184e5f35598e333bfa7de10d8fb1cebb5ee4df5bc0f970bf2b1e7c7345136426
Status: Downloaded newer image for centos:7
 ---> 1e1148e4cc2c
Step 2/3 : MAINTAINER Alexey Kurt <a.kurt@corp.mail.ru>
 ---> Running in 9f79ac38d36d
Removing intermediate container 9f79ac38d36d
 ---> ff47975cc0f2
Step 3/3 : RUN yum install -y nano
 ---> Running in dc45d8a7028e
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
 * base: mirror.reconn.ru
 * extras: mirror.reconn.ru
 * updates: centos-mirror.rbc.ru
Resolving Dependencies
--> Running transaction check
---> Package nano.x86_64 0:2.3.1-10.el7 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package        Arch             Version                   Repository      Size
================================================================================
Installing:
 nano           x86_64           2.3.1-10.el7              base           440 k

Transaction Summary
================================================================================
Install  1 Package

Total download size: 440 k
Installed size: 1.6 M
Downloading packages:
Public key for nano-2.3.1-10.el7.x86_64.rpm is not installed
warning: /var/cache/yum/x86_64/7/base/packages/nano-2.3.1-10.el7.x86_64.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-release-7-6.1810.2.el7.centos.x86_64 (@CentOS)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : nano-2.3.1-10.el7.x86_64                                     1/1 
  Verifying  : nano-2.3.1-10.el7.x86_64                                     1/1 

Installed:
  nano.x86_64 0:2.3.1-10.el7                                                    

Complete!
Removing intermediate container dc45d8a7028e
 ---> 6c050bc6f920
Successfully built 6c050bc6f920
Successfully tagged centos7_with_nano:ver2
[vagrant@localhost centos_with_nano]$ 
```

Теперь пора объяснить, что именно происходит.
На каждую строчку в файле Dockerfile докер делает нечто подобное тому, что мы делали в первом способе:
Запускает контейнер из прыдыдущего image-слоя.
Выполняет на нём инструкцию из Dockerfile.
Если она выполнилась успешно, то docker commit этого контейнера, сохраняя в промежуточный image-слой. И удаление временного контейнера.
Переходит к следующей инструкции.

Пройдемся по тому, что сделала команда docker build:

```
Sending build context to Docker daemon  2.048kB
```
Все содержимое папки, которую мы указали в качестве рабочей директории (мы указали . то есть текущая директория) отправляются в докер-демон. У нас есть только Dockerfile, но если бы существовали другие файлы (например, конфиги), они были бы отправлены тоже.

```
Step 1/3 : FROM centos:7
7: Pulling from library/centos
Digest: sha256:184e5f35598e333bfa7de10d8fb1cebb5ee4df5bc0f970bf2b1e7c7345136426
Status: Downloaded newer image for centos:7
 ---> 1e1148e4cc2c
```
Любой Dockerfile должен обязательно начинаться с инструкции FROM, в которой указывается образ, на базе которого будет выполнен build. В данном случае мы выбрали centos:7. Создание промежуточного контейнера в данном случае не нужно, просто взят образ, уже существующий у нас (1e1148e4cc2c)

```
Step 2/3 : MAINTAINER Alexey Kurt <a.kurt@corp.mail.ru>
 ---> Running in 9f79ac38d36d
Removing intermediate container 9f79ac38d36d
 ---> ff47975cc0f2
```
Чтобы выполнить инструкцию, создался контейнер с ID 9f79ac38d36d на базе образа centos:7 из прошлого пункта, выполнена инструкция, контейнер был закоммичен в image-слой с ID ff47975cc0f2 и удален

```
Step 3/3 : RUN yum install -y nano
 ---> Running in dc45d8a7028e
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
 * base: mirror.reconn.ru
....

Installed:
  nano.x86_64 0:2.3.1-10.el7                                                    

Complete!
Removing intermediate container dc45d8a7028e
 ---> 6c050bc6f920
```
Аналогично, создан контейнер dc45d8a7028e на базе образа ff47975cc0f2, выполнена команда yum install и затем из контейнера создался image-слой 6c050bc6f920.

```
Successfully built 6c050bc6f920
Successfully tagged centos7_with_nano:ver2
```
Dockerfile закончился, поэтому сборка на этом завершена, image-слой 6c050bc6f920 получает имя "centos7_with_nano:ver2"

4. История создания образа будет выглядеть в данном способе уже гораздо понятнее:

```
[vagrant@localhost centos_with_nano]sudo docker history centos7_with_nano:ver2
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
6c050bc6f920        16 minutes ago      /bin/sh -c yum install -y nano                  78MB                
ff47975cc0f2        16 minutes ago      /bin/sh -c #(nop)  MAINTAINER Alexey Kurt <a…   0B                  
1e1148e4cc2c        2 months ago        /bin/sh -c #(nop)  CMD ["/bin/bash"]            0B                  
<missing>           2 months ago        /bin/sh -c #(nop)  LABEL org.label-schema.sc…   0B                  
<missing>           2 months ago        /bin/sh -c #(nop) ADD file:6f877549795f4798a…   202MB             
```

## Сборка собственных image (Dockerfile). Простой веб-сервер.

Давайте создадим что-нибудь более интересное — веб-сервер, раздающий статическую страницу. В качестве веб-сервера возьмем nginx.

1. Создадим новую папку и в ней веб-страницу.
```
[vagrant@localhost ~]$ mkdir http_server
[vagrant@localhost ~]$ cd http_server/
[vagrant@localhost http_server]$ echo '<HTML><H1>My first http server in docker</H1></HTML>' > index.html
[vagrant@localhost http_server]$
```

2. Пишем Dockerfile
```
# version 0.0.1

FROM centos:7

MAINTAINER Alexey Kurt <a.kurt@corp.mail.ru>

RUN yum install -y ngin

ADD index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["/usr/sbin/nginx", "-g daemon off;"]
```
Здесь новые команды:
ADD: копирует файл из контекста сборки в файловую систему образа. В данном случае мы кладем нашу статическую страницу в то место, откуда nginx раздает статичные файлы.
EXPOSE: указывает докеру, что контейнеру на базе данного образа может быть открыт порт 80.
CMD: процесс, который будет запущен в контейнере по умолчанию. В данном случае мы использовали синтаксис []. Это значит, что команда будет запущена сама по себе, а не через bash, как если указать её строчкой (как в RUN). "-g daemon off;" - специальная опция у nginx, которая говорит ему не демонизироваться. Ведь если он демонизируется, то процесс /usr/sbin/nginx сразу завершится, а с ним и весь контейнер
PS: да, я специально ошибся в написании nginx.

3. Собираем

```
[vagrant@localhost http_server]sudo docker build -t http_server .
Sending build context to Docker daemon  3.072kB
Step 1/6 : FROM centos:7
 ---> 1e1148e4cc2c
Step 2/6 : MAINTAINER Alexey Kurt <a.kurt@corp.mail.ru>
 ---> Using cache
 ---> ff47975cc0f2
Step 3/6 : RUN yum install -y ngin
 ---> Running in aa35ae1a828b
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
 * base: mirror.reconn.ru
 * extras: mirror.reconn.ru
 * updates: mirror.yandex.ru
No package ngin available.
Error: Nothing to do
The command '/bin/sh -c yum install -y ngin' returned a non-zero code: 1
```

Во-первых заметьте, что на шаге 1 и 2, докер ничего не делал. Точно такой же слой уже был собран в прошлый раз, поэтому он был просто переиспользован. Чтобы отключить данный кеш при сборке нужно добавить опцию --no-cache

На третьем шаге всё пошло не так, поэтому сборка была приостановлена. Благодаря промежуточным контейнерам довольно просто дебажить подобные случаи.
Вот он, контейнер:
```
[vagrant@localhost http_server]$ sudo docker ps -a
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS                         PORTS               NAMES
aa35ae1a828b        ff47975cc0f2             "/bin/sh -c 'yum ins…"   2 minutes ago       Exited (1) 2 minutes ago                           cranky_williams
```

4. Запустим новый на базе образа ff47975cc0f2 и попробуем исправить команду

```
[vagrant@localhost http_server]$ sudo docker run --rm -ti ff47975cc0f2
[root@d7e46cdaeffd /]# yum install nginx
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
 * base: mirror.logol.ru
 * extras: mirror.reconn.ru
 * updates: dedic.sh
base                                                       | 3.6 kB  00:00:00     
extras                                                     | 3.4 kB  00:00:00     
updates                                                    | 3.4 kB  00:00:00     
(1/4): extras/7/x86_64/primary_db                          | 180 kB  00:00:00     
(2/4): base/7/x86_64/group_gz                              | 166 kB  00:00:00     
(3/4): updates/7/x86_64/primary_db                         | 2.4 MB  00:00:01     
(4/4): base/7/x86_64/primary_db                            | 6.0 MB  00:00:01     
No package nginx available.
Error: Nothing to do
```

Хм, он и не nginx, какой же пакет нам нужен?
Беглый гуглеж подсказывает нам, что нужно перед этим подключить дополнительный репозиторий. Что ж, попробуем
```
[root@d7e46cdaeffd /]# yum install -y epel-release
Loaded plugins: fastestmirror, ovl
Loading mirror speeds from cached hostfile
 * base: mirror.logol.ru
 * extras: mirror.reconn.ru
 * updates: dedic.sh
Resolving Dependencies
--> Running transaction check
---> Package epel-release.noarch 0:7-11 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

==================================================================================
 Package                Arch             Version           Repository        Size
==================================================================================
Installing:
 epel-release           noarch           7-11              extras            15 k

Transaction Summary
==================================================================================
Install  1 Package

Total download size: 15 k
Installed size: 24 k
Downloading packages:
warning: /var/cache/yum/x86_64/7/extras/packages/epel-release-7-11.noarch.rpm: Header V3 RSA/SHA256 Signature, key ID f4a80eb5: NOKEY
Public key for epel-release-7-11.noarch.rpm is not installed
epel-release-7-11.noarch.rpm                               |  15 kB  00:00:00     
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Importing GPG key 0xF4A80EB5:
 Userid     : "CentOS-7 Key (CentOS 7 Official Signing Key) <security@centos.org>"
 Fingerprint: 6341 ab27 53d7 8a78 a7c2 7bb1 24c6 a8a7 f4a8 0eb5
 Package    : centos-release-7-6.1810.2.el7.centos.x86_64 (@CentOS)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Installing : epel-release-7-11.noarch                                       1/1 
  Verifying  : epel-release-7-11.noarch                                       1/1 

Installed:
  epel-release.noarch 0:7-11                                                      

Complete!

[root@d7e46cdaeffd /]# yum install -y nginx
Loaded plugins: fastestmirror, ovl
Loading mirror speeds from cached hostfile

....

55 всяких зависимостей наставилось
...
  perl-libs.x86_64 4:5.16.3-294.el7_6                                             
  perl-macros.x86_64 4:5.16.3-294.el7_6                                           
  perl-parent.noarch 1:0.225-244.el7                                              
  perl-podlators.noarch 0:2.5.1-3.el7                                             
  perl-threads.x86_64 0:1.87-4.el7                                                
  perl-threads-shared.x86_64 0:1.43-6.el7                                         

Complete!
[root@d7e46cdaeffd /]# exit
exit
[vagrant@localhost http_server]$
```
Ура! Нужные команды найдены. Заметьте, что мы выполняли эксперименты в контейнере, который потом удалим и никакого мусора не останется.

5. Правим Dockerfile
```
# version 0.0.2

FROM centos:7

MAINTAINER Alexey Kurt <a.kurt@corp.mail.ru>

RUN yum install -y epel-release

RUN yum install -y nginx

ADD index.html /usr/share/nginx/html/index.html

EXPOSE 80

CMD ["/usr/sbin/nginx", "-g daemon off;"]
```

6. собираем
```
[vagrant@localhost http_server]$ sudo docker build -t http_server .
Sending build context to Docker daemon  3.072kB
Step 1/7 : FROM centos:7
 ---> 1e1148e4cc2c
Step 2/7 : MAINTAINER Alexey Kurt <a.kurt@corp.mail.ru>
 ---> Using cache
 ---> ff47975cc0f2
Step 3/7 : RUN yum install -y epel-release
 ---> Running in 564f528007b6
Loaded plugins: fastestmirror, ovl
Determining fastest mirrors
 * base: mirror.yandex.ru
 * extras: mirror.logol.ru
 * updates: mirror.logol.ru
Resolving Dependencies
--> Running transaction check
---> Package epel-release.noarch 0:7-11 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

.... вывод опущен

Installed:
  epel-release.noarch 0:7-11                                                    

Complete!
Removing intermediate container 564f528007b6
 ---> d12859ca612a
Step 4/7 : RUN yum install -y nginx
 ---> Running in 6f06e85e424a
Loaded plugins: fastestmirror, ovl
Loading mirror speeds from cached hostfile
 * base: mirror.yandex.ru
 * epel: mirror.yandex.ru
 * extras: mirror.logol.ru
 * updates: mirror.logol.ru
Resolving Dependencies

.... вывод опущен
  perl-threads.x86_64 0:1.87-4.el7                                              
  perl-threads-shared.x86_64 0:1.43-6.el7                                       

Complete!
Removing intermediate container 6f06e85e424a
 ---> a306a610d98d
Step 5/7 : ADD index.html /usr/share/nginx/html/index.html
 ---> 3a7a39559db6
Step 6/7 : EXPOSE 80
 ---> Running in 903aa9340ab2
Removing intermediate container 903aa9340ab2
 ---> cef12055d10a
Step 7/7 : CMD ["/usr/sbin/nginx", "-g daemon off;"]
 ---> Running in d9ee1140ec67
Removing intermediate container d9ee1140ec67
 ---> ffa343436f3e
Successfully built ffa343436f3e
Successfully tagged http_server:latest
```

Ура!

7. Запустим:

```
[vagrant@localhost http_server]$ sudo docker run -d -p 8000:80 -ti http_server
b968471ce0f30932c2f456fe0ca6742ccfabee736e91fe8cfd8911bc5fc66a51
[vagrant@localhost http_server]$
```

Опция -p задает проброс портов. В данном случае порту хостовой машины 8000 будет соответствовать порт в контейнере 80.

8. Проверяем:
```
[vagrant@localhost http_server]$ curl 127.0.0.1:8000
<HTML><H1>My first http server in docker</H1></HTML>
[vagrant@localhost http_server]$
```
