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

      # Разрешает подключение пользователей по SSH с использованием пароля
      # box.vm.provision "shell", inline: <<-SHELL
      #  sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
      #  systemctl restart sshd.service  # Перезапуск службы SSHD
      #SHELL


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