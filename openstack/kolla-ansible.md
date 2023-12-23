Install package dependency for Openstack
```
apt-get install python3-dev libffi-dev gcc libssl-dev python3-selinux python3-setuptools python3-venv -y
```

Create virtaul environment
```
mkdir openstack
cd ~/openstack
python3 -m venv os-venv
source os-venv/bin/activate
```

Upgrade pip, install ansible & install kolla-ansible
```
pip install -U pip
pip install 'ansible>=6,<8'
pip install git+https://opendev.org/openstack/kolla-ansible@master
```

Install ansible galaxy 
```
kolla-ansible install-deps
```
Copy openstack configuration from kolla directory
```
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla

cp -r os-venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
cp os-venv/share/kolla-ansible/ansible/inventory/* .
```

Create ansible configuration tune
```
mkdir -p /etc/ansible
nano /etc/ansible/ansible.cfg
---
[defaults]
host_key_checking=False
pipelining=True
forks=100
---
```

Edit multinode file 
```
nano multinode
---
[control]

ms-controller001
ms-controller002
ms-controller003

[network:children]
compute

[compute]
ms-compute001
ms-compute002
ms-compute003

[monitoring]
ms-controller001
ms-controller002
ms-controller003

[storage]
ms-controller001
ms-controller002
ms-controller003

[deployment]
localhost       ansible_connection=local
```

Test connectivity kolla-ansible to all node
```
ansible -i multinode all -m ping
```

Generate password for openstack services
```
kolla-genpwd
```

Edit kolla globals.yml
```
nano /etc/kolla/globals.yml
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "master"
kolla_internal_vip_address: "172.18.1.100"
kolla_internal_fqdn: "internal.xyz.local"
kolla_external_vip_address: "172.18.0.100"
kolla_external_fqdn: "public.xyz.local"
neutron_external_interface: "ens6"
enable_neutron_provider_networks: "yes"
enable_haproxy: "yes"
enable_keystone: "yes"
enable_horizon: "yes"
enable_openstack_core: "yes"
enable_cinder: "yes"
enable_cinder_backup: "no"
enable_fluentd: "no"
enable_masakari: "yes"
enable_masakari_instancemonitor: "yes"
enable_masakari_hostmonitor: "yes"
enable_hacluster: "yes"
external_ceph_cephx_enabled: "yes"
cinder_backend_ceph: "yes"
ceph_cinder_keyring: "ceph.client.cinder.keyring"
ceph_cinder_user: "cinder"
ceph_cinder_pool_name: "volumes"
glance_backend_ceph: "yes"
ceph_glance_keyring: "ceph.client.glance.keyring"
ceph_glance_user: "glance"
ceph_glance_pool_name: "images"
nova_backend_ceph: "yes"
ceph_nova_keyring: "ceph.client.cinder.keyring"
ceph_nova_user: "cinder"
ceph_nova_pool_name: "vms"
kolla_enable_tls_internal: "yes"
kolla_enable_tls_external: "yes"
kolla_copy_ca_into_containers: "yes"
kolla_enable_tls_backend: "yes"
openstack_cacert: "/etc/ssl/certs/ca-certificates.crt"
kolla_external_vip_interface: "ens3"
api_interface: "ens4"
tunnel_interface: "ens5"
neutron_plugin_agent: "openvswitch"
enable_neutron_dvr: "yes"
```

Copy ceph keyring to kolla config directory
```
mkdir -p /etc/kolla/config/cinder/cinder-volume/
mkdir /etc/kolla/config/nova/
mkdir /etc/kolla/config/glance/

cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-volume/
cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/nova/
cp /etc/ceph/ceph.client.glance.keyring /etc/kolla/config/glance/

* note : remove tab space in ceph.conf 

cp /etc/ceph/ceph.conf /etc/kolla/config/cinder/
cp /etc/ceph/ceph.conf /etc/kolla/config/glance/
cp /etc/ceph/ceph.conf /etc/kolla/config/nova/
```

Execute deployment command 
```
kolla-ansible -i ./multinode certificates
kolla-ansible -i ./multinode bootstrap-servers
kolla-ansible -i ./multinode prechecks
kolla-ansible -i ./multinode deploy

kolla-ansible -i ./multinode post-deploy

pip3 install openstackclient

echo "export OS_CACERT=/etc/ssl/certs/ca-certificates.crt" | tee -a /etc/kolla/admin-openrc.sh
cat /etc/kolla/certificates/ca/root.crt | sudo tee -a /etc/ssl/certs/ca-certificates.crt
```

To support vlan for provider network, change neutron-server config to add network_vlan_ranges
```
vim /etc/kolla/neutron-server/ml2_conf.ini
---
[ml2_type_vlan]
network_vlan_ranges = physnet1:100:1110
---
# restart neutron_server on all controller
docker restart neutron_server
```

