# Docker-compose

## Установка docker-compose

в виртуалке с centos7

1. Рекомендуемый способ установки: скачивание binary-файла с docker-compose

```
sudo curl -L https://github.com/docker/compose/releases/download/1.24.0-rc1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

2. Установка прав на выполнение

```
sudo chmod +x /usr/local/bin/docker-compose
```

3. Создание символической ссылки 

```
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

4. Проверяем успешность установки

```
[vagrant@localhost ~]$ docker-compose version
docker-compose version 1.24.0-rc1, build 0f3d4dda
docker-py version: 3.7.0
CPython version: 3.6.8
OpenSSL version: OpenSSL 1.1.0j  20 Nov 2018
```

## Работа с docker-compose.

Попробуем реализовать веб-сервер, который делали на прошлом занятии.
1. Если осталась с прошлого занятия папка http_server, зайдем в неё, если нет, создайте новую
```
mkdir http_server
```

2. Создаем Dockerfile

```
vi Dockerfile
```

Содержимое:
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

3. Теперь, в прошлом занятии мы поднимали его после сборки следующим образом:
```
sudo docker run -d -p 8000:80 http_server
```

Тут нужно помнить, что есть порт 80, который нужно куда-то прокинуть, помнить название сервиса и ключи, с которыми это всё нужно запустить. Когда сервисов становится больше, чем один, становится довольно сложно запомнить и воспроизвести много docker-команд по запуску этих контейнеров и связыванию их ports/volumes между собой. Для решения этой проблемы был придуман docker-compose.

Он позволяет описать структуру всех используемых в проекте контейнеров в одном yaml файле. Попробуем описать http_server.

4. Создадим файл docker-compose.yml

```
vi docker-compose.yml
```

В нем пропишем следующее:

```
version: '3'

services:
  http_server:
    build: .
    ports:
      - "8000:80"
```


5. Запуск по одной кнопке

```
[vagrant@localhost http_server]$ sudo docker-compose up
Creating network "http_server_default" with the default driver
Building http_server
Step 1/7 : FROM centos:7
 ---> 1e1148e4cc2c
Step 2/7 : MAINTAINER Alexey Kurt <a.kurt@corp.mail.ru>
 ---> Using cache
 ---> d43f40c7fac6
Step 3/7 : RUN yum install -y epel-release
 ---> Using cache
 ---> ea42a1359c1e
Step 4/7 : RUN yum install -y nginx
 ---> Using cache
 ---> fd48f748887e
Step 5/7 : ADD index.html /usr/share/nginx/html/index.html
 ---> caa8ec3f2cfc
Step 6/7 : EXPOSE 80
 ---> Running in daac3dd5e549
Removing intermediate container daac3dd5e549
 ---> b82b7361f29f
Step 7/7 : CMD ["/usr/sbin/nginx", "-g daemon off;"]
 ---> Running in c049fa4fdb7d
Removing intermediate container c049fa4fdb7d
 ---> 8eff8c05c67b
Successfully built 8eff8c05c67b
Successfully tagged http_server_http_server:latest
WARNING: Image for service http_server was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Creating http_server_http_server_1 ... done
Attaching to http_server_http_server_1
```

Как мы видим, docker-compose успешно понял, что образ нужно собрать, собрал его, запустил и правильно пробросил нужные порты

6. Проверяем, что всё в порядке

из соседней консоли
```
[vagrant@localhost ~]$ curl 127.0.0.1:8000
<html>
<h1>My first http server</h1>
</html>
```

7. Остановить можно, нажав ctr-c в консоли с docker-compose

```
^CGracefully stopping... (press Ctrl+C again to force)
Stopping http_server_http_server_1 ... done
[vagrant@localhost http_server]$
```

8. Запустим еще раз

```
[vagrant@localhost http_server]$ sudo docker-compose up
Starting http_server_http_server_1 ... done
Attaching to http_server_http_server_1
```
На этот раз docker-compose не стал собирать заново образ, а просто его запустил

9. Из соседней консоли можно посмотреть список запущенных "служб"

```
[vagrant@localhost http_server]$ sudo docker-compose ps
          Name                         Command               State          Ports
-----------------------------------------------------------------------------------------
http_server_http_server_1   /usr/sbin/nginx -g daemon off;   Up      0.0.0.0:8000->80/tcp
```

10. Можно запустить любую команду внутри запущенного контейнера 

```
[vagrant@localhost http_server]$ sudo docker-compose run http_server bash
[root@e058b7b64359 /]# date
Sun Mar 10 10:55:16 UTC 2019
```

11. Можно остановить всё вот так

```
[vagrant@localhost http_server]$ sudo docker-compose stop
Stopping http_server_http_server_1 ... done
```

12. А теперь попробуем изменить файл index.html

```
[vagrant@localhost http_server]$ vi index.html
```

```
<html>
<h1>My first http server</h1>
<h2>It changed!/h2>
</html>
```

13. Запустим 

```
[vagrant@localhost http_server]$ sudo docker-compose up
Starting http_server_http_server_1 ... done
Attaching to http_server_http_server_1
```

Мы видим, что образ не был пересобран. Docker-compose просто проверяет, есть ли образ с таким именем или нет:
```
[vagrant@localhost http_server]$ sudo docker images
REPOSITORY                TAG                 IMAGE ID            CREATED             SIZE
http_server_http_server   latest              8eff8c05c67b        14 minutes ago      451MB
```

14. Чтобы таки-пересобрать образ, нужно делать так:

```
[vagrant@localhost http_server]$ sudo docker-compose build
Building http_server
Step 1/7 : FROM centos:7
 ---> 1e1148e4cc2c
Step 2/7 : MAINTAINER Alexey Kurt <a.kurt@corp.mail.ru>
 ---> Using cache
 ---> d43f40c7fac6
Step 3/7 : RUN yum install -y epel-release
 ---> Using cache
 ---> ea42a1359c1e
Step 4/7 : RUN yum install -y nginx
 ---> Using cache
 ---> fd48f748887e
Step 5/7 : ADD index.html /usr/share/nginx/html/index.html
 ---> 4e92d4192e24
Step 6/7 : EXPOSE 80
 ---> Running in bf50e5447304
Removing intermediate container bf50e5447304
 ---> 40cfbfd853e8
Step 7/7 : CMD ["/usr/sbin/nginx", "-g daemon off;"]
 ---> Running in 9241c24a1ad0
Removing intermediate container 9241c24a1ad0
 ---> 70de52ebbc5c
Successfully built 70de52ebbc5c
Successfully tagged http_server_http_server:latest
```

15. Удалить полностью все контейнеры/образы/сети вот так

```
[vagrant@localhost http_server]$ sudo docker-compose down
Removing http_server_http_server_1 ... done
Removing network http_server_default
```


## Создание среды для разработки проекта python+django+mysql

Допустим, мы хотим начать разрабатывать приложение, которое написано на python, использует django и mysql. Мы хотим, чтобы среда для разработки этого приложения для другого разработчика поднималась по одной кнопке. Docker-compose позволяет нам это сделать. По сути это более удобный способ описывать и управлять несколькими docker-контейнерами. Всё описание хранится в одном файле, который имеет имя docker-compose.yml. После того, как в нем всё описано, одной командой 'docker-compose up' соберутся/скачаются нужные docker-образы и запустятся контейнеры, правильно связавшись между собой.

Наш будущий проект будет использовать python3. 
