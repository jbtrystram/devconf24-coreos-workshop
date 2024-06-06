# Quadlets !

As we mentionned, Fedora CoreOS is really tailored to run containers. A really good way to do that is to deploy
kubernetes on top of a cluster of coreOS machine, but that's quite overkill for a hadnful of containers at home. 

Another really nice way is to leverage podman quadlets. Let's dive a bit deeper. 

The podman quadlet generator reads a few types of files: 
- image
- container
- volume
- pod


Here is another link to the [man podman-systemd.unit](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html) documentation !
