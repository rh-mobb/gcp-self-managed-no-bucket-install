## Prepare your environment

1. Create two working directories that we will use to store important configuration:
    ```bash
    mkdir -p ocp/oidc ocp/cluster-install
    ``` 

1. Create (or copy an existing) `install-config.yaml` file into the `./ocp` directory. An example install-config.yaml file is included below:
    ```yaml
    apiVersion: v1
    baseDomain: ocp.example.com
    credentialsMode: Manual
    compute:
    - name: worker
      platform:
        gcp:
          serviceAccount: ocp-worker@gcp-project-example.iam.gserviceaccount.com
          secureBoot: true
          type: n2-standard-4
          zones:
          - us-central1-a
          - us-central1-b
          - us-central1-c
          osDisk:
            diskType: pd-ssd
            diskSizeGB: 128
      replicas: 3
    controlPlane:
      name: master
      platform:
        gcp:
          serviceAccount: ocp-control-plane@gcp-project-example.iam.gserviceaccount.com
          secureBoot: true
          type: n2-standard-4
          zones:
          - us-central1-a
          - us-central1-b
          - us-central1-c
          osDisk:
            diskType: pd-ssd
            diskSizeGB: 1024
      replicas: 3
    networking:
      machineNetwork:
      - cidr: 10.0.0.0/16
      serviceNetwork:
       - 172.30.0.0/16
      clusterNetwork:
      - cidr: 10.128.0.0/14
        hostPrefix: 23
    metadata:
      name: cluster
    platform:
      gcp:
        projectID: gcp-project-example
        region: us-central1
        controlPlaneSubnet: ocp-control-plane-subnet
        computeSubnet: ocp-worker-subnet
        network: ocp-vpc
    pullSecret: 'redacted'
    sshKey: redacted
    ```

1. Change directory to the `ocp/oidc` directory. This directory will contain the necessary configuration to allow OpenShift to use Workload Identity Federation to interact with Google Cloud APIs:

    ```bash
    cd ocp/oidc
    ``` 

1. Set environment variables that we will be using to complete the remainder of the steps:

    ```bash
    export GCP_PROJECT=gcp-project-example
    export GCP_REGION=us-central1
    export GCP_SERVICE_ACCOUNT_PREFIX=ocp
    export WORKLOAD_IDENTITY_POOL=ocp-wif-pool
    export WORKLOAD_IDENTITY_PROVIDER=ocp-wif-provider
    export RELEASE_IMAGE=quay.io/openshift-release-dev/ocp-release:4.18.17-x86_64
    ```

## Create the Workload Identity Federation configuration

1. Generate RSA keys for use when setting up the cluster's OIDC provider:

    ```bash
    ccoctl gcp create-key-pair
    ```

1. Create the GCP Workload Identity Federation pool:

    ```bash
    ccoctl gcp create-workload-identity-pool --name=${WORKLOAD_IDENTITY_POOL} --project=${GCP_PROJECT}
    ```

1. Create the necessary configuration for the Workload Identity Federation provider

    ```bash
    ccoctl gcp create-workload-identity-provider --name=${WORKLOAD_IDENTITY_PROVIDER} --region=${GCP_REGION} --project=${GCP_PROJECT} --public-key-file=serviceaccount-signer.public --workload-identity-pool=${WORKLOAD_IDENTITY_POOL} --dry-run
    ```

1. Edit the generated `05-create-workload-identity-provider.sh` script with the updated issuer URL that will contain the OIDC provider configuration:
    - Add the JWK JSON path parameter: `sed -i.bak '2s/$/ --jwk-json-path="04-keys.json"/' 05-create-workload-identity-provider.sh`
    - Update the `--issuer-url` parameter to point to the updated issuer URL.
  
1. Run the edited script:
    ```bash
    ./05-create-workload-identity-provider.sh
    ```

1. Edit the `03-openid-configuration` file:
    - Update the `issuer` and `jwks_uri` parameter to point to the updated issuer URL.

1. Rename and move OIDC configuration files to the proper location you wish to host them at:
    - `04-keys.json` should be stored at the root of the serving directory as `keys.json`
    - `03-openid-configuration` should be stored in the `.well-known` directory with the filename `openid-configuration`.
    - This means that these files should be accessible from `https://<your-oidc-provider-url>/keys.json` and `https://<your-oidc-provider-url>/.well-known/openid-configuration`

### (Optional) Build a container for Cloud Run to use

1. Copy the `keys.json` and the `openid-configuration` files to the container directory

    ```bash
    cp 04-keys.json ../container/keys.json
    cp 03-openid-configuration ../container/openid-configuration
    ```

1. Build the container

    ```bash
    podman build -t us-east1-docker.pkg.dev/gcp-project-example/container-registry/oidc ../container --platform=linux/amd64
    ```

1. Push the container to Artifact Registry.

    ```bash
    podman push us-east1-docker.pkg.dev/gcp-project-example/container-registry/oidc:latest
    ```

1. Create a new Google Cloud Run app and use the above container as a runtime.

## Create the Google Cloud service accounts for the cluster to use

1. Extract the credential requests from the OpenShift release image:

    ```bash
    oc adm release extract --cloud=gcp --credentials-requests ${RELEASE_IMAGE} --to=./credreqs
    ``` 

1. Create the Google Cloud service accounts:

    ```bash
    ccoctl gcp create-service-accounts --name=${GCP_SERVICE_ACCOUNT_PREFIX} --project=${GCP_PROJECT} --credentials-requests-dir=credreqs --workload-identity-pool=${WORKLOAD_IDENTITY_POOL} --workload-identity-provider=${WORKLOAD_IDENTITY_PROVIDER}
    ```
    
1. Edit the `manifests/cluster-authentication-02-config.yaml` file with the updated issuer URL.
     - Change `serviceAccountIssuer:` to the updated issuer URL that will contain the OIDC provider configuration.

## Create the OpenShift installation configuration files

1. Change to the `cluster-install` directory we created earlier:

    ```bash
    cd ../cluster-install
    ```

1. Copy the `install-configl.yaml` into this directory.

    ```bash
    cp ../install-config.yaml .
    ```

1. Create the necessary OpenShift installer manifests

    ```bash
    openshift-install create manifests
    ```

1. Copy the service account configuration manifests and tls directories to our `cluster-install` directory. 

    ```bash
    mkdir tls
    cp ../oidc/manifests/* manifests
    cp ../oidc/tls/* tls
    ```

## Install the cluster

```bash
openshift-install create cluster --log-level=debug
```
