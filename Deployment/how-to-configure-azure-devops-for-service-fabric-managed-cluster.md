# How to configure Azure Devops Service Fabric Managed Cluster Service Connection

The steps below describe how to configure the ADO service connection for Service Fabric managed clusters with Azure Active Directory (AAD / Azure AD). This solution requires both the use of Azure AD and the use of Azure provided build agents in ADO.

Service Fabric Managed Clusters provision and manage the 'server' certificate including the rollover process before certificate expiration.
There is currently no notification when this occurs.
Azure Devops (ADO) service connections that use X509 Certificate authentication requires the configuration of the server certificate thumbprint.
When the certificate is rolled over, the Service Fabric service connection will fail to connect to cluster causing pipelines to fail.

## Process

- Verify [Requirements](#requirements).
- In Azure Devops, create / modify the 'Service Fabric' service connection to be used with the build / release pipelines for the managed cluster.
- [Test](#test) connection.

## Requirements

- Service Fabric managed cluster security with Azure Active Directory enabled. See [Service Fabric cluster security scenarios](https://docs.microsoft.com/azure/service-fabric/service-fabric-cluster-security#client-to-node-azure-active-directory-security-on-azure) and [Service Fabric Azure Active Directory configuration in Azure portal](https://learn.microsoft.com/azure/service-fabric/service-fabric-cluster-creation-setup-azure-ad-via-portal) for additional information.

  ![sfmc enable aad](/media/how-to-configure-azure-devops-for-service-fabric-managed-cluster/sfmc-enable-aad.png)

- Azure Devops user configured to use the 'Cluster' App Registration that is configured for the managed cluster.

- Azure Devops build agent with 'Hosted' (not 'Self-Hosted') pool type. For hosted, 'Azure virtual machine scale set' is the pool type to be used.

  ![sfmc ado pool type](/media/how-to-configure-azure-devops-for-service-fabric-managed-cluster/sfmc-ado-pool-type.png)

- Connectivity from ADO agent to cluster. This can be done by adding the ADO agent IP address to the cluster Network Security Group (NSG) inbound rule for the cluster endpoint port. See [Azure Network Security Group (NSG) Configuration](#azure-network-security-group-nsg-configuration) for more information.

## Azure Network Security Group (NSG) Configuration

The 'AzureCloud' [Service Tag](https://learn.microsoft.com/azure/virtual-network/service-tags-overview) can be used when configuring a Network Security Group (NSG) for access to cluster. If using a self-hosted ADO agent, the agent IP address will need to be added to the NSG inbound rule for the cluster endpoint port. If using a ADO agent pool, the agent pool IP address will need to be added to the NSG inbound rule for the cluster endpoint port. A list of IP ranges being used by ADO can be found [here](https://docs.microsoft.com/azure/devops/organizations/security/allow-list-ip-url?view=azure-devops#ip-ranges).

> **Important**
> If using a Service Tag for Azure Devops access to cluster, ensure the NSG inbound rule is using service tag 'AzureCloud' (not 'AzureDevops') and is at minimum configured for the cluster gateway endpoint port. The default port is 19000.

- Source: Service Tag
- Source service tag: AzureCloud
- Source port ranges: *
- Destination: Any
- Service: Custom
- Destination port ranges: 19000
- Protocol: TCP
- Action: Allow
- Priority: 110
- Name: AzureDevopsDeployment

![nsg inbound rule](/media/how-to-configure-azure-devops-for-service-fabric-cluster/ado-nsg-service-tag.png)

### Service Fabric Service Connection

Create / Modify the Service Fabric Service Connection to provide connectivity to Service Fabric managed cluster from ADO pipelines.
For maintenance free configuration, only 'Azure Active Directory credential' authentication  and 'Common Name' server certificate lookup is supported.

#### Service Fabric Service Connection Properties

- **Authentication method:** Select 'Azure Active Directory credential'.
- **Cluster Endpoint:** Enter connection endpoint for cluster. This is in the format of tcp://{{cluster name}}.{{azure region}}.cloudapp.azure.com:{{cluster endpoint port}}.
  - Example: tcp://mysftestcluster.eastus.cloudapp.azure.com:19000
- **Server Certificate Lookup (optional):** Select 'Common Name'.
- **Server Common Name** Enter the managed cluster server certificate common name. The common name format is {{cluster guid id with no dashes}}.sfmc.azclient.ms. This name can also be found in the cluster manifest in Service Fabric Explorer (SFX).
  - Example: d3cfe121611d4c178f75821596a37056.sfmc.azclient.ms

    ![sfmc cluster id](/media/how-to-configure-azure-devops-for-service-fabric-managed-cluster/sfmc-cluster-id.png)

- **Username:** Enter an Azure AD user that has been added to the managed clusters 'Cluster' App Registration in UPN format. This can be tested by connecting to SFX as the Azure AD user.
- **Password:** Enter Azure AD users password. If this is a new user, ensure account is not prompting for a password change. This can be tested by connecting to SFX as the Azure AD user.
- **Service connection name:** Enter a descriptive name of connection.

  ![sfmc ado service connection](/media/how-to-configure-azure-devops-for-service-fabric-managed-cluster/sfmc-ado-service-connection.png)

## Test

Use builtin task 'Service Fabric PowerShell' in pipeline to test connection.

```yaml
trigger:
  - main

pool:
  vmImage: "windows-latest"

variables:
  System.Debug: true
  sfmcServiceConnectionName: serviceFabricConnection

steps:
  - task: ServiceFabricPowerShell@1
    inputs:
      clusterConnection: $(sfmcServiceConnectionName)
      ScriptType: "InlineScript"
      Inline: |
        $psVersionTable
        $env:connection
        [environment]::getEnvironmentVariables().getEnumerator()|sort Name
```

## Troubleshooting

- Error: ##[debug]System.AggregateException: One or more errors occurred. ---> System.Fabric.FabricTransientException: Could not ping any of the provided Service Fabric gateway endpoints. ---> System.Runtime.InteropServices.COMException: Exception from HRESULT: 0x80071C49
- Test network connectivity. Add a powershell task to pipeline to run 'test-netConnection' command to cluster endpoint, providing tcp port. Default port is 19000.

  - Example:

  ```yaml
  - powershell: |
      $psVersionTable
      [environment]::getEnvironmentVariables().getEnumerator()|sort Name
      $publicIp = (Invoke-RestMethod https://ipinfo.io/json).ip
      write-host "---`r`ncurrent public ip:$publicIp" -ForegroundColor Green
      write-host "test-netConnection $env:clusterEndpoint -p $env:clusterPort"
      $result = test-netConnection $env:clusterEndpoint -p $env:clusterPort
      write-host "test net connection result: $($result | fl * | out-string)"
      if(!($result.TcpTestSucceeded)) { throw }
    errorActionPreference: stop
    displayName: "PowerShell Troubleshooting Script"
    failOnStderr: true
    ignoreLASTEXITCODE: false
    env:
      clusterPort: 19000
      clusterEndpoint: xxxxxx.xxxxx.cloudapp.azure.com
  ```

- Verify configured Azure AD user is able to logon successfully to cluster using SFX or powershell. The 'servicefabric' module is installed as part of Service Fabric SDK.

  ```powershell
  import-module servicefabric
  import-module az.resources

  $clusterEndpoint = 'mysftestcluster.eastus.cloudapp.azure.com:19000'
  $clusterName = 'mysftestcluster'

  $clusterResource = Get-AzResource -Name $clusterName -ResourceType 'Microsoft.ServiceFabric/managedclusters'
  $serverCertThumbprint = $clusterResource.Properties.clusterCertificateThumbprints

  Connect-ServiceFabricCluster -ConnectionEndpoint $clusterEndpoint `
    -AzureActiveDirectory `
    -ServerCertThumbprint $serverCertThumbprint `
    -Verbose
  ```

- Use logging from task to assist with issues.
- Enabling System.Debug in build yaml or in release variables will provide additional output.
