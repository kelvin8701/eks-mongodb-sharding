# eks-mongodb-sharding
Setup for mongoDB sharding on Amazon Elastic Kubernetes Service (EKS).

This configuration include :
- 1 config server
- 2 shards with 1 node each as replica set
- 2 statefull nodes as mongos routers

# How to Run
## Prerequisites

Ensure the following tools are installed before starting:

1. [AWS-CLI](https://docs.aws.amazon.com/eks/latest/userguide/install-awscli.html)
2. [kubectl and eksctl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

---
## Steps

### 1. Configure EKS Cluster

Follow these steps to set up your EKS cluster:

1. Run the command to create the cluster:
    ```bash
    eksctl create cluster -f  ../eks-config/cluster-config.yaml
    ```

---

### 2. Configure StorageClass and PVC

#### **Creating StorageClass**
1. Apply the StorageClass configuration:
    ```bash
    kubectl apply -f ../eks-config/storageclass.yaml
    ```

#### **Creating Persistent Volume Claim (PVC)**
1. Apply the PVC configuration:
    ```bash
    kubectl apply -f ../pvc/
    ```

---

### 3. Configure Pods

1. Deploy the pods with the following command:
    ```bash
    kubectl apply -f ../pods-config/
    ```

---

### 4. Configure Sharding

Sharding is a crucial part of MongoDB scalability. Follow these steps to configure sharding:

#### **1. Config Server**
-

#### **2. Configure Each Shard Server**

Repeat the following steps for every shard pod (replace `shard#` with the actual shard pod number, e.g., `shard1`, `shard2`, etc.):

1. **SSH into the Shard Pod:**
    ```bash
    kubectl exec --stdin --tty mongodb-shard#-0 -- /bin/bash
    ```

2. **SSH into MongoDB:**
    ```bash
    mongo --port 27017
    ```

3. **Configure ReplicaSets:**
    ```javascript
    rs.initiate({
        _id: "shard#",
        version: 1,
        members: [
            { _id: 0, host: "mongodb-shard#-0.mongodb-shard#-headless-service.default.svc.cluster.local:27017" }
        ]
    });
    ```

4. **Wait until the primary node for every shard is promoted.**

#### **3. Route Server**

1. **SSH into the Router Pod:**
    ```bash
    kubectl exec -it $(kubectl get pod -l "tier=routers" -o jsonpath='{.items[0].metadata.name}') -c mongos-container bash
    ```

2. **SSH into MongoDB:**
    ```bash
    mongo --port 27017
    ```

3. **Add Shards to the Cluster:**
    ```javascript
    sh.addShard("shard1/mongodb-shard1-0.mongodb-shard1-headless-service.default.svc.cluster.local:27017");
    sh.addShard("shard2/mongodb-shard2-0.mongodb-shard2-headless-service.default.svc.cluster.local:27017");
    ```

4. **Check Sharding Status in Routers:**
    ```javascript
    sh.status();
    ```

5. **Add Database to be Sharded:**
    ```javascript
    sh.enableSharding("db-name");
    ```


---

## Notes

- Ensure all configurations are correct before applying them to the cluster.
- Monitor the status of your pods, ReplicaSets, and clusters to ensure everything is running smoothly.
