# Monitoring Springboot Application with Prometheus and Grafana

This demo showcases how you monitor Springboot Application with Promethus and Granfana.

## Demo Overview

This demo uses a sample Springboot REST API to illustrate how you can monitor Springboot Application with Prometheus and Granfana.

This steps have been tested on Oracle Cloud Infrastructure Gen 2 as of 4 Feb 2020. Should you encounter any problems, please feel free to feedback to Eugenia (eugenia.pay@oracle.com).

[Special Note] This is a work-in-progress version, more details will be provided in the near future. Please stay tuned.

## Pre-Requisite

Before you can proceed with the following demo, please ensure that you have the following aaccounts:-
- OCI Gen 2 Account with Container Pipeline (Wercker), OKE, OCI Registry Access.
- GitHub Account

In addition, you should also have completed the following two guides:-
- http://github.com/yyeung13/oke_ocir_demo: Setup OCIR/OKE as well as the OCI CLI to manage the kubernetes cluster
- https://github.com/yyeung13/wercker_oke_demo: Use Wercker, Oracle Kubernetes Engine and OCI Registry for complete DevOps of a demo Springboot application
  
Lastly, you should have Promethus and Grafana images pushed into OCIR.

## Demo Details

1. Fork the demo from Github https://github.com/yyeung13/spring-rest-service into your own GitHub.

2. Update 'pom.xml' to add Spring Boot Actuator and micrometer dependencies 

- From Github web console, click on the file 'pom.xml' and edit it
- This XML file contains information about the project and configuration details used by Maven to build the project.
- Add the following lines to 'dependencies' node.

```
		<!-- Spring boot actuator to expose metrics endpoint -->
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-actuator</artifactId>
		</dependency>
		<!-- Micormeter core dependecy  -->
		<dependency>
			<groupId>io.micrometer</groupId>
			<artifactId>micrometer-core</artifactId>
		</dependency>
		<!-- Micrometer Prometheus registry  -->
		<dependency>
			<groupId>io.micrometer</groupId>
			<artifactId>micrometer-registry-prometheus</artifactId>
		</dependency>
    
```

  
3. Update 'build.gradle'

- From Github web console, click on the file 'build.gradle' and edit it
- This file is the build configuration script.
- Add the following 2 lines to 'dependencies' section.

```
		implementation 'org.springframework.boot:spring-boot-starter-actuator'
		implementation 'io.micrometer:micrometer-registry-prometheus'
```			

4. Create 'application.properties' to enable the actuator and Prometheus endpoints to be exposed

- From Github web console, navigate to 'src/main'.
- Create a new folder called 'resources'.
- Create a new file called 'application.properties' and add the following lines:-

```
		management.endpoint.metrics.enabled=true
		management.endpoints.web.exposure.include=*
		management.endpoint.prometheus.enabled=true
		management.metrics.export.prometheus.enabled=true
		management.security.enabled=false

```

5. Re-build

- Ensure all existing spring-rest's pods, deployments and services are deleted from OKE.
- From Github web console, click 'readme.MD' and make any changes and commit it to trigger a build in wercker automatically. 
- Ensure the image is pushed to OCIR and its pod and Load Balancer are cerated in OKE.
- From OCI CLI terminal, run 'kubectl get po' to confirm the Pods are running
- At the same console, run 'kubectl get services' to confirm the service for 'spring_rest_service' is running and identify its public IP. In case the public IP (or external IP as it shows on screen) is shown as 'pending', try again later as sometimes it takes time to secure the public IP. If the status is still pending after 5 minutes, please check if you have any other pods not running smoothly. You may need to delete those pods first and try again as I encounter similar problem with trial OCI Gen 2 accounts
- From a browser, acccess http://<public_ip>:8080/greeting?user=Mary. You should see something like below:- (Note that the ID may change as well as its greetings because it is constantly used for various demos)

```
'{"id":3,"content":"Good Morning, World!"}'

```

- From another browser, access http://<public_ip>:8080/actuator. You should see a JSON response. Ensure '/prometheus' is availalble in the list of endpoints. 

- You have got the monitoring metrics ready!
  
6. Setup Prometheus 

