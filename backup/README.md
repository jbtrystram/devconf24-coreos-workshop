# Storage

You can attach an additionnal disk to the VM using `launch_coreos_vm -d abcd123` where abcd123 is the serial number of the disk.
Knowning the serial of the disk allows us to use udev stable identifiers to manipulate the disk through ignition. 

Here is an example of formatting and mounting the disk through butane :

```
variant: fcos
version: 1.5.0
storage:
  disks:
    - device: /dev/disk/by-id/virtio-abc123   (1)
      wipe_table: true                        (2)
      partitions:
        - number: 1
          label: backup                       (3)
  filesystems:
    - path: /var/backup                       (4)
      device: /dev/disk/by-partlabel/backup   (5)
      format: btrfs                           (6)
      wipe_filesystem: true
      label: backup
      with_mount_unit: true                   (7)
```

Let´s break it down: 
(1) Get the block device we want to format by the id. Here the `virtio-` prefix is due to how we virtualize the disk. 
In real life this would be `nvme-Samsung_SSD_970_EVO...` \
(2) Wipe the partition table. \
(3) Here we create a partition and assign it the label `backup` \
(4) (5) (6) will format the partition identified with the label `backup` (created at (3)) with a btrfs filesystem and mount
it at `/var/backup` \
(7) create a systemd mount unit that will ensure the partition is mounted at each boot.


So now that we have a backupo disk available, let's set up a scheduled job to save the contents of our website ! 

We want to copy the content of our application, so the heart of it would be: 
```
rsync -a /var/syncthing/ /var/backup/syncthing
```

Let's build that into a systemd service, triggered by a timer:
```
# syncthing-backup.service:
  [Unit]
  Description=runs rsync backup of /var/syncthing
   # Stop syncthing to avoid data conflicts
  Conflicts=syncthing.service

  [Service]
  Type=oneshot
  ExecStart=rsync -av /var/syncthing /var/backup/syncthing
  [Install]
  WantedBy=multi-user.target

# syncthing-backup.timer:
  [Unit]
  Description=runs syncthing-backup.service nightly

  [Timer]
  OnCalendar=*-*-* 3:00:00
  Persistent=true
  Unit=syncthing-backup.service

  [Install]
  WantedBy=timers.target
```

Try to incorporate this into the butane config ! *hint:* search for the `systemd.unit` section in [the butane documentation](https://coreos.github.io/butane/config-fcos-v1_5/)

