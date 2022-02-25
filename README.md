# Ceph 5 Install
### Demo video on YouTube [here](https://youtu.be/phvRNOtxNkM)

During this install, I've configured _Ceph 5 on VMware 6.7U3_ using 4 VM's. This first node is **cephadm**, this node is used for configuring and managing your Ceph cluster. The next three VM's are specifically for the OSD's and will be labeled for services to support high availablity.

1. cephadm
    - 2 vCPU
    - 4 GB Memory
    - 36 GB HD
      - /var, 15g, ext2
      - /, 15g, ext2
      - /boot/efi, 2g
      - swap, 4g
    - Network (172/26, 10g)
2. ceph[0,1,2]
    - 6 vCPU
    - 24 GB Memory
    - 36 GB HD
      - /var, 15g, ext2
      - /, 15g, ext2
      - /boot/efi, 2g
      - swap, 4g
    - Network (172/26, 10g)
    - 100 GB, /dev/sdb (unpartitioned)
    - 100 GB, /dev/sdc (unpartitioned)

```
#### Note: Setting the filesystems are using ext2 to disable the overhead of disk journaling and is a recommend tune for all Ceph installs.
```

### Let's get this party started!

1. Access all of your nodes, patch.
    - dnf update -y
    - reboot
3. Install packages on **cephadm**
    - dnf install ceph podman -y
4. Install packages on **ceph[0,1,2]**
    - dnf install podman -y
5. Sync root's ssh_key for passwordless login
    - On **cephadm** switch to root `sudo su -`
    - ssh-keygen
    - ssh-copy-id root@[all_nodes]
6. Disable TCP timestamps
    - echo -n "net.ipv4.tcp_timestamps=0" > /etc/sysctl.d/99-ceph.conf
    - scp /etc/sysctl.d/99-ceph.conf ceph[0,1,2]:/etc/sysctl.d/99-ceph.conf
7. Add hosts to the /etc/hosts file, and replicate to each node
    - vi /etc/hosts
    - scp /etc/hosts ceph[0,1,2]:/etc/hosts
8. ✨ Bootstrap! ✨
    - cephadm bootstrap --mon-ip 172.16.1.39 `cephadm's IP address`
9. Add the cephadm public key to all nodes.
    - ceph cephadm get-pub-key > ~/ceph.pub
    - ssh-copy-id -f -i ~/ceph.pub root@ceph[0,1,2]
10. Add OSD nodes
    - ceph orch host add ceph0 172.16.1.40
    - ceph orch host add ceph1 172.16.1.41
    - ceph orch host add ceph2 172.16.1.42
11. Label the nodes for Ceph services
    - ceph orch host label add cephadm [_admin,mon,mgr]
    - ceph orch host label add ceph[0,1] mon
    - ceph orch host label add ceph[1,2] mgr
#### Optional: Add services directly to the nodes `ceph orch apply mon --placement "ceph0,ceph1"`
12. Apply the services
    - ceph orch apply mon label:mon
    - ceph orch apply mgr label:mgr
13. Configure OSD's
    - ceph orch daemon add osd ceph[0,1,2]:/dev/sd[b,c]
#### Optional: Find and apply all available disks as OSD's, this will also 'auto discover' new disks when they're added: `ceph orch apply osd --all-available-devices`

### Commands of interest
1. Ceph Full Status : ceph status
2. List devices : ceph orch device ls --wide --refresh
3. Display OSD Status : ceph osd tree
4. List Ceph Services : ceph orch ls
5. List Ceph Processes : ceph orch ps
