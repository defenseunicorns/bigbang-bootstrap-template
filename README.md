# Bigbang Bootstrap Template

This repository contains an example bigbang deployment configuration.

## Pre-Requisites
In order to deploy the bootstrap repository you will need the following:

* Kubernetes cluster ready for [Big Bang](https://repo1.dso.mil/platform-one/big-bang/bigbang/-/blob/master/docs/d_prerequisites.md)
* [Kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) (configured to use the target cluster)
* [Kustomize](https://kubectl.docs.kubernetes.io/installation/kustomize/)
* [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)

The following additional dependencies are recommended for development:
* [`Flux`](https://fluxcd.io/docs/installation/)
* [SOPS](https://github.com/mozilla/sops)
* [Iron Bank Credentials](https://registry1.dso.mil)

## Glossary
| Term | Summary | Docs |
| ---- | ----------- | ---- |
| Environment | Referencing separate clusters or deployments (e.g. dev, staging and prod) | n/a |
| Bases and Overlays | A hierarchy of configuration settings which allows custom settings to be defined in the "base", while allowing environment specific configuration in the "overlays" | [link](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/#bases-and-overlays) |
|

## Project Structure
The top level folder structure is as follows:
```bash
.
├── apps/
│   ├── pod-info/
│   └── ...
├── bigbang-base/
└── environment-overlays/
    ├── dev/
    │   └── flux-system/
    └── ...
```

### Apps
The `apps` directory should contain a folder for each package (e.g. `podinfo`) to be deployed alongside bigbang. Each package folder should contain the following required items:

* `deploy.yaml` to define the following manifests
    * `namespace` - this should logically match the package name
    * `GitRepository` - this should reference an upstream helm chart
    * `HelmRelease` - this should reference the defined `GitRepository` and define a configmap to be populated by `kustomize`
* `values.yaml` to define common configuration settings (used by `kustomize` to populate the config map defined in the `HelmRelease`)
* Additional package specific required resources such as secrets, custom credentials - some of which may need to be protected by SOPS encryption.
> NOTE: Package definitions in this repository should be limited to manifests which are required to configure a deployment. The manifests which actually define the structure of the app should be maintained in the upstream helm chart.
* `kustomization.yaml` which references the defined resources within the package and handles any required map generation or patches.

For certmanager, the following are required:

* `certificates/cert.yaml` - Defines certmanager Certificate. Update `dnsNames` field with appropriate values for deployment.
* `certificates/issuer.yaml`- Defines certmanager ClusterIssuer. Use acme staging server until ready for prod, then replace with acme server. Update
`solvers/selector/dnsZones` with appropriate values for deployment. Update `dns01` field based on dns provider per instructions in (https://cert-manager.io/docs/configuration/acme/dns01/).
* `certificates/kustomization.yaml`- references the defined resources within the certificates folder.
* `certmanager/cert-manager-helm.yaml` - defines namespace, HelmRepository and HelmRelease for deploying certmanager.
* `certmanager/kustomization.yaml`- references the defined resources within the certmanager folder

### Bigbang Base
`bigbang-base` defines the common configuration elements for bigbang. This configuration will be used as the basis for bigbang deployments across each environment. Contents:

* `ips.yaml` - the definition of the image pull secret ("ips") expected by Big Bang containing valid Iron Bank pull credentials. This file should be protected by SOPS encryption.
* `values.yaml` - common Big Bang configuration settings
* `kustomization.yaml` - which references the defined resources and handles any required map generation or patches.

### Cluster Overlays
`cluster-overlays` allow each deployment environment (e.g. dev, staging, prod etc) to be customized. Each environment should contain the following required items:

* `values.yaml` with non-sensitive bigbang configurations.
* `kustomization.yaml` which references `bigbang-base` and any desired packages as `resources`
* Additional environment specific required resources such as secrets, custom credentials - some of which may need to be protected by SOPS encryption. For certmanager, include 
certmanager-secret for service account key per https://cert-manager.io/docs/configuration/acme/dns01/.

### Misc. / Additional
* `.sops.yaml` contains the sops configuration rules mapping folders (or specific files) to SOPS encryption/decryption resources. 

## Deployment Instructions
Assumptions:
* An existing Kubernetes clusterhttps://cert-manager.io/docs/configuration/acme/dns01/
* User has kubectl access to the cluster
* User has kustomize installed 

The following steps can be used to install flux and deploy bigbang:
1. Create the flux system namespace 
    * `kubectl create ns flux-system`

2. Add required secrets to the `flux-system` namespace:
    * `private-registry` - IronBank pull credentials
      ```
      kubectl create secret docker-registry private-registry -n flux-system \
        --docker-server=registry1.dso.mil \
        --docker-username=<username> \
        --docker-password=<password> \
        --docker-email=<email>
      ```
    * `flux-system` - Git repository credentials ([ssh credentials example](https://fluxcd.io/docs/components/source/gitrepositories/#ssh-authentication) - other formats, such as https are possible) to allow flux system components to pull from the bootstrap/configuration repository
      ```
      kubectl create secret generic -n flux-system flux-system \
        --from-file=identity=./identity \
        --from-file=identity.pub=./identity.pub \
        --from-file=known_hosts=./known_hosts
      ```
    * `sops-creds` - AWS access key to allow flux to decrypt SOPS protected resources
      ```
      kubectl create secret generic -n flux-system sops-creds --from-literal=access_key_id=<key_id> --from-literal=access_key_secret=<key>
      ```
      > The `sops-creds` secret is optional / situational, and may not be required for your environment.

3. Install flux
    ```
    kustomize build 'https://repo1.dso.mil/platform-one/big-bang/bigbang.git/base/flux?ref=1.18.0' | kubectl apply -f -
    ```
    > NOTE: the version `ref` here should match the one defined in `cluster-overlays/<env>/flux-system/kustomization.yaml` which in turn should match the targeted version of bigbang defined in `bigbang-base/kustomization.yaml`

4. Install the GitRepository and Kustomization objects for the environment
    ```
    kustomize build https://github.com/defenseunicorns/bigbang-bootstrap-template.git/environment-overlays/dev/flux-system?ref=main | kubectl apply -f -
    ```