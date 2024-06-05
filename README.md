# Maintenance-free self-hosting with FCOS

Welcome to the examples config files for the DevConf.cz 2024 CoreOS workshop ! 
This repositories lays out the basic to get started and contains some example butane config files so you can 
build on them to hack on your own coreOS instance ! 


## 1 - Transpile your first butane config to ignition

If you have podman or docker installed you can simply run butane from a container: 
```
# using standard input/output
podman run -i --rm quay.io/coreos/butane:release --strict < your_config.bu > result_config.ign
# using files
podman run --rm -v /path/to/your_config.bu:/config.bu:z quay.io/coreos/butane:release --strict /config.bu > result_config.ign
```

Otherwise, run `dnf/apk install butane`, then: 
```
butane --strict config.bu > result_config.ign
```

Here is a minimal butane config for you to try:
```yaml
variant: fcos
version: 1.5.0
passwd:
  users:
    - name: core
      ssh_authorized_keys:
        - ssh-rsa AABBza75L... <your public key here>
```
This will allow you to SSH into the machine.

Make sure to pen the [butane specification](https://coreos.github.io/butane/specs/), you will probably need it later in this workshop ! 

## 2 - Boot your first VM with the ignition config ! 

ssh into the host that will be displayed by the workshop host, using credentials you have been given. 

Once in, create your butane config in your home directory. 

You can now launch a coreOS VM with : 

```

```
TA-DA ! 

You should think about coreOS like a container. If you want a change, update the config and re-provision the node.
Doing ad-hoc changes works, but changing the butane/ignition config is easier to track, and always reproducable !


## 3 - Write a quadlet to run containers

Now that you have the basic flow working, let's elaborate that ignition config a bit. 

You can rely on podman quadlets to generate systemd services that will handle the lifecycle of containers.
This use the systemd-generator feature : at boot, the quadlet-generator of podman will scan some well-known directories
for `.container` files and generate systemd unit files. 
    
The appropriate podman commands will be wrapped in a systemd unit so your containers are just a `systmectl start mycontainer` away!

Small example : 
```
[Container]
Image=docker.io/nginx (1)
Volume=/var/nginx/html:/usr/share/nginx/html:ro (2)
[Service]
Restart=always (3) 
[Install]
WantedBy=multi-user.target (4)
```
1: The container will use the image at `docker.io/nginx` 
2: mount /var/nginx/html inside the container at /usr/share/nginx/html (read only)
3: If the service exit, always restart it
4: Ties the service to a well-known target so it is started at boot.
Simply create this in `/etc/containers/systemd/nginx.container` for example.

See the [quadlet folder](./quadlet/) for other examples and implementation in a butane config !

More documentation at [man podman-systemd.unit](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)


## 4 - Set up some backups

Now that we have a nice service up and running, let's set a systemd timer (cron job alternative) to back up the data ! 

So we would have: a systemd timer tied to systemd unit that does the backup work. If the backup script is more complex
than a one-liner, feel free to break out a script and call it from the systemd unit. 

See [the backup folder](./backup) for an example

[man systemd.unit](https://www.man7.org/linux/man-pages/man5/systemd.unit.5.html)
[man systemd.timer](https://www.man7.org/linux/man-pages/man5/systemd.timer.5.html)

## 5 - Tune the update schedule

There isn't much about coreOS update. We strongly encourage you to leave the auto-updates on.
You can however tune the schedule and update windows. Here is a config example.

```toml
[updates]
strategy = "periodic"
[[updates.periodic.window]]
days = [ "Sat", "Sun" ]
start_time = "22:30"
length_minutes = 60
```
The file should be placed in `/etc/zincati/config.d/your-custom-config.toml`. I'll let you cook up the appropriate 
butane snippet!

More documentation on [coreOS updates](https://docs.fedoraproject.org/en-US/fedora-coreos/auto-updates/)
[Zincati documentation](https://coreos.github.io/zincati/)

## 6 - Experiment ! 

Feel free to experiment with the coreOS instance you have access to, it will be destroyed after the workshop ends.
