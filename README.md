# SNO with LACP and VLAN
This is a step by step to deploy a SNO lab environment with:

- a Single Node OpenShift
- a link aggregation (LACP)
- a VLAN on the link aggregation (as a second step)
- adding a worker
- deploying LVM on NVMe
- deploying OpenShift Virtualization


## Environment w/o VLAN

```
                             Router with DHCP (192.168.10.0/24)
                                      | (Port1)
                                      |
                                      | (Port1)
                                    Switch (192.168.10.104)
                                      |
            |-------------------------|--------------------------|
          Port2                LACP (Port4,Port8)            LACP (Port3, Port6)
            |                         |                          |
            |                         |                          |
          DNS/LB                     SNO                       Worker
      192.168.10.10             192.168.10.51              192.168.10.61
```

### BOM
- AC1200 Wi-Fi Router
- GS108TV3 Switch
- MeLe Quieter2Q for DNS/LB running CentOS Stream 9
- Beelink SER5 for SNO and Worker 

## dnsmasq to handle internal DNS

The configuration file ```/etc/dnsmasq.conf```
```
domain-needed
bogus-priv
server=1.1.1.1
server=8.8.8.8
local=/beezy.local/
listen-address=127.0.0.1,192.168.10.10

expand-hosts
domain=beezy.local

address=/lb.beezy.local/192.168.10.10
ptr-record=10.10.168.192.in-addr.arpa,lb.beezy.local

address=/bst.beezy.local/192.168.10.50
address=/c01.beezy.local/192.168.10.51
address=/c02.beezy.local/192.168.10.52
address=/c03.beezy.local/192.168.10.53
address=/n01.beezy.local/192.168.10.61
address=/n02.beezy.local/192.168.10.62
address=/n03.beezy.local/192.168.10.63

address=/api.beezy.local/192.168.10.51
ptr-record=51.10.168.192.in-addr.arpa,api.beezy.local

address=/api-int.beezy.local/192.168.10.51
ptr-record=51.10.168.192.in-addr.arpa,api-int.beezy.local

address=/apps.beezy.local/192.168.10.51
address=/console-openshift-console.apps.beezy.local/192.168.10.51
address=/oauth-openshift.apps.beezy.local/192.168.10.51
address=/etcd-0.apps.beezy.local/192.168.10.51

srv-host=_etcd-server-ssl._tcp.beezy.local,etcd-0
```
To enable it: ```systemctl enable --now dnsmasq.service``` 
To check the status: ```systemctl status dnsmasq.service```

Expected output:
```
● dnsmasq.service - DNS caching server.
     Loaded: loaded (/usr/lib/systemd/system/dnsmasq.service; disabled; preset: disabled)
     Active: active (running) since Fri 2024-07-19 14:07:26 CEST; 1h 34min ago
    Process: 2278 ExecStart=/usr/sbin/dnsmasq (code=exited, status=0/SUCCESS)
   Main PID: 2281 (dnsmasq)
      Tasks: 1 (limit: 47720)
     Memory: 596.0K
        CPU: 7ms
     CGroup: /system.slice/dnsmasq.service
             └─2281 /usr/sbin/dnsmasq

Jul 19 14:07:26 lb dnsmasq[2281]: using only locally-known addresses for domain beezy.local
Jul 19 14:07:26 lb dnsmasq[2281]: using nameserver 8.8.8.8#53
Jul 19 14:07:26 lb dnsmasq[2281]: using nameserver 1.1.1.1#53
Jul 19 14:07:26 lb dnsmasq[2281]: reading /etc/resolv.conf
Jul 19 14:07:26 lb systemd[1]: Started DNS caching server..
Jul 19 14:07:26 lb dnsmasq[2281]: using only locally-known addresses for domain beezy.local
Jul 19 14:07:26 lb dnsmasq[2281]: using nameserver 8.8.8.8#53
Jul 19 14:07:26 lb dnsmasq[2281]: using nameserver 1.1.1.1#53
Jul 19 14:07:26 lb dnsmasq[2281]: ignoring nameserver 192.168.10.10 - local interface
Jul 19 14:07:26 lb dnsmasq[2281]: read /etc/hosts - 2 addresses
```

