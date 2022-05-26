# Manually install a 3-node Ceph cluster

## Hardening CentOS 7

Run hardening on all nodes of cluster

- YUM update first
- Modify `/etc/hosts` and `/etc/resolv.conf`

```bash
echo "192.168.0.101 node1" >> /etc/hosts
echo "192.168.0.102 node2" >> /etc/hosts
echo "192.168.0.103 node3" >> /etc/hosts
echo "nameserver 8.8.8.8" >> /etc/resolv.conf
```

- Disable `selinux`, `firewalld` and ipv6

```bash
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
systemctl stop firewalld
systemctl disable firewalld
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf
sysctl -p
```

## Get packages

- Add key

```bash
rpm --import 'https://download.ceph.com/keys/release.asc'
```

- Create a ceph repo file `/etc/yum.repos.d/ceph.repo`

```
[ceph]
name=Ceph packages for $basearch
baseurl=https://download.ceph.com/rpm-octopus/el7/$basearch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-noarch]
name=Ceph noarch packages
baseurl=https://download.ceph.com/rpm-octopus/el7/noarch
enabled=1
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=https://download.ceph.com/rpm-octopus/el7/SRPMS
enabled=0
priority=2
gpgcheck=1
gpgkey=https://download.ceph.com/keys/release.asc
```

- Add EPEL repo

```bash
yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
```

- Install pre-requisite packages and then install ceph

```bash
yum update -y
yum install -y yum-plugin-priorities snappy leveldb gdisk python-argparse gperftools-libs
yum install -y ceph
```

## Setup ceph cluster

Replace `<cluster_name>` with the name of ceph cluster, default cluster name is "ceph".

These steps were done on the initial monitor node - node1

- Generate ceph config file `/etc/ceph/<cluster_name>.conf` 

```bash
touch /etc/ceph/<cluster_name>.conf
echo "[global]" > /etc/ceph/<cluster_name>.conf
echo "fsid = `uuidgen`" >> /etc/ceph/<cluster_name>.conf
echo "mon initial members = node1,node2,node3" >> /etc/ceph/<cluster_name>.conf
echo "192.168.0.101,192.168.0.102,192.168.0.103" >> /etc/ceph/<cluster_name>.conf
```

Example of `/etc/ceph/ceph.conf`

```
[global]
fsid = 1ccdcab3-6a20-400b-bee2-7448b9643334
mon initial members = node1,node2,node3
mon host = 192.168.0.101,192.168.0.102,192.168.0.103
```

- Create the keyring for the cluster and generate the Monitor secret key. Monitors communicate with each other via a secret key

```bash
ceph-authtool --create-keyring /etc/ceph/<cluster_name>.mon.keyring --gen-key -n mon. --cap mon 'allow *'
```

- Generate an administrator keyring, generate a `client.admin` user and add the user to the keyring to use `ceph` CLI tools

```bash
ceph-authtool --create-keyring /etc/ceph/<cluster_name>.client.admin.keyring --gen-key -n client.admin --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow *' --cap mgr 'allow *'
```

- Add the `<cluster_name>.client.admin.keyring` key to the `<cluster_name>.mon.keyring`

```bash
ceph-authtool /etc/ceph/<cluster_name>.mon.keyring --import-keyring /etc/ceph/<cluster_name>.client.admin.keyring
```

- Generate the Monitor map. Specify using the node name, IP address and the `fsid`, of the initial Monitor and save it as `/etc/ceph/monmap`

```bash
FSID=$(cat /etc/ceph/ceph_manual.conf | grep fsid | awk '{print $3}')
monmaptool --create --add node1 192.168.0.101 --fsid $FSID /etc/ceph/monmap
```

- On the initial Monitor node, create a default data directory

```bash
mkdir /var/lib/ceph/mon/<cluster_name>-node1
```

- Populate the initial Monitor daemon with the Monitor map and keyring

```bash
ceph-mon --cluster <cluster_name> --mkfs -i node1 --monmap /etc/ceph/monmap --keyring /etc/ceph/<cluster_name>.mon.keyring
```

- Update the owner and group permissions on the newly created directories

```bash
chown -R ceph:ceph /var/lib/ceph
chown -R ceph:ceph /var/log/ceph
chown -R ceph:ceph /var/run/ceph
chown -R ceph:ceph /etc/ceph
```

- For storage clusters with custom names, add the the following line

```bash
echo "CLUSTER=<custom_cluster_name>" >> /etc/sysconfig/ceph
```

- Start and enable the ceph-mon process on the initial Monitor node

```bash
systemctl enable ceph-mon.target
systemctl enable ceph-mon@node1
systemctl start ceph-mon@node1
```

- Verify the monitor daemon is running

```bash
systemctl status ceph-mon@node1
```

### Add a monitor node

These steps were done on the initial node to setup on node2 (do the same on node3)

- Setup ssh key pair

```bash
ssh-keygen -t rsa
ssh-copy-id root@node2
```

- Add node2 to monmap

```bash
monmaptool --add node2 192.168.0.102 --fsid $FSID /etc/ceph/monmap
```

- Copy config and keyring to node2

```bash
scp /etc/ceph/<cluster_name>.conf node2:/etc/ceph
scp /etc/ceph/<cluster_name>.mon.keyring node2:/etc/ceph/<cluster_name>.mon.keyring
scp /etc/ceph/monmap node2:/etc/ceph
```

- Populate the Monitor daemon with the Monitor map and keyring

```bash
ssh node2 "ceph-mon --cluster <cluster_name> --mkfs -i node2 --monmap /etc/ceph/monmap --keyring /etc/ceph/<cluster_name>.mon.keyring"
```

- Update the owner and group permissions on the newly created directories

```bash
ssh node2 "chown -R ceph:ceph /etc/ceph /var/lib/ceph /var/log/ceph /var/run/ceph"
```

- Start monitor daemon

```bash
ssh node2 'echo "CLUSTER=<custom_cluster_name>" >> /etc/sysconfig/ceph'
ssh node2 "systemctl enable ceph-mon@node2"
ssh node2 "systemctl start ceph-mon@node2"
```

- Verify the monitor daemon is running

```bash
ssh node2 "systemctl status ceph-mon@node2"
```

After setting up, check cluster health with command `ceph -s --cluster <cluster_name>`
