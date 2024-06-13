# Quadlets !

As we mentionned, Fedora CoreOS is really tailored to run containers. A really good way to do that is to deploy
kubernetes on top of a cluster of coreOS machine, but that's quite overkill for a handful of containers at home. 

Another really nice way is to leverage podman quadlets. Let's dive a bit deeper. 

The podman quadlet generator reads a few types of files: 
- image
- container
- volume
- pod


Here is a more complex example deploying a wordpress service: 
- WordPress network that holds the containers
- WordPress database container
- WordPress container
- 1 volume per container

 
First we have `wordpress.network`:
```
[Unit]
Description=WordPress Container Network

[Network]
Label=app=wordpress
```
Note that grouping the container into a `Pod` can be an alternative to `network`.

`wordpress-app.volume`:
```
[Unit]
Description=WordPress Container Volume

[Volume]
Label=app=wordpress
```
`wordpress-db.volume`:
```
[Unit]
Description=WordPress Database Container Volume

[Volume]
Label=app=wordpress
```

`wordpress-db.container`:
```
[Unit]
Description=WordPress Database Container

[Container]
Label=app=wordpress
ContainerName=wordpress-db
Image=docker.io/library/mariadb:10
Network=wordpress.network
Volume=wordpress-db.volume:/var/lib/mysql
Environment=MARIADB_RANDOM_ROOT_PASSWORD=1
Environment=MARIADB_USER=wordpress
Environment=MARIADB_DATABASE=wordpress
Environment=MARIADB_PASSWORD=password

[Install]
WantedBy=multi-user.target default.target
```

`wordpress-app.container` (note the dependencies at the top):
```
[Unit]
Description=Wordpress App Container
Requires=wordpress-db.service
After=wordpress-db.service

[Container]
Label=app=wordpress
ContainerName=wordpress-app
Image=docker.io/library/wordpress:6
Network=wordpress.network
Volume=wordpress-app.volume:/var/www/html
Environment=WORDPRESS_DB_HOST=wordpress-db
Environment=WORDPRESS_DB_USER=wordpress
Environment=WORDPRESS_DB_NAME=wordpress
# This one should be stored in a secret
Environment=WORDPRESS_DB_PASSWORD=password
PublishPort=8080:80

[Install]
WantedBy=multi-user.target default.target
```
All those files can go into `/etc/containers/systemd/wordpress`. (The `wordpress` subfolder is optionnal`).

That is a good showcase of the podman quadlet features. Feel free to experiment ! 
Any changes can be applies with `sudo systemctl daemon-reload`.

Source: https://blog.while-true-do.io/podman-quadlets/

# Rootless mode

Quadlets can also run in rootless mode. For this simply create the `.container` file under `/etc/containers/systemd/users/<$UID>/`
There are several others paths that can be used for user level services, you can look at `podman-systemd.unit` documentation.

**Note**: by default, users services only start when the user session starts (user logs in).
Systemd (loginctl enable-linger)[https://www.man7.org/linux/man-pages/man1/loginctl.1.html] enable slinger for a user, here is how we can do it through butane : 
```
variant: fcos
version: 1.5.0
passwd:
  users:
    - name: wordpress
storage:
  files:
    - path: /var/lib/systemd/linger/wordpress
      mode: 0644
```
This config allow lingering for the user `wordpress`. I'll let you look up in the butane spec how to manually specify a user ID ;) 


More details [in the fedora coreOS docs](https://docs.fedoraproject.org/en-US/fedora-coreos/tutorial-user-systemd-unit-on-boot/)

# Resources
[A red hat blog post about quadlets](https://www.redhat.com/sysadmin/container-systemd-persist-reboot) \
Here is another link to the [man podman-systemd.unit](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) documentation !
