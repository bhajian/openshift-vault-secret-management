# openshift-vault-secret-management

Openshift is the most advanced container management product as of now which is based on Kubernetes. 
The product is opensource and free to use with a community license. However, the secret management 
in Openshift is a complex problem that is not solved in native Kubernetes and can be solved 
with a third party product such as Hashicorp Vault.

The reason for that is that Kubernetes has Kubernetes secret which is not really a safe place to 
store secrets due to lack of encryption in Kubernetes. Kubernetes secret obfuscates the secrets
using the Base64 encoding. 

Hashicorp Vault is a de facto product designed for secret management. Vault can be integrated with
many different cloud providers and products including Kubernetes which makes it a unique safe secret 
management solution that supports Kubernetes service account to authenticate with Vault. 

In this article, first the installation of Vault in Openshift is described then we show how to integrate
Vault with Kubernetes API to validate service account tokens for authentication with Vault.
Finally, we show how to verify the integration and a simple application is installed in the cluster 
that uses service account to authenticate with Vault and receives secrets from Vault.

## Vault and Kubernetes authentication workflow 
Vault provides a couple of different ways to authenticate the client. There are two main approaches for authenticating a client:

1. Platform integration
2. Trusted orchestration platform

The former is used when we are using vault in integration with one of the major cloud providers: GCP, AWS, Azure. The latter is our focus in the following sections, these methods are described in more detail.

### Openshift + Vault using Kubernetes service account workflow:
Vault and kubernetes authentication is a method to be used when vault can trust kubernetes as a
trusted orchestrator. For more info: https://learn.hashicorp.com/vault/identity-access-management/iam-secure-intro 

![alt text](./kubernetes-vault-multi-tenancy-arc.png "Openshift + Vault architecture")

### Vault app-role workflow:
Approle is another pattern that can be used in order to authenticate with Vault. App role has more flexibility 
to deliver a secret 0 to the container for authenticating with Vault. This method needs two pieces of information 
for authentication with Vault: 1- Secret 2- Role Id 

For more info: https://www.vaultproject.io/docs/auth/approle.html   

![alt text](./app-role-pattern.png "Hashicorp Vault architecture")

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

Now, your client is ready to use for installing Vault server in your Kubernetes cluster. 

### Setup Helm

1- First you need to create a project in Openshift to install tiller:

```
$ oc new-project tiller
 Now using project "tiller" on server "https://...".
 ...
```
 
Setup tiller namespace using the following command:
 
 `$ export TILLER_NAMESPACE=tiller`

2- Install the Helm client locally.

```
$ curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.14.0-linux-amd64.tar.gz | tar xz
 $ cd linux-amd64
 $ ./helm init --client-only
 ...
 $HELM_HOME has been configured at /.../.helm.
 Not installing Tiller due to 'client-only' flag having been set
 Happy Helming!
```
 
 3- Install tiller server:
 
 `oc process -f https://github.com/openshift/origin/raw/master/examples/helm/tiller-template.yaml -p TILLER_NAMESPACE="${TILLER_NAMESPACE}" -p HELM_VERSION=v2.14.0 | oc create -f -`
 
 Note: make sure you set appropriate tiller/helm version in the command above.
 
 ```
 $ oc rollout status deployment tiller
  Waiting for rollout to finish: 0 of 1 updated replicas are available...
  deployment "tiller" successfully rolled out
```
Get the helm version  

```
   $ ./helm version
   Client: &version.Version{SemVer:"v2.14.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}
   Server: &version.Version{SemVer:"v2.14.0", GitCommit:"f6025bb9ee7daf9fee0026541c90a6f557a3e0bc", GitTreeState:"clean"}
```
   
 4- Create a separate project where we’ll install a Helm Chart for Vault
 
 ```
  $ oc new-project vault
  Now using project "vault" on server "https://...".
  ...
```
 
 5- Grant the Tiller server edit access to the current project
 
```
 $ oc policy add-role-to-user edit "system:serviceaccount:${TILLER_NAMESPACE}:tiller"
```
 
 6- Add admin policy privilege to the project:
 
 `oc adm policy add-scc-to-user privileged -z vault -n vault` 
 
 7- Install a Helm Chart following this document: https://www.vaultproject.io/docs/platform/k8s/helm.html
 or https://www.hashicorp.com/blog/announcing-the-vault-helm-chart
 
``` 
  $ git clone https://github.com/hashicorp/vault-helm.git
  $ cd vault-helm 
  $ git checkout v0.1.2
  $ helm install . --name=vault
```

8- Initialize and Unseal Vault:

- Check the status:

``` 
  # Check status
  $ oc exec -it vault-0 -- vault status
```

- Initialize Vault:

```
# Initialize
$ kubectl exec -it vault-0 -- vault operator init -n 1 -t 1
```

