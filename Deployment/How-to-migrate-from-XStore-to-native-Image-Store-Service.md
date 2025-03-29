# How to migrate from XStore to the native Image Store Service

The following steps outline the process for migrating from specific XStore to the native Image Store Service:

## Pre-requisite

### Restriction of image store migration

Any change to the image store should not be allowed during the migration. If the content is copied from the original image store to the target image store, and before the cluster is switched to the target image store by fabric upgrade; any new changes on the original image store, brought by provision/un-provision/creation/upgrade/deletion, will be lost at the target image store and cause image store inconsistency issue.


### Ensure that the current cluster manifest has been provisioned

It’s just the same as what any fabric upgrade demands. In case of failure of fabric upgrade, the provision of the original cluster manifest is required for the rollback.  If it hasn’t been done yet, using the following commands to copy current cluster package to image store.


```json

Copy-WindowsFabricClusterPackage -ClusterManifestPath xxx\CurrentClusterManifest.xml -CodePackagePath xxx\CodePackage\WindowsFabricRC.3.3.45.9490.msi -ImageStoreConnectionString "file:C:\ProgramData\Windows Fabric\ImageStore"

Register-WindowsFabricClusterPackage -ClusterManifestPath CurrentClusterManifest.xml -CodePackagePath WindowsFabricRC.3.3.45.9490.msi

```

If running on PAAS V1, Port 445 should be added as input endpoint for SMB copy

## Migration Steps

1. Prepare 2 updated cluster manifest files and do the provision. Updating the current cluster manifest by adding the following “ImageStoreService” configuration. Note: set the “Enabled” flag to “true” to create the new native image store.
   
```json
<Section Name="ImageStoreService">
<Parameter Name="Enabled" Value="true" />
```
2.	Do first fabric upgrade to create the native image store. The purpose of the first fabric upgrade is to create an extra native image store beside the current image store. Actually, the new native image store is empty and unused, however it’s still observable by querying the system service.
   
```json

Get-WindowsFabricService -ApplicationName fabric:/System

```

3.	Use image store copy tool to copy the content from source image store to the target native image store.

```json

ImageStoreCopier.exe /ConnectionEndpoint:"MININT-8MRQ409.redmond.corp.microsoft.com:19000" /CredentialType:X509 /ServerCommonName:"WinfabDevClusterCert" /FindType:"FindBySubjectName" /FindValue:"CN=WinfabDevClusterCert" /StoreLocation:"LocalMachine" /StoreName:"My" /SourceImageStore:"file:C:\ProgramData\Windows Fabric\ImageStore" /DestinationImageStore:"fabric:ImageStore"

```

(Note: once the copy is completed, a manual comparison between two image stores is strongly recommended. The location of native image store can be found by querying the primary replica: Get-WindowsFabricReplica -PartitionId 00000000-0000-0000-0000-000000003000)

4.	Do a 2nd fabric upgrade to make the native image store officially connected. Once the upgrade is completed, the verification can be done by invoking image store operation like doing provision/un-provision/creation/upgrade/deletion; the change of content at native image store is expected. Use image store tool compare command to verify the consistence by comparing the before – and – after view of original image store; since the tool generates a snapshot file called “imageStoreSnapshot.sp” which contains serialized content objects. When the 2nd upgrade is completed, execute the following command and comparison result will be printed out to the screen.

```json

ImageStoreCopier.exe /ConnectionEndpoint:"localhost:19000" /SourceImageStore:"file:C:\ProgramData\Windows Fabric\ImageStore" /Compare

```
