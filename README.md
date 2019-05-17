# Relax and Recover

Testing the rescue solution Relax and Recover, ReaR.

The test environment has 2 machines: one is the master, _rear01_, and also export the a NFS share so the second machine, _rear02_, can use to save the backup files.

## Adding a disk

The NFS export is a disk added via Vagrant configuration.

It seems that the order of definitions is import, and the order is:

1. Add the disk

1. Create a new controller to don't interfere with the disk created by Vagrant when starting the VM.

1. Attach the disk to the controller.

Also, the disk must be created only once, this means adding a check if the disk already exists when powering up or down the machines.