- Ensure Prometheus image is pushed into OCIR.
- Create a file called 'clusterRole.yaml' (to assign clsuter reader permission so that Prometheus can fetch metrics from Kubernetes API's) with the following content:- 
```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: prometheus
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: default

```
- Change the namespace if you are using another one.
- From OCI CLI terminal, run 'kubectl create -f clusterRole.yaml'.

- Create a file called 'config-map.yaml' (contain all the prometheus scrape config and alerting rules, which will be mounted to the Prometheus container in /etc/prometheus as prometheus.yaml and prometheus.rules files) with the following contents:-
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-server-conf
  labels:
    name: prometheus-server-conf
data:
  prometheus.rules: |-
    groups:
    - name: devopscube demo alert
      rules:
      - alert: High Pod Memory
        expr: sum(container_memory_usage_bytes) > 1
        for: 1m
        labels:
          severity: slack
        annotations:
          summary: High Memory Usage
  prometheus.yml: |-
    global:
      scrape_interval: 5s
      evaluation_interval: 5s
    rule_files:
      - /etc/prometheus/prometheus.rules
    alerting:
      alertmanagers:
      - scheme: http
        static_configs:
        - targets:
          - "alertmanager.monitoring.svc:9093"

    scrape_configs:
      - job_name: 'spring_micrometer'
        metrics_path: '/actuator/prometheus'
        scrape_interval: 5s
        static_configs:
        - targets: ['[SPRING_REST_SERVICE_PUBLIC_IP]:8080']
        
      - job_name: 'kubernetes-apiservers'

        kubernetes_sd_configs:
        - role: endpoints
        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https

      - job_name: 'kubernetes-nodes'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics

      
      - job_name: 'kubernetes-pods'

        kubernetes_sd_configs:
        - role: pod

        relabel_configs:
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
          action: replace
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
          target_label: __address__
        - action: labelmap
          regex: __meta_kubernetes_pod_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod_name
      
      - job_name: 'kube-state-metrics'
        static_configs:
          - targets: ['kube-state-metrics.kube-system.svc.cluster.local:8080']

      - job_name: 'kubernetes-cadvisor'

        scheme: https

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        kubernetes_sd_configs:
        - role: node

        relabel_configs:
        - action: labelmap
          regex: __meta_kubernetes_node_label_(.+)
        - target_label: __address__
          replacement: kubernetes.default.svc:443
        - source_labels: [__meta_kubernetes_node_name]
          regex: (.+)
          target_label: __metrics_path__
          replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
      
      - job_name: 'kubernetes-service-endpoints'

        kubernetes_sd_configs:
        - role: endpoints

        relabel_configs:
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels: [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: ([^:]+)(?::\d+)?;(\d+)
          replacement: $1:$2
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_name
```

- Change the address and port of the 'targets' for 'spring_micrometer' job. The values are from Step 5.
- From OCI CLI terminal, run 'kubectl create -f config-map.yaml'.

- Create a file called 'prometheus-deployment.yaml' with the following contents:-
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  labels:
    app: prometheus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
        - name: prometheus
          image: <region_code>.ocir.io/<tenancy_namespace>/prometheus:latest
          imagePullPolicy: Always
          args:
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
      imagePullSecrets:
        - name: ocirsecret
      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf
        - name: prometheus-storage-volume
          emptyDir: {}
```
- Change the image location to match the location used for docker push in the above yaml file.
- If the OCIR secret is not named 'ocirsecret', change the secret name in the above yaml file accordingly too.
- From OCI CLI terminal, run 'kubectl create -f prometheus-deployment.yaml'.
- To verify deployment, use 'kubectl get po' to check if the prometheus pod is running.

- Create a file called 'prometheus-service.yaml' with the following contents:-
```
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  annotations:
      prometheus.io/scrape: 'true'
      prometheus.io/port:   '9090'
spec:
  type: LoadBalancer
  ports:
  - port: 80
    protocol: TCP
    targetPort: 9090
  selector:
    app: prometheus
```

- From OCI CLI terminal, run 'kubectl create -f prometheus-service.yaml'.
- To verify the service is created, use 'kubectl get services' to check if the load balancer is running. Note down the public IP of this load balancer.

- Open a browser and go to http://<public_ip>:8080, you should see Prometheus loaded.
- From the menu bar, click 'Status' > 'Targets'. You should see 'spring_micrometer' is in the list and has a status of 'up'.
- Your Prometheus server is running and scraping your Springboot application metrics now!