## haproxy to handle the load balancing part

The configuration file ```/etc/haproxy/haproxy.conf```

```
# Global settings
#---------------------------------------------------------------------
global
    maxconn     20000
    log         /dev/log local0 info
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          300s
    timeout server          300s
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 20000

listen stats
    bind :9000
    mode http
    stats enable
    stats uri /

frontend okd4_k8s_api_fe
    bind :6443
    default_backend okd4_k8s_api_be
    mode tcp
    option tcplog

backend okd4_k8s_api_be
    balance source
    mode tcp
    server      c01 192.168.10.51:6443 check

frontend okd4_machine_config_server_fe
    bind :22623
    default_backend okd4_machine_config_server_be
    mode tcp
    option tcplog

backend okd4_machine_config_server_be
    balance source
    mode tcp
    server      c01 192.168.10.51:22623 check

frontend okd4_http_ingress_traffic_fe
    bind :80
    default_backend okd4_http_ingress_traffic_be
    mode tcp
    option tcplog

backend okd4_http_ingress_traffic_be
    balance source
    mode tcp
    server      c01 192.168.10.51:80 check

frontend okd4_https_ingress_traffic_fe
    bind *:443
    default_backend okd4_https_ingress_traffic_be
    mode tcp
    option tcplog

backend okd4_https_ingress_traffic_be
    balance source
    mode tcp
    server      c01 192.168.10.51:443 check
```
To enable it: ```systemctl enable --now haproxy.service``` 
To check the status: ```systemctl status haproxy.service```

Expected output:
```
● haproxy.service - HAProxy Load Balancer
     Loaded: loaded (/usr/lib/systemd/system/haproxy.service; enabled; preset: disabled)
     Active: active (running) since Fri 2024-07-19 13:54:48 CEST; 1h 48min ago
    Process: 1039 ExecStartPre=/usr/sbin/haproxy -f $CONFIG -f $CFGDIR -c -q $OPTIONS (code=exited, status=0/SUCCESS)
   Main PID: 1055 (haproxy)
      Tasks: 5 (limit: 47720)
     Memory: 17.9M
        CPU: 1.384s
     CGroup: /system.slice/haproxy.service
             ├─1055 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -f /etc/haproxy/conf.d -p /run/haproxy.pid
             └─1076 /usr/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -f /etc/haproxy/conf.d -p /run/haproxy.pid

Jul 19 13:54:50 lb haproxy[1076]: [NOTICE]   (1076) : haproxy version is 2.4.22-f8e3218
Jul 19 13:54:50 lb haproxy[1076]: [NOTICE]   (1076) : path to executable is /usr/sbin/haproxy
Jul 19 13:54:50 lb haproxy[1076]: [ALERT]    (1076) : sendmsg()/writev() failed in logger #1: No such file or directory (errno=2)
Jul 19 13:54:50 lb haproxy[1076]: [ALERT]    (1076) : backend 'okd4_k8s_api_be' has no server available!
Jul 19 13:54:50 lb haproxy[1076]: [WARNING]  (1076) : Server okd4_machine_config_server_be/c01 is DOWN, reason: Layer4 timeout, check duration: 2001ms. 0 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Jul 19 13:54:50 lb haproxy[1076]: [ALERT]    (1076) : backend 'okd4_machine_config_server_be' has no server available!
Jul 19 13:54:51 lb haproxy[1076]: [WARNING]  (1076) : Server okd4_http_ingress_traffic_be/c01 is DOWN, reason: Layer4 timeout, check duration: 2001ms. 0 active and 0 backup servers left. 0 sessions active, 0 requeued, 0 remaining in queue.
Jul 19 13:54:51 lb haproxy[1076]: [ALERT]    (1076) : backend 'okd4_http_ingress_traffic_be' has no server available!
Jul 19 13:54:51 lb haproxy[1076]: [WARNING]  (1076) : Server okd4_https_ingress_traffic_be/c01 is DOWN, reason: Layer4 connection problem, info: "No route to host", check duration: 1570ms. 0 active and 0 backup servers left. 0 sessions active,>
Jul 19 13:54:51 lb haproxy[1076]: [ALERT]    (1076) : backend 'okd4_https_ingress_traffic_be' has no server available!
```

