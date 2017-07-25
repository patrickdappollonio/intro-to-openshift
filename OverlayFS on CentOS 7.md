# OverlayFS on CentOS 7 and OpenShift

Stop Docker and remove the `devicemapper` folder from your hard drive:

```bash
# Stop Docker
systemctl stop docker

# Remove Docker folder with devicemapper
rm -rf /var/lib/docker
```

Once that's done, edit `/etc/sysconfig/docker` and change the `OPTIONS` part, there should be a part saying `--selinux-enabled`, which you should switch to `--selinux-enabled=false`.

Edit `/etc/sysconfig/docker-storage` file to use `overlay`, by editing the options like `DOCKER_STORAGE_OPTIONS=' -s overlay'`.

Start Docker once again with `systemctl start docker` and run a `docker info` to see if the storage mechanism is Overlay, it'll say something like:

```text
Storage Driver: overlay
 Backing Filesystem: xfs
```

Then edit the Ansible hosts file (usually `/etc/ansible/hosts`) and add the following two variables:

```yaml
docker_storage_driver=overlay
openshift_docker_selinux_enabled=False
```

We disable SELinux in Containers [due to a kernel issue](https://www.projectatomic.io/blog/2015/06/notes-on-fedora-centos-and-docker-storage-drivers/), but SELinux is still installed on the Host OS.
