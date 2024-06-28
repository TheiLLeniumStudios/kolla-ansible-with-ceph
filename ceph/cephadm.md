## 1. Generate SSH-Key and distribute to all host.
```
# run only on controller001
ssh-keygen -t rsa
ssh-copy-id -f -i ~/.ssh/id_rsa.pub root@ms-controller002
ssh-copy-id -f -i ~/.ssh/id_rsa.pub root@ms-controller003
ssh-copy-id -f -i ~/.ssh/id_rsa.pub root@ms-compute001
ssh-copy-id -f -i ~/.ssh/id_rsa.pub root@ms-compute002
ssh-copy-id -f -i ~/.ssh/id_rsa.pub root@ms-compute003
```

## 2. Install cephadm 
```
# run only on controller001
CEPH_RELEASE=18.2.0 # replace this with the active release
curl --silent --remote-name --location https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm

chmod +x cephadm

./cephadm add-repo --release reef
./cephadm install

```

## 3. Bootstrap ceph cluster
```
cephadm bootstrap --mon-ip 172.16.16.11 --cluster-network 172.20.23.0/24 --log-to-file --allow-overwrite
```

## 4. Install ceph client 
```
cephadm shell
exit
cephadm shell -- ceph -s
cephadm add-repo --release reef
cephadm install ceph-common
```

## 5. Copy ceph public key to all node ceph cluster:
```
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ms-controller002
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ms-controller003
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ms-compute001
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ms-compute002
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ms-compute003
```

## 6. Add additional ceph-mon
```
ceph orch apply mon --unmanaged
ssh ms-controller002 apt install podman -y
ssh ms-controller003 apt install podman -y

ceph orch host add ms-controller002 172.16.16.12 --labels _admin
ceph orch host add ms-controller003 172.16.16.13 --labels _admin

ceph orch daemon add mon ms-controller002:172.16.16.12
ceph orch daemon add mon ms-controller003:172.16.16.13
```

## 7 Add node compute as storage disk server 
```
ssh ms-compute001 apt install podman -y
ssh ms-compute002 apt install podman -y

ceph orch host add ms-compute001 172.16.16.14
ceph orch host add ms-compute002 172.16.16.15

ceph orch device ls
# add disk avaialble as osd
ceph orch daemon add osd *<host>*:*<device-path>*
ex: ceph orch daemon add osd ms-compute001:/dev/sdb
ceph -s

# if we want add disk automaticly
ceph orch apply osd --all-available-devices

```

## 8. Create openstack keyring for openstack usage
```
ceph -s
ceph osd pool create volumes
ceph osd pool set volumes size 2
ceph osd pool set volumes min_size 1

ceph osd pool create images
ceph osd pool set images size 2
ceph osd pool set images min_size 1

ceph osd pool create backups
ceph osd pool set backups size 2
ceph osd pool set backups min_size 1

ceph osd pool create vms
ceph osd pool set vms size 2
ceph osd pool set vms min_size 1

rbd pool init volumes
rbd pool init images
rbd pool init backups
rbd pool init vms

ceph auth get-or-create client.glance mon 'profile rbd' osd 'profile rbd pool=images' mgr 'profile rbd pool=images'
ceph auth get-or-create client.cinder mon 'profile rbd' osd 'profile rbd pool=volumes, profile rbd pool=vms, profile rbd-read-only pool=images' mgr 'profile rbd pool=volumes, profile rbd pool=vms'

ceph auth get client.glance > /etc/ceph/ceph.client.glance.keyring
ceph auth get client.cinder > /etc/ceph/ceph.client.cinder.keyring

ceph osd lspools
```




