TripleO 部署 Overcloud  排错方法和思路
---

1. 部署overcloud过程中失败，先查看heat resource list ，确认哪些组件部署失败。
```
heat resource-list overcloud
+-----------------------------------+-----------------------------------------------+---------------------------------------------------+-----------------+----------------------+
| resource_name                     | physical_resource_id                          | resource_type                                     | resource_status | updated_time         |
+-----------------------------------+-----------------------------------------------+---------------------------------------------------+-----------------+----------------------+
| BlockStorage                      | 9e40a1ee-96d3-4920-868d-683d3788e129          | OS::Heat::ResourceGroup                           | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| BlockStorageAllNodesDeployment    | 2c453f6b-7378-44c8-a0ad-57de57d9c57f          | OS::Heat::StructuredDeployments                   | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| BlockStorageNodesPostDeployment   |                                               | OS::TripleO::BlockStoragePostDeployment           | INIT_COMPLETE   | 2015-04-06T21:15:20Z |
| CephClusterConfig                 | 1684e7a3-0e42-44fe-9db4-7543b742fbfc          | OS::TripleO::CephClusterConfig::SoftwareConfig    | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| CephStorage                       | 48b3460c-bf9a-4663-99fc-2b4fa01b8dc1          | OS::Heat::ResourceGroup                           | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| CephStorageAllNodesDeployment     | 76beb3a9-8327-4d2e-a206-efe12f1613fb          | OS::Heat::StructuredDeployments                   | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| CephStorageCephDeployment         | af8fb02a-5bc6-468c-8fac-fbe7e5b2c689          | OS::Heat::StructuredDeployments                   | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| CephStorageNodesPostDeployment    |                                               | OS::TripleO::CephStoragePostDeployment            | INIT_COMPLETE   | 2015-04-06T21:15:20Z |
| Compute                           | e5e6ec84-197f-4bf6-b8ac-eb11fe494cdf          | OS::Heat::ResourceGroup                           | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ComputeAllNodesDeployment         | e6d44fbf-9683-4765-acbb-4a3d31c8fd48          | OS::Heat::StructuredDeployments                   | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ControllerNodesPostDeployment     | e551e472-f2db-4468-b586-0374678d71a3          | OS::TripleO::ControllerPostDeployment             | CREATE_FAILED   | 2015-04-06T21:15:20Z |
| ComputeCephDeployment             | 673608d5-70d7-453a-ac78-7987bc2c0158          | OS::Heat::StructuredDeployments                   | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ComputeNodesPostDeployment        | 1078e3e3-9f6f-48b9-8961-a30f44098856          | OS::TripleO::ComputePostDeployment                | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ControlVirtualIP                  | 6402b396-84aa-4cf6-9849-305205755604          | OS::Neutron::Port                                 | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| Controller                        | ffc45352-9708-486d-81ac-3b60efa8e8b8          | OS::Heat::ResourceGroup                           | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ControllerAllNodesDeployment      | f73c6e33-3dd2-46f1-9eca-0d2981a4a986          | OS::Heat::StructuredDeployments                   | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ControllerBootstrapNodeConfig     | 01ce5b6a-794a-4828-bad9-49d5fbfd55bf          | OS::TripleO::BootstrapNode::SoftwareConfig        | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ControllerBootstrapNodeDeployment | c963d53d-879b-4a41-a10a-9000ac9f02a1          | OS::Heat::StructuredDeployments                   | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ControllerCephDeployment          | 2d4281df-31ea-4433-820d-984a6dca6eb1          | OS::Heat::StructuredDeployments                   | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ControllerClusterConfig           | 719c0d30-a4b8-4f77-9ab6-b3c9759abeb3          | OS::Heat::StructuredConfig                        | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ControllerClusterDeployment       | d929aa40-1b73-429e-81d5-aaf966fa6756          | OS::Heat::StructuredDeployments                   | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ControllerSwiftDeployment         | cf28f9fe-025d-4eed-b3e5-3a5284a2aa60          | OS::Heat::StructuredDeployments                   | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| HeatAuthEncryptionKey             | overcloud-HeatAuthEncryptionKey-5uw6wo7kavnq  | OS::Heat::RandomString                            | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| MysqlClusterUniquePart            | overcloud-MysqlClusterUniquePart-vazyj2s4n2o5 | OS::Heat::RandomString                            | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| MysqlRootPassword                 | overcloud-MysqlRootPassword-nek2iky7zfdm      | OS::Heat::RandomString                            | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ObjectStorage                     | 47327c98-533e-4cc2-b1f3-d8d0eedba822          | OS::Heat::ResourceGroup                           | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ObjectStorageAllNodesDeployment   | 7bb691aa-fa93-4f10-833e-6edeccc61408          | OS::Heat::StructuredDeployments                   | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ObjectStorageNodesPostDeployment  | d4d16f39-384a-4d6a-9719-1dd9b2d4ff09          | OS::TripleO::ObjectStoragePostDeployment          | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| ObjectStorageSwiftDeployment      | afc87385-8b40-4097-b529-2a5bc81c94c8          | OS::Heat::StructuredDeployments                   | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| PublicVirtualIP                   | 4dd92878-8f29-49d8-9d3d-bc0cd44d26a9          | OS::Neutron::Port                                 | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| RabbitCookie                      | overcloud-RabbitCookie-uthzbos3l66v           | OS::Heat::RandomString                            | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| SwiftDevicesAndProxyConfig        | e2141170-bb77-4509-b8bd-58447b2cd15f          | OS::TripleO::SwiftDevicesAndProxy::SoftwareConfig | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
| allNodesConfig                    | cbd42692-fffa-4527-a519-bd4014ebf0fb          | OS::TripleO::AllNodes::SoftwareConfig             | CREATE_COMPLETE | 2015-04-06T21:15:20Z |
+-----------------------------------+-----------------------------------------------+---------------------------------------------------+-----------------+----------------------+
```




REF：[Troubleshooting a Failed Overcloud Deployment](http://docs.openstack.org/developer/tripleo-docs/troubleshooting/troubleshooting-overcloud.html)