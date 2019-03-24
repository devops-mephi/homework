# Ansible

## 
На этот раз нам не нужна та огромная машина с gitlab, создадим несколько маленьких.

1. Создадим папку для Vagrantfile

```
iMac-Aleksej:develop kurt$ mkdir ansible_main
iMac-Aleksej:develop kurt$ cd ansible_main/
iMac-Aleksej:ansible_main kurt$ vi Vagrantfile
```

На этот раз воспользуемся возможностями Vagrant для provision машин.
```
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.network "private_network", ip: "10.2.0.10"
  config.vm.provider "virtualbox" do |v|
    v.memory = 256
    v.cpus = 1
  config.vm.provision "shell",
    inline: "yum -y install epel-release && yum -y install ansible"
  end
end
```

```
iMac-Aleksej:ansible_main kurt$ vagrant up
Bringing machine 'default' up with 'virtualbox' provider...
==> default: Importing base box 'centos/7'...
==> default: Matching MAC address for NAT networking...
==> default: Checking if box 'centos/7' version '1901.01' is up to date...
==> default: Setting the name of the VM: ansible_main_default_1553422452303_80356

....


    default: Installed:
    default:   ansible.noarch 0:2.7.9-1.el7                                                  
    default: 
    default: Dependency Installed:
    default:   PyYAML.x86_64 0:3.10-11.el7                                                   
    default:   libtomcrypt.x86_64 0:1.17-26.el7                                              
    default:   libtommath.x86_64 0:0.42.0-6.el7                                              
    default:   libyaml.x86_64 0:0.1.4-11.el7_0                                               
    default:   python-babel.noarch 0:0.9.6-8.el7                                             
    default:   python-backports.x86_64 0:1.0-8.el7                                           
    default:   python-backports-ssl_match_hostname.noarch 0:3.5.0.1-1.el7                    
    default:   python-cffi.x86_64 0:1.6.0-5.el7                                              
    default:   python-enum34.noarch 0:1.0.4-1.el7                                            
    default:   python-httplib2.noarch 0:0.9.2-1.el7                                          
    default:   python-idna.noarch 0:2.4-1.el7                                                
    default:   python-ipaddress.noarch 0:1.0.16-2.el7                                        
    default:   python-jinja2.noarch 0:2.7.2-2.el7                                            
    default:   python-keyczar.noarch 0:0.71c-2.el7                                           
    default:   python-markupsafe.x86_64 0:0.11-10.el7                                        
    default:   python-paramiko.noarch 0:2.1.1-9.el7                                          
    default:   python-ply.noarch 0:3.4-11.el7                                                
    default:   python-pycparser.noarch 0:2.14-1.el7                                          
    default:   python-setuptools.noarch 0:0.9.8-7.el7                                        
    default:   python-six.noarch 0:1.9.0-2.el7                                               
    default:   python2-crypto.x86_64 0:2.6.1-15.el7                                          
    default:   python2-cryptography.x86_64 0:1.7.2-2.el7                                     
    default:   python2-jmespath.noarch 0:0.9.0-3.el7                                         
    default:   python2-pyasn1.noarch 0:0.1.9-7.el7                                           
    default:   sshpass.x86_64 0:1.06-2.el7                                                   
    default: 
    default: Complete!
```

Мы видим, что после поднятия машины запустился provision-скрипт, который установил ansible в эту машину. Можно провиженить уже запущенную машину по команде 
```
vagrant provision
```

2. Зайдем на машину и убедимся, что ansible работает

```
iMac-Aleksej:ansible_main kurt$ vagrant ssh
[vagrant@localhost ~]$ ansible --help
Usage: ansible <host-pattern> [options]

Define and run a single task 'playbook' against a set of hosts

...
```

3. Выйдем из машины и создадим еще одну виртуалку, которую будем настраивать через ansible.

```
iMac-Aleksej:develop kurt$ mkdir ansible_slave
iMac-Aleksej:develop kurt$ cd ansible_slave/
iMac-Aleksej:ansible_slave kurt$ vi Vagrantfile
```

```
Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.network "private_network", ip: "10.2.0.11"
  config.vm.provider "virtualbox" do |v|
    v.memory = 256
    v.cpus = 1
  end
end
```

запустим её
```
vagrant up
```

4. Вернемся на машину с ansible и проверим, что из неё доступна эта свежесозданная машина по IP адресу

```
[vagrant@localhost ~]$ ping 10.2.0.11
PING 10.2.0.11 (10.2.0.11) 56(84) bytes of data.
64 bytes from 10.2.0.11: icmp_seq=1 ttl=64 time=0.643 ms
```

5. Теперь попробуем осуществить ping всех машин, прописанных в inventory файле

```
[vagrant@localhost ansible]$ ansible -i inventory.yml all -m ping
ansible_slave | UNREACHABLE! => {
    "changed": false, 
    "msg": "Failed to connect to the host via ssh: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).", 
    "unreachable": true
}
```

Как мы видим, авторизация не удалась.

6. Настроим авторизацию. Авторизация будет происходить по ssh-ключам. На машине с ansible нужно сгенерировать пару публичный-приватный ключи

