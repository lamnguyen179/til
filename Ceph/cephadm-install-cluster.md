# Install 3-node ceph cluster using cephadm

## Hardening Ubuntu 20.04

- Config static IP addresses and DNS Server
- Disable IPv6, ufw

Add to the end of line `GRUB_CMDLINE_LINUX_DEFAULT` with `ipv6.disable=1`

Disable ufw with command `ufw disable`

- Install docker ubuntu

```bash
apt update
apt install -y ca-certificates curl gnupg lsb-release
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

## Install octopus ceph cluster

Running on the initial node: `node1`

- Install `cephadm`

```bash
curl --silent --remote-name --location https://github.com/ceph/ceph/raw/octopus/src/cephadm/cephadm
chmod +x cephadm
./cephadm add-repo --release octopus
./cephadm install
```

- Bootstrap ceph cluster

```bash
cephadm prepare-host
cephadm --image ceph/ceph:v16.2 bootstrap --mon-ip 192.168.0.101 --cluster-network 192.168.26.0/24 --allow-mismatched-release
```

- Enable ceph CLI

```bash
cephadm add-repo --release octopus
cephadm install ceph-common
```

## Add Monitors

- Add host

```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@node3
ceph orch host add node2 192.168.0.102 --labels _admin
ceph orch host add node3 192.168.0.103 --labels _admin
```

- Deploy monitors

```bash
ceph orch apply mon --unmanaged
ceph orch daemon add mon node2:192.168.0.102
ceph orch daemon add mon node3:192.168.0.103
```

## Add OSDs

- Check local disk status

```bash
ceph orch device ls --wide --refresh
```

- Add OSD if disk available

```bash
ceph orch daemon add osd node1:/dev/sdb
ceph orch daemon add osd node2:/dev/sdb
ceph orch daemon add osd node3:/dev/sdb
```

## Add RGWs

- Deploy `DESIGNATED GATEWAYS`

```bash
ceph orch host label add node2 rgw
ceph orch host label add node3 rgw
ceph orch apply rgw rgw_test '--placement=label:rgw count-per-host:1' --port=8000
```

- Create user to view use on Ceph Dashboard. Dump access_key and secret_key to file and config Dashboard

```bash
radosgw-admin user create --uid=user_test --display-name=user_test --system
radosgw-admin user info --uid=user_test | grep access_key
radosgw-admin user info --uid=user_test | grep secret_key
ceph dashboard set-rgw-api-access-key -i ACCESS_KEY_FILE
ceph dashboard set-rgw-api-secret-key -i SECRET_KEY_FILE
```
