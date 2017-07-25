# OverlayFS on CentOS 7

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
