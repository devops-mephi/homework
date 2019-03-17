# Gitlab-CI

## Continius integration

Continius integration (CI) — практика, при которой код помещается в общий репозиторий, а на каждое изменение в этом репозитории автоматически прогоняется сборка/тестирование написанного кода.

Gitlab CI — встроенный в Gitlab инструмент для реализации практики CI.
Он состоит из двух частей: Runner'ы и файл .gitlab-ci.yml

Runner — машина, желательно отдельная (мы будем ставить в ту же виртуальную машину, так как у нас нет столько ресурсов на ноутбуках), которая занимается выполнением задач, описанных в .gitlab-ci.yml файле.
Установим runner'а внутри виртуальной машины vagrant

1. Подключение репозитория
```
[vagrant@localhost banners]$ curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh | sudo bash
```

2. Установка

```
[vagrant@localhost banners]$ sudo yum install -y gitlab-runner
```

3. Регистрация

Откройте настройки проекта Settings - CI / CD

![Runners](images/runners.png)

Запустите
```
sudo gitlab-runner register
```

Он будет задавать вопросы, отвечайте тем, что видите на экране настроек проекта
```
Runtime platform                                    arch=amd64 os=linux pid=29782 revision=4745a6f3 version=11.8.0
Running in system-mode.                            
                                                   
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://127.0.0.1/
Please enter the gitlab-ci token for this runner:
7pNpywCPRfNdNgwMBij9
Please enter the gitlab-ci description for this runner:
[localhost.localdomain]: vagrant
Please enter the gitlab-ci tags for this runner (comma separated):
vagrant
Registering runner... succeeded                     runner=7pNpywCP
Please enter the executor: shell, ssh, virtualbox, docker+machine, kubernetes, docker, docker-ssh, parallels, docker-ssh+machine:
shell
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! 
[vagrant@localhost banners]$
```

На вопрос, какой executor выбрать, ответим "shell". Это значит, что команды от gitlab-ci будут выполнены прямо в shell'е на машине, на которой установлен runner.
Есть другие опции вроде виртуальных машин (будет поднята машина на каждую сборку) или docker (будет создан контейнер под каждую сборку)

После окончания регистрации на странице настроек проекта должен быть виден настроенный runner.

![Runner activated](images/runner_activated.png)

4. Добавление прав пользователю gitlab-runner.
Мы будем хотеть запускать команды докера из раннера. Поэтому нужно добавить его в группу "docker"

```
sudo usermod -aG docker gitlab-runner
```

5. Теперь добавим файл .gitlab-ci.yml в корень проекта
```
stages:
  - build

build:
  stage: build
  script:
    - whoami
    - docker build -t banners_web:$CI_BUILD_REF_NAME .
  tags:
    - vagrant
```

6. Когда мы запушим изменение из пункта 5, должна автоматически запуститься сборка 

![Gitlab CI job](images/gitlab-ci-job.png)


## Docker runner

Схема с shell-раннером хороша своей простотой, но у неё есть несколько недостатков: нельзя делать параллельные запуски (так как они будут друг друга аффектить) и существует зависимость от состояния машины. Например, если прошлая сборка что-то положила в кеш докера, то новая будет брать из этого кеша. Можно запускать на каждую сборку виртуальную машину, но это довольно тяжело. Будем запускать каждую сборку в отдельном докер-контейнере.
Но цель нашей сборки — собрать докер контейнер, то есть нам потребуется запускать докер внутри докера. К счастью, существует возможность так сделать.

1. Настроить такой runner уже придется вручную:
```
sudo gitlab-runner register -n \
--url http://gitlab.mephi/ \
--registration-token J_HKEz_YhzwApzCjtb5T \
--description "Vagrant Docker" \
--tag-list docker_in_docker \
--executor docker \
--docker-image "docker:stable" \
--docker-privileged
```

Итого у нас получилось 2 раннера на одной машине:

![Gitlab CI two runners](images/gitlab-two-runners.png)

2. Допишем .gitlab-ci файл:

```
stages:
  - build

build:
  stage: build
  script:
    - whoami
    - docker build -t banners_web:$CI_BUILD_REF_NAME .
  tags:
    - vagrant
    
build_in_docker:
  services:
    - docker:dind
  stage: build
  variables:
    DOCKER_HOST: tcp://docker:2375/
  script:
    - whoami
    - docker build -t banners_web:$CI_BUILD_REF_NAME .
  tags:
    - docker_in_docker
```

