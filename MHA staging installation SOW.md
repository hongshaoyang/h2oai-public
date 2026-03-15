### Prerequisites

* Kubernetes cluster (v1.30)  
  * Test ingresses  
  * Test storage class  
* `kubectl` installed and configured  
* `helm` (v3+) installed  
* **Base installation files**  
  * **Terraform bundle:** `haic-installer-bundle-24-07-1.tgz`  
  * **Umbrella Helm Chart:** `h2oai-chart-1.2.0.tgz`  
  * **Zalando Helm Chart**: `postgres-operator-1.12.2`  
  * GPU Operator Helm Chart: `gpu-operator-v24.3.0`  
  * ….  
  * ….  
  * ….	  
* Base configuration files  
  * Infra  
    * Postgres operator custom values  
    * Postgres cluster CR yaml  
    * GPU Operator custom values  
  * Terraform bundle  
    * All files under under config/ folder  
    * resources/emissary/crds/emissary-crds.yaml  
    * terraform/modules/applications/appstore/resources/\*.yaml  
    * terraform/modules/applications/mlops-helm/resources/\*.yaml  
    * terraform/modules/applications/telemetry/resources/\*.yaml  
    * terraform/modules/kubernetes/tools/keycloak/\*.yaml  
    * terraform/templates/applications/\*.\*  
    * terraform/templates/kubernetes/\*.\*  
    * AES CRDs  
  * Umbrella bundle  
    * Umbrella 1.2.0 values YAML  
* Certs  
  * CA (to check if MHA is using self-signed)  
  * key  
  * Private cert  
* Credentials  
  * External S3 (PureStorage) credentials  
* License  
* Waveapps  
  * Waveapp configuration files  
    * All app.toml files under waveapps/ folder  
* Customizations  
  * All custom ingresses (TF installer disable ingresses)  
    * eg mlapi  
* k8s resources that needs to be copied out  
  *  mlops \- secret \-  h2oai-mlops-environment  
  * mlops \- deployment \- h2oai-mlops-storage

* **Docker Images:** Present in the private registry accessible from the cluster.  
* **Domain Configuration:**  
  * Domain name configured to point to the ingress service.  
  * We recommend using a wildcard domain, e.g., `sandbox.aluskydomain`, `*.sandbox.aluskydomain`.  
* **List of Domains used in this installation:**  
  * `sandbox.aluskydomain`  
  * `auth.sandbox.aluskydomain`  
  * `authz-gateway.sandbox.aluskydomain`  
  * `h2ogpte.sandbox.aluskydomain`  
  * `modelhub.sandbox.aluskydomain`  
  * `storage.sandbox.aluskydomain`  
  * `sts.sandbox.aluskydomain`  
  * `user-drive.sandbox.aluskydomain`  
  * `workspace-drive.sandbox.aluskydomain`  
* **TLS Certificates:**  
  * We recommend SAN certificates with domains like `sandbox.aluskydomain`, `*.sandbox.aluskydomain`.  
  * Example command:

```shell
openssl req -x509 -nodes -subj '/CN=*.sandbox.aluskydomain' -addext 'subjectAltName = DNS:*.sandbox.aluskydomain, DNS:sandbox.aluskydomain' -newkey rsa:4096 -keyout key.pem -out cert.pem -sha256 -days 3650
```

  *   
* Pre-created namespace (if applicable)