## SNO Deployment

### Tooling
Make sure to have on your system:  
* ```oc```, the OpenShift CLI 
* ```openshift-installer```, the OpenShift installer helper

All available here on the [OpenShift Mirror](https://mirror.openshift.com/pub/)

### Prep'ing

Make a directory called ```mycluster``` with another one called ```ocp```: 

```
mkdir -p mycluster/ocp
```

The configuration ```install-config.yaml``` to keep in the root directory for backup and to copy in ```ocp```:
```
apiVersion: v1
baseDomain: beezy.local
compute: 
- hyperthreading: Enabled 
  name: worker
  replicas: 0 
controlPlane: 
  hyperthreading: Enabled 
  name: master
  replicas: 1 
metadata:
  name: sno 
networking:
  clusterNetwork:
  - cidr: 10.128.0.0/14 
    hostPrefix: 23 
  networkType: OVNKubernetes 
  serviceNetwork: 
  - 172.30.0.0/16
platform:
  none: {} 
bootstrapInPlace:
  installationDisk: [/dev/sda|/dev/nvme0n1]
fips: false 
pullSecret: '' 
sshKey: '' 
```
***Note: insert your own ```pullSecret``` and ```sshKey```***

Create the ignition templaes:

```
openshift-install --dir=ocp create single-node-ignition-config
```

Expected output:
``` 
INFO Consuming Install Config from target directory 
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings 
INFO Single-Node-Ignition-Config created in: ocp and ocp/auth
```
Expected folder structure:
```
ls -al ocp/
total 2152
drwxr-xr-x 1 rovandep rovandep     226 Jul 24 09:14 .
drwxr-xr-x 1 rovandep rovandep     242 Jul 24 09:08 ..
drwxr-x--- 1 rovandep rovandep      56 Jul 24 09:14 auth
-rw-r----- 1 rovandep rovandep  288780 Jul 24 09:14 bootstrap-in-place-for-live-iso.ign
-rw-r----- 1 rovandep rovandep     134 Jul 24 09:14 metadata.json
-rw-r--r-- 1 rovandep rovandep   73930 Jul 24 09:14 .openshift_install.log
-rw-r----- 1 rovandep rovandep 1825411 Jul 24 09:14 .openshift_install_state.json
-rw-r----- 1 rovandep rovandep    1717 Jul 24 09:14 worker.ign
```

Get the latest Red Hat CoreOS OpenShift ISO available on the [OpenShift Mirror](https://mirror.openshift.com/pub/)

Then create an alias to the coreos-installer container image to modify the ISO:

```
alias coreos-installer='podman run --privileged --pull always --rm \
-v /dev:/dev -v /run/udev:/run/udev -v $PWD:/data \
-w /data quay.io/coreos/coreos-installer:release'
```

Embed the ignition template in the ISO
```
coreos-installer iso ignition embed -fi ocp/bootstrap-in-place-for-live-iso.ign rhcos-live-sno.x86_64.iso
```

Expected output:
```
Trying to pull quay.io/coreos/coreos-installer:release...
Getting image source signatures
Copying blob 6bb4232ad7af skipped: already exists  
Copying blob 74bb9e361085 skipped: already exists  
Copying blob 0af9cdd8df93 skipped: already exists  
Copying config f0af5149de done   | 
Writing manifest to image destination
```

Burn the ISO on a USB disk - in my case, running Fedora, I am using the Fedora Media Writer app. 
