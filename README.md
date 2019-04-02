# fabric-kubernetes-resource



A Concourse resource for updating hyperledger fabric chaincode on top of Kubernetes cluster.

*This resource supports AWS EKS.*

## Versions

The version of this resource is based on `zlabjp/kubernetes-resource`.
## Source Configuration

### kubeconfig

- `kubeconfig`: *Optional.* A kubeconfig file.
    ```yaml
    kubeconfig: |
      apiVersion: v1
      clusters:
      - cluster:
        ...
    ```
- `context`: *Optional.* The context to use when specifying a `kubeconfig` or `kubeconfig_file`

### cluster configs

- `server`: *Optional.* The address and port of the API server.
- `token`: *Optional.* Bearer token for authentication to the API server.
- `namespace`: *Optional.* The namespace scope. Defaults to `default`. If set along with `kubeconfig`, `namespace` will override the namespace in the current-context
- `certificate_authority`: *Optional.* A certificate file for the certificate authority.
    ```yaml
    certificate_authority: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
    ```
- `insecure_skip_tls_verify`: *Optional.* If true, the API server's certificate will not be checked for validity. This will make your HTTPS connections insecure. Defaults to `false`.
- `use_aws_iam_authenticator`: *Optional.* If true, the aws_iam_authenticator, required for connecting with EKS, is used. Requires `aws_eks_cluster_name`. Defaults to `false`.
- `aws_eks_cluster_name`: *Optional.* the AWS EKS cluster name, required when `use_aws_iam_authenticator` is true.

## Behavior

### `check`: Do nothing.

### `in`: Do nothing.

### `out`: Control the Kubernetes cluster.

Control the Kubernetes cluster like `kubectl apply`, `kubectl delete`, `kubectl label` and so on.

## Example

```yaml
resource_types:
- name: kube-fabric
  type: docker-image
  source:
    repository: midoblgsm/kube-fabric
    tag: latest
resources:
- name: data-marketplace-chaincode
  type: git
  source:
    uri: git@github.com:lgsvl/data-marketplace-chaincode.git
    branch: dev

- name: fabric-tools-deployment
  type: git
  source:
    uri: git@github.com:lgsvl/data-marketplace-deployment.git
    branch: dev
    paths: [./fabric-tools-deployment-org1.yaml,./fabric-tools-deployment-org2.yaml] # you can add other orgs to make the update in parallel

- name: kubeconfig-deployment #This should contain your kubeconfig file to allow the pipeline to deploy on your behalf
  type: git
  source:
    uri: git@github.com:lgsvl/deployment.git
    branch: master
    paths: [./kubeconfig/config]

- name: version
  type: semver
  source:
    driver: git
    uri: git@github.com:lgsvl/data-marketplace-chaincode.git
    branch: version-dev
    file: VERSION

jobs:
- name: run-units-for-chaincode
  public: true
  plan:
  - get: data-marketplace-chaincode
    trigger: true
  - task: unit-tests
    file: data-marketplace-chaincode/ci/unit_tests.yaml
# bump version
- name: bump-version
  public: true
  plan:
  - get: data-marketplace-chaincode
    trigger: true
    passed: [run-units-for-chaincode]
  - put: version
    params: {bump: patch}

- name: upgrade-chaincode-org1
  plan:
  - get: data-marketplace-chaincode
    passed: [run-units-for-chaincode]
  - get: fabric-tools-deployment
    trigger: false
  - get: kubeconfig-deployment
    trigger: false
  - get: version
    passed: [bump-version]
    trigger: true
  - put: kubernetes-resource
    params:
      kubectl: apply -f fabric-tools-deployment/fabric-tools-deployment-org1.yaml
      wait_until_ready_selector: app=fabric-tools-org1
      kubeconfig_file: kubeconfig-deployment/kubeconfig/config
      namespace: dev
  - put: kubernetes-resource
    params:
      kubectl: exec fabric-tools-org1 -- mkdir -p /opt/gopath/src/github.com/lgsvl/
      kubeconfig_file: kubeconfig-deployment/kubeconfig/config
      namespace: dev

  # copy chaincode to fabric-tools
  - put: kube-fabric-resource
    params:
      kubectl: cp data-marketplace-chaincode fabric-tools-org1:/opt/gopath/src/github.com/lgsvl/
      kubeconfig_file: kubeconfig-deployment/kubeconfig/config
      namespace: dev
      branch: dev
      do_pull: "true" # pull the code before executing kubectl command

# peer install
  - put: kube-fabric-resource
    params:
      kubectl: exec fabric-tools-org1 -- peer chaincode install -n ${CHAINCODE_NAME} -v ${CHAINCODE_VERSION} -p ${CHAINCODE_PATH}
      kubeconfig_file: kubeconfig-deployment/kubeconfig/config
      namespace: dev
      branch: dev
      chaincode_name: dmp
      channel_name: dmp
      chaincode_path: github.com/lgsvl/data-marketplace-chaincode/
      core_peer_address: blockchain-org1peer1:30110
      do_pull: "false"
# peer upgrade
  - put: kube-fabric-resource
    params:
      kubectl: exec fabric-tools-org1 -- peer chaincode upgrade -o ${ORDERER_URL} -C ${CHANNEL_NAME} -n ${CHAINCODE_NAME} -v ${CHAINCODE_VERSION} -p ${CHAINCODE_PATH} -c '{"Args":["init","dmpoken","10000"]}'
      kubeconfig_file: kubeconfig-deployment/kubeconfig/config
      namespace: dev
      branch: dev
      chaincode_name: dmp
      channel_name: dmp
      orderer_url: blockchain-orderer:31010
      chaincode_path: github.com/lgsvl/data-marketplace-chaincode/
      core_peer_address: blockchain-org1peer1:30110
      do_pull: "false"
```


