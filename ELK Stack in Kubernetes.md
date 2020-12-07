# ELK Stack 구성 in Kubernetes

안녕하세요. 서비스 운영시 모니터링 환경이 필요성을 느끼게 되어

Kubernetes 환경에서 ElasticSearch, Kibana를 Yaml을 이용하여 구성하는 방법을 다룹니다.

이번 글 에서는 ElasticSearch와 Kibana 설치에 대해 다루겠습니다.

Logstash, Filebeat는 다른 페이지에서 다룹니다.

(틀린 내용이 많을 수 있으니 피드백은 언제든지 남겨주세요~)

# 해보자

## 구성

![ELK%20Stack%20in%20Kubernetes/Untitled.png](ELK%20Stack%20in%20Kubernetes/Untitled.png)

ElasticSearch, Logstash, Kibana, Filebeat 버젼은 7.9.3으로 구성을 할 예정입니다.

- 어플리케이션 서버들이 로그 전용 PVC에 남기도록 한 후 Filebeat 전용 서버에서 Logstash로 전송합니다.
- Filebeat에서 보낸 로그를 Logstash에서 필터링 후 ElasticSearch로 전송합니다.
- Kibana를 통해 원하는 값들에 대해 시각화 표현을 합니다.
- ElasticSearch는 총 5개의 노드로 구성하며, 모두 다 마스터(마스터,데이터 구분해야하나) 노드입니다.
- 하단에 작성해놓은 Yaml 파일들을 수정하신 후 `kubectl create -f 파일명.yaml` 명령어를 수행하시면 됩니다.

리소스를 마음껏 사용할수 있는 환경이 아니여서 일반적인 구성과 살짝 다르게 구성합니다.

## PVC 생성

ElasticSearch와 Log파일을 저장하기 위한 PVC를 생성합니다.

### Yaml

- PVC(ElasticSearch)

    ```yaml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: efk-stack-pvc
      labels:
        pvc: monitor
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 512Gi #사용할 용량 기입

      storageClassName: hdd-ceph-fs #스토리지 클래스 명 기입
      volumeMode: Filesystem
    ```

- PVC(Log)

    ```yaml
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: log
      labels:
        pvc: log
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 1Ti #사용할 용량 기입
      storageClassName: ssd-ceph-fs #스토리지 클래스 명 기입
      volumeMode: Filesystem
    ```

## Service 생성

각 서버에 필요한 Service를 생성합니다. (자신의 환경에 맞도록 해주셔도 됩니다.)

이번 구성에는 Kibana만 LoadBalancer를 사용하고 나머지는 ClusterIP로 구성합니다. 

### Yaml

- ClusterIP(ElasticSearch)

    ```yaml
    kind: Service
    apiVersion: v1
    metadata:
      name: elasticsearch
      labels:
        app: elasticsearch
    spec:
      ports:
        - name: http
          protocol: TCP
          port: 9200
          targetPort: 9200
        - name: transport
          protocol: TCP
          port: 9300
          targetPort: 9300
      selector:
        app: elasticsearch
      type: ClusterIP
    ```

- ClusterIP(Logstash)

    ```yaml
    kind: Service
    apiVersion: v1
    metadata:
      name: logstash
      labels:
        app: logstash
    spec:
      ports:
        - name: logstash
          protocol: TCP
          port: 5044
          targetPort: 5044
        - name: logstash-2
          protocol: TCP
          port: 80
          targetPort: 80
        - name: logstash-3
          protocol: TCP
          port: 8080
          targetPort: 8080
        - name: logstash-4
          protocol: TCP
          port: 443
          targetPort: 443
        - name: logstash-5
          protocol: TCP
          port: 9600
          targetPort: 9600
      selector:
        app: logstash
      type: ClusterIP
    ```

- LoadBalancer(Kibana)

    ```yaml
    kind: Service
    apiVersion: v1
    metadata:
      name: kibana
    spec:
      ports:
        - name: kibana
          protocol: TCP
          port: 5601
          targetPort: 5601
          nodePort: 31459
      selector:
        app: kibana
      type: LoadBalancer
    ```

## Elasticsearch 생성

ElasticSearch는 StatefulSet 형식으로 생성합니다.

### Yaml

