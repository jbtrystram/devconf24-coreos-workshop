# Maintenance-free self-hosting with FCOS

Welcome to the examples config files for the DevConf.cz 2024 CoreOS workshop ! 
This repositories lays out the basic to get started and contains some example butane config files so you can 
build on them to hack on your own coreOS instance ! 

## 1 - Get into the workshop environnement

Ssh into the host that will be displayed by the workshop host, using credentials you have been given. 

Once in, you are able to launch coreOS virtual machines with:  

```
launch_coreos_vm
```
TA-DA ! 

You should notice that you cannot log into it for now. Let's fix that.

**TIP**: You can escape out of the console by pressing `Ctrl + ]` \
As long as you use this script, VMs will be destroyed when you escape the console, so you can launch another.

**TIP2**: Sometimes, the console output may break the terminal output. Nothing `reset` won't fix!
## 2 - Create your first ignition config 

The provisionned environment (a coreOS machine) you've been given access to should have all the tools you need.

Create a first butane file with the following contents: 
```yaml
variant: fcos
version: 1.5.0
systemd:
  units:
    - name: serial-getty@ttyS0.service
      dropins:
      - name: autologin-core.conf
        contents: |
          [Service]
          # Override Execstart in main unit
          ExecStart=
          # Add new Execstart with `-` prefix to ignore failure`
          ExecStart=-/usr/sbin/agetty --autologin core --noclear %I $TERM
```
This will allow you to use login into the machine through the serial console.

You can now transpile this in the ignition format with:
```
butane --pretty --strict autologin.bu --output autologin.ign
```

Then boot the VM as instructed before, but pass the ignition file as argument:
```
launch_coreos_vm -i autologin.ign

# you can even run the transpiling step at the same time with: 
launch_coreos_vm -b autologin.bu
```

Make sure to open the [butane specification](https://coreos.github.io/butane/specs/), you will probably need it later in this workshop ! 

You should think about coreOS like a container. If you want a change, update the config and re-provision the node.
Doing ad-hoc changes works, but changing the butane/ignition config is easier to track, and always reproducible !


## 3 - Write a quadlet to run containers


Now that you have the basic flow working, let's elaborate that ignition config a bit. 

You can rely on podman quadlets to generate systemd services that will handle the lifecycle of containers.
This use the systemd-generator feature : at boot, the quadlet-generator of podman will scan some well-known directories
for `.container` files and generate systemd unit files. 
    
The appropriate podman commands will be wrapped in a systemd unit so your containers are just a `systemctl start mycontainer` away!

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
1: The container will use the image at `docker.io/nginx` \
2: mount /var/nginx/html inside the container at /usr/share/nginx/html (read only) \
3: If the service exit, always restart it \
4: Ties the service to a well-known target so it is started at boot. \

Create this in `/etc/containers/systemd/nginx.container` using a `storage.files` butane key:
```
variant: fcos
version: 1.5.0
passwd:
  users:
    - name: wordpress
storage:
  files:
    - path: /etc/containers/systemd...
      contents:
        inline: |
          [Container]
          Image=quay.io....
```

See the [quadlet page](./quadlet.md) for other examples and implementation in a butane config !

**TIP**: If you mess up the quadlet file, you can use: 
`/usr/libexec/podman/quadlet -dryrun` from within the VM to debug your file. 

More documentation at [man podman-systemd.unit](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)


## 3b - Merge butane configs together

At this point you probably booted a few VMs and have a few butane/ignition files laying around. \
You can conveniently merge ignition configs into butane: 
```
variant: fcos
version: 1.5.0
ignition
  config:
    merge:
      - local: autologin.ign 
      - local: some_other_config.ign 
      - source: http://example.com/remote_config.ign 
```

See the `ignition` key in the butane documentation for more details.

## 4 - Set up some backups - storage

Now that we have a nice service up and running, let's set a systemd timer (cron job alternative) to back up the data ! 

So we would have: a systemd timer tied to systemd unit that does the backup work. If the backup script is more complex
than a one-liner, feel free to break out a script and call it from the systemd unit. 

See [the backup doc](./backup.md) for an example

[man systemd.unit](https://www.man7.org/linux/man-pages/man5/systemd.unit.5.html) \
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

## 7 - How to do this at home ?

Coreos-installer will help you create ISOs with your ignition config embedded. 

We also have instructions for a bunch of platforms in the [coreOS documentation](https://docs.fedoraproject.org/en-US/fedora-coreos/bare-metal/)

# Useful links

Butane examples: https://coreos.github.io/butane/examples/
CoreOS tutorials: https://docs.fedoraproject.org/en-US/fedora-coreos/tutorial-autologin/

