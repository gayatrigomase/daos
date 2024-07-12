•	Perform the additional steps on all nodes:
#yum install -y epel-release

•	Copy the following repos to all the servers (storage & client):  
#wget -O /etc/yum.repos.d/daos-packages.repo https://packages.daos.io/v2.4/CentOS7/packages/x86_64/daos_packages.repo 

OR
echo < cat /etc/yum.repos.d/daos-packages.repo
[daos-packages]
name=DAOS v2.4.1 Packages Packages
baseurl=https://packages.daos.io/v2.4.1/EL8/packages/x86_64
enabled=1
gpgcheck=1
protect=1
gpgkey=https://packages.daos.io/RPM-GPG-KEY-2023
Daos server Installation:

•	Install daos-server, daos-devel 
We have configured four daos servers each with 8 NVMe drives.(We have configured daos-admin in one of the daos servers)

#yum install  daos-devel daos daos-server daos-admin

•	Create the daos server configuration file:

Perform network scan with a provider, in this case, we want InfiniBand so ofi_verbs;ofi_rxm is our provider in the server.yml

 

[root@login01 ~]# cat  /etc/daos/daos_server.yml

name: daos_server
access_points: [ip of access point]
provider: [provider] 

port: 10001
transport_config:
  allow_insecure: true
  client_cert_dir: /etc/daos/certs/clients
  ca_cert: /etc/daos/certs/daosCA.crt
  cert: /etc/daos/certs/server.crt
  key: /etc/daos/certs/server.key
control_metadata:
  path: /var/daos/config #md-on-ssd
engines:
- targets: 16
  nr_xs_helpers: 4
  first_core: 0
  log_file: /tmp/daos_engine.0.log
  storage:
  - class: ram
    scm_mount: /mnt/daos0
  - class: nvme
    bdev_roles:
    - wal
    - meta
    bdev_list:
    - 0000:3d:00.0
  - class: nvme
    bdev_roles:
    - data
    bdev_list:
    - 0000:3e:00.0
    - 0000:3f:00.0
    - 0000:40:00.0
    - 0000:60:00.0
    - 0000:61:00.0
    - 0000:62:00.0
    - 0000:63:00.0
  fabric_iface: ib0
  fabric_iface_port: 31416
  pinned_numa_node: 1
disable_vfio: false
disable_vmd: false
enable_hotplug: false
nr_hugepages: 16384
disable_hugepages: false
control_log_mask: INFO
control_log_file: /tmp/daos_server.log
core_dump_filter: 19

#systemctl start daos_server

Certificate Configuration

Generate this on daos storage server and copy it on all the client nodes
#/usr/lib64/daos/certgen/gen_certificates.sh

admin certificates on each admin node:

chown daos_agent:daos_agent /etc/daos/certs/agent.\*
chown $USER:$USER /etc/daos/certs/admin.*

client certificates on each client node:

chown $USER:$USER /etc/daos/certs/daosCA.crt
chown daos_agent:daos_agent /etc/daos/certs/agent.*

server certificates on each server node:

chown daos_server:daos_server /etc/daos/certs/daosCA.crt
chown daos_server:daos_server /etc/daos/certs/server.*
chown daos_server:daos_server /etc/daos/certs/clients/agent.crt
chown daos_server:daos_server /etc/daos/certs/clients
 
Client Installation:
•	Install the DAOS client RPMs on the client and admin nodes:
       #yum install -y daos-client
[root@login01 daos]# cat /etc/daos/daos_control.yml
name: daos_server
port: 10001
hostlist: [access point ip]
transport_config:
    allow_insecure: true
    ca_cert: /etc/daos/certs/daosCA.crt
    cert: /etc/daos/certs/admin.crt
    key: /etc/daos/certs/admin.key

#systemctl start daos_agent
Admin Installation:
•	Install the DAOS client RPMs on admin nodes
#yum install daos-admin -y 
[root@login01 daos]# cat /etc/daos/daos_agent.yml
name: daos_server
access_points: [access point ip]
port: 10001
transport_config:
    allow_insecure: true
    ca_cert: /etc/daos/certs/daosCA.crt
    cert: /etc/daos/certs/agent.crt
    key: /etc/daos/certs/agent.key
log_file: /tmp/daos_agent.log

When you run dmg system query -v, the DAOS Management tool performs a query on the DAOS system to retrieve the current status and configuration details. 
 
dmg storage query usage, the command queries the DAOS system to retrieve details about storage usage.
 

To create the Pool and Container
#dmg pool create -r0 -z 90 % daos_pool

 
#daos cont create daos_pool App_test1 –type=POSIX
 

Mount the container to the client node

#dfuse --singlethread --multi-user --pool=daos_pool --container=App_test -mountpoint=(mount point)
Note: Unless you use the multi-user mode, the fuse mount point is only accessible to the user that mounted it.
Also make sure that when you submit the job, your current directory from which submit is not in DAOS, Because Slurm will fail to change into that directory on the compute nodes at job startup.
Umount the container from the client node

#fusermount3 -u (mount point)
