
The purpose of this overview is to demostrate the integration of AKS with Azure AD and and removing the use of the cluster-admin certificate based authentication from day-to-day activities.

### **Prerequisites**

The steps below are performed on an AKS cluster with AKS Managed Azure AD Integration enabled. The following will need to be created in Azure AD:

1. User Account for the Developer
2. Group for Cluster Administrators
3. Group for Developers of ProductX

For the cluster administrator you can use the account that has Owner/Contributor access to the 'sandbox' subscription you will test this in, this will allow the request of cluster credentials without the need to assign separate roles.

Once the groups have been created and membership granted the cluster can be deployed as below:

**Obtain the obejctId for the cluster administrators group**

    $ az ad group list --display-name "AKS - ClusterAdmins" --query [].objectId
    [
    "6b47b8f8-81ce-49b3-8b43-c205ab1d3355"
    ]

**Create a new resource group for the cluster**

    $ az group create --name rgAksClusters --location uksouth

**Deploy the Cluster using the objectId from above - Note: this will also add Azure Policy**

    $ az aks create -g rgAksClusters -n myAADCluster --enable-aad --aad-admin-group-object-ids 6b47b8f8-81ce-49b3-8b43-c205ab1d3355 --enable-addons azure-policy --node-count 1

Once the cluster has completed deployment confirm it's visible

    $ az aks list -o table
    Name           Location    ResourceGroup    KubernetesVersion    ProvisioningState
    -------------  ----------  ---------------  -------------------  ------------------- 
    myAADCluster   uksouth     rgAksClusters    1.18.14              Succeeded

## Authentication

We will now step through the process of authenticating against Azure AD. If you would like to validate the group(s) being used for clsuter administration the following CLI commands will provide this information:

    $ az aks show --name myAADCluster --resource-group rgAksClusters --query aadProfile.adminGroupObjectIds
    [
    "6b47b8f8-81ce-49b3-8b43-c205ab1d3355"
    ]
    $ az ad group member list --group 6b47b8f8-81ce-49b3-8b43-c205ab1d3355 --query [].userPrincipalName -o table
    Result
    ----------------------------
    someone@yourdomain.com

**Firstly, confirm you have valid cluster admin credentials**

    $ az aks get-credentials --resource-group rgAksClusters --name myAADCluster
    Merged "myAADCluster" as current context in /home/anwilladmin/.kube/config
 
 The config should now contain the various attributes required for Azure AD authentication:
 
    $ kubectl config view
    
    -- Sample --

    user:
    auth-provider:
      config:
        apiserver-id: 6d4e4ef8-4368-4678-94ff-3960e28e3630
        client-id: 80faf921-1508-4b52-b5ef-a8e7bedfc67a
        config-mode: "1"
        environment: AzurePublicCloud
        tenant-id: ced9c432-317f-46a7-9asd-e473b151220a
      name: azure

We will now send a command to the API Server, this will cause a prompt for signin to Azure AD and issue of an access token

    $ kubectl get pods -n kube-system
    To sign in, use a web browser to open the page https://microsoft.com/devicelogin and enter the code DZSFYGE7W to authenticate

System pods should be displayed following authentication confirming cluster access has been granted.

If you would like to explore how this is being granted you can review the aks-cluster-admin-binding-aad clusterrolebinding that linked cluster-admin ClusterRole to the Azure AD group created earlier.

    $ kubectl describe clusterrolebindings.rbac.authorization.k8s.io aks-cluster-admin-binding-aad

**Granting access to our developer**

We will now step through the process for granting access to our developer, John Smith, if you've previously completed the standalone Kubernetes steps this is closely aligned.

**Create a namespace and deploy a simple pod**

    $ kubectl create namespace productx
    namespace/productx created
    $ kubectl run frontend --image=nginx --port=80 -n productx --labels=app=web
    pod/frontend created

**Create role and binding**

    $ az ad group show --group "AKS - ProductX Developers" --query objectId
    $ kubectl create role developer-role -n productx --resource=pods --verb=get,update,list
    role.rbac.authorization.k8s.io/developer-role created 
    $ kubectl create rolebinding Productx-Devs -n productx --group=70eb9450-f850-4822-b0e8-95f218065352 --role=developer-role
    rolebinding.rbac.authorization.k8s.io/Productx-Devs created

**Login as Developer to confirm RBAC**

    $ az aks get-credentials --resource-group rgAksClusters --name myAADCluster
    $ kubectl get pods
    $ kubectl get pods -n productx

**Confirm no access to default namespace:**
