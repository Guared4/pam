# Vagrant+ansible-стенд c PAM

## Цель домашнего задания  
Научиться создавать пользователей и добавлять им ограничения.

### Описание домашнего задания  

1. Запретить всем пользователям кроме группы admin логин в выходные (суббота и воскресенье), без учета праздников   

Задание со звездочкой   
2. * дать конкретному пользователю права работать с докером и возможность перезапускать докер сервис.   

## Введение:   

Почти все операционные системы Linux — многопользовательские. Администратор Linux должен уметь создать и настраивать пользователей.   
В Linux есть 3 группы пользователей:    
•	Администраторы — привилегированные пользователи с полным доступом к системе. По умолчанию в ОС есть такой пользователь — root   
•	Локальные пользователи — их учетные записи создает администратор, их права ограничены. Администраторы могут изменять права локальных пользователей   
•	Системные пользователи — учетный записи, которые создаются системой для внутренних процессов и служб. Например пользователь — nginx   

У каждого пользователя есть свой уникальный идентификатор — UID.    
Чтобы упростить процесс настройки прав для новых пользователей, их объединяют в группы. Каждая группа имеет свой набор прав и ограничений. Любой пользователь, создаваемый или добавляемый в такую группу,   
автоматически их наследует. Если при добавлении пользователя для него не указать группу, то у него будет своя, индивидуальная группа — с именем пользователя. Один пользователь может одновременно входить в несколько групп.   
Информацию о каждом пользователе сервера можно посмотреть в файле /etc/passwd   
Для более точных настроек пользователей можно использовать подключаемые модули аутентификации (PAM)   
PAM (Pluggable Authentication Modules - подключаемые модули аутентификации) — набор библиотек, которые позволяют интегрировать различные методы аутентификации в виде единого API.   
PAM решает следующие задачи:    
•	Аутентификация — процесс подтверждения пользователем своей подлинности. Например: ввод логина и пароля, ssh-ключ и т д.    
•	Авторизация — процесс наделения пользователя правами   
•	Отчетность — запись информации о произошедших событиях   
PAM может быть реализован несколькими способами:    
•	Модуль pam_time — настройка доступа для пользователя с учетом времени   
•	Модуль pam_exec — настройка доступа для пользователей с помощью скриптов   
•	И т.д.   


## Формат сдачи:   
Vagrantfile + ansible   

# Выполнение:  
## 1. Создал Vagranfile

```vagrantfile

Vagrant.configure("2") do |config|
  # Дистрибутив Ubuntu 22.04
  MACHINES = {
    :"pam" => {
      :box_name => "ubuntu/jammy64",
      :cpus => 2,
      :memory => 2048,
      :ip => "192.168.57.10",
    }
  }

  MACHINES.each do |boxname, boxconfig|
    config.vm.synced_folder ".", "/vagrant", disabled: true
    config.vm.network "private_network", ip: boxconfig[:ip]
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s

      box.vm.provider "virtualbox" do |v|
        v.memory = boxconfig[:memory]
        v.cpus = boxconfig[:cpus]
      end
      
      # Настройка SSH для аутентификации по паролю
      box.vm.provision "shell", inline: <<-SHELL
        if ! grep -q "^PasswordAuthentication yes" /etc/ssh/sshd_config; then
          echo "PasswordAuthentication yes" >> /etc/ssh/sshd_config
        fi
        systemctl restart sshd.service
      SHELL

      # Настройка Ansible provisioner
      box.vm.provision "ansible" do |ansible|
        ansible.playbook = "pam.yml"      # Указывает путь к playbook
        ansible.verbose = "-v"            # Включает подробный вывод
        ansible.become = true             # Включает выполнение задач с правами sudo
      end
    end
  end
end

```

## 2. Написал Playbook pam.yml   