- Unseal Vault:

```
# Unseal vault
$ kubectl exec -it vault-0 -- vault operator unseal <unsealkey>
```

### Vault in HA mode:


In order to enable Vault in HA mode, we need to install consul to store helm data in its persistence layer.

![alt text](ha-architecture.png "Hashicorp Vault HA architecture")

For more info: https://www.vaultproject.io/docs/platform/k8s/helm.html
The following document shows how to install consul by its helm chart: https://github.com/hashicorp/consul-helm

or running the following command:
```
  $ git clone https://github.com/hashicorp/consul-helm
  $ cd consul-helm
  $ helm install ./consul-helm
```

Once the consul is installed we can install vault after changing values.yaml and enable the ha mode to enabled.

9- Create a route to access Vault UI:

```
  $ oc expose service/vault

  $ oc get route
```

Now, vault is successfully installed and ready to be integrated with Kubernetes (Openshift).

## Integrating Vault with Openshift

In order to use Kubernetes authentication method implemented by service account tokens, we may need to enable
Kubernetes backend (a.k.a., Kubernetes token review API) with Vault. The following document describes Kubernetes auth
method: https://www.vaultproject.io/docs/auth/kubernetes.html 

In order to enable Openshift auth method using service account we would need to follow these steps:

1- setup Vault client on the jumpbox to point to the Vault server:

you can get $VAULT_KUBERNETES_ADDRESS by running `oc get route` and paste the address
 in the bellow command.

`export VAULT_ADDR=http://$VAULT_KUBERNETES_ADDRESS`

Or run:

`echo "export VAULT_ADDR=http://$Vault_Kubernetes_address" >> ~/.bash_profile`

2- Integrate vault with kubernetes API:
 
``` 
# create a service account
$ oc create sa vault-auth

# add a cluster admin policy to the service account created above
$ oc adm policy add-cluster-role-to-user system:auth-delegator -z vault-auth

# get token and secret in an environment variable
$ secret=`oc describe sa vault-auth | grep 'Tokens:' | awk '{print $2}'`
$ token=`oc describe secret $secret | grep 'token:' | awk '{print $2}'`
$ pod=`oc get pods | grep vault | awk '{print $1}'`

# Save kubernetes service account certificate in a local file
$ oc exec $pod -- cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt >> ca.crt
$ export VAULT_TOKEN=$ROOT_TOKEN_YOU_GET_FROM_VAULT_INIT

# enable kubernetes verification method in vault
$ vault auth enable -tls-skip-verify kubernetes


# Integrate vault with kubernetes token review API 
$ vault write -tls-skip-verify auth/kubernetes/config token_reviewer_jwt=$token kubernetes_host=https://kubernetes.default.svc:443 kubernetes_ca_cert=@ca.crt
rm ca.crt


# Set Vault policy for kubernetes service account:
$ vault write -tls-skip-verify auth/kubernetes/role/test bound_service_account_names=default bound_service_account_namespaces='*' policies=default ttl=1h 
```

Now, we try to test the service account token to authenticate with Vault:

```
$ secret=`oc describe sa default | grep 'Tokens:' | awk '{print $2}'`
$ token=`oc describe secret $secret | grep 'token:' | awk '{print $2}'`
$ vault write -tls-skip-verify auth/kubernetes/login role=test jwt=$token
```

## Testing Vault as secret management solution for containers running in openshift

![alt text](./app-vault.png "Hashicorp Vault HA architecture")

1- Create an app:

