###########################################################################################################################################################################################

                                                                               ----ELK AND EFK Stack------ (Completed)  

###########################################################################################################################################################################################

-------------------------------------------------------------------------------------------------------------------
Elasticsearch is a scalable and distributed search engine that is commonly used to store large amounts of log data. 
              It is a NoSQL database. Its primary function is to store and retrieve logs from fluent bit.

Fluent Bit is a logging and metrics processor and forwarder that is extremely fast, lightweight, and highly scalable. 
           Because of its performance-oriented design, it is simple to collect events from various sources and ship them to various destinations without complexity.

Kibana is a graphical user interface (GUI) tool for data visualization, querying, and dashboards. 
       It is a query engine that lets you explore your log data through a web interface, create visualizations for event logs, and filter data to detect problems. 
       Kibana is being used to query elasticsearch indexed data.
-------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------
Why do we need EFK Stack?

->Using the EFK stack in your Kubernetes cluster can make it much easier to collect, store, and analyze log data from all the pods and nodes in your cluster, making it more 
  manageable and  more accessible for different users.
->The kubectl logs command is useful for looking at logs from individual pods, but it can quickly become unwieldy when you have a large number of pods running in your cluster.
->With the EFK stack, you can collect logs from all the nodes and pods in your cluster and store them in a central location. It allows you to quickly troubleshoot issues and identify 
  patterns in your log data.
->It also enables people who are not familiar with using the command line to check logs and keep track of the Kubernetes cluster and the applications that are deployed on it.
->It also allows you to easily create alerts, dashboards, and create monitoring and reporting capabilities that can give you an overview of your system’s health and performance, and It 
  will notify you in real-time if something goes wrong.
-------------------------------------------------------------------------------------------------------------------

-------------------------------------------------------------------------------------------------------------------
In this tutorial, we will be deploying EFK components as follows:

->Elasticsearch is deployed as statefulset as it stores the log data.
->Kibana is deployed as deployment and connects to elasticsearch service endpoint.
->Fluent-bit is deployed as a daemonset to gather the container logs from every node. It connects to the Elasticsearch service endpoint to forward the logs.
-------------------------------------------------------------------------------------------------------------------

-->Create a t2.medium instance and the run these cmds
-->Now to Setup Kubernetes on Amazon EKS
   -->1. Setup kubectl
      2. Setup eksctl 
      3. Create an IAM Role and attache it to EC2 instance
      4. Create your cluster and nodes
-->curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
-->chmod +x ./kubectl 
-->sudo mv ./kubectl /usr/local/bin/kubectl
-->Now as already kubectl is already set and we will attach a admin iam role to the instance for now
-->After attaching the role now we should set up eksctl
-->sudo su -
-->curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
-->sudo mv /tmp/eksctl /usr/local/bin
-->eksctl version
   -->0.176.0
-->vi clusterconfig.yaml
   -->
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: "uat-dev"
  region: "ap-south-1"
  version: "1.23"
nodeGroups:
  - name: ng-1
    instanceType: t3.medium
    desiredCapacity: 3
    volumeSize: 30
    ssh:
      publicKeyName: Mumbai_1
   
   -->Here public key should be in ap_south
-->eksctl create cluster -f clusterconfig.yaml

-->Visit EKS cluster console and find the IAM role attached to the cluster -> add administer level access to it
-->Open any one of node — go to security — click on IAM role — Add administrator access permission to the existing policy.
-->Change the cluser security groups inbond rule to all traffic. source is anywere-ipv4
-->Now install Helm
  -->wget https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz
  -->tar -zxvf helm-v3.14.0-linux-amd64.tar.gz
  -->sudo mv linux-amd64/helm /usr/local/bin/helm
  -->chmod 777 /usr/local/bin/helm
-->Add helm repo to the cluster and update it.
  -->helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
  -->helm repo update
