# Deploy сервиса в продакшен

## Docker registry

Существующие сборки в Gitlab CI, которые мы делали на прошлых занятиях, собирают и хранят образ локально. Для деплоя нужно перенести собранный образ на продакшен-машину. Это делают с помощью docker registry.
Мы не будем поднимать свой собственный, а воспользуемся официальным Docker Hub.

1. Зарегистрируйтесь на https://hub.docker.com/signup

2. После успешной регистрации создайте репозиторий, который будет называться banners. В итоге вы сможете пушить образ в ИМЯ_ПОЛЬЗОВАТЕЛЯ/banners.
В нашем примере это будет devopsmephi/banners

3. Нужно сделать docker login на машине с gitlab-ci раннерами. Они у нас установлены в машине с Gitlab CI.

```
[vagrant@localhost ~]$ sudo su gitlab-runner

[gitlab-runner@localhost vagrant]$ sudo docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: devopsmephi
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

```

4. Добавим в .gitlab ci третий стейдж "deploy"

```
stages:
  - build
  - test
  - deploy
```

и сборку, которая пока будет просто вешать тег latest и пушить в репозиторий. Мы будем вешать еще один тег, с хешом коммита в git, чтобы потом было легче расследовать проблемы.

```
deploy_to_prod:
  stage: deploy
  tags:
    - vagrant
  script:
    - docker tag banners_web:$CI_BUILD_REF_NAME devopsmephi/banners:$CI_COMMIT_SHORT_SHA
    - docker tag banners_web:$CI_BUILD_REF_NAME devopsmephi/banners:latest
    - docker push devopsmephi/banners:$CI_COMMIT_SHORT_SHA
    - docker push devopsmephi/banners:latest

  when: manual
```

5. Запустим её


```
Running with gitlab-runner 11.8.0 (4745a6f3)
  on vagrant wC8Ldt2b
Using Shell executor...
Running on localhost.localdomain...
Fetching changes...
HEAD is now at 1fbc7d9 Update .gitlab-ci.yml
Checking out 1fbc7d97 as master...
Skipping Git submodules setup
$ docker tag banners_web:$CI_BUILD_REF_NAME devopsmephi/banners:$CI_COMMIT_SHORT_SHA
$ docker tag banners_web:$CI_BUILD_REF_NAME devopsmephi/banners:latest
$ docker push devopsmephi/banners:$CI_COMMIT_SHORT_SHA
The push refers to repository [docker.io/devopsmephi/banners]
5f737ef5aac8: Preparing
877c3b8b4986: Preparing
f4d7791fcb9d: Preparing
f61166e33473: Preparing
bb839e9783c7: Preparing
237ce60325c6: Preparing
1b976700da1f: Preparing
bde41e1d0643: Preparing
7de462056991: Preparing
3443d6cf0f1f: Preparing
237ce60325c6: Waiting
1b976700da1f: Waiting
bde41e1d0643: Waiting
7de462056991: Waiting
f3a38968d075: Preparing
a327787b3c73: Preparing
5bb0785f2eee: Preparing
3443d6cf0f1f: Waiting
f3a38968d075: Waiting
a327787b3c73: Waiting
bb839e9783c7: Mounted from library/python
5f737ef5aac8: Pushed
237ce60325c6: Mounted from library/python
f61166e33473: Pushed
f4d7791fcb9d: Pushed
7de462056991: Mounted from library/python
1b976700da1f: Mounted from library/python
3443d6cf0f1f: Mounted from library/python
bde41e1d0643: Mounted from library/python
f3a38968d075: Mounted from library/python
a327787b3c73: Mounted from library/python
5bb0785f2eee: Mounted from library/python
877c3b8b4986: Pushed
1fbc7d97: digest: sha256:2ee907f50abedadb8a5e5975eebd265d2ad8fb1c5ff604ca333e876bdae014d6 size: 3052
$ docker push devopsmephi/banners:latest
The push refers to repository [docker.io/devopsmephi/banners]
5f737ef5aac8: Preparing
877c3b8b4986: Preparing
f4d7791fcb9d: Preparing
f61166e33473: Preparing
bb839e9783c7: Preparing
237ce60325c6: Preparing
1b976700da1f: Preparing
bde41e1d0643: Preparing
7de462056991: Preparing
3443d6cf0f1f: Preparing
f3a38968d075: Preparing
a327787b3c73: Preparing
5bb0785f2eee: Preparing
bde41e1d0643: Waiting
7de462056991: Waiting
3443d6cf0f1f: Waiting
f3a38968d075: Waiting
a327787b3c73: Waiting
5bb0785f2eee: Waiting
237ce60325c6: Waiting
1b976700da1f: Waiting
f61166e33473: Layer already exists
877c3b8b4986: Layer already exists
5f737ef5aac8: Layer already exists
f4d7791fcb9d: Layer already exists
bb839e9783c7: Layer already exists
1b976700da1f: Layer already exists
237ce60325c6: Layer already exists
7de462056991: Layer already exists
bde41e1d0643: Layer already exists
3443d6cf0f1f: Layer already exists
a327787b3c73: Layer already exists
5bb0785f2eee: Layer already exists
f3a38968d075: Layer already exists
latest: digest: sha256:2ee907f50abedadb8a5e5975eebd265d2ad8fb1c5ff604ca333e876bdae014d6 size: 3052
Job succeeded
```

Как мы видим, сборка успешно запушила образ с тегом 