```yml

- hosts: all
  become: yes
  tasks:

    # Создание пользователей otusadm и otus
    - name: Create user otusadm
      user:
        name: otusadm
        state: present

    - name: Create user otus
      user:
        name: otus
        state: present

    # Установка паролей для пользователей с помощью chpasswd
    - name: Set password for otusadm
      shell: echo "otusadm:Otus2022!" | chpasswd

    - name: Set password for otus
      shell: echo "otus:Otus2022!" | chpasswd

    # Создание группы admin
    - name: Ensure admin group exists
      group:
        name: admin
        state: present

    # Добавление пользователей в группу admin
    - name: Add otusadm to admin group
      user:
        name: otusadm
        groups: admin
        append: yes

    - name: Add root to admin group
      user:
        name: root
        groups: admin
        append: yes

    - name: Add vagrant to admin group
      user:
        name: vagrant
        groups: admin
        append: yes

    # Настройка PAM для ограничения доступа по выходным для пользователя otus
    - name: Ensure /etc/security/time.conf exists
      file:
        path: /etc/security/time.conf
        state: touch
        owner: root
        group: root
        mode: '0644'

    - name: Restrict login for otus on weekends
      lineinfile:
        path: /etc/security/time.conf
        line: 'sshd;*;otus;!Wd0000-2400'  # Блокировка доступа по выходным
        create: yes
        state: present

    - name: Enable pam_time in PAM configuration
      lineinfile:
        path: /etc/pam.d/sshd
        line: 'account required pam_time.so'
        insertafter: '^account'
        state: present

    # Перезапуск SSHD
    - name: Restart SSH service
      service:
        name: sshd
        state: restarted

    # Копирование SSH-ключа для пользователей otus и otusadm
    - name: Ensure .ssh directory exists for otus
      file:
        path: /home/otus/.ssh
        state: directory
        owner: otus
        group: otus
        mode: '0700'

    - name: Copy SSH public key to otus
      copy:
        content: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
        dest: /home/otus/.ssh/authorized_keys
        owner: otus
        group: otus
        mode: '0600'

    - name: Ensure .ssh directory exists for otusadm
      file:
        path: /home/otusadm/.ssh
        state: directory
        owner: otusadm
        group: otusadm
        mode: '0700'

    - name: Copy SSH public key to otusadm
      copy:
        content: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
        dest: /home/otusadm/.ssh/authorized_keys
        owner: otusadm
        group: otusadm
        mode: '0600'

    # Установка Docker и добавление пользователя otusadm в группу docker

    # Обновление списка пакетов
    - name: Update apt package cache
      apt:
        update_cache: yes

    # Установка зависимостей для добавления репозитория Docker
    - name: Install dependencies for Docker repository
      apt:
        name:
          - ca-certificates
          - curl
          - gnupg
          - lsb-release
        state: present

    # Ручная загрузка GPG-ключа Docker
    - name: Download Docker GPG key manually
      shell: curl -fsSL http://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

    # Добавление GPG-ключа Docker
    - name: Add Docker GPG key
      apt_key:
        file: /usr/share/keyrings/docker-archive-keyring.gpg
        state: present

    # Добавление репозитория Docker
    - name: Add Docker repository
      apt_repository:
        repo: "deb [arch=amd64] https://download.docker.com/linux/ubuntu jammy stable"
        state: present

    # Обновление списка пакетов после добавления репозитория
    - name: Update apt package cache after adding Docker repository
      apt:
        update_cache: yes

    # Установка Docker
    - name: Install Docker
      apt:
        name: docker-ce
        state: present

    # Проверка, что Docker запущен и включен в автозагрузку
    - name: Ensure Docker is running and enabled
      service:
        name: docker
        state: started
        enabled: yes

    # Добавление пользователя otusadm в группу docker
    - name: Ensure docker group exists
      group:
        name: docker
        state: present

    - name: Add otusadm to docker group
      user:
        name: otusadm
        groups: docker
        append: yes

    # Разрешение пользователю otusadm перезапускать Docker через sudo
    - name: Validate sudoers file before modification
      command: visudo -cf /etc/sudoers
      changed_when: false

    - name: Allow otusadm to restart Docker service via sudo
      lineinfile:
        path: /etc/sudoers
        line: 'otusadm ALL=(ALL) NOPASSWD: /bin/systemctl restart docker'
        validate: 'visudo -cf %s'

    # Перезагрузка systemd и перезапуск Docker
    - name: Reload systemd
      systemd:
        daemon_reload: yes

    - name: Restart Docker service
      service:
        name: docker
        state: restarted

```    

## 3. На хосте создал публичный ключ SSH, это необходимо для доступа пользователей otus и otusadm по ssh на ВМ и отработки playbook без ошибок   
*в моем случае без этого действия ВМ на отрез отказывалась давать доступ на подключение, ломал голову как это исправить, в итоге пришел к такому решению*

```shel

root@guared:/home/guared/pam# ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
/root/.ssh/id_rsa already exists.
Overwrite (y/n)? y
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:ckDOO6bI1Pf9I2GycSGI02LZ9ZTgxQBlO6w9Ym6ZOeM your_email@example.com
The key's randomart image is:
+---[RSA 4096]----+
|      oo=+..     |
|     + +.o+      |
|     =+o=o       |
|   .* +=..o      |
|  ...oX S. .     |
| o . * Xoo+      |
|  o . O .*..     |
|     o o. ...    |
|      E    ...   |
+----[SHA256]-----+

```

## 4. Далее выполнил команду vagrant up   

В результате выполнения команды, была развернута ВМ и playbook отработал без ошибок

