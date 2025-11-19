
require "yaml"
vagrant_root = File.dirname(File.expand_path(__FILE__))
settings = YAML.load_file "#{vagrant_root}/settings.yaml"

IP_SECTIONS = settings["network"]["control_ip"].match(/^([0-9.]+\.)([^.]+)$/)
# First 3 octets including the trailing dot:
IP_NW = IP_SECTIONS.captures[0]
# Last octet excluding all dots:
IP_START = Integer(IP_SECTIONS.captures[1])
NUM_WORKER_NODES = settings["nodes"]["workers"]["count"]

Vagrant.configure("2") do |config|
  config.vm.provision "shell", env: { "IP_NW" => IP_NW, "IP_START" => IP_START, "NUM_WORKER_NODES" => NUM_WORKER_NODES }, inline: <<-SHELL
      apt-get update -y
      echo "$IP_NW$((IP_START)) controlplane" >> /etc/hosts
      for i in `seq 1 ${NUM_WORKER_NODES}`; do
        echo "$IP_NW$((IP_START+i)) node0${i}" >> /etc/hosts
      done
  SHELL
### SSH config for host ###
  # If running on Windows host, run the SSH config generator automatically after `vagrant up`
  require 'rbconfig'
  if RbConfig::CONFIG['host_os'] =~ /mswin|mingw|cygwin/
    config.trigger.after :up do |t|
      t.info = "Generating host SSH config for easy 'ssh <vm>' access"
      t.run = { inline: "powershell -NoProfile -ExecutionPolicy Bypass -File \"#{vagrant_root}/scripts/generate-ssh-config.ps1\"" }
    end
    # Also run after reload to keep config in sync
    config.trigger.after :reload do |t|
      t.info = "Regenerating host SSH config after reload"
      t.run = { inline: "powershell -NoProfile -ExecutionPolicy Bypass -File \"#{vagrant_root}/scripts/generate-ssh-config.ps1\"" }
    end
  end
### end ssh config ###
  if `uname -m`.strip == "aarch64"
    config.vm.box = settings["software"]["box"] + "-arm64"
  else
    config.vm.box = settings["software"]["box"]
  end
  config.vm.box_check_update = true

  config.vm.define "controlplane" do |controlplane|
    controlplane.vm.hostname = "controlplane"
    controlplane.vm.network "private_network", ip: settings["network"]["control_ip"]
    # Set the VirtualBox VM name to match the hostname
    controlplane.vm.provider "virtualbox" do |vb_name_setter|
      vb_name_setter.gui = false
      vb_name_setter.customize ["modifyvm", :id, "--name", "controlplane"]
    end
    if settings["shared_folders"]
      settings["shared_folders"].each do |shared_folder|
        controlplane.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
      end
    end
    controlplane.vm.provider "virtualbox" do |vb|
        vb.cpus = settings["nodes"]["control"]["cpu"]
        vb.memory = settings["nodes"]["control"]["memory"]
        if settings["cluster_name"] and settings["cluster_name"] != ""
          vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
        end
    end
    controlplane.vm.provision "shell",
      env: {
        "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
        "ENVIRONMENT" => settings["environment"],
        "KUBERNETES_VERSION" => settings["software"]["kubernetes"],
        "KUBERNETES_VERSION_SHORT" => settings["software"]["kubernetes"][0..3],
        "OS" => settings["software"]["os"]
      },
      path: "scripts/common.sh"
    controlplane.vm.provision "shell",
      env: {
        "CALICO_VERSION" => settings["software"]["calico"],
        "CONTROL_IP" => settings["network"]["control_ip"],
        "POD_CIDR" => settings["network"]["pod_cidr"],
        "SERVICE_CIDR" => settings["network"]["service_cidr"]
      },
      path: "scripts/master.sh"
    # Ensure the default 'vagrant' user's password is set to 'vagrant'
    controlplane.vm.provision "shell", inline: <<-SHELL
      echo "vagrant:vagrant" | chpasswd || true
    SHELL
      # Reboot the VM once provisioning completes (run in background so provisioner doesn't hang)
      controlplane.vm.provision "shell", inline: <<-SHELL
        sudo nohup bash -c 'sleep 15; shutdown -r now' >/dev/null 2>&1 &
      SHELL
  end

  (1..NUM_WORKER_NODES).each do |i|

    config.vm.define "node0#{i}" do |node|
      node.vm.hostname = "node0#{i}"
      node.vm.network "private_network", ip: IP_NW + "#{IP_START + i}"
      if settings["shared_folders"]
        settings["shared_folders"].each do |shared_folder|
          node.vm.synced_folder shared_folder["host_path"], shared_folder["vm_path"]
        end
      end
      node.vm.provider "virtualbox" do |vb|
          vb.cpus = settings["nodes"]["workers"]["cpu"]
          vb.memory = settings["nodes"]["workers"]["memory"]
          if settings["cluster_name"] and settings["cluster_name"] != ""
            vb.customize ["modifyvm", :id, "--groups", ("/" + settings["cluster_name"])]
          end
          # Ensure the VirtualBox VM name matches the hostname for workers
          vb.customize ["modifyvm", :id, "--name", "node0#{i}"]
      end
      node.vm.provision "shell",
        env: {
          "DNS_SERVERS" => settings["network"]["dns_servers"].join(" "),
          "ENVIRONMENT" => settings["environment"],
          "KUBERNETES_VERSION" => settings["software"]["kubernetes"],
          "KUBERNETES_VERSION_SHORT" => settings["software"]["kubernetes"][0..3],
          "OS" => settings["software"]["os"]
        },
        path: "scripts/common.sh"
      node.vm.provision "shell", path: "scripts/node.sh"

      # Ensure the default 'vagrant' user's password is set to 'vagrant' on workers
      node.vm.provision "shell", inline: <<-SHELL
        echo "vagrant:vagrant" | chpasswd || true
      SHELL

      # Only install the dashboard after provisioning the last worker (and when enabled).
      if i == NUM_WORKER_NODES and settings["software"]["dashboard"] and settings["software"]["dashboard"] != ""
        node.vm.provision "shell", path: "scripts/dashboard.sh"
      end
        # Reboot the VM once provisioning completes (run in background so provisioner doesn't hang)
        node.vm.provision "shell", inline: <<-SHELL
          sudo nohup bash -c 'sleep 15; shutdown -r now' >/dev/null 2>&1 &
        SHELL
    end

  end
end 
