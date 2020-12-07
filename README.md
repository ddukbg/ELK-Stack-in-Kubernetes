# ELK-Stack-in-Kubernetes
ELK Stack 구성 in Kubernetes

# 참고사항
Kubernetes단의 모니터링 보다는 Kubernetes 환경에서 운영되는 어플리케이션 기준으로 작성하였습니다.

실제 운영환경에서는 Elasticseach 노드 구성시 전부 Master Node로 구성 X

(여기선 단순 구성을 위한 목적으로 모든 노드를 Master로 설정함)

Filebeat, Logstash를 Ubuntu 이미지를 이용하여 각각 한대 씩 구성 

(효율적으로 구성 하기 위해선 Daemonsets, Sidecar 방식을 사용하겠죠?)

[Elasticsearch, Kibana 구성](https://github.com/ddukbg/ELK-Stack-in-Kubernetes/blob/main/ELK%20Stack%20in%20Kubernetes.md)

[Filebeat, Logstash 구성](https://github.com/ddukbg/ELK-Stack-in-Kubernetes/blob/main/ELK%20Stack%20in%20Kubernetes.md)
