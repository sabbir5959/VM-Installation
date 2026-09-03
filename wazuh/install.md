# Wazuh Lab Setup

This is a simple guide to create a local Wazuh lab using Vagrant and Ubuntu virtual machines.

## 1. Project Structure

First, create a folder named `VM`. Inside it, create another folder named `wazuh-lab`.

```text
VM/
└── wazuh-lab/
    └── Vagrantfile
```

The `Vagrantfile` contains the VM settings, such as the name, CPU, memory, and network configuration.

---

## 2. Create the Vagrant File

Open the file `Vagrantfile` and add the following configuration:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"

  # Disable default synced folder
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Default provider settings
  config.vm.provider :libvirt do |lv|
    lv.storage_pool_name = "default"
  end

  # =========================================================
  # Wazuh Manager
  # =========================================================
  config.vm.define "wazuh-manager" do |manager|
    manager.vm.hostname = "wazuh-manager"
    manager.vm.network "private_network",
      ip: "192.168.121.10"

    manager.vm.provider :libvirt do |lv|
      lv.memory = 6144
      lv.cpus = 4
      lv.cpu_mode = "host-passthrough"
    end
  end

  # =========================================================
  # Ubuntu Agent
  # =========================================================
  config.vm.define "ubuntu-01" do |agent|
    agent.vm.hostname = "ubuntu-01"
    agent.vm.network "private_network",
      ip: "192.168.121.11"

    agent.vm.provider :libvirt do |lv|
      lv.memory = 2048
      lv.cpus = 2
      lv.cpu_mode = "host-passthrough"
    end
  end

  # =========================================================
  # Attacker Machine
  # =========================================================
  config.vm.define "attacker-01" do |attacker|
    attacker.vm.hostname = "attacker-01"
    attacker.vm.network "private_network",
      ip: "192.168.121.12"

    attacker.vm.provider :libvirt do |lv|
      lv.memory = 2048
      lv.cpus = 2
      lv.cpu_mode = "host-passthrough"
    end
  end

  # =========================================================
  # Grafana Server
  # =========================================================
  config.vm.define "grafana-01" do |grafana|
    grafana.vm.hostname = "grafana-01"
    grafana.vm.network "private_network",
      ip: "192.168.121.13"

    grafana.vm.provider :libvirt do |lv|
      lv.memory = 2048
      lv.cpus = 2
      lv.cpu_mode = "host-passthrough"
    end
  end

  # =========================================================
  # PrestaShop E-Commerce Server
  # =========================================================
  config.vm.define "prestashop-01" do |prestashop|
    prestashop.vm.hostname = "prestashop-01"

    prestashop.vm.network "private_network",
      ip: "192.168.121.14"

    prestashop.vm.provider :libvirt do |lv|
      lv.memory = 2048
      lv.cpus = 2
      lv.cpu_mode = "host-passthrough"
    end
  end

  # =========================================================
  # TheHive Server
  # =========================================================
  config.vm.define "thehive-01" do |thehive|
    thehive.vm.hostname = "thehive-01"

    thehive.vm.network "private_network",
      ip: "192.168.121.15"

    thehive.vm.provider :libvirt do |lv|
      lv.memory = 8192
      lv.cpus = 4
      lv.cpu_mode = "host-passthrough"
    end
  end
end
```

This file creates 6 virtual machines:

- `wazuh-manager` at `192.168.121.10`
- `ubuntu-01` at `192.168.121.11`
- `attacker-01` at `192.168.121.12`
- `grafana-01` at `192.168.121.13`
- `prestashop-01` at `192.168.121.14`
- `thehive-01` at `192.168.121.15`

Each machine has a private network and a specific amount of CPU and memory.

---

## 3. Go to the Lab Folder

Open a terminal and move to the lab directory:

```bash
cd ~/VM/wazuh-lab
```

---

## 4. Create the Virtual Machines

Start the virtual machines with Vagrant:

```bash
vagrant up
```

After the setup is complete, check if the VMs are running:

```bash
vagrant status
```

---

## 5. Connect to the Wazuh Manager

Use this command to connect to the Wazuh Manager VM:

```bash
vagrant ssh wazuh-manager
```

Now you are inside the Ubuntu VM where Wazuh will be installed.

---

## 6. Install Wazuh

Download the Wazuh installation script:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
```

Make the script executable:

```bash
chmod +x wazuh-install.sh
```

Run the installer:

```bash
sudo ./wazuh-install.sh -a
```

This installs the following components:

- Wazuh Manager
- Wazuh Indexer
- Wazuh Dashboard
- Filebeat

---

## 7. Check the Installation

