自定义Overcloud roles
---

#限制
- You can assign any systemd managed service to a supported standalone custom role.
- You cannot split Pacemaker-managed services. This is because the Pacemaker manages the same set of services on each node within the Overcloud cluster. Splitting Pacemaker-managed services can cause cluster deployment errors. These services should remain on the Controller role.
- You cannot change to custom roles and composable services during the upgrade process from Red Hat OpenStack Platform 9 to 10. The upgrade scripts can only accommodate the default Overcloud roles.
- You can create additional custom roles after the initial deployment and deploy them to scale existing services.
- You cannot modify the list of services for any role after deploying an Overcloud. Modifying the service lists after Overcloud deployment can cause deployment errors and leave orphaned services on nodes.