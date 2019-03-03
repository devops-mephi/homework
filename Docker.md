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

