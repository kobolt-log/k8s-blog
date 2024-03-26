
Vagrant.configure("2") do |config|
  #test  
  config.vm.box = "rockylinux/8"
  # Disk拡張設定を追加
  config.disksize.size = "50GB"

  config.vbguest.auto_update = false
  config.vm.synced_folder "./", "/vagrant", disabled: true

  config.vm.provision :shell, privileged: true, inline: $install_default
  config.vm.define "master-node" do |master|
    master.vm.hostname = "k8s-master"
    master.vm.network "private_network", ip: "192.168.56.30"
	master.vm.provider :virtualbox do |vb|
      vb.memory = 6144
      vb.cpus = 4
	  vb.customize ["modifyvm", :id, "--firmware", "efi"]
	end
    master.vm.provision :shell, privileged: true, inline: $install_master
  end

end

$install_default = <<-SHELL

echo '======== [4] Rocky Linuxの基本設定 ========'
echo '======== [4-1] パッケージの更新 ========'
yum -y update

echo '======== [4-2] タイムゾーンの設定 ========'
timedatectl set-timezone Asia/Seoul

echo '======== [4-3] Disk拡張 / Bug: soft lockup設定を追加========'
yum install -y cloud-utils-growpart
growpart /dev/sda 4
xfs_growfs /dev/sda4
echo 0 > /proc/sys/kernel/hung_task_timeout_secs
echo "kernel.watchdog_thresh = 20" >> /etc/sysctl.conf

echo '======== [4-4] [WARNING FileExisting-tc]: tc not found in system path ログ関連のアップデート ========'
yum install -y yum-utils iproute-tc

echo '======= [4-4] hostsの設定 =========='
cat << EOF >> /etc/hosts
192.168.56.30 k8s-master
EOF

echo '======== [5] kubeadmインストール前の事前作業 ========'
echo '======== [5] ファイアウォール解除 ========'
systemctl stop firewalld && systemctl disable firewalld

echo '======== [5] Swapの無効化 ========'
swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab


echo '======== [6] コンテナランタイムのインストール ========'
echo '======== [6-1] コンテナランタイムインストール前の事前作業 ========'
echo '======== [6-1] iptableの設定 ========'
cat <<EOF |tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF |tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

echo '======== [6-2] コンテナランタイム(containerdインストール) ========'
echo '======== [6-2-1] containerdパッケージのインストール (option2) ========'
echo '======== [6-2-1-1] docker engine インストール ========'
echo '======== [6-2-1-1] repoの設定 ========'
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

echo '======== [6-2-1-1] containerdのインストール ========'
yum install -y containerd.io-1.6.21-3.1.el8
systemctl daemon-reload
systemctl enable --now containerd

echo '======== [6-3] コンテナランタイム：criを有効化 ========'
# defualt cgroupfsからsystemdに変更する。 (kubernetes defaultは systemd)
containerd config default > /etc/containerd/config.toml
sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd



echo '======== [7] kubeadm インストール ========'
echo '======== [7] repo 設定 ========'
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.27/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.27/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF


echo '======== [7] SELinux 設定 ========'
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

echo '======== [7] kubelet, kubeadm, kubectlパッケージのインストール ========'
yum install -y kubelet-1.27.2-150500.1.1.x86_64 kubeadm-1.27.2-150500.1.1.x86_64 kubectl-1.27.2-150500.1.1.x86_64 --disableexcludes=kubernetes
systemctl enable --now kubelet

SHELL



$install_master = <<-SHELL

echo '======== [8] kubeadmでクラスターを作成  ========'
echo '======== [8-1] クラスター初期化 (Pod Networkの設定) ========'
kubeadm init --pod-network-cidr=20.96.0.0/12 --apiserver-advertise-address 192.168.56.30

echo '======== [8-2] kubectlの利用設定 ========'
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

echo '======== [8-3] Pod Networkのインストール (calico) ========'
kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/calico-3.26.4/calico.yaml
kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/calico-3.26.4/calico-custom.yaml

echo '======== [8-4] MasterにPodを生成できるように設定 ========'
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane-


echo '======== [9] Kubernetesの便利機能をインストール ========'
echo '======== [9-1] kubectlの補完スクリプト ========'
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc

echo '======== [9-2] Dashboardのインストール ========'
kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/dashboard-2.7.0/dashboard.yaml

echo '======== [9-3] Metrics Serverのインストール ========'
kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/metrics-server-0.6.3/metrics-server.yaml
SHELL
