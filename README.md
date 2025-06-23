## Prepare environment and set variables

Create directories we will be using

```bash
mkdir oidc
mkdir cluster-install
``` 

copy / create an install-config.yaml in this root directory

Change to the OIDC Directory which will contain manifests and settings required to use a custom OIDC provider

```bash
cd oidc
``` 

Set environment variables

```bash
export GCP_PROJECT=mobb-demo
export GCP_REGION=us-east1
export GCP_SERVICE_ACCOUNT_PREFIX=kmc-oidc-sa
export WORKLOAD_IDENTITY_POOL=kmc-wip2
export WORKLOAD_IDENTITY_PROVIDER=kmc-wip-oidc
export RELEASE_IMAGE=quay.io/openshift-release-dev/ocp-release:4.18.18-x86_64
```

Create a GCP Key Pair

```bash
ccoctl gcp create-key-pair
```

Create the GCP workload identity pool

```bash
ccoctl gcp create-workload-identity-pool --name=${WORKLOAD_IDENTITY_POOL} --project=${GCP_PROJECT}
```

## Create resources needed to create an identity provider

```bash
ccoctl gcp create-workload-identity-provider --name=${WORKLOAD_IDENTITY_PROVIDER} --region=${GCP_REGION} --project=${GCP_PROJECT} --public-key-file=serviceaccount-signer.public --workload-identity-pool=${WORKLOAD_IDENTITY_POOL} --dry-run
```

### Edit 05-create-workload-identity-provider.sh
Change --issuer-uri= to the value of the URL that will contain the oidc public configuration files

add the following parameter to the end of the file

```
--jwk-json-path="04-keys.json"
```

Run the above command after editing

```bash
sudo chmod +x 05-create-workload-identity-provider.sh
./05-create-workload-identity-provider.sh
``` 

### Edit 03-openid-configuration file
Change the issuer and jwks_uri to the URL you would like to use.
copy keys.json to the root directory of the url you want to use
copy 03-openid-configuration ( after editing ) to .well-known/openid-configuration to the root directory of the url you want to use.


### ( Optional ) Build a container for Google Run to use

Copy keys.json and the openid configuration to the container directory

```bash
cp 04-keys.json ../container/keys.json
cp 03-openid-configuration ../container/openid-configuration
```

Build the container

```bash
podman build -t us-east1-docker.pkg.dev/mobb-demo/kmc-registry/oidc ../container --platform=linux/amd64
```

Push the container

```bash
podman push us-east1-docker.pkg.dev/mobb-demo/kmc-registry/oidc:latest
```

Create a new Google Cloud Run app and use the above container as a runtime

###
Create credentials requests

```bash
oc adm release extract --cloud=gcp --credentials-requests $RELEASE_IMAGE --to=./credreqs
``` 

### Create service accounts

```bash
ccoctl gcp create-service-accounts --name=${GCP_SERVICE_ACCOUNT_PREFIX} --project=${GCP_PROJECT} --credentials-requests-dir=credreqs --workload-identity-pool=${WORKLOAD_IDENTITY_POOL} --workload-identity-provider=${WORKLOAD_IDENTITY_PROVIDER}
```

## Create install configuration files
Switch to the installer directory we created

```bash
cd ../cluster-install
```

Copy the install-configl.yaml to this directory.

```bash
cp ../install-config.yaml .
```

Create installer manifests

``` bash
openshift-install create manifests
```

Copy OIDC manifests and tls directories over

```bash
mkdir tls
cp ../oidc/manifests/* manifests
cp ../oidc/tls/* tls
```

## Install the cluster

```bash
openshift-install create cluster --log-level=debug
```