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