-->Install this helm repo to the cluster.
  -->helm upgrade --install aws-ebs-csi-driver --namespace kube-system aws-ebs-csi-driver/aws-ebs-csi-driver
     -->Release "aws-ebs-csi-driver" does not exist. Installing it now.
        NAME: aws-ebs-csi-driver
        LAST DEPLOYED: Mon May  6 08:17:28 2024
        NAMESPACE: kube-system
        STATUS: deployed
        REVISION: 1
        NOTES:
        To verify that aws-ebs-csi-driver has started, run:

            kubectl get pod -n kube-system -l "app.kubernetes.io/name=aws-ebs-csi-driver,app.kubernetes.io/instance=aws-ebs-csi-driver"

        NOTE: The [CSI Snapshotter](https://github.com/kubernetes-csi/external-snapshotter) controller and CRDs will no longer be installed as part of this chart and moving forward will be 
        a prerequisite of using the snap shotting functionality.
-->Now Create a namespace
-->vi custom-ns.yaml
   -->
kind: Namespace
apiVersion: v1
metadata:
 name: kube-logging

-->kubectl apply -f custom-ns.yaml
-->Create StorageClass named “ebs-storage“
-->vi storage-cls.yaml
   -->
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
 name: ebs-storage
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
parameters:
 type: gp2

-->kubectl apply -f storage-cls.yaml
-->vi elsaticsearch-sts.yaml
   -->
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: es-cluster
  namespace: kube-logging
spec:
  serviceName: elasticsearch
  replicas: 3
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
        - name: elasticsearch
          image: docker.elastic.co/elasticsearch/elasticsearch:7.2.0
          resources:
            limits:
              cpu: 1000m
              memory: 2Gi
            requests:
              cpu: 500m
              memory: 1Gi
          ports:
            - containerPort: 9200
              name: rest
              protocol: TCP
            - containerPort: 9300
              name: inter-node
              protocol: TCP
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
          env:
            - name: cluster.name
              value: k8s-logs
            - name: node.name
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: discovery.seed_hosts
              value: "es-cluster-0.elasticsearch,es-cluster-1.elasticsearch,es-cluster-2.elasticsearch"
            - name: cluster.initial_master_nodes
              value: "es-cluster-0,es-cluster-1,es-cluster-2"
            - name: ES_JAVA_OPTS
              value: "-Xms512m -Xmx512m"
      initContainers:
        - name: fix-permissions
          image: busybox
          command: ["sh", "-c", "chown -R 1000:1000 /usr/share/elasticsearch/data"]
          securityContext:
            privileged: true
          volumeMounts:
            - name: data
              mountPath: /usr/share/elasticsearch/data
        - name: increase-vm-max-map
          image: busybox
          command: ["sysctl", "-w", "vm.max_map_count=262144"]
          securityContext:
            privileged: true
        - name: increase-fd-ulimit
          image: busybox
          command: ["sh", "-c", "ulimit -n 65536"]
          securityContext:
            privileged: true
  volumeClaimTemplates:
    - metadata:
        name: data
        labels:
          app: elasticsearch
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: ebs-storage
        resources:
          requests:
            storage: 10Gi

-->kubectl apply -f elsaticsearch-sts.yaml
-->Now we create a headless service for the statefulset
   -->Now Let’s set up elasticsearch, a Kubernetes headless service that will define a DNS domain for pods.
   -->A headless service lacks load balancing and has no static IP address.
   -->Let’s create a Headless Service for an elasticsearch
-->vi elasticsearch-svc.yaml
   -->
kind: Service
apiVersion: v1
metadata:
  name: elasticsearch
  namespace: kube-logging
  labels:
    app: elasticsearch
spec:
  selector:
    app: elasticsearch
  clusterIP: None
  ports:
    - port: 9200
      name: rest
    - port: 9300
      name: inter-node

-->kubectl apply -f elasticsearch-svc.yaml
-->kubectl get pod -n kube-logging
   -->NAME           READY   STATUS    RESTARTS   AGE
      es-cluster-0   1/1     Running   0          3m34s
      es-cluster-1   1/1     Running   0          2m41s
      es-cluster-2   1/1     Running   0          109s

-->"Creating the Kibana Deployment"
    Kibana can be set up as a simple Kubernetes deployment. 
    If you look at the Kibana deployment manifest file, 
    you’ll notice that we have an env variable ELASTICSEARCH_URL defined to configure the Elasticsearch cluster endpoint. 
    Kibana communicates to elasticsearch via the endpoint URL.
-->vi kibana-deploy.yaml
   -->
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: kube-logging
  labels:
    app: kibana
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
        - name: kibana
          image: docker.elastic.co/kibana/kibana:7.2.0
          resources:
            limits:
              cpu: 1000m
              memory: 1Gi
            requests:
              cpu: 700m
              memory: 1Gi
          env:
            - name: ELASTICSEARCH_URL
              value: http://elasticsearch:9200
          ports:
            - containerPort: 5601

-->kubectl apply -f kibana-deploy.yaml
-->"Creating the Kibana Service":
    Let’s make a NodePort service to access the Kibana UI via the node IP address.
    For demonstration purposes or testing, however,
    it’s not considered a best practice for actual production use.
    The Kubernetes ingress with a ClusterIP service is a more secure and way to expose
    the Kibana UI.
-->vi kibana-svc.yaml
   -->
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: kube-logging
  labels:
    app: kibana
spec:
  type: NodePort
  ports:
    - port: 5601
  selector:
    app: kibana

-->kubectl apply -f kibana-svc.yaml
-->Now Setup FluentBit
-->vi fluentbit-cr.yaml
   -->
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit
  labels:
    app: fluent-bit
rules:
  - apiGroups: [""]
    resources:
      - pods
      - namespaces
    verbs: ["get", "list", "watch"]

-->kubectl apply -f fluentbit-cr.yaml

-->"Creating the Fluent-bit Role Binding"
    ClusterRoleBinding to bind this ClusterRole to Service Account, 
    which will give that ServiceAccount the permissions defined in the ClusterRole.

-->vi fluentbit-crb.yaml
   -->
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: fluent-bit
roleRef:
  kind: ClusterRole
  name: fluent-bit
  apiGroup: rbac.authorization.k8s.io
subjects:
  - kind: ServiceAccount
    name: fluent-bit
    namespace: kube-logging

-->kubectl apply -f fluentbit-crb.yaml

-->"Creating the Fluent-bit Service Account"
    A Service Account is a Kubernetes resource that allows you to control access to the
    Kubernetes API
    for a set of pods, which determines what the pods are allowed to do.
    You can attach roles and role bindings to the service account,
    to give it specific permissions to access the Kubernetes API,
    this is done through Kubernetes Role and Rolebinding resources.
-->vi fluentbit-sa.yaml
   -->
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: kube-logging
  labels:
    app: fluent-bit

-->kubectl apply -f fluentbit-sa.yaml

-->"Creating the Fluent-bit ConfigMap"
    This ConfigMap is used to configure a Fluent-bit pod, 
    by specifying the ConfigMap field in the pod definition. 
    This way when the pod starts it will use the configurations defined in the configmap. 
    ConfigMap can be updates and changes, 
    it will reflect in the pod without the need to recreate the pod itself.
-->vi fluentbit-cm.yaml
   -->
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: kube-logging
  labels:
    k8s-app: fluent-bit
data:
  # Configuration files: server, input, filters and output
  # ======================================================
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020
    @INCLUDE input-kubernetes.conf
    @INCLUDE filter-kubernetes.conf
    @INCLUDE output-elasticsearch.conf
  input-kubernetes.conf: |
    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10
  filter-kubernetes.conf: |
    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Kube_Tag_Prefix     kube.var.log.containers.
        Merge_Log           On
        Merge_Log_Key       log_processed
        K8S-Logging.Parser  On
        K8S-Logging.Exclude Off
  output-elasticsearch.conf: |
    [OUTPUT]
        Name            es
        Match           *
        Host            ${FLUENT_ELASTICSEARCH_HOST}
        Port            ${FLUENT_ELASTICSEARCH_PORT}
        Logstash_Format On
        Replace_Dots    On
        Retry_Limit     False
  parsers.conf: |
    [PARSER]
        Name   apache
        Format regex
        Regex  ^(?<host>[^ ]*) [^ ]* (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z
    [PARSER]
        Name   apache2
        Format regex
        Regex  ^(?<host>[^ ]*) [^ ]* (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^ ]*) +\S*)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z
    [PARSER]
        Name   apache_error
        Format regex
        Regex  ^\[[^ ]* (?<time>[^\]]*)\] \[(?<level>[^\]]*)\](?: \[pid (?<pid>[^\]]*)\])?( \[client (?<client>[^\]]*)\])? (?<message>.*)$
    [PARSER]
        Name   nginx
        Format regex
        Regex ^(?<remote>[^ ]*) (?<host>[^ ]*) (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z
    [PARSER]
        Name   json
        Format json
        Time_Key time
        Time_Format %d/%b/%Y:%H:%M:%S %z
    [PARSER]
        Name        docker
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep   On
    [PARSER]
        Name        syslog
        Format      regex
        Regex       ^\<(?<pri>[0-9]+)\>(?<time>[^ ]* {1,2}[^ ]* [^ ]*) (?<host>[^ ]*) (?<ident>[a-zA-Z0-9_\/\.\-]*)(?:\[(?<pid>[0-9]+)\])?(?:[^\:]*\:)? *(?<message>.*)$
        Time_Key    time
        Time_Format %b %d %H:%M:%S

-->kubectl apply -f fluentbit-cm.yaml

-->Now create a Daemon set
-->vi fluentbit-ds.yaml
   -->
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: kube-logging
  labels:
    k8s-app: fluent-bit-logging
    version: v1
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    matchLabels:
      k8s-app: fluent-bit-logging
  template:
    metadata:
      labels:
        k8s-app: fluent-bit-logging
        version: v1
        kubernetes.io/cluster-service: "true"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "2020"
        prometheus.io/path: /api/v1/metrics/prometheus
    spec:
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:1.3.11
          imagePullPolicy: Always
          ports:
            - containerPort: 2020
          env:
            - name: FLUENT_ELASTICSEARCH_HOST
              value: "elasticsearch"
            - name: FLUENT_ELASTICSEARCH_PORT
              value: "9200"
          volumeMounts:
            - name: varlog
              mountPath: /var/log
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: fluent-bit-config
              mountPath: /fluent-bit/etc/
      terminationGracePeriodSeconds: 10
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: fluent-bit-config
          configMap:
            name: fluent-bit-config
      serviceAccountName: fluent-bit
      tolerations:
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
        - operator: "Exists"
          effect: "NoExecute"
        - operator: "Exists"
          effect: "NoSchedule"

-->kubectl apply -f fluentbit-ds.yaml
-->Now Kbana access
   -->Then take node ip and nodeport of kibana svc
-->kubectl get pods -n kube-logging
   -->NAME                     READY   STATUS    RESTARTS   AGE
      es-cluster-0             1/1     Running   0          24m
      es-cluster-1             1/1     Running   0          24m
      es-cluster-2             1/1     Running   0          23m
      fluent-bit-bch6n         1/1     Running   0          13s
      fluent-bit-g6tqn         1/1     Running   0          13s
      fluent-bit-pwq2w         1/1     Running   0          13s
      kibana-9cbd998d7-pb4pq   1/1     Running   0          18m
-->kubectl get svc -n kube-logging
   -->NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
      elasticsearch   ClusterIP   None            <none>        9200/TCP,9300/TCP   24m
      kibana          NodePort    10.100.37.203   <none>        5601:30278/TCP      16m
-->Now login with Node_ip and port_n0 -> 3.108.226.159:30278
-->Now Create the index pattern in Kibana dashboard then logs will be shown in Kibana.
   -->click on explore on my own -> Go to management -> Under Kibana select Index Patterns -> click on create index pattern -> give index pattern as * -> click on next step
   -->in step 2 select @timestamp -> click on create index pattern -> Now it will show us to the logs parameters 
-->Now go to discover -> Now it will show us the logs
-->Now You can create any Deployments we can see the pods logs information
-->vi existingspace.yaml
   -->
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  namespace: kube-logging
  labels:
      app: webapp
      type: front-end
spec:
  containers:
  - name: nginx-container
    image: nginx
-->kubectl apply -f existingspace.yaml
-->Now we can see the pods logs information

-->eksctl delete cluster uat-dev  --region ap-south-1