- ElasticSearch

    ```yaml
    kind: StatefulSet
    apiVersion: apps/v1
    metadata:
      name: es-cluster
    spec:
      replicas: 5 #구성할 개수 기입
      selector:
        matchLabels:
          app: elasticsearch
      template:
        metadata:
          labels:
            app: elasticsearch
        spec:
          volumes:
            - name: data
              persistentVolumeClaim:
                claimName: efk-stack-pvc #생성한 PVC명 기입
          initContainers:
            - resources:
                limits:
                  cpu: '1'
                  memory: 1Gi
                requests:
                  cpu: '1'
                  memory: 1Gi
              terminationMessagePath: /dev/termination-log
              name: fix-permissions
              command:
                - sh
                - '-c'
                - 'chown -R 1000:1000 /usr/share/elasticsearch/data'
              securityContext:
                privileged: true
              imagePullPolicy: Always
              volumeMounts:
                - name: data
                  mountPath: /usr/share/elasticsearch/data
              terminationMessagePolicy: File
              image: busybox
            - name: increase-vm-max-map
              image: busybox
              command:
                - sysctl
                - '-w'
                - vm.max_map_count=262144 #디폴트 값은 65536이나 정상 기동이 안될시 늘려야 함
              resources:
                limits:
                  cpu: '1'
                  memory: 1Gi
                requests:
                  cpu: '1'
                  memory: 1Gi
              terminationMessagePath: /dev/termination-log
              terminationMessagePolicy: File
              imagePullPolicy: Always
              securityContext:
                privileged: true
            - name: increase-fd-ulimit
              image: busybox
              command:
                - sh
                - '-c'
                - ulimit -n 65536
              resources:
                limits:
                  cpu: '1'
                  memory: 1Gi
                requests:
                  cpu: '1'
                  memory: 1Gi
              terminationMessagePath: /dev/termination-log
              terminationMessagePolicy: File
              imagePullPolicy: Always
              securityContext:
                privileged: true
          containers:
            - resources: #적용할 사양 기입
                limits:
                  cpu: '4'
                  memory: 16Gi
                requests:
                  cpu: '2'
                  memory: 12Gi
              name: elasticsearch
              env:
                - name: cluster.name
                  value: k8s-logs
                - name: node.name
                  valueFrom:
                    fieldRef:
                      apiVersion: v1
                      fieldPath: metadata.name
                - name: discovery.seed_hosts
                  value: elasticsearch
                - name: cluster.initial_master_nodes #구성할 개수만큼 0번부터 기입
                  value: 'es-cluster-0,es-cluster-1,es-cluster-2,es-cluster-3,es-cluster-4'
                - name: node.max_local_storage_nodes #구성할 개수 기입
                  value: '5'
                - name: ES_JAVA_OPTS #권장 옵션은 메모리의 50% 이하로 설정
                  value: '-Xms8g -Xmx8g'
              ports:
                - name: rest
                  containerPort: 9200
                  protocol: TCP
                - name: inter-node
                  containerPort: 9300
                  protocol: TCP
              imagePullPolicy: IfNotPresent
              volumeMounts:
                - name: data
                  mountPath: /usr/share/elasticsearch/data
              terminationMessagePolicy: File
              image: 'docker.elastic.co/elasticsearch/elasticsearch:7.9.3'
          restartPolicy: Always
          terminationGracePeriodSeconds: 30
          dnsPolicy: ClusterFirst
          securityContext: {}
          schedulerName: default-scheduler
      serviceName: elasticsearch
      podManagementPolicy: OrderedReady
      updateStrategy:
        type: RollingUpdate
        rollingUpdate:
          partition: 0
      revisionHistoryLimit: 10
    ```

정상적으로 ElasticSearch Pod들이 생성이 되었다면, 아무 노드에 터미널로 접속합니다.

EX)  `kubectl exec -it es-cluster-0 -- /bin/bash` 

이후 `curl [localhost:9200/_cluster/health?pretty=true](http://localhost:9200/_cluster/health?pretty=true)` 명령어를 날려 아래 사진과 같이 status가 Green 상태 그리고 number_of_nodes(구성한 갯수)를 확인합니다. 제대로 조회가 된다면 정상적으로 구성 완료입니다.

![ELK%20Stack%20in%20Kubernetes/Untitled%201.png](ELK%20Stack%20in%20Kubernetes/Untitled%201.png)

## Kibana 생성

Kibana는 Deployment 형식으로 생성합니다.

### Yaml

- Kibana

    ```yaml
    kind: Deployment
    apiVersion: apps/v1
    metadata:
      name: kibana
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
              image: 'docker.elastic.co/kibana/kibana:7.9.3'
              ports:
                - containerPort: 5601
                  protocol: TCP
              env:
                - name: ELASTICSEARCH_URL
                  value: 'http://elasticsearch:9200'
              resources: #설정할 사양 기입
                limits:
                  cpu: '2'
                  memory: 4Gi
                requests:
                  cpu: '1'
                  memory: 2Gi
    ```

정상적으로 Kibana Pod가 기동이 되었다면 웹브라우저에서 LoadBalancer(Kibana) IP:5601 입력하여 접속합니다. 

![ELK%20Stack%20in%20Kubernetes/Untitled%202.png](ELK%20Stack%20in%20Kubernetes/Untitled%202.png)

접속했을 때 위 사진처럼 **"Kibana server is not ready yer"** 이라는 문장이 나타난다면

`kubectl logs {Kibana Pod명}` 입력후 아래 로그가 찍혀있는지 확인합니다.

별다른 에러 메세지가 없다면 아직 ElasticSearch 클러스터 구성중에 있을수도 있기에 기다려줍니다.

하지만 대부분 아래 로그와 같은 에러가 나타납니다.

`{"type":"log","@timestamp":"2020-11-19T04:24:18Z","tags":["info","savedobjects-service"],"pid":1,"message":"Creating index **.kibana_1**."}
{"type":"log","@timestamp":"2020-11-19T04:24:48Z","tags":["warning","savedobjects-service"],"pid":1,"message":"Unable to connect to Elasticsearch. Error: Request Timeout after 30000ms"}`

ElasticSearch Pod 터미널에 접속한 후 문제가 발생한 index에 대해 삭제 후 

추가(alias 포함)를 한 후 Kibana Pod를 재기동합니다.

```bash
curl -X POST https://localhost:9200/_aliases --insecure -H 'Content-Type: application/json' -d'
{
    "actions" : [
        { "remove" : { "index" : ".kibana_1", "alias" : ".kibana" } },
        { "add" : { "index" : ".kibana_1", "alias" : ".kibana" } }
    ]
}
'
```

그리고 다시 URL로 접속해보면 Kibana 화면이 정상적으로 보이게 됩니다!

![ELK%20Stack%20in%20Kubernetes/Untitled%203.png](ELK%20Stack%20in%20Kubernetes/Untitled%203.png)

## Logstash, Filebeat

Logstash, Filebeat 설치 방법은 아래 페이지에서 다루도록 하겠습니다.
[Logstash, Filebeat](https://github.com/ddukbg/ELK-Stack-in-Kubernetes/blob/main/Logstash%20Filebeat.md)
