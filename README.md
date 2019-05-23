# Relax and Recover

Testing the rescue solution Relax and Recover, ReaR.

The test environment has 2 machines: one is the master, _rear01_, and also export the a NFS share so the second machine, _rear02_, can use to save the backup files.

## Adding a disk

The NFS export is a disk added via Vagrant configuration.

It seems that the order of definitions is import, and the order is:

1. Add the disk

1. Create a new controller to don't interfere with the disk created by Vagrant when starting the VM.

1. Attach the disk to the controller.

First we create a file that will represent the disk, a VDI file. But create the file only once

    file_to_disk = File.realpath(".").to_s + "/rear01b.vdi"
    if !File.exist?(file_to_disk)
        vb.customize [
            'createhd',
            '--filename', file_to_disk,
            '--format', 'VDI',
            '--size', 15 * 1024 # 15GB
        ]

Then we create a new controller for the disk

    vb.customize [
        'storagectl', :id,
        '--name', 'rearctl',
        '--add', 'sata'
    ]

Finally we add the disk to the controller

    vb.customize [
        'storageattach', :id,
        '--storagectl', 'rearctl',
        '--port', 0,
        '--type', 'hdd', '--medium',
        file_to_disk
    ]

## NFS share 

After adding the disk, we configure it to be exported via NFS from `rear01`. This area will be used to store the backup files created by ReaR running on `rear02`.

We start creating the partition on the disk and configuring the NFS server on `rear01`

    rear01.vm.provision "shell", inline: $parted_sdb
    rear01.vm.provision "shell", inline: "mkfs.ext3 /dev/sdb1 > /dev/null 2>&1"
    rear01.vm.provision "shell", inline: $nfs_server

Where `$parted_sdb` creates the partition on the disk

```bash
    $parted_sdb = <<-SCRIPT
    parted --script /dev/sdb \
    unit % \
    mklabel msdos \
    mkpart primary ext3 1 100% \
    quit
    SCRIPT
```

And `$nfs_server` configure the NFS server

```bash
    $nfs_server = <<-SCRIPT
    # Create directory before exporting it.
    if [ ! -d /data/ ]; then
        echo "Creating directory '/data'"
        mkdir /data/
    fi
    mount /dev/sdb1 /data
    # Install NFS package
    yum -y --quiet install nfs-utils
    systemctl enable nfs-server.service
    systemctl start nfs-server.service
    echo "/data   192.168.50.11(rw,sync,no_root_squash,no_subtree_check)" > /etc/exports
    /usr/sbin/exportfs -a
    SCRIPT
```

## NFS client and ReaR

The NFS client is configured and the ReaR package is installed on `rear02`

    rear02.vm.provision "shell", inline: $nfs_client
    rear02.vm.provision "shell", inline: "yum -y --quiet install rear"

Where `$nfs_client` is define as

```bash
    $nfs_client = <<-SCRIPT
    # Install NFS package
    yum -y --quiet install nfs-utils
    # Create mount point for the NFS export.
    if [ ! -d /backup/ ]; then
        echo "Creating directory '/backup'"
        mkdir /backup/
    fi
    # Check if fstab already configured.
    grep 'backup' /etc/fstab > /dev/null 2>&1
    if [ "$?" -eq 1 ]; then
        echo "192.168.50.10:/data  /backup   nfs rw,sync,hard,intr  0     0" >> /etc/fstab
    fi
    mount /backup
    SCRIPT
```

Now the cluster is ready to be a test environment for ReaR.