```
# Create application policy for app1
cat <<EOF > app1-policy.hcl
  path "secret/app1" {
    capabilities = ["read", "list"]
  }
  path "database/creds/app1" {
    capabilities = ["read", "list"]
  }
EOF

# Create application policy for app2
cat <<EOF > app2-policy.hcl
  path "secret/app2" {
    capabilities = ["read", "list"]
  }
  path "database/creds/app2" {
    capabilities = ["read", "list"]
  }
EOF

# set vault token in your client machine
export VAULT_TOKEN=$YOUR_ROOT_TOKEN

vault policy write app1-policy app1-policy.hcl
vault policy write app2-policy app2-policy.hcl
vault policy read app1-policy
# Write an example static secret
vault secrets enable -path=secret -version=1 kv
vault kv put secret/app1 username=app1 password=supasecr3t
vault kv put secret/app2 username=app2 password=That1sWhatSheSaid
vault read secret/app1

# Configure the app1 role in Vault
vault write "auth/kubernetes/role/app1-role" \
  bound_service_account_names="default,app1" \
  bound_service_account_namespaces="*" \
  policies="app1-policy" ttl=1h

# Configure the app2 role in Vault
vault write "auth/kubernetes/role/app2-role" \
  bound_service_account_names="default,app2" \
  bound_service_account_namespaces="*" \
  policies="app2-policy" ttl=1h

oc new-project vault-demo

oc create sa app1
oc create sa app2

# unset vault token in your client machine
export VAULT_TOKEN=

# Login to app1 role - vault CLI
vault write "auth/kubernetes/login" role="app1-role" \
  jwt="$(oc sa get-token app1)"

# Read a secret using Vault token - vault CLI
VAULT_TOKEN="<token-from-login>" vault read secret/app1

# Login to app1 role - curl
cat <<EOF > payload.json
  { "role":"app1-role", "jwt":"$(oc sa get-token app1)" }
EOF

curl --request POST --data @payload.json \
   "${VAULT_ADDR}/v1/auth/kubernetes/login"

# Read a secret using Vault token - curl
curl -H "X-Vault-Token: <token-from-login>" \
   "${VAULT_ADDR}/v1/secret/app1"

# error: "service account name not authorized"
vault write "auth/kubernetes/login" role="app1-role" \
  jwt="$(oc sa get-token app2)"

# error: "permission denied"
vault write "auth/kubernetes/login" role="app2-role" \
  jwt="$(oc sa get-token app2)"
VAULT_TOKEN="<token-from-login>" vault read secret/app2

# ///////////////////////////////
# Create a deployment.yaml file:
cat <<EOF> deployment.yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: basic-example
  namespace: vault-demo
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: basic-example
    spec:
      serviceAccountName: app1
      containers:
        - name: app
          image: "kawsark/vault-example-init:0.0.7"
          imagePullPolicy: Always
          env:
            - name: VAULT_ADDR
              value: "${VAULT_ADDR}"
            - name: VAULT_ROLE
              value: "app1-role"
            - name: SECRET_KEY
              value: "secret/app1"
            - name: VAULT_LOGIN_PATH
              value: "auth/kubernetes/login"
EOF

# Display contents of deployment.yaml and ensure that the values of 
# these 4 environment variables are correct: VAULT_ADDR, VAULT_ROLE,
# SECRET_KEY and VAULT_LOGIN_PATH. These values will be read by the
# application.
cat deployment.yaml
# Create deployment and check that the application pod is running
oc create -f deployment.yaml
oc get pods


oc get pods
# You will see a pod name starting with "basic-example-"
export pod=<pod_name>
oc logs ${pod}
# Example successful response
[ec2-user@ip-10-0-1-201 ~]$ oc get pods
NAME                            READY     STATUS    RESTARTS   AGE
basic-example-6785d4cc4-qlnpx   1/1       Running   0          3m
[ec2-user@ip-10-0-1-201 ~]$ export pod=basic-example-6785d4cc4-qlnpx
[ec2-user@ip-10-0-1-201 ~]$ oc logs ${pod}
2019/07/20 15:55:32 ==> WARNING: Don't ever write secrets to logs.
2019/07/20 15:55:32 ==>          This is for demonstration only.
2019/07/20 15:55:32 s.ptVI20KkPBAJv0J6VkMM3ZHA
2019/07/20 15:55:32 secret secret/app1 -> &{c2fe72f4-147d-0361-1834-7acca397cc74  2764800 false map[username:app1 password:supasecr3t] [] <nil> <nil>}
2019/07/20 15:55:32 Starting renewal loop
2019/07/20 15:55:32 Successfully renewed: &api.RenewOutput{RenewedAt:time.Time{wall:0x3a21573c, ext:63699234932, loc:(*time.Location)(nil)}, Secret:(*api.Secret)(0xc00006a6c0)}
``` 
## Writing Secrets to Vault using CICD pipeline

### Creating a CICD pipeline to generate secrets and save them in vault using kuberntes service account
1- Create a new cicd project
`oc new-project cicd`
2- Create Jenkins and pipeline infrastructure app in the project

```
$ oc new-app jenkins-persistent
$ oc get routes
```

3- Create a buildconfig to setup your pipeline using our template

```
$ cat <<EOF > cicd.hcl
  path "database/creds/*" {
    capabilities = ["list", "create"]
  }
  path "secret/*" {
    capabilities = ["create", "read", "update", "delete", "list"]
  }
EOF

$ vault policy write cicd cicd.hcl

$ vault write "auth/kubernetes/role/cicd" bound_service_account_names="default,jenkins" bound_service_account_namespaces="*" policies="cicd" ttl=1h

$ oc create -f https://raw.githubusercontent.com/bhajian/openshift-vault-secret-management/master/buildconfig.yaml
```

Once the pipeline is created run the pipeline but it won't deliver the secret to Vault because Jenkins admin needs to
approve Jenkins script to use a certain library. You may need to go to Jenkins configurations and accept script 
approvals and run the pipeline again.

# Appendix

![alt text](./vault-k8s-auth-workflow.png "Hashicorp Vault architecture")