3. Запушим, попробуем запустить:

```
Running with gitlab-runner 11.8.0 (4745a6f3)
  on Vagrant Docker jGJ96CGz
Using Docker executor with image docker:stable ...
Starting service docker:dind ...
Pulling docker image docker:dind ...
Using docker image sha256:85e924caedbd3e5245ad95cc7471168e923391b22dcb559decebe4a378a06939 for docker:dind ...
Waiting for services to be up and running...
Pulling docker image docker:stable ...
Using docker image sha256:639de9917ae1f2e4a586893c9a6ea3f21fd774bc4037184ecac35f3153a293b5 for docker:stable ...
Running on runner-jGJ96CGz-project-2-concurrent-0 via st91.i...
Cloning repository...
Cloning into '/builds/devops-mephi/banners'...
fatal: unable to access 'http://gitlab-ci-token:xxxxxxxxxxxxxxxxxxxx@gitlab.mephi/devops-mephi/banners.git/': Could not resolve host: gitlab.mephi
/bin/bash: line 65: cd: /builds/devops-mephi/banners: No such file or directory
ERROR: Job failed: exit code 1
```

Хм, получилось так, что докер внутри докера пытается обратиться по gilab.mephi и не знает, куда пойти, потому что нет записи в /etc/hosts.

4. Добавить эту запись можно следующим образом в конфиг gitlab-runner'а

```
sudo vi /etc/gitlab-runner/config.toml
```

допишем следующую строчку в runners.runners.docker
```
    extra_hosts = ["gitlab.mephi:127.0.0.1"]
```
и перезапустим раннеры

```
[vagrant@localhost banners]$ sudo gitlab-runner restart
Runtime platform                                    arch=amd64 os=linux pid=18901 revision=4745a6f3 version=11.8.0
```

5. Пробуем заново:

теперь новая ошибка
```
Cloning into '/builds/devops-mephi/banners'...
fatal: unable to access 'http://gitlab-ci-token:xxxxxxxxxxxxxxxxxxxx@gitlab.mephi/devops-mephi/banners.git/': Failed to connect to gitlab.mephi port 80: Connection refused
/bin/bash: line 65: cd: /builds/devops-mephi/banners: No such file or directory
ERROR: Job failed: exit code 1
```

Подсоединиться к хосту удалось, но там нет 80 порта. Хм.
Разгадка такова: для докера внутри докера 127.0.0.1 это не локальная машина vagrant, а контейнер. Нам нужно прописать IP именно машины с vagrant внутри docker-сети.
Узнать нужный IP можно следующим образом:

```
[vagrant@st91 banners]$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:75:dc:3d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global noprefixroute dynamic eth0
       valid_lft 79731sec preferred_lft 79731sec
    inet6 fe80::5054:ff:fe75:dc3d/64 scope link
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:c6:c4:39:a1 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:c6ff:fec4:39a1/64 scope link
       valid_lft forever preferred_lft forever
```

Нас интересует сеть docker0. Пропишем этот IP: 172.17.0.1

```
sudo vi /etc/gitlab-runner/config.toml
```

в runners.runners.docker
```
    extra_hosts = ["gitlab.mephi:172.17.0.1"]
```

6. Пробуем еще раз. Успех!

![Gitlab CI success](images/gitlab-docker-success.png)

PS: если бы наш гитлаб жил на нормальной отдельной машине с собственным настоящим доменным именем, это шаманство с IP адресами бы не потребовалось.

7. Отключим пока автоматический запуск этой сборки (потому что она слишком долгая и будет нам мешаться)

Допишем в .gitlab-ci.yaml строчку "when: manual"

```
stages:
  - build

build:
  stage: build
  script:
    - whoami
    - docker build -t banners_web:$CI_BUILD_REF_NAME .
  tags:
    - vagrant
    
build_in_docker:
  services:
    - docker:dind
  stage: build
  variables:
    DOCKER_HOST: tcp://docker:2375/
  script:
    - whoami
    - docker build -t banners_web:$CI_BUILD_REF_NAME .
  tags:
    - docker_in_docker
  when: manual
```

Видим, что теперь сборка автоматически не запустилась, а появилась кнопка, по которой её можно запустить

![Gitlab CI manual](images/gitlab-manual.png)
