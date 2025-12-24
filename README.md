# Quickstart to run Loki locally

## 快速构建Grafana+Loki+Alloy日志系统
> 参考官方文档：https://grafana.com/docs/loki/latest/get-started/quick-start/quick-start/
> 
**To install Loki locally, follow these steps:**

1. Create a directory called `evaluate-loki` for the demo environment.
   Make `evaluate-loki` your current working directory:

   ```bash
   mkdir evaluate-loki
   cd evaluate-loki
   ```

2. Download `loki-config.yaml`, `alloy-local-config.yaml`, and `docker-compose.yaml`:

   ```bash
   wget https://raw.githubusercontent.com/grafana/loki/main/examples/getting-started/loki-config.yaml -O loki-config.yaml
   wget https://raw.githubusercontent.com/grafana/loki/main/examples/getting-started/alloy-local-config.yaml -O alloy-local-config.yaml
   wget https://raw.githubusercontent.com/grafana/loki/main/examples/getting-started/docker-compose.yaml -O docker-compose.yaml
   ```

   ```bash
   curl https://raw.githubusercontent.com/grafana/loki/main/examples/getting-started/loki-config.yaml --output loki-config.yaml
   curl https://raw.githubusercontent.com/grafana/loki/main/examples/getting-started/alloy-local-config.yaml --output alloy-local-config.yaml
   curl https://raw.githubusercontent.com/grafana/loki/main/examples/getting-started/docker-compose.yaml --output docker-compose.yaml
   ```

   <!-- INTERACTIVE ignore END -->


3. Deploy the sample Docker image.

   With `evaluate-loki` as the current working directory, start the demo environment using `docker compose`:

   ```bash
   docker compose up -d
   ```

   At the end of the command, you should see something similar to the following:

   ```console
   ✔ Network evaluate-loki_loki          Created      0.1s
   ✔ Container evaluate-loki-minio-1     Started      0.6s
   ✔ Container evaluate-loki-flog-1      Started      0.6s
   ✔ Container evaluate-loki-backend-1   Started      0.8s
   ✔ Container evaluate-loki-write-1     Started      0.8s
   ✔ Container evaluate-loki-read-1      Started      0.8s
   ✔ Container evaluate-loki-gateway-1   Started      1.1s
   ✔ Container evaluate-loki-grafana-1   Started      1.4s
   ✔ Container evaluate-loki-alloy-1     Started      1.4s
   ```


4. (Optional) Verify that the Loki cluster is up and running.

   - The read component returns `ready` when you browse to [http://localhost:3101/ready](http://localhost:3101/ready).
     The message `Query Frontend not ready: not ready: number of schedulers this worker is connected to is 0` shows until the read component is ready.
   - The write component returns `ready` when you browse to [http://localhost:3102/ready](http://localhost:3102/ready).
     The message `Ingester not ready: waiting for 15s after being ready` shows until the write component is ready.

5. (Optional) Verify that Grafana Alloy is running.
   - You can access the Grafana Alloy UI at [http://localhost:12345](http://localhost:12345).


<!-- INTERACTIVE page step1.md END -->

<!-- INTERACTIVE page step2.md START -->

## 在grafana中进行简单语句查询

1. The Loki query editor has two modes (3):

   - Builder mode, which provides a visual query designer.
   - Code mode, which provides a feature-rich editor for writing LogQL queries.

   Next we’ll walk through a few simple queries using both the builder and code views.

1. Click **Code** (3) to work in Code mode in the query editor.

   Here are some sample queries to get you started using LogQL.
   These queries assume that you followed the instructions to create a directory called `evaluate-loki`.

   If you installed in a different directory, you’ll need to modify these queries to match your installation directory.

   After copying any of these queries into the query editor, click **Run Query** (4) to execute the query.

   1. View all the log lines which have the container label `evaluate-loki-flog-1`:
      <!-- INTERACTIVE copy START -->
      ```bash
      {container="evaluate-loki-flog-1"}
      ```
      <!-- INTERACTIVE copy END -->
      In Loki, this is a log stream.

      Loki uses [labels](https://grafana.com/docs/loki/<LOKI_VERSION>/get-started/labels/) as metadata to describe log streams.

      Loki queries always start with a label selector.
      In the previous query, the label selector is `{container="evaluate-loki-flog-1"}`.

   1. To view all the log lines which have the container label `evaluate-loki-grafana-1`:
      <!-- INTERACTIVE copy START -->
      ```bash
      {container="evaluate-loki-grafana-1"}
      ```
      <!-- INTERACTIVE copy END -->
   1. Find all the log lines in the `{container="evaluate-loki-flog-1"}` stream that contain the string `status`:
      <!-- INTERACTIVE copy START -->
      ```bash
      {container="evaluate-loki-flog-1"} |= `status`
      ```
      <!-- INTERACTIVE copy END -->
   1. Find all the log lines in the `{container="evaluate-loki-flog-1"}` stream where the JSON field `status` has the value `404`:
      <!-- INTERACTIVE copy START -->
      ```bash
      {container="evaluate-loki-flog-1"} | json | status=`404`
      ```
      <!-- INTERACTIVE copy END -->
   1. Calculate the number of logs per second where the JSON field `status` has the value `404`:
      <!-- INTERACTIVE copy START -->
      ```bash
      sum by(container) (rate({container="evaluate-loki-flog-1"} | json | status=`404` [$__auto]))
      ```
      <!-- INTERACTIVE copy END -->

## 收集k8s日志
> 示例为收集docker日志，再此基础上收集k8s日志
> 
> 参考官方文档：https://grafana.com/docs/alloy/latest/collect/logs-in-kubernetes/
>
```yaml
// alloy-local-config.yaml

discovery.docker "flog_scrape" {
        host             = "unix:///var/run/docker.sock"
        refresh_interval = "5s"
}

discovery.relabel "flog_scrape" {
        targets = []

        rule {
                source_labels = ["__meta_docker_container_name"]
                regex         = "/(.*)"
                target_label  = "container"
        }
}

loki.source.docker "flog_scrape" {
        host             = "unix:///var/run/docker.sock"
        targets          = discovery.docker.flog_scrape.targets
        forward_to       = [loki.write.default.receiver]
        relabel_rules    = discovery.relabel.flog_scrape.rules
        refresh_interval = "5s"
}

loki.write "default" {
        endpoint {
                url       = "http://gateway:3100/loki/api/v1/push"
                tenant_id = "tenant1"
        }
        external_labels = {}
}

local.file_match "node_logs" {
  path_targets = [{
      __path__  = "/var/log/syslog",
      job       = "node/syslog",
      node_name = sys.env("HOSTNAME"),
      cluster   = "production",
  }]
}

loki.source.file "node_logs" {
  targets    = local.file_match.node_logs.targets
  forward_to = [loki.write.default.receiver]
}
```
```yaml
// docker-compose.yaml文件中alloy的部分修改如下

  alloy:
    image: grafana/alloy:latest
    volumes:
      - ./alloy-local-config.yaml:/etc/alloy/config.alloy:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/log:/var/log:ro
      # 挂载 k8s 系统日志目录
      - /var/log/pods:/var/log/pods:ro
      # 添加 K8s Pod 日志路径
    command:  run --server.http.listen-addr=0.0.0.0:12345 --storage.path=/var/lib/alloy/data /etc/alloy/config.alloy
    ports:
      - 12345:12345
    depends_on:
      - gateway
    networks:
      - loki