```shell

root@guared:/home/guared/pam# vagrant up
Bringing machine 'pam' up with 'virtualbox' provider...
==> pam: Importing base box 'ubuntu/jammy64'...
==> pam: Matching MAC address for NAT networking...
==> pam: Checking if box 'ubuntu/jammy64' version '20241002.0.0' is up to date...
==> pam: Setting the name of the VM: pam_pam_1737224818492_80326
==> pam: Clearing any previously set network interfaces...
==> pam: Preparing network interfaces based on configuration...
    pam: Adapter 1: nat
    pam: Adapter 2: hostonly
==> pam: Forwarding ports...
    pam: 22 (guest) => 2222 (host) (adapter 1)
==> pam: Running 'pre-boot' VM customizations...
==> pam: Booting VM...
==> pam: Waiting for machine to boot. This may take a few minutes...
    pam: SSH address: 127.0.0.1:2222
    pam: SSH username: vagrant
    pam: SSH auth method: private key
    pam: Warning: Connection reset. Retrying...
    pam: Warning: Connection reset. Retrying...
    pam: Warning: Connection reset. Retrying...
    pam: Warning: Connection reset. Retrying...
    pam: Warning: Connection reset. Retrying...
    pam:
    pam: Vagrant insecure key detected. Vagrant will automatically replace
    pam: this with a newly generated keypair for better security.
    pam:
    pam: Inserting generated public key within guest...
    pam: Removing insecure key from the guest if it's present...
    pam: Key inserted! Disconnecting and reconnecting using new SSH key...
==> pam: Machine booted and ready!
==> pam: Checking for guest additions in VM...
    pam: The guest additions on this VM do not match the installed version of
    pam: VirtualBox! In most cases this is fine, but in rare cases it can
    pam: prevent things such as shared folders from working properly. If you see
    pam: shared folder errors, please make sure the guest additions within the
    pam: virtual machine match the version of VirtualBox you have installed on
    pam: your host and reload your VM.
    pam:
    pam: Guest Additions Version: 6.0.0 r127566
    pam: VirtualBox Version: 6.1
==> pam: Setting hostname...
==> pam: Configuring and enabling network interfaces...
==> pam: Running provisioner: shell...
    pam: Running: inline script
==> pam: Running provisioner: ansible...
    pam: Running ansible-playbook...
...
PLAY RECAP *********************************************************************
pam                        : ok=31   changed=23   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

```   

# 5.  Пробую подключиться с хостовой машины на ВМ и поменять дату на 27 августа 2022 года   

```shell

root@guared:/home/guared/pam# ssh otus@192.168.57.10
The authenticity of host '192.168.57.10 (192.168.57.10)' can't be established.
ED25519 key fingerprint is SHA256:ckDOO6bI1Pf9I2GycSGI02LZ9ZTgxQBlO6w9Ym6ZOeM.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.57.10' (ED25519) to the list of known hosts.
Enter passphrase for key '/root/.ssh/id_rsa':
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-130-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Jan 17 16:38:02 UTC 2025

  System load:  0.56              Processes:               108
  Usage of /:   5.7% of 38.70GB   Users logged in:         0
  Memory usage: 13%               IPv4 address for enp0s3: 10.0.2.15
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

3 updates can be applied immediately.
3 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '24.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.


The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

$ whoami
otus
$ exit
Connection to 192.168.57.10 closed.
...
$ whoami
otusadm
$ sudo date 082712302022.00
Sat Aug 27 12:30:00 UTC 2022
$ date
Sat Aug 27 12:30:14 UTC 2022
$ exit
Connection to 192.168.57.10 closed.

```   

# 6 Пробую подключиться на ВМ, чтобы убедиться что после смены даты условие запрета доступа для пользователя otus в выходные дни выполняется   

```shell  

root@guared:/home/guared/pam# ssh otus@192.168.57.10
Enter passphrase for key '/root/.ssh/id_rsa':
Connection closed by 192.168.57.10 port 22

```
*а для пользователя otusadm запрета нет*   

```shell  

root@guared:/home/guared# ssh otusadm@192.168.57.10
Enter passphrase for key '/root/.ssh/id_rsa':
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-130-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Jan 17 16:39:48 UTC 2025

  System load:  0.16              Processes:               110
  Usage of /:   5.7% of 38.70GB   Users logged in:         0
  Memory usage: 14%               IPv4 address for enp0s3: 10.0.2.15
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

3 updates can be applied immediately.
3 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

New release '24.04.1 LTS' available.
Run 'do-release-upgrade' to upgrade to it.



The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

$ whoami
otusadm

```

# Задание со зведочкой. Попытка перезапуска docker от имени пользователя otusadm   

```shell

$ systemctl restart docker
==== AUTHENTICATING FOR org.freedesktop.systemd1.manage-units ===
Authentication is required to restart 'docker.service'.
Multiple identities can be used for authentication:
 1.  Ubuntu (ubuntu)
 2.  otusadm
 3.  ,,, (vagrant)
Choose identity to authenticate as (1-3): 2
Password:
==== AUTHENTICATION COMPLETE ===
$ systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2022-08-27 12:54:07 UTC; 13s ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 6258 (dockerd)
      Tasks: 9
     Memory: 20.7M
        CPU: 13.913s
     CGroup: /system.slice/docker.service
             └─6258 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

```   
____________________   

end
