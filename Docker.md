1. Установка docker
в виртуалке с centos7

установка зависимостей
```
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

добавление репозитория
```
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

установка
```
sudo yum install docker-ce
```

включение докера в автозагрузку
```
sudo systemctl enable docker.service
```

старт сервиса докера
```
sudo systemctl start docker.service
```

проверка, что всё прошло хорошо
```
sudo docker run hello-world
```