```
[vagrant@localhost ansible]$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vagrant/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/vagrant/.ssh/id_rsa.
Your public key has been saved in /home/vagrant/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:kCeh0nLOJRfrLz+Vp7YTN0uxDc34RQdnCFD0xnRyfls vagrant@localhost.localdomain
The key's randomart image is:
+---[RSA 2048]----+
|      o   .++.+o=|
|   . . =     +.Bo|
|  o = B .    ++.E|
|   * = +    +.o =|
|    o . S  . * o |
|       .  + * o  |
|      . .. * o   |
|       o. + .    |
|        .o.o     |
+----[SHA256]-----+
```

Теперь публичный ключ (.ssh/id_rsa.pub) нужно положить на все машины, на которые нужно иметь доступ в файл .ssh/authorized_keys того же пользователя.
Ходить будем пользователем vargrant, который уже есть в системе.

7. На хост-машине в папке ansible_main сделаем следующее:

```
iMac-Aleksej:ansible_main kurt$ vagrant ssh-config
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2222
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /Users/kurt/develop/ansible_main/.vagrant/machines/default/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL

```

В ответ vagrant сказал нам настройки, использовав которые, можно залогиниться на машину (он их использует в команде vagrant ssh)
Воспользуемся ими для передачи файла
```
iMac-Aleksej:ansible_main kurt$ scp -P 2222 -i /Users/kurt/develop/ansible_main/.vagrant/machines/default/virtualbox/private_key vagrant@127.0.0.1:/.ssh/id_rsa.pub .
The authenticity of host '[127.0.0.1]:2222 ([127.0.0.1]:2222)' can't be established.
ECDSA key fingerprint is SHA256:DwdoCV7zmZpRqcwLjLjaTUWERWqyJ0+FAN9Hrtj1wXQ.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[127.0.0.1]:2222' (ECDSA) to the list of known hosts.
id_rsa.pub                                                                           100%  411   528.1KB/s   00:00    
```

Успех!

Теперь перейдем в папку ansible_slave и сделаем тоже самое, но уже загружая файл

```
iMac-Aleksej:ansible_slave kurt$ vagrant ssh-config
Host default
  HostName 127.0.0.1
  User vagrant
  Port 2200
  UserKnownHostsFile /dev/null
  StrictHostKeyChecking no
  PasswordAuthentication no
  IdentityFile /Users/kurt/develop/ansible_slave/.vagrant/machines/default/virtualbox/private_key
  IdentitiesOnly yes
  LogLevel FATAL
```

Копируем

```
iMac-Aleksej:ansible_slave kurt$ scp -P 2200 -i /Users/kurt/develop/ansible_slave/.vagrant/machines/default/virtualbox/private_key ../ansible_main/id_rsa.pub vagrant@127.0.0.1:/home/vagrant/
The authenticity of host '[127.0.0.1]:2200 ([127.0.0.1]:2200)' can't be established.
ECDSA key fingerprint is SHA256:oICdbo++Uf/dWOT8/zyMw+6lpPmMa5wDQF0hl0t8iOs.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '[127.0.0.1]:2200' (ECDSA) to the list of known hosts.
id_rsa.pub                                                                           100%  411   453.0KB/s   00:00
```

Теперь на машине slave делаем
```
[vagrant@localhost ~]$ cat id_rsa.pub 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDqH/+bq5vX+ao36WhfAyvqeDOr25fKRiqlw7PS5DLlaP1oQGfYM2DiNWxMJ3vWfeb1mFpZH2RX781LKxaxc3Z3rqdwjz0W2aj8IYRkezC4AiHmpXdOaOp+Ig9f81kDOCCFFOWzROkpgU7/J4yhvfFAGmXDDlQRRDKE2/Avj6XzhDkVOTa9H94sfXfBQO3rbWrCoFX/V6jrJY0btMo0hulHEqOYF8jkYJsLSdtIRTPCLe/EZObZ/4OWWZmoggYbzDCTBIWvV9p894q9Tnb8e34yxZm/UTwKXTD/78nc6iQnrtte6elr1vL9cnBsYmMvoz+ycPTeXOQ6Yp0BT5h78KWL vagrant@localhost.localdomain
[vagrant@localhost ~]$ cat id_rsa.pub >> .ssh/authorized_keys
```

7. Проверяем с машины с ansible

```
[vagrant@localhost ansible]$ ansible -i inventory.yml all -m ping
ansible_slave | SUCCESS => {
    "changed": false, 
    "ping": "pong"
}
```

Попробуем запустить команду (посмотрим, под каким пользователем мы)

```
[vagrant@localhost ansible]$ ansible -i inventory.yml all -a "whoami"
ansible_slave | CHANGED | rc=0 >>
vagrant
```

Если мы хотим выполнять команду от root, нужно добавить --become, тогда сделается sudo в другого пользователя (по умолчанию root)

```
[vagrant@localhost ansible]$ ansible -i inventory.yml all -a "whoami" --become          
ansible_slave | CHANGED | rc=0 >>
root
```

8. Не хочется каждый раз прописывать файл с хостами и флаг --become (скорее всего настраивать машину будет от root). Поэтому заведем файл с конфигурацией:
```
[vagrant@localhost ansible]$ vi ansible.cfg
```

```
[defaults]
inventory=inventory.yml

[privilege_escalation]
become=True
```


Теперь можно делать просто вот так
```
[vagrant@localhost ansible]$ ansible all -a "whoami"
ansible_slave | CHANGED | rc=0 >>
root
```
