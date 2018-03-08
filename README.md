## 基于Kubernetes的日志监控系统安装

## 1.场景

我们在生产环境中需要对系统的各种日志进行采集、查询和分析。本例演示使用`Fluentd`进行日志采集，`Elasticsearch`进行日志存储，`Kibana`进行日志查询分析。

## 2.安装

### 2.1 创建dashboard用户

`sa.yml`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: dashboard
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: dashboard
    namespace: kube-system

```

### 2.2 创建PersistentVolume

创建`PersistentVolume`用作`Elasticsearch`存储所用的磁盘:

`pv.yml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: elk-log-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
   nfs:
     path: /opt/data/kafka0
     server: 192.168.1.140
     readOnly: false
```

### 2.3 fluentd的配置(configmap)
若无此步的话，`fluentd-es`镜像会有下面的错误：

```
2018-03-07 08:35:18 +0000 [info]: adding filter pattern="kubernetes.**" type="kubernetes_metadata"
2018-03-07 08:35:19 +0000 [error]: config error file="/etc/td-agent/td-agent.conf" error="Invalid Kubernetes API v1 endpoint https://172.21.0.1:443/api: SSL_connect returned=1 errno=0 state=error: certificate verify failed"
2018-03-07 08:35:19 +0000 [info]: process finished code=256
```

所以需要配置configmap：

`cm.yml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-conf
  namespace: kube-system
