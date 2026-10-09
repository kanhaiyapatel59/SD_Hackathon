# Diagram 8: Deployment Diagram

```mermaid
graph TB
    Internet(("🌐 Internet Users")) --> CDN["CDN / WAF (Edge Caching & DDoS)"]
    CDN --> LB["Load Balancer"]

    subgraph AppCluster ["Horizontal Stateless Application Tier"]
        App1["App Instance 1"]
        App2["App Instance 2"]
        App3["App Instance 3"]
    end

    LB --> App1
    LB --> App2
    LB --> App3

    subgraph CacheCluster ["In-Memory Cache Cluster"]
        CacheMaster[("Cache Primary")]
        CacheSlave[("Cache Replica")]
        CacheMaster -.-> CacheSlave
    end

    subgraph MQCluster ["Message Queue Cluster"]
        Broker1{{"Message Broker 1"}}
        Broker2{{"Message Broker 2"}}
        Broker1 <--> Broker2
    end

    subgraph DBCluster ["Relational Database Tier"]
        DBMaster[("SQL Database Primary")]
        DBReplica[("SQL Read Replica")]
        DBMaster -.-> DBReplica
    end

    App1 --> CacheMaster
    App2 --> CacheMaster
    App3 --> CacheMaster

    App1 --> Broker1
    App2 --> Broker1
    App3 --> Broker2

    App1 --> DBMaster
    App2 --> DBMaster
    App3 --> DBMaster

    App1 -.-> DBReplica
    App2 -.-> DBReplica
    App3 -.-> DBReplica
```

### Key Highlights
* **Horizontal Scaling:** Stateless application instances scale out behind the Load Balancer to handle traffic surges.
* **Cache Primary-Replica:** Fast-path writes target Cache Primary; replicas stand by for sub-second automated failover.
* **Durable Message Clustering:** Message brokers are clustered to guarantee zero message loss during broker restarts.
* **Database Read/Write Splitting:** Heavy read queries target the Read Replica, reserving the Primary for critical inventory and payments.
