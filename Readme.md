### Proyecto Final de Sistemas Operativos

**Autores:**
- Simón Colonia Amador - 22155291
- Alejandro Clavijo Arce - 2215402
- Juan David Moreno Mañunga - 2215290
- Juan Pablo Castaño - 2215929

#### Parte 1: Configuración de la Primera Máquina Virtual

**Configuración de recursos, sistema operativo y red:**
```ruby
config.vm.define :vm1 do |vm1|
  vm1.vm.box = "ubuntu/bionic64"
  vm1.vm.network "private_network", ip: "192.168.56.21"
  vm1.vm.synced_folder "./html", "/var/www/html"

  vm1.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
  end
end
```

**Creación de usuarios:**
Crea tres usuarios (`admin`, `tester`, `appUser`), donde `admin` forma parte del grupo de administradores y tiene acceso a sudo sin contraseña. Cada usuario tiene un directorio home con configuración SSH:
```ruby
vm1.vm.provision "shell", inline: <<-SHELL
  sudo useradd -m -g admin -s /bin/bash admin 
  echo "admin:password_admin" | sudo chpasswd

  sudo useradd -m -s /bin/bash tester 
  echo "tester:password_tester" | sudo chpasswd

  sudo useradd -m -s /bin/bash appUser 
  echo "appUser:password_appUser" | sudo chpasswd

  for user in admin tester appUser; do
      sudo mkdir -p /home/$user/.ssh
      sudo chown -R $user:$user /home/$user/.ssh
      sudo chmod 700 /home/$user/.ssh
  done

  ssh-keygen -t rsa -N "" -f /tmp/host_ssh_key

  for user in admin tester appUser; do
      sudo cp /tmp/host_ssh_key.pub /home/$user/.ssh/authorized_keys
      sudo chown $user:$user /home/$user/.ssh/authorized_keys
      sudo chmod 600 /home/$user/.ssh/authorized_keys
  done

  echo "admin ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/admin

  sudo cp /tmp/host_ssh_key /vagrant/host_ssh_key
  rm /tmp/host_ssh_key /tmp/host_ssh_key.pub
SHELL
```

**Conexión SSH:**
1. Si ya te has conectado antes, limpia la clave anterior:
   ```bash
   ssh-keygen -f "/home/%tu usuario%/.ssh/known_hosts" -R "[127.0.0.1]:2222"
   ```
2. Ajusta permisos y conéctate:
   ```bash
   chmod 600 host_ssh_key
   ssh -i host_ssh_key -p 2222 admin@127.0.0.1
   ssh -i host_ssh_key -p 2222 tester@127.0.0.1
   ssh -i host_ssh_key -p 2222 appUser@127.0.0.1
   ```

#### Parte 2: Configuración de la Segunda Máquina Virtual y Ansible

**Configuración de `vm2`:**
```ruby
config.vm.define :vm2 do |vm2|
  vm2.vm.box = "ubuntu/bionic64"
  vm2.vm.network "private_network", ip: "192.168.56.22"

  vm2.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = 2
  end
end
```

**Ejecución del `playbook.yml` con Ansible:**
```ruby
config.vm.provision "ansible" do |ansible|
  ansible.playbook = "playbook.yml"
  ansible.inventory_path = "inventory.ini"
end
```

**Archivo `inventory.ini`:**
```ini
[vm1]
192.168.56.21 ansible_user=admin ansible_ssh_private_key_file=host_ssh_key

[vm2]
192.168.56.22 ansible_user=admin ansible_ssh_private_key_file=host_ssh_key
```

**Configuración de `playbook.yml` para Prometheus y Grafana en `vm2`:**
```yaml
- name: Configurar la segunda máquina virtual
  hosts: vm2
  become: yes
  tasks:
    - name: Descargar Grafana
      get_url:
        url: https://dl.grafana.com/oss/release/grafana_7.5.10_amd64.deb
        dest: /tmp/grafana_7.5.10_amd64.deb

    - name: Instalar Grafana
      apt:
        deb: /tmp/grafana_7.5.10_amd64.deb
        state: present

    - name: Iniciar y habilitar Grafana
      systemd:
        name: grafana-server
        state: started
        enabled: yes

    - name: Instalar Prometheus
      apt:
        update_cache: yes
        name: prometheus
        state: present

    - name: Iniciar y habilitar Prometheus
      systemd:
        name: prometheus
        state: started
        enabled: yes

    - name: Configurar prometheus.yml
      copy:
        content: |
          global:
            scrape_interval: 15s
          scrape_configs:
            - job_name: 'node'
              static_configs:
                - targets: ['192.168.56.21:9100']
        dest: /etc/prometheus/prometheus.yml
        owner: prometheus
        group: prometheus
        mode: '0644'

    - name: Reiniciar y habilitar Prometheus
      systemd:
        name: prometheus
        state: restarted
        enabled: yes
```

#### Parte 3: Configuración de Servidor Web Estático y Pruebas de Carga con JMeter

**Directorio sincrónico para HTML estático en `vm1`:**
```ruby
vm1.vm.synced_folder "./html", "/var/www/html"
```

Para verificar el contenido estático, visita: [http://192.168.56.21/](http://192.168.56.21/)

**Archivo de prueba `test.jmx` para Apache JMeter:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="Test Plan">
      <elementProp name="TestPlan.user_defined_variables" elementType="Arguments">
        <collectionProp name="Arguments.arguments"/>
      </elementProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="Thread Group">
        <elementProp name="ThreadGroup.main_controller" elementType="LoopController">
          <boolProp name="LoopController.continue_forever">false</boolProp>
          <stringProp name="LoopController.loops">10</stringProp>
        </elementProp>
        <stringProp name="ThreadGroup.num_threads">100</stringProp>
        <stringProp name="ThreadGroup.ramp_time">10</stringProp>
        <boolProp name="ThreadGroup.scheduler">false</boolProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy">
          <elementProp name="HTTPsampler.Arguments" elementType="Arguments">
            <collectionProp name="Arguments.arguments"/>
          </elementProp>
          <stringProp name="HTTPSampler.domain">192.168.56.21</stringProp>
          <stringProp name="HTTPSampler.port">80</stringProp>
          <stringProp name="HTTPSampler.protocol">http</stringProp>
          <stringProp name="HTTPSampler.path">/</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
        </HTTPSamplerProxy>
        <hashTree/>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

**Acceso a Grafana en `vm2`:**
Accede en: [http://192.168.56.22:3000/](http://192.168.56.22:3000/)

Se debe agregar a prometeus como fuente de datos: [http://localhost:9090](http://localhost:9090)

**Visualización de métricas:**

- Para ver el tráfico de red (peticiones recibidas): agrega en el dashboard la siguiente métrica: node_network_receive_packets_total
- Para ver el uso de memoria: node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes
- Para ver el uso de disco: node_filesystem_size_bytes{mountpoint="/"} - node_filesystem_free_bytes{mountpoint="/"}

**Ejecución de la prueba de carga con JMeter:**
```bash
rm -rf test-results/web-report/
mkdir -p test-results/web-report/
jmeter -n -t test.jmx -l test-results/results.jtl -e -o test-results/web-report
```
Si, como yo, tienes Jmeter 5.6.3 instalado en esta misma carpeta:
```bash
rm -rf test-results/web-report/
mkdir -p test-results/web-report/
apache-jmeter-5.6.3/bin/jmeter -n -t test.jmx -l test-results/results.jtl -e -o test-results/web-report
```