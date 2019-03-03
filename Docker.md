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
TAG — тег образа. Обычно в качестве тега указывают версию. Самый последний тег также называется latest.
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

