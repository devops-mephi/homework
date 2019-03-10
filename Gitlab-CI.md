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
