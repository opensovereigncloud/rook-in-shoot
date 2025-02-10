# Ceph Cluster Deployment Documentation on Gardener Shoot

This document describes how to deploy a Ceph cluster on a Gardener Shoot and configure it for access from other Shoots. It specifically covers **High Availability (HA) configurations within the Ceph cluster itself (using Availability Zones)** and demonstrates how to set up **external access for consumer Shoots** residing in different Gardener projects. This documentation focuses on the essential configurations and provides a streamlined `Shoot` YAML for clarity, while highlighting critical settings for robust deployments.  This guide demonstrates how a Ceph cluster in a dedicated **Service Project (svc)** can be accessed by a **Test Shoot (test) in a Development Project (dev)**.  The Ceph cluster in the `svc` project is designed for HA using multiple Availability Zones, while the `test` Shoot example represents a consumer accessing it.

**Important Disclaimer:**  This documentation is provided as a **starting point to understand the possibilities of deploying Ceph on Open Sovereign Cloud (OSC)**.  **Open Sovereign Cloud (OSC) provides no official support or warranty for this specific Ceph deployment method.**  Use this documentation and the configurations at your own risk.

**Important Notes:**

*   **Rook Version and Internal Load Balancer:** This deployment leverages a specific, modified version of Rook that enables the use of Open Sovereign Cloud (OSC) internal load balancers.  This functionality is not available in the standard upstream Rook release. The source code for this modified Rook version can be found on GitHub: [https://github.com/opensovereigncloud/rook/commits/release-1.16-osc/](https://github.com/opensovereigncloud/rook/commits/release-1.16-osc/).  This modified version allows you to configure an internal load balancer for Ceph services using the `internalLoadBalancerService` setting within the Rook `Cluster` custom resource, as demonstrated below.
*   **OSC PaaS Release Dependency:** This deployment requires **OSC PaaS Release >= v1.0.0** due to the dependency on internal load balancer features and potentially other OSC-specific functionalities. Ensure your Gardener environment meets this minimum OSC PaaS release version.

**Prerequisites**

Before you begin, ensure you have the following prerequisites in place:

1.  **Access to a Gardener Environment**: You need access to a Gardener environment where you can create and manage Shoots.
2.  **kubectl**:  The Kubernetes command-line tool (`kubectl`) must be installed and configured to interact with your Gardener environment and Shoots.
3.  **kustomize**:  Kustomize should be installed for Kubernetes configuration management.
4.  **KUBECONFIG**: You should be able to access both the central Gardener environment and the Shoots using `KUBECONFIG`. Login to ceph and test shoots is always done via export of `KUBECONFIG`.
5.  **osc-rook-in-shoot repository**:  You should have access to the `osc-rook-in-shoot` repository which contains necessary deployment configurations and scripts.
6.  **jq**: `jq` command-line JSON processor is needed for some scripts.

**Note on Tools:**  Python 3 and the `ceph-tools` pod are necessary for exporting external cluster configurations. These are deployed as part of the Rook Ceph deployment process itself and do not need to be pre-installed prerequisites on your local machine or Gardener environment.

## Setup

This setup guide is divided into several steps to deploy a Ceph cluster within a Gardener Shoot.

### 1. Shoot Creation

First, you need to create a Gardener Shoot for your Ceph cluster in a dedicated **Service Project (svc)**. Below is an example `Shoot` manifest highlighting the important aspects of worker pools and networking configuration for a Ceph deployment. This Shoot will host the Ceph cluster that can be accessed by other Shoots in different projects.

```yaml
kind: Shoot
apiVersion: core.gardener.cloud/v1beta1
metadata:
  name: ceph # Name of your ceph service shoot
  namespace: garden-svc # Namespace of your Service Project (svc)
spec:
  kubernetes:
    version: "1.29.12" # Kubernetes version - important to specify
  networking:
    nodes: 10.10.0.0/25 # Node CIDR - ensure this does not overlap with other networks!
  provider:
    type: onmetal # Provider type
    workers: # Defines worker pools, each AZ as a pool
      - name: rook-az1 # Worker pool for AZ1
        zones:
          - ffm1-stage # Availability Zone
        machine:
          type: vm-dedicated-4-16 # Machine type for AZ1 workers
        labels: # Labels for worker nodes in this pool
          dedicated: rook
          role: osd-node
        taints: # Taints to dedicate nodes for Rook OSDs
          - key: dedicated
            value: rook
            effect: NoSchedule
        volume: # Volume configuration for worker nodes
          type: fast
          size: 50Gi
        systemComponents: # Control system component allowance on these workers
          allow: false # Disallow system components on dedicated OSD workers
        machineControllerManager: # Settings for Machine Controller Manager specific to this pool
          machineHealthTimeout: 168h0m0s # Extended machine health timeout to prevent premature node recreation
          maxEvictRetries: 30
      - name: rook-az2 # Worker pool for AZ2
        zones:
          - ffm2-stage # Availability Zone
        machine:
          type: vm-dedicated-4-16 # Machine type for AZ2 workers
        labels: # Labels for worker nodes in this pool
          dedicated: rook
          role: osd-node
        taints: # Taints to dedicate nodes for Rook OSDs
          - key: dedicated
            value: rook
            effect: NoSchedule
        volume: # Volume configuration for worker nodes
          type: fast
          size: 50Gi
        systemComponents: # Control system component allowance on these workers
          allow: false # Disallow system components on dedicated OSD workers
        machineControllerManager: # Settings for Machine Controller Manager specific to this pool
          machineHealthTimeout: 168h0m0s # Extended machine health timeout to prevent premature node recreation
          maxEvictRetries: 30
      - name: rook-az3 # Worker pool for AZ3
        zones:
          - ffm3-stage # Availability Zone
        machine:
          type: vm-dedicated-4-16 # Machine type for AZ3 workers
        labels: # Labels for worker nodes in this pool
          dedicated: rook
          role: osd-node
        taints: # Taints to dedicate nodes for Rook OSDs
          - key: dedicated
            value: rook
            effect: NoSchedule
        volume: # Volume configuration for worker nodes
          type: fast
          size: 50Gi
        systemComponents: # Control system component allowance on these workers
          allow: false # Disallow system components on dedicated OSD workers
        machineControllerManager: # Settings for Machine Controller Manager specific to this pool
          machineHealthTimeout: 168h0m0s # Extended machine health timeout to prevent premature node recreation
          maxEvictRetries: 30
      - name: pool1 # General purpose worker pool
        zones: # Can span multiple AZs
          - ffm1-stage
          - ffm2-stage
          - ffm3-stage
        machine:
          type: vm-dedicated-2-4 # Machine type for general pool
        systemComponents: # Control system component allowance on these workers
          allow: true # Allow system components on general purpose workers

```

**Key parts of the `Shoot` manifest (Detailed Explanation):**

*   **`spec.kubernetes.version`**: Specifies the Kubernetes version for your Shoot cluster.  Choose a version compatible with Rook and your Gardener environment.
*   **`spec.networking.nodes`**: Defines the Node CIDR for your Shoot's network (`10.10.0.0/25` in this example). **Critical**:  Ensure that the Node CIDR you specify here **does not overlap** with any other network ranges in your environment, including other Shoot networks or on-premise networks, to avoid network routing conflicts.  **Node CIDRs of different Shoots must not overlap!**
*   **`spec.provider.workers`**: This section is crucial and defines the worker pools for your Shoot. For a Ceph deployment, carefully configure the worker pools:
    *   **Dedicated AZ Worker Pools (`rook-az1`, `rook-az2`, `rook-az3`)**:  These pools are designed for Ceph OSD nodes to ensure High Availability across Availability Zones.  Each pool is configured with:
        *   **`zones`**: Specifies the Availability Zone where nodes in this pool will be created (e.g., `ffm1-stage`).
        *   **`machine.type`**:  Defines the machine type for the worker nodes. For OSD nodes, choose machine types with sufficient resources (e.g., `vm-dedicated-4-16`).
        *   **`labels`**:  `dedicated: rook` and `role: osd-node` labels are added to nodes in these pools. These labels are important for node selection and workload placement within the cluster (e.g., Rook OSD pods can target these labels).
        *   **`taints`**:  The `dedicated=rook:NoSchedule` taint is applied. This ensures that only workloads with a matching toleration (like Rook OSD pods) will be scheduled on these nodes, effectively dedicating them for Ceph OSDs.
        *   **`volume`**: Configures the root volume for worker nodes. For OSD nodes, ensure sufficient size and appropriate `type` (e.g., `fast`, `volume types` depend on your cloud provider).
        *   **`systemComponents.allow: false`**:  This setting is **highly recommended** for dedicated OSD worker pools. By setting `allow: false`, you prevent Gardener system components (like kube-proxy, coredns, etc.) from being scheduled on these dedicated OSD nodes, ensuring that resources are primarily available for Ceph.
        *   **`machineControllerManager.machineHealthTimeout: 168h0m0s`**:  This setting significantly increases the `machineHealthTimeout`.  **Importance**: In cloud environments, temporary infrastructure issues can sometimes cause nodes to become temporarily unhealthy. The default `machineHealthTimeout` might trigger premature node deletion and recreation in such situations, potentially leading to unnecessary cluster disruptions, especially during Ceph OSD deployment and initial setup. **Extending this timeout to `168h0m0s` (7 days) provides a much longer grace period, reducing the risk of automatic node recreation due to transient issues and enhancing cluster stability**, especially during the initial Ceph deployment phase. Adjust this value based on your environment and risk tolerance.
    *   **General Purpose Worker Pool (`pool1`)**:  This pool is an example of a worker pool for general workloads or allowing system components. It spans multiple AZs and uses a different, potentially smaller, `machine.type` (`vm-dedicated-2-4` in this example).  `systemComponents.allow: true` in this pool would allow system components to be scheduled on these nodes if needed.

Apply this `Shoot` manifest using `kubectl`:

```bash
kubectl apply -f <shoot-manifest.yaml> # Replace <shoot-manifest.yaml> with the actual file name
```

Wait for the Shoot to be successfully created and the cluster to be ready. You can check the status using Gardener Dashboard or `kubectl`.

### 2. Deploy Rook Operator

Navigate to the `<path-to-osc-rook-in-shoot>/deployment` directory within your `osc-rook-in-shoot` repository.  **Replace `<path-to-osc-rook-in-shoot>` with the actual path to your local repository checkout.**

```bash
cd <path-to-osc-rook-in-shoot>/deployment # Adjust the path to your local repository
```

Deploy the Rook operator using `kustomize` and `kubectl`:

```bash
kustomize build rook-operator | kubectl apply -f -
```

This command builds the Rook operator manifests using `kustomize` and applies them to your Kubernetes cluster, setting up the necessary operator to manage the Ceph cluster. **This deployment uses a modified Rook version (release-1.16-osc, see Important Note at the beginning of this document) that enables the use of OSC internal load balancers.**

### 3. Prepare Cluster for Ceph

Prepare the cluster for Ceph deployment by applying necessary configurations.

```bash
kustomize build cluster-prepare | kubectl apply -f -
```

This step sets up essential prerequisites within the cluster, such as RBAC roles and configurations needed by the Rook operator to deploy and manage Ceph.

### 4. Configure Ceph Cluster

Configure the Ceph cluster itself by applying the cluster manifests.  **This configuration will enable the internal load balancer for Ceph services, leveraging the capabilities of the modified Rook version.**

Within your Ceph `Cluster` manifest (part of `kustomize build cluster`), ensure you have the following `network` settings to enable the internal load balancer:

```yaml
network:
  connections:
    encryption:
      enabled: false # Encryption settings - adjust as needed
    compression:
      enabled: false # Compression settings - adjust as needed
  provider: host # Network provider - 'host' is typical for on-premise/internal clouds
  internalLoadBalancerService: # Enable internal load balancer service
    enabled: true
    labels:
      service.beta.kubernetes.io/onmetal-load-balancer-internal: "true" # OSC internal LB annotation
```

Apply the cluster manifests:

```bash
kustomize build cluster | kubectl apply -f -
```

This command deploys the core Ceph cluster components, including monitors (MONs), managers (MGRs), and Object Storage Devices (OSDs), based on the configurations within the `cluster` directory, now with the internal load balancer enabled.

### 5. Export Configuration for External Cluster (Test Shoot in Development Project)

If you want to access this Ceph cluster from a **Test Shoot** in a **Development Project (dev)**, you need to export the configuration required for external access. **This step requires using Python 3 and the `ceph-tools` pod, which are deployed as part of the Rook operator setup. You do not need to install these separately.**

First, find the name of the `rook-ceph-tools` pod in the `rook-ceph` namespace. This pod is deployed by the Rook operator and contains the necessary tools, including Python 3, to generate external cluster configurations:

```bash
TOOLS_POD=$(kubectl get po -n rook-ceph -l app=rook-ceph-tools -o jsonpath='{.items[0].metadata.name}')
echo "Tools pod name: $TOOLS_POD"
```

Once you have the pod name, execute a bash shell inside the tools pod:

```bash
kubectl exec -n rook-ceph $TOOLS_POD -i -t -- bash
```

Now, **inside the `rook-ceph-tools` pod's shell**, run the configuration export script.  This script uses Python 3, which is available within the tools pod image:

```bash
python3 ./etc/rook-external/create-external-cluster-resources.py \
  --cephfs-filesystem-name cephfs \
  --namespace rook-ceph \
  --format bash \
  --rbd-data-pool-name .mgr \
  --alias-rbd-data-pool-name data \
  --k8s-cluster-name test \
  --restricted-auth-permission true \
  --cephfs-data-pool-name cephfs-erasurecoded
```

**Note**: In this script, `--k8s-cluster-name` is set to `test` to reflect the name of our consumer Shoot.

This script generates shell scripts and environment variables containing the necessary configuration details to connect to the Ceph cluster from an external Kubernetes environment.  **Ensure you execute this script within the `rook-ceph-tools` pod environment as it relies on specific tools and configurations available there.**

After running the script inside the tools pod, you will get output that you need to use in your shell environment (outside the tools pod) for the next steps.  Set the following environment variables. You can copy and paste the output from the script, or adjust the values as needed for your specific Test Shoot setup. **Note:** These are example values and must be adjusted to match your actual Ceph cluster and environment.

```bash
export ARGS="[Configurations]
verbose = True
k8s-cluster-name = test
namespace = rook-ceph
rgw-pool-prefix = default
restricted-auth-permission = true
format = bash
cephfs-filesystem-name = cephfs
cephfs-metadata-pool-name = cephfs-metadata
cephfs-data-pool-name = cephfs-erasurecoded
rbd-data-pool-name = .mgr
alias-rbd-data-pool-name = data
"
export NAMESPACE=rook-ceph
export ROOK_EXTERNAL_FSID=0d74224d-1819-4817-80f4-eb5809fe63de
export ROOK_EXTERNAL_USERNAME=client.healthchecker
export ROOK_EXTERNAL_CEPH_MON_DATA=a=10.10.0.7:6789 # Adjust to your monitor IP or hostname
export ROOK_EXTERNAL_USER_SECRET=AQCy+6lnWgpvABAAu7X4GVBKFIghDpe52bxubQ== # Replace with the actual secret
export ROOK_EXTERNAL_DASHBOARD_LINK=[https://10.10.0.7:8443/](https://10.10.0.7:8443/) # Adjust to your dashboard link
export CSI_RBD_NODE_SECRET=AQCy+6lnimczARAAi0FjSpZcsHJYW9eUtZ1SPQ== # Replace with the actual secret
export CSI_RBD_NODE_SECRET_NAME=csi-rbd-node-test-data
export CSI_RBD_PROVISIONER_SECRET=AQCy+6lnrBLXARAA7jMwvh/6DzxnzBQPYJeAdA== # Replace with the actual secret
export CSI_RBD_PROVISIONER_SECRET_NAME=csi-rbd-provisioner-test-data
export CEPHFS_POOL_NAME=cephfs-erasurecoded
export CEPHFS_METADATA_POOL_NAME=cephfs-metadata
export CEPHFS_FS_NAME=cephfs
export RESTRICTED_AUTH_PERMISSION=true
export CSI_CEPHFS_NODE_SECRET=AQCy+6ln9byWAhAANbt1R1ZmiJMywdQiBnjTMg== # Replace with the actual secret
export CSI_CEPHFS_PROVISIONER_SECRET=AQCy+6lnqgZXAxAA9Utr3qXdkDcNBzhwKCoweA== # Replace with the actual secret
export CSI_CEPHFS_NODE_SECRET_NAME=csi-cephfs-node-test-cephfs
export CSI_CEPHFS_PROVISIONER_SECRET_NAME=csi-cephfs-provisioner-test-cephfs
export MONITORING_ENDPOINT=10.10.0.7 # Adjust to your monitor IP for monitoring
export MONITORING_ENDPOINT_PORT=9283
export RBD_POOL_NAME=.mgr
export RGW_POOL_PREFIX=default
```

If you are using a LoadBalancer service for `external-mgr` service in `rook-ceph` namespace, you can dynamically determine the monitoring endpoint IP:

```bash
MGR_IP=$(kubectl get svc -n rook-ceph external-mgr -o json | jq -r '.status.loadBalancer.ingress[0].ip')
echo "export MONITORING_ENDPOINT=$MGR_IP"
echo "export ROOK_EXTERNAL_DASHBOARD_LINK=https://$MGR_IP:8443/"
```

Alternatively, or to override the LoadBalancer IP, you can manually set `MONITORING_ENDPOINT` and `ROOK_EXTERNAL_DASHBOARD_LINK` with the correct IP or hostname and dashboard link for your Ceph cluster.  Ensure `ROOK_EXTERNAL_CEPH_MON_DATA` is also correctly configured, typically pointing to one of your Ceph monitor IPs and ports.

```bash
export MONITORING_ENDPOINT=10.10.0.2 # Replace with actual monitoring endpoint IP/hostname
export ROOK_EXTERNAL_DASHBOARD_LINK=[https://10.10.0.2:8443/](https://10.10.0.2:8443/) # Replace with actual dashboard link
export ROOK_EXTERNAL_CEPH_MON_DATA=a=10.10.0.3:3300 # Replace with actual monitor data, e.g., a=MONITOR_IP:MONITOR_PORT
```

### 6. Create Peering between Shoots (For External Access)

To enable access to the Ceph cluster from the **Test Shoot** in the **Development Project (dev)**, you need to establish Shoot Peering between the Ceph Service Shoot and the Test Shoot.  **Important Note:** Currently, setting up Shoot Peering as described below requires a change request to OSC Operations. We are actively working to enable self-managed Shoot Peering in the future, which will simplify this process significantly.

In this example, we are peering the Ceph Service Shoot in the `svc` project with a **Test Shoot (`test`) in a Development Project (`dev`)**. This `test` Shoot will act as the consumer of the Ceph cluster.  You can extend this peering to multiple consumer Shoots in different projects as needed.

Create the following `ShootPeering` resources in both relevant Shoot namespaces.

**In the Ceph Service Shoot's namespace (`shoot--garden-svc--ceph`):**

```yaml
apiVersion: [cluster.osc.t-systems.com/v1](https://www.google.com/search?q=https://cluster.osc.t-systems.com/v1)
kind: ShootPeering
metadata:
  name: svc-ceph--garden-dev-test # Name for peering, adjust as needed, clearly indicating source and target
  namespace: shoot--garden-svc--ceph # Namespace of the Ceph Service Shoot (svc project)
spec:
  # Left side - Ceph Service Shoot configuration (svc project)
  shoot_namespace: smr30-0-shoots
  shoot_network: shoot--garden-svc--ceph
  prefix: shoot--garden-svc--ceph # Prefix for network identification

  # Right side - Development Project Shoot (dev project, test Shoot) configuration
  namespace: smr30-0-shoots
  network: shoot--garden-dev--test # Corrected project name: garden-dev
```

Apply this `ShootPeering` manifest within the Ceph Service Shoot's namespace using `kubectl`.

**In the Development Project Shoot's namespace (`shoot--garden-dev--test`):**

```yaml
apiVersion: [cluster.osc.t-systems.com/v1](https://www.google.com/search?q=https://cluster.osc.t-systems.com/v1)
kind: ShootPeering
metadata:
  name: garden-dev-test--svc-ceph # Name for peering, adjust as needed, clearly indicating target and source
  namespace: shoot--garden-dev--test # Namespace of the Development Project Shoot (dev project)
spec:
  # Left side - Development Project Shoot (dev project, test Shoot) configuration
  shoot_namespace: smr30-0-shoots
  shoot_network: shoot--garden-dev--test # Corrected project name: garden-dev
  prefix: shoot--garden-dev--test # Prefix for network identification

  # Right side - Ceph Service Shoot configuration (svc project)
  namespace: smr30-0-shoots
  network: shoot--garden-svc--ceph # Note: svc instead of tsi
```

Apply this `ShootPeering` manifest within the Development Project Shoot's namespace using `kubectl`.

**Important Networking Consideration:** Ensure that the `spec.networking.nodes` CIDR blocks defined in your Shoot manifests **never overlap**. Overlapping CIDRs will lead to severe networking issues and prevent successful peering and communication between your Shoots and potentially other networks. Plan your network ranges carefully to avoid conflicts. **Node CIDRs of different Shoots must not overlap!**

### 7. Import Configuration for External Cluster

If you performed step 5 to export configuration for external access, now you need to import that configuration into the **Development Project's (dev)** Kubernetes cluster (specifically, the `test` Shoot) which will consume the Ceph storage.

First, ensure you are currently logged in to the **Development Project's (dev)** consumer/external cluster (the `test` Shoot) using `kubectl`.

Navigate to the `<path-to-osc-rook-in-shoot>/deployment` directory in your `osc-rook-in-shoot` repository.  **Replace `<path-to-osc-rook-in-shoot>` with the actual path to your local repository checkout.**

```bash
cd <path-to-osc-rook-in-shoot>/deployment # Adjust the path to your local repository
```

Apply the external Rook operator manifests in the **Development Project's (dev)** cluster:

```bash
kustomize build rook-operator-external | kubectl apply -f -
```

Next, execute the import script. Ensure all the `export` commands from step 5 have been run in your current shell session to set the required environment variables.

```bash
./scripts/import-external-cluster.sh
```

Finally, apply the external cluster configuration manifests in the **Development Project's (dev)** cluster:

```bash
kustomize build cluster-external | kubectl apply -f -
```

This completes the configuration import on the **Development Project's (dev)** cluster, enabling it to access the external Ceph cluster deployed in the **Service Project (`svc`)** Gardener Shoot.

### 8. Test the Deployment

To validate the Ceph deployment and external access from the **Development Project's (dev)** `test` Shoot to the **Service Project's (`svc`)** `ceph` Shoot, apply the provided test manifests in the **`garden-dev` Shoot**.

```bash
kustomize build test | kubectl apply -f -
```