Use these commands to see if the services are running:

```bash
sudo systemctl status wazuh-manager
sudo systemctl status wazuh-indexer
sudo systemctl status wazuh-dashboard
sudo systemctl status filebeat
```

If all services show `active (running)`, the installation is successful.

---

## 8. Summary

The main steps are:

1. Create the `VM/wazuh-lab` folder
2. Add the `Vagrantfile` configuration
3. Run `vagrant up` to create the VMs
4. Connect with `vagrant ssh wazuh-manager`
5. Install Wazuh
6. Verify all services with `systemctl status`

This setup creates a small lab with Wazuh, an Ubuntu agent, an attacker machine, Grafana, PrestaShop, and TheHive.

---

## 9. Wazuh Manager Custom `ossec.conf`

This is the custom manager configuration used in this lab. It is based on the default Wazuh configuration for Ubuntu 24.04, but with extra monitoring and security settings added.

Save this file at:

```bash
/etc/ossec/etc/ossec.conf
```

```xml
<!--
  Wazuh - Manager - Default configuration for ubuntu 24.04
  More info at: https://documentation.wazuh.com
  Mailing list: https://groups.google.com/forum/#!forum/wazuh
-->

<ossec_config>
  
  <syscheck>
    <disabled>no</disabled>

    <frequency>43200</frequency>

    <scan_on_start>yes</scan_on_start>

    <alert_new_files>yes</alert_new_files>

    <auto_ignore frequency="10" timeframe="3600">no</auto_ignore>

    <directories realtime="yes" report_changes="yes">/etc/passwd</directories>
    <directories realtime="yes" report_changes="yes">/etc/shadow</directories>
    <directories realtime="yes" report_changes="yes">/etc/group</directories>
    <directories realtime="yes" report_changes="yes">/etc/gshadow</directories>

    <directories realtime="yes" report_changes="yes">/etc/sudoers</directories>
    <directories realtime="yes" report_changes="yes">/etc/sudoers.d</directories>

    <directories realtime="yes" report_changes="yes">/etc/ssh/sshd_config</directories>
    <directories realtime="yes" report_changes="yes">/etc/ssh/sshd_config.d</directories>

    <directories realtime="yes" report_changes="yes">/etc/pam.d</directories>
    <directories realtime="yes" report_changes="yes">/etc/security</directories>

    <directories realtime="yes" report_changes="yes">/etc/crontab</directories>
    <directories realtime="yes" report_changes="yes">/etc/cron.d</directories>
    <directories realtime="yes" report_changes="yes">/etc/cron.daily</directories>
    <directories realtime="yes" report_changes="yes">/etc/cron.hourly</directories>
    <directories realtime="yes" report_changes="yes">/etc/cron.monthly</directories>
    <directories realtime="yes" report_changes="yes">/etc/cron.weekly</directories>

    <directories realtime="yes" report_changes="yes">/var/ossec/etc/ossec.conf</directories>
    <directories realtime="yes" report_changes="yes">/var/ossec/etc/rules</directories>
    <directories realtime="yes" report_changes="yes">/var/ossec/etc/decoders</directories>
    <directories realtime="yes" report_changes="yes">/var/ossec/etc/lists</directories>
    <directories realtime="yes" report_changes="yes">/var/ossec/integrations</directories>

    <directories>/etc,/usr/bin,/usr/sbin</directories>
    <directories>/bin,/sbin,/boot</directories>

  </syscheck>

```

### Important notes

- The first `<ossec_config>` section contains the main manager settings.
- The second `<ossec_config>` section adds extra local log monitoring for `journald`, `active-responses.log`, and `dpkg.log`.
- `rootcheck`, `syscheck`, `vulnerability-detection`, and `syscollector` are enabled to improve visibility on the server.
- `cis-cat` and `osquery` are kept disabled in this lab because they are not needed for the basic setup.
- The `cluster` section is disabled because this is a single-node lab environment.
- The `remote` section keeps agent communication active on port `1514`.

### After editing the file

After saving the file, restart Wazuh Manager:

```bash
sudo systemctl restart wazuh-manager
```

Then check the service status:

```bash
sudo systemctl status wazuh-manager
```

If the service is running normally, the custom configuration is active.

---

## 10. Final Lab Flow

A basic flow for this lab is:

1. Start all VMs with `vagrant up`
2. Connect to `wazuh-manager`
3. Install Wazuh Manager and dependencies
4. Update the custom `ossec.conf`
5. Restart the manager
6. Connect agents and monitor events from the dashboard

This gives you a complete local monitoring lab with Wazuh, logs, and multiple test machines.