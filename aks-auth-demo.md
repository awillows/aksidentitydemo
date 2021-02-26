Prerequisites

AKS Cluster with Azure AD integration deployed.
AKS User Role granted to resource group where AKS clusters reside.
Open https://docs.microsoft.com/en-us/azure/role-based-access-control/resource-provider-operations#microsoftcontainerservice

AKS - ClusterAdmins


Login as Cluster Admin

    $ az aks list -o table
    $ az aks show --name myAADCluster --resource-group rgAksClusters --query aadProfile
    $ az ad group show --group 6b47b8f8-81ce-49b3-8b43-c205ab1d3355
    $ az ad group member list --group 6b47b8f8-81ce-49b3-8b43-c205ab1d3355 --query [].userPrincipalName -o table
    $ az aks get-credentials --resource-group rgAksClusters --name myAADCluster
    $ kubectl config view
    $ kubectl get pods -n kube-system

Complete prompt for sign-in, after pods displayed then show the updated token:

    $ kubectl config view

Show ClusterRole binding that is responsible:

    $ az ad group show --group "AKS - ClusterAdmins" --query objectId
    $ kubectl describe clusterrolebindings.rbac.authorization.k8s.io aks-cluster-admin-binding-aad

Create a namespace and deploy a simple pod:

    $ kubectl create namespace productx
    $ kubectl run frontend --image=nginx --port=80 -n productx --labels=app=web

Create role and binding:

    $ az ad group show --group "AKS - ProductX Developers" --query objectId
    $ kubectl create role developer-role -n productx --resource=pods --verb=get,update,list 
    $ kubectl create rolebinding Productx-Devs -n productx --group=70eb9450-f850-4822-b0e8-95f218065352 --role=developer-role

Login as Developer to confirm RBAC:

    $ az aks get-credentials --resource-group rgAksClusters --name myAADCluster
    $ kubectl get pods
    $ kubectl get pods -n productx

Confirm no access to default namespace:

Show Token:

$ kubectl config view - copy/paste to https://jwt.ms/
