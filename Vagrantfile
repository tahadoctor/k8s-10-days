 Vagrant.configure("2") do |config|
 # ARM64 Ubuntu box with vmware_desktop provider (Apple Silicon)
  BOX = "bento/ubuntu-22.04"

  MASTER_HOSTNAME = "master"
  WORKER_COUNT    = 3

  # Kubernetes version can be pinned if needed
  KUBE_VERSION    = ""  # e.g., "=1.29.6-1.1" or leave ""

  # Pod CIDR compatible with Calico default
  POD_CIDR        = "192.168.0.0/16"

  # Common bootstrap for all nodes
  COMMON_SCRIPT = <<-SHELL
    set -euo pipefail

    # Basic deps
    apt-get update
    apt-get install -y curl gnupg2 ca-certificates apt-transport-https software-properties-common

    # Disable swap now and on reboot
    swapoff -a
    sed -i.bak '/\\sswap\\s/s/^/#/' /etc/fstab
    (crontab -l 2>/dev/null; echo "@reboot /sbin/swapoff -a") | crontab - || true

    # Install containerd (via Docker repo for consistent containerd.io)
    if ! test -f /etc/apt/keyrings/docker.gpg; then
      install -m 0755 -d /etc/apt/keyrings
      curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
      chmod a+r /etc/apt/keyrings/docker.gpg
    fi
    echo "deb [arch=arm64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" > /etc/apt/sources.list.d/docker.list
    apt-get update
    apt-get install -y containerd.io

    # Generate default config.toml and enable systemd cgroup
    mkdir -p /etc/containerd
    containerd config default | tee /etc/containerd/config.toml >/dev/null
    sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml

    # Ensure CRI plugin is enabled (remove disabled_plugins line if present)
    sed -i '/disabled_plugins = \\["cri"\\]/d' /etc/containerd/config.toml

    systemctl restart containerd
    systemctl enable containerd

    # Install Kubernetes (pkgs.k8s.io repo)
    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' > /etc/apt/sources.list.d/kubernetes.list

    apt-get update
    apt-get install -y kubelet#{KUBE_VERSION} kubeadm#{KUBE_VERSION} kubectl#{KUBE_VERSION}
    apt-mark hold kubelet kubeadm kubectl

    systemctl daemon-reload
    systemctl enable --now kubelet || true

    echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-k8s.conf
    sysctl --system
  SHELL

  # Master-specific script: init, Calico download/apply (first time), kubeconfig export
  MASTER_SCRIPT = <<-SHELL
    set -euo pipefail

    MASTER_IP=$(ip -4 addr show dev eth0 | awk '/inet /{print $2}' | cut -d/ -f1 | head -n1)

    if [ ! -f /etc/kubernetes/admin.conf ]; then
      kubeadm init --apiserver-advertise-address=${MASTER_IP} --pod-network-cidr=#{POD_CIDR} --v=5
    fi

    # kubeconfig for vagrant and root
    sudo -u vagrant mkdir -p ~vagrant/.kube
    cp -i /etc/kubernetes/admin.conf ~vagrant/.kube/config
    chown vagrant:vagrant ~vagrant/.kube/config
    mkdir -p /root/.kube
    cp -i /etc/kubernetes/admin.conf /root/.kube/config

    # Prepare shared configs directory
    mkdir -p /vagrant/configs
    cp -f /etc/kubernetes/admin.conf /vagrant/configs/config

    # Download Calico manifest for manual or first-time apply
    curl -fsSL https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml -o /vagrant/configs/calico.yaml

    # Apply Calico once on first init (you can re-apply manually too)
    if ! sudo -u vagrant kubectl get ns calico-system >/dev/null 2>&1; then
      sudo -u vagrant kubectl apply -f /vagrant/configs/calico.yaml
    fi
  SHELL

  # Worker-specific script: no auto-join; just prepared for manual join
  WORKER_SCRIPT = <<-SHELL
    set -euo pipefail
    echo "Worker is prepared. Run the kubeadm join command manually from the master."
  SHELL

  # Global defaults
  config.vm.box = BOX
  config.vm.synced_folder ".", "/vagrant", type: "rsync"

  # Master node
  config.vm.define MASTER_HOSTNAME do |m|
    m.vm.hostname = MASTER_HOSTNAME
    m.vm.network "private_network", type: "dhcp"
    m.vm.provider "vmware_desktop" do |v|
      v.memory = 4096
      v.cpus   = 2
    end
    m.vm.provision "shell", inline: COMMON_SCRIPT
    m.vm.provision "shell", inline: MASTER_SCRIPT, run: "always"
  end

  # Worker nodes
  (1..WORKER_COUNT).each do |i|
    name = "worker-#{i}"
    config.vm.define name do |w|
      w.vm.hostname = name
      w.vm.network "private_network", type: "dhcp"
      w.vm.provider "vmware_desktop" do |v|
        v.memory = 2048
        v.cpus   = 2
      end
      w.vm.provision "shell", inline: COMMON_SCRIPT
      w.vm.provision "shell", inline: WORKER_SCRIPT, run: "always"
    end
  end
end
