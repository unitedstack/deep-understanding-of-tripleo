After full reboot of the host running the compute nodes, cinder-volume stays with error: Unable to update stats, LVMVolumeDriver -3.0.0
---


When openstack-cinder-volume uses an LVM backend and the Overcloud nodes reboot, the file-backed loopback device is not recreated. As a workaround, manually recreate the loopback device:

```
$ sudo losetup /dev/loop2 /var/lib/cinder/cinder-volumes
```
Then restart openstack-cinder-volume. Note that openstack-cinder-volume only runs on one node at a time in a high availability cluster of Overcloud Controller nodes. However, the loopback device should exist on all nodes.

https://bugzilla.redhat.com/show_bug.cgi?id=1241644