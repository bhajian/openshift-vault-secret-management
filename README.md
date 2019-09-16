# openshift-vault-secret-management

Openshift is the most advanced container management product as of now which is based on Kubernetes. 
The product is opensource and free to use with a community license. However, the secret management 
in Openshift is a complex problem that is not solved in native Kubernetes and can be solved 
with a third party product such as Hashicorp Vault.

The reason for that is that Kubernetes has Kubernetes secret which is not really a safe place to 
store secrets due to lack of encryption in Kubernetes. Kubernetes secret obfuscates the secrets
using the Base64 hash function. 

Hashicorp Vault is a de facto product designed for secret management. Vault can be integrated with
many different cloud providers and products including Kubernetes which makes it a unique safe secret 
management solution that supports Kubernetes service account to authenticate with Vault. 

In this article, first the installation of Vault in Openshift is described then we show how to integrate
Vault with Kubernetes API to validate service account tokens for authentication with Vault.
Finally, we show how to verify the integration and a simple application is installed in the cluster 
that uses service account to authenticate with Vault and receives secrets from Vault.

## Openshift Installation

**Openshift** can be installed by following this document: https://docs.openshift.com/container-platform/4.1/installing/installing_aws/installing-aws-default.html 

The installation is pretty much straightforward and easy to follow. You may need an AWS account, register a 
domain name and install the Openshift using your AWS account credentials.

## Install Hashicorp Vault

Hashicorp suggests installing Vault in Kubernetes(Openshift) using their helm charts. However, before starting
using helm and tiller in Openshift you may need to follow the following steps to enable helm and tiller.

You may need an extra EC2 small instance as a client machine. On the client machine you need to install
- OC CLI compatible with the openshift version: https://docs.openshift.com/container-platform/3.6/cli_reference/get_started_cli.html  
- git: https://git-scm.com/book/en/v2/Getting-Started-Installing-Git
- Helm: https://github.com/helm/helm/blob/master/docs/install.md
- Vault CLI compatible with the Vault server version: https://www.vaultproject.io/downloads.html 

### Login to the Openshift using oc CLI
Now, you need to login to your Openshift server following these steps:
https://docs.openshift.com/container-platform/4.1/installing/installing_aws/installing-aws-default.html 

### Setup Helm

1- First you need to create a project in Openshift to install tiller:

`$ oc new-project tiller
 Now using project "tiller" on server "https://...".
 ...`
 
Setup tiller namespace using the following command:
 
 `$ export TILLER_NAMESPACE=tiller`

2- Install the Helm client locally.

`$ curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.9.0-linux-amd64.tar.gz | tar xz
 $ cd linux-amd64
 $ ./helm init --client-only
 ...
 $HELM_HOME has been configured at /.../.helm.
 Not installing Tiller due to 'client-only' flag having been set
 Happy Helming!`
 
 3- Install tiller server:
 
 `oc process -f https://github.com/openshift/origin/raw/master/examples/helm/tiller-template.yaml -p TILLER_NAMESPACE="${TILLER_NAMESPACE}" -p HELM_VERSION=v2.9.0 | oc create -f -`
 
 Note: make sure you set appropriate tiller/helm version in the command above.
 
 `$ oc rollout status deployment tiller
  Waiting for rollout to finish: 0 of 1 updated replicas are available...
  deployment "tiller" successfully rolled out`
  
  `$ ./helm version
   Client: &version.Version{SemVer:"v2.9.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}
   Server: &version.Version{SemVer:"v2.9.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}`
   
 4- Create a separate project where we’ll install a Helm Chart for Vault
 
 `$ oc new-project vault
  Now using project "myapp" on server "https://...".
  ...`
 
 5- Grant the Tiller server edit access to the current project
 
 `$ oc policy add-role-to-user edit "system:serviceaccount:${TILLER_NAMESPACE}:tiller"`
 
 6- 
 
 7-  Install a Helm Chart.
 
 `$ ./helm install https://github.com/jim-minter/nodejs-ex/raw/helm/helm/nodejs-0.1.tgz -n nodejs-ex
  NAME: nodejs-ex
  LAST DEPLOYED: ...
  NAMESPACE: myapp
  STATUS: DEPLOYED
  ...`
  
 

Now, your client is ready to use for installing Vault server in your Kubernetes cluster. 
