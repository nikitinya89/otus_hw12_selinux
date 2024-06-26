# Otus Homework 12. SELinux
### Цель домашнего задания
Тренировка умения работать с SELinux: диагностировать проблемы и модифицировать политики SELinux для корректной работы приложений, если это требуется.
### Описание домашнего задания

1. Запустить nginx на нестандартном порту 3-мя разными способами:
- переключатели setsebool;
- добавление нестандартного порта в имеющийся тип;
- формирование и установка модуля SELinux.

2. Обеспечить работоспособность приложения при включенном selinux.
- развернуть приложенный стенд https://github.com/mbfx/otus-linux-adm/tree/master/selinux_dns_problems;
- выяснить причину неработоспособности механизма обновления зоны (см. README);
- предложить решение для данной проблемы;
- выбрать одно из решений для реализации, предварительно обосновав выбор;
- реализовать выбранное решение и продемонстрировать его работоспособность.

## Выполнение
### Запустить nginx на нестандартном порту 3-мя разными способами
С помощью *Vagrant* создаем виртуальную машину.
```
# -*- mode: ruby -*-
# vim: set ft=ruby :


MACHINES = {
  :selinux => {
        :box_name => "centos/7",
        :box_version => "2004.01",       
  },
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

      config.vm.define boxname do |box|

        box.vm.box = boxconfig[:box_name]
        box.vm.box_version = boxconfig[:box_version]

        box.vm.host_name = "selinux"
        box.vm.network "forwarded_port", guest: 4881, host: 4881

        box.vm.provider :virtualbox do |vb|
              vb.customize ["modifyvm", :id, "--memory", "1024"]
              needsController = false
        end

        box.vm.provision "shell", inline: <<-SHELL
          yum install -y epel-release
          yum install -y nginx
          sed -ie 's/:80/:4881/g' /etc/nginx/nginx.conf
          sed -i 's/listen       80;/listen       4881;/' /etc/nginx/nginx.conf
          systemctl start nginx
          systemctl status nginx
          ss -tlpn | grep 4881
        SHELL
    end
  end
end
```

Результатом выполнения команды *vagrant up* будет виртуальная машина с установленным *nginx*, работающим на 4881 порту и рабоютающим *SELinux*. Порт проброшен в хостовую ОС.  
Подключимся к ВМ по *ssh*. Убедимся, что SELinux включен, конфигурация nginx без ошибок, но служба не запускается.

![1.1](1.1.jpg)  
Из строчки
```bash
nginx: [emerg] bind() to 0.0.0.0:4881 failed (13: Permission denied)
```
мы уже можем сделать вывод, что с правами доступа к сокету. Попробуем решить эту проблему тремя способами.
#### Переключатель setsebool:
Проанализируем лог с помощью утилиты **audit2why**  
```bash
cat /var/log/audit/audit.log | grep 4881 | audit2why
```
![1.2](1.2.jpg)  

Из чего следует, что необходимо включить параметр *nis_enabled*
```bash
setsebool -P nis_enabled 1
```
После чего **nginx** успешно стартует  

![1.3](1.3.jpg)  

Вернем все, как было, чтобы проверить другие способы  
```bash
setsebool -P nis_enabled off
```
#### Добавление нестандартного порта в имеющийся тип:

Найдем тип для http трафика командой
```bash
semanage port -l | grep http
```
![1.4](1.4.jpg)  

Добавить необходимый порт к существущему типу можно командой:
```bash
semanage port -a -t http_port_t -p tcp 4881
```
*nginx* снова запускается корректно  
  
![1.5](1.5.jpg)  

Для демонстрации третьего способа опять все сломаем. Удаляем порт командой
```bash
semanage port -d -t http_port_t -p tcp 4881
```
#### Формирование и установка модуля SELinux:
Воспользуемся утилитой **audit2allow**
```bash
cat /var/log/audit/audit.log | grep 4881 | audit2allow
```
![1.6](1.6.jpg)  

Утилита сформировала модуль. Запустим его 
```bash
semodule -i nginx.pp
```
*nginx* снова работает. Так же убедимся в этом, перейдя по адресу http://127.0.0.1:4881 в хостовой ОС:  

![1.7](1.7.jpg)  

### Обеспечить работоспособность приложения при включенном selinux
На хост с устарновленным *ansible* клонируем репозиторий и переходим в нужный каталог:
```bash
git clone https://github.com/mbfx/otus-linux-adm.git
cd otus-linux-adm/selinux_dns_problems
```
Создаем стенд командой *vagrant up*. Результатом работы команды будет две созданные виртуальные машины: сервер с IP-адресом 192.168.50.10 и клиент с IP-адресом 192.168.50.15

```bash
vagrant status
Current machine states:

ns01                      running (virtualbox)
client                    running (virtualbox)
```
Попробуем внести изменения, введя команду
```bash
nsupdate -k /etc/named.zonetransfer.key
```
Но получаем ошибку **update failed: SERVFAIL**  
  
![2.1](2.1.jpg)  
Команда `cat /var/log/audit/audit.log | audit2why` ничего не показывает, следовательно ошибок со стороны клиента нет. Выполним ту же команду на сервере.  
  
![2.2](2.2.jpg)  
Из ошибки следует, что вместо типа **named_t** используется **etc_t**
```bash
ls -laZ /etc/named
```
![2.3](2.3.jpg)

Измненим тип:
```bash
chcon -R -t named_zone_t /etc/named

ls -laZ /etc/named
```
![2.4](2.4.jpg)  
Теперь повторное выполнение команды с клиента будет успешно:  
  
![2.5](2.5.jpg)  
  
Убедимся в этом, выполнив DNS-запрос с помощью утилиты **dig**:  
  
![2.6](2.6.jpg)  
