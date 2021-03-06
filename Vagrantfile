# -*- mode: ruby -*-
# vi: set ft=ruby :

$common = <<SCRIPT
echo "* Add hosts ..."
echo "192.168.99.101 k8s1.dof.lab k8s1" >> /etc/hosts
echo "192.168.99.102 k8s2.dof.lab k8s2" >> /etc/hosts
echo "192.168.99.103 k8s3.dof.lab k8s3" >> /etc/hosts

echo "* Stop Firewall ..."
systemctl stop firewalld
systemctl disable firewalld

echo "* Change SELinux state ..."
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

echo "* Install Prerequisites ..."
dnf install -y bash-completion wget tc

echo "* Add Docker repository ..."
dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

echo "* Add Kubernetes repository ..."
cat << EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

echo "* Install containerd.io package ..."
dnf install -y https://download.docker.com/linux/centos/7/x86_64/stable/Packages/containerd.io-1.2.13-3.1.el7.x86_64.rpm

echo "* Install Docker and Kubernetes ..."
dnf install -y docker-ce kubeadm kubectl

echo "* Start Docker ..."
systemctl enable docker
systemctl start docker

echo "* Start Kubernetes ..."
systemctl enable kubelet
systemctl start kubelet

echo "* Change some system settings ..."
cat << EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

echo "* Turn off the swap ..."
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

echo "* Add vagrant user to docker group ..."
usermod -aG docker vagrant
SCRIPT

Vagrant.configure(2) do |config|
    
  config.ssh.insert_key = false
  
  # We can use this block if we want all machines to be the same
  #
  # config.vm.provider "virtualbox" do |v|
  #   v.memory = 2048
  #   v.cpus = 2
  # end

  config.vm.define "k8s1" do |k8s1|
    k8s1.vm.provider "virtualbox" do |v|
      v.memory = 2048
      v.cpus = 2
    end
    k8s1.vm.box = "shekeriev/centos-8-minimal"
    k8s1.vm.hostname = "k8s1.dof.lab"
    k8s1.vm.network "private_network", ip: "192.168.99.101"
    k8s1.vm.synced_folder "vagrant/", "/vagrant"
    k8s1.vm.provision "shell", inline: $common
    k8s1.vm.provision "shell", inline: <<EOS

echo "* Initialize Kubernetes cluster ..."
kubeadm init --apiserver-advertise-address=192.168.99.101 --pod-network-cidr 10.244.0.0/16

echo "* Copy configuration for root ..."
mkdir -p /root/.kube
cp -i /etc/kubernetes/admin.conf /root/.kube/config
chown root:root /root/.kube/config

echo "* Copy configuration for vagrant ..."
mkdir -p /home/vagrant/.kube
cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
chown vagrant:vagrant /home/vagrant/.kube/config

echo "* Install POD network plugin (Calico) ..."
kubectl create -f https://docs.projectcalico.org/manifests/tigera-operator.yaml
wget https://docs.projectcalico.org/manifests/custom-resources.yaml -O /tmp/custom-resources.yaml
sed -i 's/192.168.0.0/10.244.0.0/g' /tmp/custom-resources.yaml
kubectl create -f /tmp/custom-resources.yaml

echo "* Install Dashboard ..."
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml

echo "* Create Dashboard admin user ..."
cat << EOF > /vagrant/dashboard-admin-user.yml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
EOF

echo "* Create Dashboard admin user role ..."
cat << EOF > /vagrant/dashboard-admin-role.yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
EOF

echo "* Add both the user and role ..."
kubectl apply -f /vagrant/dashboard-admin-user.yml
kubectl apply -f /vagrant/dashboard-admin-role.yml

echo "* Save the user token ..."
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}') > /vagrant/admin-user-token.txt

echo "* Create custom token ..."
kubeadm token create abcdef.1234567890abcdef

echo "* Save the hash to a file ..."
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //' > /vagrant/hash.txt

EOS
  end

  config.vm.define "k8s2" do |k8s2|
    k8s2.vm.provider "virtualbox" do |v|
      v.memory = 4096
      v.cpus = 2
    end
    k8s2.vm.box = "shekeriev/centos-8-minimal"
    k8s2.vm.hostname = "k8s2.dof.lab"
    k8s2.vm.network "private_network", ip: "192.168.99.102"
    k8s2.vm.synced_folder "vagrant/", "/vagrant"
    k8s2.vm.provision "shell", inline: $common
    k8s2.vm.provision "shell", inline: <<EOS

echo "* Join the worker node (k8s2) ..."
kubeadm join 192.168.99.101:6443 --token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:`cat /vagrant/hash.txt`

EOS
  end
  
  config.vm.define "k8s3" do |k8s3|
    k8s3.vm.provider "virtualbox" do |v|
      v.memory = 4096
      v.cpus = 2
    end
    k8s3.vm.box = "shekeriev/centos-8-minimal"
    k8s3.vm.hostname = "k8s3.dof.lab"
    k8s3.vm.network "private_network", ip: "192.168.99.103"
    k8s3.vm.synced_folder "vagrant/", "/vagrant"
    k8s3.vm.provision "shell", inline: $common
    k8s3.vm.provision "shell", inline: <<EOS

echo "* Join the worker node (k8s3) ..."
kubeadm join 192.168.99.101:6443 --token abcdef.1234567890abcdef --discovery-token-ca-cert-hash sha256:`cat /vagrant/hash.txt`
 
EOS
  end
 
end
