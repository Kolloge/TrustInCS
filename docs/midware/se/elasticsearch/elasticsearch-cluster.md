## Elasticsearch集群相关知识点

### 节点和集群的概念

- 节点
  - 节点是Elasticsearch运行中的实例
- 集群
  - 集群包含一个或多个具有相同cluster.name的节点，节点间协同工作，共享数据，共同分担工作负荷。

###集群的健康状态

- 集群的健康状态共分为三种
  - green
    - 所有的主分片和从分片都可用
  - yellow
    - 所有的主分片可用，但是存在不可用的从分片
  - red
    - 存在不可用的主分片
- 如何查看集群状态
  - ```http request
    GET /_cluster/health
    ```
  - ```json
    {
        "cluster_name": "test-cluster-kg",
        "status": "yellow",
        "timed_out": false,
        "number_of_nodes": 1,
        "number_of_data_nodes": 1,
        "active_primary_shards": 6,
        "active_shards": 6,
        "relocating_shards": 0,
        "initializing_shards": 0,
        "unassigned_shards": 6,
        "delayed_unassigned_shards": 0,
        "number_of_pending_tasks": 0,
        "number_of_in_flight_fetch": 0,
        "task_max_waiting_in_queue_millis": 0,
        "active_shards_percent_as_number": 50
    }
    ```
