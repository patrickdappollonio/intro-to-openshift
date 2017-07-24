# OverlayFS on CentOS 7

Perform the following steps to update the kernel to use version 4.x where OverlayFS is available:

```bash
# Import ELRepo GPG Keys
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org

# Import ELRepo Yum repository
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm

# Clean the Yum cache
yum clean all

# Enable the repo and install `kernel-ml`
yum --enablerepo=elrepo-kernel install kernel-ml -y

# Once installed, change the default grub boot order to use
# the recently installed kernel
grub2-set-default 0

# Enable Overlay to load on boot
echo "overlay" > /etc/modules-load.d/overlay.conf
```

Once that's done, change the Docker configuration on CentOS to use `overlay` 
(this will only work on Docker installed from EPEL, and running on version 1.12):

```bash
echo "DOCKER_STORAGE_OPTIONS='--storage-driver=overlay'" > /etc/sysconfig/docker-storage
```

Reboot the machine to enable the `overlay` module and the new kernel, and then
either enable and/or restart the Docker service with `systemctl`. You can verify it worked by
executing `docker info` and checking the `Storage` section, it'll be something like:

```text
Server Version: 1.12.6
Storage Driver: overlay
 Backing Filesystem: xfs
```
