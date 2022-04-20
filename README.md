# IBM PowerVM NovaLink i PowerVC 2.0 Workshop #

Please change labX string to your lab seat number.

## Take a look on our lab environment ##

### Login to Novalink using ssh as neo user ###
```
pvmctl sys list
pvmctl vm list
pvmctl scsi list
pvmctl vea list
pvmctl repo list
pvmctl io list
```

### Login to PowerVC using ssh as neo user ###
```
openstack hypervisor list
openstack server list
openstack server show 5437aix1
openstack network list
openstack subnet list
openstack image list
openstack flavor list
openstack volume list
openstack volume type list
```

# Novalink installation using Ubuntu Linux iso and Novalink internet repo (co-managed) #

## Create VIOS partitions and install VIOSs ##

## Create Novalink partition using HMC ##
```
chsyscfg -m S824-5437 -r lpar -o apply --id 3 -n default_profile
chcomgmt -m S824-5437 -o setcontroller -t norm ​
chsyscfg -m S824-5437 -r lpar -i lpar_id=3,powervm_mgmt_capable=1
```

## Install Ubuntu 18.04 LTS ##
1. Create neo user during installation
2. RAID1 for OS

## Set controller to Novalink ##
`chcomgmt -o relcontroller -m S824-5437`

## Run Novalink installation commands as root ##
```
apt-get install --yes openssh-server net-tools gnupg

cat > /etc/apt/sources.list <<EOF
deb http://us.ports.ubuntu.com/ubuntu-ports/ bionic main restricted
deb http://us.ports.ubuntu.com/ubuntu-ports/ bionic-updates main restricted
deb http://us.ports.ubuntu.com/ubuntu-ports/ bionic universe
deb http://us.ports.ubuntu.com/ubuntu-ports/ bionic-updates universe
deb http://us.ports.ubuntu.com/ubuntu-ports/ bionic multiverse
deb http://us.ports.ubuntu.com/ubuntu-ports/ bionic-updates multiverse
deb http://us.ports.ubuntu.com/ubuntu-ports/ bionic-backports main restricted universe multiverse
EOF

cat > /etc/apt/sources.list.d/pvm.list <<EOF
deb http://public.dhe.ibm.com/systems/virtualization/Novalink/debian/ novalink_2.0.2.1 non-free optional
EOF

wget http://public.dhe.ibm.com/systems/virtualization/Novalink/debian/novalink-gpg-pub.key 
apt-key add novalink-gpg-pub.key

apt-get update

ln -s /usr/bin/python3 /usr/bin/python

apt-get install --yes pvm-cli pvm-core pvm-core-5250 pvm-novalink pvm-rest-app pvm-rest-server pypowervm
usermod -G pvm_admin neo
sed -i "s/\(root\)\(.*ALL\)$/\1\2\nneo\2/" /etc/sudoers

ls /sys/class/net/ibmveth6
if [ $? -ne 0 ]; then
    cat -<< EOF >> /etc/netplan/01-netcfg.yaml
    ibmveth3:
      addresses: [ 192.168.128.1/17 ]
EOF
else
    cat -<< EOF >> /etc/netplan/01-netcfg.yaml
    ibmveth6:
      addresses: [ 192.168.128.1/17 ]
EOF
fi

add-apt-repository ppa:ibmpackages/rsct
apt-get update
apt-get install src rsct.core.utils rsct.core
reboot
```

# IBM PowerVC installation on RHEL 8.4 ppc64le #

## Add host entry in hosts file ##
`IP_ADDRESS FQDN SHORT_NAME`

## Enable addtional repos ##
```
yum repolist all 
subscription-manager repos --enable=rhel-8-for-ppc64le-highavailability-rpms
subscription-manager repos --enable=rhel-8-for-ppc64le-supplementary-rpms
subscription-manager repos --enable=ansible-2.9-for-rhel-8-ppc64le-rpms
```

## Install and run tmux ##
```
yum install tmux
tmux
```

## Run PowerVC installation ##
```
export HOST_INTERFACE=env2
tar -xvf powervc-opsmgr-rhel-ppcle-2.0.2.1.tgz 
cd powervc-opsmgr-2.0.2.1/
./setup_opsmgr.sh 
./powervc-opsmgr inventory -c iiclab
powervc-opsmgr inventory -c 5437s824
powervc-opsmgr install -c 5437s824
```

## Apply all latest iFixes ##
```
powervc-opsmgr apply-ifix --ifix /root/IT39534_IT39712-2.0.2.1 -c 5437s824
```

# Useful Novalink commands #
```
pvmctl lpar create --name 5437s824labX --proc-type shared --proc 1 --sharing-mode uncapped --type AIX/Linux --mem 8192 --proc-unit 1
pvmctl lpar update -i name=5437s824labX --set-fields \
SharedProcessorConfiguration.min_units=0.1 \
SharedProcessorConfiguration.max_units=2 \
SharedProcessorConfiguration.desired_units=1 \
SharedProcessorConfiguration.min_virtual=2 \
SharedProcessorConfiguration.max_virtual=6 \
SharedProcessorConfiguration.desired_virtual=4 \
PartitionMemoryConfiguration.max=32768 \
PartitionMemoryConfiguration.min=1024 \
PartitionMemoryConfiguration.desired=32768

pvmctl vea list
pvmctl vea create --pvid=100 --vswitch=ETHERNET0 --parent-id name=5437s824pvcbis
pvmctl vea list

pvmctl scsi list
pvmctl scsi create --type lv --lpar name=powervc001 --stor-id name=pvc2lv --parent-id name=vios03
pvmctl scsi list

pvmctl repo list
pvmctl repo create -sp rootvg -size 20G --parent-id name=vios03
pvmctl vom upload --vios vios03 --data <path to .iso image of Red Hat 8.3> --name rhel83 
pvmctl repo list
pvmctl scsi create --type vopt --lpar name=powervc001 --stor-id rhel83 -p name=vios03

mkvterm -p powervc001

pvmctl io attach -p id=1 --drc-names U78C9.001.WZS01BP-P1-C9

pvmctl media upload --name centos73le --vios 6018s822l-v1 --data CentOS-7-AltArch-ppc64le-Minimal-1611.iso

```