data:
  td-agent.conf: |
    <match fluent.**>
      type null
    </match>
    # Example:
    # {"log":"[info:2016-02-16T16:04:05.930-08:00] Some log text here\n","stream":"stdout","time":"2016-02-17T00:04:05.931087621Z"}
    <source>
      type tail
      path /var/log/containers/*.log
      pos_file /var/log/es-containers.log.pos
      time_format %Y-%m-%dT%H:%M:%S.%NZ
      tag kubernetes.*
      format json
      read_from_head true
    </source>
    <filter kubernetes.**>
      type kubernetes_metadata
      verify_ssl false
    </filter>
    <source>
      type tail
      format syslog
      path /var/log/messages
      pos_file /var/log/messages.pos
      tag system
    </source>
    <match **>
       type elasticsearch
       user "#{ENV['FLUENT_ELASTICSEARCH_USER']}"
       password "#{ENV['FLUENT_ELASTICSEARCH_PASSWORD']}"
       log_level info
       include_tag_key true
       host elasticsearch-logging
       port 9200
       logstash_format true
       # Set the chunk limit the same as for fluentd-gcp.
       buffer_chunk_limit 2M
       # Cap buffer memory usage to 2MiB/chunk * 32 chunks = 64 MiB
       buffer_queue_limit 32
       flush_interval 5s
       # Never wait longer than 5 minutes between retries.
       max_retry_wait 30
       # Disable the limit on the number of retries (retry forever).
       disable_retry_limit
       # Use multiple threads for processing.
       num_threads 8
    </match>


```

### 2.4 整体配置

`logging.yml`:

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: elasticsearch-logging-v1
  namespace: kube-system
  labels:
    k8s-app: elasticsearch-logging
    version: v1
    kubernetes.io/cluster-service: "true"
spec:
  replicas: 2
  selector:
    k8s-app: elasticsearch-logging
    version: v1
  template:
    metadata:
      labels:
        k8s-app: elasticsearch-logging
        version: v1
        kubernetes.io/cluster-service: "true"
    spec:
      serviceAccount: dashboard
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/google-containers/elasticsearch:v2.4.1-1
        name: elasticsearch-logging
        resources:
          # need more cpu upon initialization, therefore burstable class
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        ports:
        - containerPort: 9200
          name: db
          protocol: TCP
        - containerPort: 9300
          name: transport
          protocol: TCP
        volumeMounts:
        - name: es-persistent-storage
          mountPath: /data
        env:
        - name: "NAMESPACE"
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
      volumes:
      - name: es-persistent-storage
        persistentVolumeClaim:
          claimName: elk-log

---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: elk-log
  namespace: kube-system
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  #selector:
  #  matchLabels:
  #    release: "stable"
  #  matchExpressions:
  #    - {key: environment, operator: In, values: [dev]}

---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch-logging
  namespace: kube-system
  labels:
    k8s-app: elasticsearch-logging
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "Elasticsearch"
spec:
  ports:
  - port: 9200
    protocol: TCP
    targetPort: db
  selector:
    k8s-app: elasticsearch-logging

---
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: fluentd-es-v1.22
  namespace: kube-system
  labels:
    k8s-app: fluentd-es
    kubernetes.io/cluster-service: "true"
    version: v1.22
spec:
  template:
    metadata:
      labels:
        k8s-app: fluentd-es
        kubernetes.io/cluster-service: "true"
        version: v1.22
      # This annotation ensures that fluentd does not get evicted if the node
      # supports critical pod annotation based priority scheme.
      # Note that this does not guarantee admission on the nodes (#40573).
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
        scheduler.alpha.kubernetes.io/tolerations: '[{"key": "node.alpha.kubernetes.io/ismaster", "effect": "NoSchedule"}]'
    spec:
      serviceAccount: dashboard
      containers:
      - name: fluentd-es
        image: registry.cn-hangzhou.aliyuncs.com/google-containers/fluentd-elasticsearch:1.22
        command:
          - '/bin/sh'
          - '-c'
          - '/usr/sbin/td-agent 2>&1 >> /var/log/fluentd.log'
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
        - name: config-volume
          mountPath: /etc/td-agent/
          readOnly: true
      #nodeSelector:
      #  alpha.kubernetes.io/fluentd-ds-ready: "true"
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config-volume
        configMap:
          name: fluentd-conf

---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kibana-logging
  namespace: kube-system
  labels:
    k8s-app: kibana-logging
    kubernetes.io/cluster-service: "true"
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: kibana-logging
  template:
    metadata:
      labels:
        k8s-app: kibana-logging
    spec:
      containers:
      - name: kibana-logging
        image: registry.cn-hangzhou.aliyuncs.com/google-containers/kibana:v4.6.1-1
        resources:
          # keep request = limit to keep this container in guaranteed class
          limits:
            cpu: 100m
          requests:
            cpu: 100m
        env:
          - name: "ELASTICSEARCH_URL"
            value: "http://elasticsearch-logging:9200"
          - name: "KIBANA_BASE_URL"
            value: "/api/v1/proxy/namespaces/kube-system/services/kibana-logging"
        ports:
        - containerPort: 5601
          name: ui
          protocol: TCP

---
apiVersion: v1
kind: Service
metadata:
  name: kibana-logging
  namespace: kube-system
  labels:
    k8s-app: kibana-logging
    kubernetes.io/cluster-service: "true"
    kubernetes.io/name: "Kibana"
spec:
  ports:
  - port: 5601
    protocol: TCP
    targetPort: ui
  selector:
    k8s-app: kibana-logging
  type: ClusterIP
```

### 2.5 安装

```yaml
kubectl apply -f sa.yml
kubectl apply -f cm.yml
kubectl apply -f pv.yml
kubectl apply -f logging.yml
```

### 2.6 验证

- 上步完成后要等待相当长的时间，请耐心等待。

- 查看`kibana`与`Elasticsearch`访问地址:`kubectl cluster-info`

    ```yaml
    Kubernetes master is running at https://Master-IP:6443
    Elasticsearch is running at https://Master-IP:6443/api/v1/namespaces/kube-system/services/elasticsearch-logging/proxy
    Heapster is running at https://Master-IP:6443/api/v1/namespaces/kube-system/services/heapster/proxy
    Kibana is running at https://Master-IP:6443/api/v1/namespaces/kube-system/services/kibana-logging/proxy
    KubeDNS is running at https://Master-IP:6443/api/v1/namespaces/kube-system/services/kube-dns/proxy
    monitoring-influxdb is running at https://Master-IP:6443/api/v1/namespaces/kube-system/services/monitoring-influxdb/proxy
    
    ```

- 启动客户端代理`kubectl proxy`,访问:`http://localhost:8001/api/v1/namespaces/kube-system/services/kibana-logging/proxy`


