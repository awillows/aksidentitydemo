# Authentication and Authorization in Kubernetes

The purpose of this overview is to demostrate the native options available in Kubernetes to implement authentication and authorization control.

### **Prerequisites**

The steps below are performed on a standalone cluster to which cluster-admin access is available. In my overview the cluster was built using kubeadm on an Azure VM running Ubuntu 18.04. Flannel was used as the overlay network.

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

## Authentication

**Firstly, confirm you have valid cluster credentials**

    $ kubectl config current-context
    kubernetes-admin@kubernetes

**Create a namespace and deploy a simple pod, this will be used to test developer access**

    $ kubectl create namespace productx
    namespace/productx created

    $ kubectl run frontend --image=nginx --port=80 -n productx
    pod/frontend created

We now need to create certificates for the end user. There are various ways to achieve this but to keep it simple we will use the built in Linux tools.

The following creates the key pairs for a user we will call JohnSmith and signing requests in order to have the cluster signing.

**Create Certificates**

    $ mkdir client-certs
    $ openssl genrsa -out client-certs/JohnSmith.key 2048
    Generating RSA private key, 2048 bit long modulus (2 primes)
    .............+++++
    ..+++++
    e is 65537 (0x010001)

    $ openssl req -new -key client-certs/JohnSmith.key -out client-certs/JohnSmith.csr -subj "/CN=JohnSmith/O=productx"

    $ sudo openssl x509 -req -in client-certs/JohnSmith.csr -CA /etc/kubernetes/pki/ca.crt -CAkey /etc/kubernetes/pki/ca.key  -CAcreateserial -out client-certs/JohnSmith.crt -days 45
    Signature ok
    subject=CN = JohnSmith, O = productx
    Getting CA Private Key

We will now add a reference to the certificate to our kubeconfig file so we can use them for validating access.

**Update the kubeconfig file to include newly created certificates**

    $ kubectl config set-credentials JohnSmith --client-certificate client-certs/JohnSmith.crt --client-key client-certs/JohnSmith.key
    User "JohnSmith" set.
    $ kubectl config set-context JohnSmith-context --cluster=kubernetes --namespace=productx --user=JohnSmith
    Context "JohnSmith-context" modified.

**You can now test the auth - this should result in a forbidden error as JohnSmith has not been granted access to the cluster**

    $ kubectl get pods --context=JohnSmith-context 
    Error from server (Forbidden): pods is forbidden: User "JohnSmith" cannot list resource "pods" in API group "" in the namespace "productx"

## Authorization

We now have certificates in place to authenticate JohnSmith. The next step is to use Kubernetes RBAC to grant access to the 'productx' namespace. We will create a role and rolebinding for this purpose (Note: there are anumber of predefined ClusterRoles that are also available the can be bound into a namespace)

**Create roles for JohnSmith**

    $ kubectl create role developer-role -n productx --resource=pods --verb=get,update,list
    role.rbac.authorization.k8s.io/developer-role created
    $ kubectl create rolebinding dev-JohnSmith -n productx --user=JohnSmith --role=developer-role
    rolebinding.rbac.authorization.k8s.io/dev-JohnSmith created

**Confirm JohnSmith access to productx**

    $ kubectl auth can-i list pods -n productx --as=JohnSmith
    yes
    $ kubectl get pods --context=JohnSmith-context
    NAME       READY   STATUS    RESTARTS   AGE
    frontend   1/1     Running   0          23m

## Cleanup

 To remove the objects created via the steps above you can use the commands below:

 **Remove Context and User from kubeconfig**

    $ kubectl config delete-context JohnSmith-context
    deleted context JohnSmith-context from /home/anwilladmin/.kube/config
    $ kubectl config unset users.JohnSmith
    Property "users.JohnSmith" unset.

 **Remove Certificates (Please ensure this directory doesn't contain anything other that the demo certificates prior to removal)**

     $ rm -rf client-certs

**Remove Namespace - this will also cleanup the roles and rolebindings**

    $ kubectl delete ns productx
    namespace "productx" deleted