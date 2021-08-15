# Setting up custom ISO storage space for your new proxmox install

**For sake of example we will pretend proxmox is being installed to a 120GB SSD and that its dev path is /dev/sda**

* Fire up Proxmox installer either boot from USB flash or CD/DVD

* During install when you get to the screen with the Options button, click that and under 'HDSIZE' set it to say, 60GB. (this will reserve only 60gb for proxmox to install into leaving the rest of the SSD empty)

* After installation is complete log into proxmox and go to the CLI either from the Web UI or over SSH/local console.

* enter into cfdisk to create the partition

    `cfdisk /dev/sda`

* select free space and choose create below, accepting the default options for size and format being Linux (ext4)

* that should only take a moment.  back at the main screen if the new partition looks good at the bottom arrow over to "WRITE" and enter "yes" when asked to confirm. (this doesnt affect anything on proxmox itself we are adding a partition to free space not touching anything already there)

* We must format the new partition by creating the file system on it

    `mke2fs.ext4 /dev/sdaX`

> *Where "X" is the number of your newly created partition.If you arent sure what it is you can run `lsblk` to list all the partitions*

* Make a new path to mount this data in on the host
    `mkdir /mnt/iso-images`

* Find the new partition's UUID
    `blkid /dev/sdaX`

* Add entry to /etc/fstab to auto-mount on boot

    `nano /etc/fstab`

    ```conf
    # <file system> <mount point> <type> <options> <dump> <pass>
    /dev/pve/root / ext4 errors=remount-ro 0 1
    /dev/pve/swap none swap sw 0 0
    proc /proc proc defaults 0 0
    UUID="f6005985-739f-4d55-aa6e-4a32450b43a1" /mnt/iso_images ext4 defaults 0 1
    ```

* *The last line above is the new one*

* Now re-run mount so it mounts the new space immediately
`mount -a`

* That returns successful then go to proxmox UI and under DATACENTER => Storage, click "add", under the drop down select "directory"

* Input the mount path `/mnt/iso-images` that we created above and name your storage something like iso-images.  be sure to open the content drop down and highlight ISO Images and CT Templates (in case you want to d/l LXC templates)  Un-Highlight anything we dont want to go here like vm disks or backups, etc. because this is for ISO's

* Save and done!

## Bonus section

**If you have multiple Proxmox servers and would like to access the same set of ISO's from them all...**

* install NFS on this server

`apt install nfs-kernel-server`

* add the export

`nano /etc/exports`

```conf
# /etc/exports: the access control list for filesystems which may be exported
#               to NFS clients.  See exports(5).
#
# Example for NFSv2 and NFSv3:
# /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
#
# Example for NFSv4:
# /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
# /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
#
/mnt/iso_images         pve05(ro)
```

> Notice here I have put the hostname of my other proxmox server which will connect as a client to this ISO share over NFS and I have set it to RO for read only so it doesnt bother to try to overwrite anything by accident

`service nfs-kernel-server restart`

* from the other server go to datacenter => Storage and add then click NFS as the option, enter in a name for it and the IP or host name from the first proxmox server.

* the iso-images path should appear when you click into the export path field if the steps were followed correctly.

* again here also make sure to select the ISO images and ct templates from the content type drop down and save.

* if all worked well you should see your ISO's from the other server now under that storage item when you browse it on the 2nd server.