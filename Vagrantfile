VMS = [
    {
        name: "directus",
        box: "ubuntu/jammy64",
        public_ip: "192.168.56.10",
        private_ip: "192.168.56.10",
        ssh_port: 2201,
        memory: 1024,
        cpus: 1,
        forwarded_ports: [
            {guest: 8055, host: 8055},
            {guest: 22, host: 2201}
        ]
    },
    {
        name: "node1-postgresql",
        box: "ubuntu/jammy64",
        public_ip: "192.168.56.20",
        private_ip: "192.168.56.20",
        ssh_port: 2202,
        memory: 1024,
        cpus: 1,
        forwarded_ports: [
            {guest: 22, host: 2202}
        ]
    },
    {
        name: "node2-postgresql",
        box: "ubuntu/jammy64",
        public_ip: "192.168.56.30",
        private_ip: "192.168.56.30",
        ssh_port: 2204,
        memory: 1024,
        cpus: 1,
        forwarded_ports: [
            {guest: 22, host: 2204}
        ]
    },
    {
        name: "node3-monitoring",
        box: "ubuntu/jammy64",
        public_ip: "192.168.56.40",
        private_ip: "192.168.56.40",
        ssh_port: 2205,
        memory: 2048,
        cpus: 2,
        forwarded_ports: [
            {guest: 22, host: 1105},
            {guest: 3000, host: 3000},  # Grafana
            {guest: 9090, host: 9090}   # Prometheus
        ]
    },
    {
        name: "etcd1",  # Etcd кластер (1-нода, для теста)
        box: "ubuntu/jammy64",
        public_ip: "192.168.56.50",
        private_ip: "192.168.56.50",
        ssh_port: 1106,
        memory: 512,
        cpus: 1,
        forwarded_ports: [
            {guest: 2379, host: 2379}, # Etcd client port
            {guest: 2380, host: 2380}, # Etcd peer port
            {guest: 22, host: 1106}
        ]
    },
    {
        name: "haproxy1",  # HAProxy (балансировщик для Patroni)
        box: "ubuntu/jammy64",
        public_ip: "192.168.56.60",
        private_ip: "192.168.56.60",
        ssh_port: 1107,
        memory: 512,
        cpus: 1,
        forwarded_ports: [
            {guest: 5432, host: 7654}, # Прокси для PostgreSQL
            {guest: 8008, host: 8008}, # Patroni API через HAProxy
            {guest: 22, host: 1107}
        ]
    },
    {
        name: "patroni-node1",  # Новая ВМ: Patroni + PostgreSQL (будущий лидер)
        box: "ubuntu/jammy64",
        public_ip: "192.168.56.80",
        private_ip: "192.168.56.80",
        ssh_port: 1108,
        memory: 1024,
        cpus: 1,
        forwarded_ports: [
            {guest: 22, host: 1108},
            {guest: 5432, host: 5433},  # PostgreSQL
            {guest: 8008, host: 8020}   # Patroni API
        ]
    },
    {
        name: "patroni-node2",  # Новая ВМ: Patroni + PostgreSQL (будущая реплика)
        box: "ubuntu/jammy64",
        public_ip: "192.168.56.81",
        private_ip: "192.168.56.81",
        ssh_port: 1109,
        memory: 1024,
        cpus: 1,
        forwarded_ports: [
            {guest: 22, host: 1109},
            {guest: 5432, host: 5434},  # PostgreSQL
            {guest: 8008, host: 8021}   # Patroni API
        ]
    },
    {
        name: "patroni-node3",  # Новая ВМ: Patroni + PostgreSQL (будущая реплика)
        box: "ubuntu/jammy64",
        public_ip: "192.168.56.82",
        private_ip: "192.168.56.82",
        ssh_port: 1110,
        memory: 1024,
        cpus: 1,
        forwarded_ports: [
            {guest: 22, host: 1110},
            {guest: 5432, host: 5435},  # PostgreSQL
            {guest: 8008, host: 8022}   # Patroni API
        ]
    }
]

Vagrant.configure("2") do |config|
    VMS.each do |vm|
        config.vm.define vm[:name] do |v|
            v.vm.box = vm[:box]

            # Сетевые настройки
            v.vm.network "private_network", ip: vm[:private_ip]
            v.vm.network "public_network", bridge: "enp4s0", ip: vm[:public_ip]

            # Проброс портов
            vm[:forwarded_ports].each do |fp|
                v.vm.network "forwarded_port",
                    guest: fp[:guest],
                    host: fp[:host],
                    auto_correct: false
            end

            # Настройки VirtualBox
            v.vm.provider "virtualbox" do |vb|
                vb.name = vm[:name]
                vb.memory = vm[:memory]
                vb.cpus = vm[:cpus]
                vb.customize ["modifyvm", :id, "--ioapic", "on"]
                vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
                vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
            end

            # Provisioning: создание пользователя devopsuser и настройка SSH
            v.vm.provision "shell", inline: <<-SHELL
                # Создаем нового пользователя devopsuser
                useradd -m -s /bin/bash devopsuser

                # Добавляем пользователя в группу sudo
                usermod -aG sudo devopsuser

                # Настроим sudo без пароля
                echo 'devopsuser ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/99_devopsuser

                # Создаем .ssh директорию для devopsuser
                mkdir -p /home/devopsuser/.ssh

                # Добавляем публичный ключ в authorized_keys для devopsuser
                echo "#{File.read('devopsuser_rsa.pub')}" >> /home/devopsuser/.ssh/authorized_keys

                # Устанавливаем правильные права доступа
                chown -R devopsuser:devopsuser /home/devopsuser/.ssh
                chmod 700 /home/devopsuser/.ssh
                chmod 600 /home/devopsuser/.ssh/authorized_keys
            SHELL
        end
    end
end

require 'yaml'

inventory = {
  "all" => {
    "hosts" => {}
  }
}

VMS.each do |vm|
  inventory["all"]["hosts"][vm[:name]] = {
    "ansible_host" => "127.0.0.1",
    "ansible_port" => vm[:ssh_port],
    "ansible_user" => "devopsuser",
    "ansible_ssh_private_key_file" => "devopsuser_rsa",
    "internal_host" => vm[:private_ip],
    "ansible_ssh_extra_args" => "-o StrictHostKeyChecking=no"
  }
end

File.write("./ansible/inventory/inventory.yml", inventory.to_yaml)