yum install -y git

git clone git@github.com:atwin140/OKD-Install.git

# OKD-Install
1 Master 4 Node cluster


# ESXi cloneing disk commands

# After creating a master vm (I used this to become my bastion host) update and install users, 
ansible-playbook bootstrap.Centos.yml -l OKD 

# Build the master, infra, and app nodes but do not create HDD with the vm. we will copy that in next.

# Shutdown Bastian host.

# ssh to the ESXi server and run these commands to copy the master.vmdk to all the other VMs.

vmkfstools -i /vmfs/volumes/{DataStore}/{MasterVM}/{MasterVM}.vmdk /vmfs/volumes/{DataStore}/master/master.vmdk

# examples:
vmkfstools --deletevirtualdisk /vmfs/volumes/SP_1TB/master/master.vmdk; 
vmkfstools --deletevirtualdisk /vmfs/volumes/SP_1TB/infra01/infra01.vmdk; 
vmkfstools --deletevirtualdisk /vmfs/volumes/SP_1TB/infra02/infra02.vmdk; 
vmkfstools --deletevirtualdisk /vmfs/volumes/SP_1TB/app01/app01.vmdk ; 
vmkfstools --deletevirtualdisk /vmfs/volumes/SP_1TB/app02/app02.vmdk; 
vmkfstools -i /vmfs/volumes/120_SSD/spirit/spirit.vmdk /vmfs/volumes/SP_1TB/master/master.vmdk
vmkfstools -i /vmfs/volumes/120_SSD/spirit/spirit.vmdk /vmfs/volumes/SP_1TB/infra01/infra01.vmdk
vmkfstools -i /vmfs/volumes/120_SSD/spirit/spirit.vmdk /vmfs/volumes/SP_1TB/infra02/infra02.vmdk 
vmkfstools -i /vmfs/volumes/120_SSD/spirit/spirit.vmdk /vmfs/volumes/SP_1TB/app01/app01.vmdk 
vmkfstools -i /vmfs/volumes/120_SSD/spirit/spirit.vmdk /vmfs/volumes/SP_1TB/app02/app02.vmdk 

# From ESXi manager add the disks you just copied to each host.
	# - Add hdd
	# - Boot
	# - Configure Network
	    vi /etc/sysconfig/network-scripts/ifcfg-ens192.  #ens-192 is the adaptor id and can change
	    or
		nmtui
		systemctl restart network

# From the Bastian in the OKD-Intall directory
  - Check the all the hosts are working
      ansible nodes -a hostname

# pre install
yum install -y  wget git zile nano net-tools docker-1.13.1\
				bind-utils iptables-services \
				bridge-utils bash-completion \
				kexec-tools sos psacct openssl-devel \
				httpd-tools NetworkManager \
				python-cryptography python2-pip python-devel  python-passlib \
				java-1.8.0-openjdk-headless "@Development Tools"

# install epel
yum -y install epel-release

# Disable the EPEL repository globally so that is not accidentally used during later steps of the installation
sed -i -e "s/^enabled=1/enabled=0/" /etc/yum.repos.d/epel.repo

yum -y --enablerepo=epel install pyOpenSSL

# Install ansible 2.6.5
curl -o ansible.rpm https://releases.ansible.com/ansible/rpm/release/epel-7-x86_64/ansible-2.6.5-1.el7.ans.noarch.rpm
yum -y --enablerepo=epel install ansible.rpm

# Clone the opeshift-ansible repo for 3.11
git clone https://github.com/openshift/openshift-ansible.git -b release-3.11 --depth=1


ansible masters -m shell -a "{mkdir -p /etc/origin/master/; touch /etc/origin/master/htpasswd}"
mkdir -p /etc/origin/master/
touch /etc/origin/master/htpasswd


ansible-playbook dockerInstall.yml


ansible-playbook openshift-ansible/playbooks/prerequisites.yml

ansible-playbook dockerPrePull.yml

# Shutdown all the systems and take a snapshot
ansible nodes -a 'shutdown now'

# From the ESXi server via ssh

vim-cmd vmsvc/getallvms | awk '/master|infra|app/ {print "vim-cmd vmsvc/snapshot.create "$1" PreDeploy  # Name-"$2}'
vim-cmd vmsvc/snapshot.create 3 PreDeploy  # Name-master
vim-cmd vmsvc/snapshot.create 4 PreDeploy  # Name-infra01
vim-cmd vmsvc/snapshot.create 5 PreDeploy  # Name-infra02
vim-cmd vmsvc/snapshot.create 6 PreDeploy  # Name-app01
vim-cmd vmsvc/snapshot.create 7 PreDeploy  # Name-app02

# This will create a list of commands to run copy and paste.


# Turn on all vms

vim-cmd vmsvc/getallvms | awk '/master|infra|app/ {print "vim-cmd vmsvc/power.on "$1" # Name-"$2}'


# Run the deploy from the Bastion host in the OKD-Install directory
ansible-playbook openshift-ansible/playbooks/deploy_cluster.yml


# From the master add the users
htpasswd -b /etc/origin/master/htpasswd ${USERNAME} ${PASSWORD}
oc adm policy add-cluster-role-to-user cluster-admin ${USERNAME}




# ESXi clean up if needed vim-cmd /vmsvc/unregister <id> 

vim-cmd vmsvc/getallvms | awk '/master|infra|app/ {print "vim-cmd /vmsvc/unregister "$1" # Name-"$2}'














