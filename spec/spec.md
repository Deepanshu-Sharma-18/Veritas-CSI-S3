## Objective

The objective of this CSI specification is to define a standard interface for using a local file-system cache to improve the performance and reduce costs associated with accessing data from S3 object storage. The specification allows Kubernetes (K8S) workloads to seamlessly utilize this caching mechanism by integrating it into the Container Storage Interface (CSI). Additionally, the specification provides flexibility in configuring the caching behavior, including write-back or write-through modes, and introduces the possibility of leveraging AI/ML for predictive caching based on access patterns or workload analysis.

## Terminology

| Term              | Definition                                                                                                           |
| ----------------- | -------------------------------------------------------------------------------------------------------------------- |
| Volume            | A unit of storage that will be made available inside of a CO-managed container, via the CSI.                         |
| Block Volume      | A volume that will appear as a block device inside the container.                                                    |
| Mounted Volume    | A volume that will be mounted using the specified file system and appear as a directory inside the container.        |
| CO                | Container Orchestration system, communicates with Plugins using CSI service RPCs.                                    |
| SP                | Storage Provider, the vendor of a CSI plugin implementation.                                                         |
| RPC               | [Remote Procedure Call](https://en.wikipedia.org/wiki/Remote_procedure_call).                                        |
| Node              | A host where the user workload will be running, uniquely identifiable from the perspective of a Plugin by a node ID. |
| Plugin            | Aka “plugin implementation”, a gRPC endpoint that implements the CSI Services.                                       |
| Plugin Supervisor | Process that governs the lifecycle of a Plugin, MAY be the CO.                                                       |
| Workload          | The atomic unit of "work" scheduled by a CO. This MAY be a container or a collection of containers.                  |

## Goals in MVP

Enable dynamic provisioning and deprovisioning of a volume that uses a local file-system as a cache for S3 object storage.
Support attaching and detaching volumes from nodes.
Mount and unmount volumes from nodes.
Sync data between the local cache and the S3 backend.
Configure the caching mode (write-back or write-through).
Optionally, integrate AI/ML models for predictive caching.

# CSI Plugin Capabilities

The S3 Object Storage Cache CSI Plugin MUST expose the following capabilities:

Dynamic Provisioning: Enable dynamic provisioning of cache volumes to store S3 object data.
Volume Expansion: Support for expanding the size of cache volumes to accommodate growing data.
Mounting and Unmounting: Ability to mount and unmount cache volumes to/from Kubernetes Pods.
Caching Operations: Provide mechanisms for caching operations such as read, write, and synchronization with S3 object storage.
Configuration Options: Allow configuration of caching behavior, including write-back or write-through modes.
Predictive Caching (Optional): Optionally support AI/ML-based predictive caching to optimize cache utilization based on access patterns or workload analysis.

# CSI RPC Interface

## Identity Service

GetPluginInfo: This RPC returns information about the S3 Object Storage Cache CSI Plugin, such as its name, version, and other metadata.

GetPluginCapabilities: Retrieves the capabilities supported by the Plugin, providing information about the functionalities it offers, such as dynamic provisioning, volume expansion, caching operations, and predictive caching.

Probe: Checks the availability and readiness of the Plugin, allowing Kubernetes to determine whether the Plugin is healthy and operational.

## Controller Service

CreateVolume: Creates a cache volume for storing S3 object data. This RPC provisions a new cache volume with specified parameters, such as size and access mode.

DeleteVolume: Deletes a cache volume, removing it from the storage system. This RPC is used to release resources associated with a cache volume that is no longer needed.

ControllerPublishVolume: Publishes a cache volume to a Kubernetes Pod, making it available for use by applications running within the Pod. This RPC establishes the connection between the cache volume and the Pod.

ControllerUnpublishVolume: Unpublishes a cache volume from a Kubernetes Pod, terminating the connection between the volume and the Pod. This RPC is called when a Pod no longer requires access to the cache volume.

ValidateVolumeCapabilities: Validates the capabilities of a cache volume to ensure compatibility with the requirements of the requesting Pod. This RPC checks whether the cache volume supports the specified access mode and other attributes.

ListVolumes: Lists cache volumes available in the storage system, providing information about each volume, such as its ID, size, and status.

GetCapacity: Retrieves the capacity of cache volumes, indicating the total amount of storage space available for caching S3 object data.

ControllerGetCapabilities: Retrieves the capabilities of the Controller Service, providing information about the functionalities supported by the Controller, such as volume expansion and snapshot management.

ControllerExpandVolume: Expands the size of a cache volume to accommodate growing data requirements. This RPC increases the capacity of an existing cache volume without losing existing data.

CreateSnapshot: Creates a snapshot of a cache volume for backup or data protection purposes. This RPC captures the current state of the volume, allowing it to be restored later if needed.

DeleteSnapshot: Deletes a snapshot of a cache volume, removing it from the storage system. This RPC is used to reclaim storage space occupied by unused snapshots.

ListSnapshots: Lists snapshots of cache volumes, providing information about each snapshot, such as its ID, creation time, and associated volume.

ControllerModifyVolume: Modifies properties of a cache volume, such as access mode or size. This RPC allows for dynamic adjustments to volume configurations without disrupting existing data.

## Node Service

NodeStageVolume: Prepares a cache volume for mounting to a Kubernetes Pod, performing any necessary setup or initialization steps required before the volume can be accessed by applications within the Pod.

NodeUnstageVolume: Unstages a cache volume from a Kubernetes Pod, performing cleanup or teardown operations to release resources associated with the volume.

NodePublishVolume: Publishes a cache volume to a Kubernetes Pod, making it accessible to applications running within the Pod. This RPC establishes the connection between the cache volume and the Pod at the node level.

NodeUnpublishVolume: Unpublishes a cache volume from a Kubernetes Pod, terminating the connection between the volume and the Pod at the node level.

NodeGetVolumeStats: Retrieves statistics of a cache volume, providing information about the volume's usage, such as capacity, usage, and available space.

NodeExpandVolume: Expands the size of a cache volume on the node, adjusting the volume's capacity to accommodate additional data.

NodeGetCapabilities: Retrieves the capabilities of the Node Service, providing information about the functionalities supported by the Node, such as volume staging, publishing, and expansion.

NodeGetInfo: Retrieves information about the node, such as its ID, hostname, and available resources, allowing the Plugin to determine the node's capabilities and suitability for cache volume operations.


# Solution 

## Architecture

![image](/arch.png)

## Cache Synchronization Mechanism

The S3 Object Storage Cache CSI Plugin MUST implement a cache synchronization mechanism to ensure that data cached locally is synchronized with the corresponding objects in the S3 object storage. This mechanism should support both write-back and write-through modes, which can be configured based on user preferences or workload requirements.

Write-Back Mode: In this mode, changes made to cached data are first written to the local cache and then asynchronously synchronized with the S3 object storage in the background. This mode provides low-latency access to data and reduces the number of requests sent to the S3 storage, thereby optimizing performance.
Write-Through Mode: In this mode, every write operation is immediately propagated to both the local cache and the S3 object storage synchronously. While this ensures data consistency between the cache and the storage, it may introduce higher latency for write operations due to network overhead.
Predictive Caching (Optional):
Optionally, the S3 Object Storage Cache CSI Plugin can integrate AI/ML capabilities for predictive caching. This involves analyzing past access patterns or workload characteristics using machine learning algorithms to predict which data objects are likely to be accessed in the future. Predictive caching can optimize cache utilization by proactively caching frequently accessed or anticipated data, thereby further improving performance and reducing latency.

## Example

### Persistent Volume

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage:  10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/cache
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - <node-name>
```

### Persistent Volume Claim

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
     storage: 10Gi
```

### Pod

```
apiVersion: v1
kind: Pod
metadata:
  name: local-cache-pod
spec:
  containers:
    - name: app
      image: <app-image-with-s3-protocol>
      volumeMounts:
        - name: local-storage
          mountPath: /cache
  volumes:
    - name: local-storage
      persistentVolumeClaim:
        claimName: local-pvc
```

### Storage Volume

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
parameters:
  s3Bucket: "my-s3-bucket"
  s3Region: "ap-south-1"
  localCachePath: "/mnt/cache"
  cacheMode: "write-through" # or "write-back"
  predictiveCaching: "true" # or "false"
```
