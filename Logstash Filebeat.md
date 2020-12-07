# Logstash, Filebeat

---

![Logstash%20Filebeat/Untitled.png](Logstash%20Filebeat/Untitled.png)

Logstash와 Filebeats 전용 이미지를 이용하지 않고 우분투 이미지를 이용하여 설치를 하도록 하겠습니다. 

우분투 기본 이미지를 통해 설치시 Pod를 재생성하면 기존의 데이터가 싹 날아가게 됩니다.

이를 대안하기 위해 Config라는 PVC를 마운트하여 설치, 설정 파일을 넣어두고 쉘스크립트를 작성하여 Pod 기동시 자동으로 설치되도록 설정합니다.

## Filebeat

각 web server, was에서 떨궈주는 Log 경로를 Log전용 PVC를 생성하여 마운트 하고 변경해줍니다.

저는 해당 우분투 Pod를 로그전용 Pod로 이용할 예정입니다.

**PVC Mount Path**

- Log → 어플리케이션 로그가 남는 경로
- Config → /config

wget을 이용하여 [https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.9.3-amd64.deb](https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.9.3-amd64.deb) 설치 파일을 `/config/log_setting/` 디렉토리에 다운로드합니다.

Filebeat를 설치합니다.

`dpkg -i /config/log_setting/filebeat-7.9.3.deb;`

Filebeat 설정 파일을 수정하기 위해 PVC영역으로 복사합니다.

`cp /etc/filebeat/filebeat.yml /config/log_setting/filebeat.yml`

Filebeat 설정 파일을 열고 수정합니다.

`vi /config/log_setting/filebeat.yml`

설정파일 하단부에 Elasticsearch와 Logstash 관련 설정 부분이 있는데, 

우리는 Logstash에서 필터링된 데이터를 Elasticsearch에 보낼 것이기 때문에 

Logstash 부분에 생성했던 Service(`미리 Logstash 항목의 Service를 생성해야함`)를 기입하고, Elasticsearch 부분은 주석처리 해줍니다.

### Filebeat 설정

![Logstash%20Filebeat/Untitled%201.png](Logstash%20Filebeat/Untitled%201.png)

로그 수집을 위한 설정을 진행하기 위해 윗 부분에 `filebeat.inputs:` 부분으로 옵니다.

`filebeat.inputs:` 안에서 `- type: log`을 분기로 수집할 로그 경로, 필드, 태그명을 설정합니다.

![Logstash%20Filebeat/Untitled%202.png](Logstash%20Filebeat/Untitled%202.png)

**해당 로그 파일 내용(웹서버)**

![Logstash%20Filebeat/Untitled%203.png](Logstash%20Filebeat/Untitled%203.png)

만약 멀티라인으로 로그가 구분된다면 아래 사진 처럼 설정하면 됩니다.

멀티라인 관련 참고 URL: [https://elkstack.tistory.com/14](https://elkstack.tistory.com/14)

여기까지가 Filebeat에서 로그 수집을 위한 준비가 끝난 것입니다.

![Logstash%20Filebeat/Untitled%204.png](Logstash%20Filebeat/Untitled%204.png)

**해당 로그 파일 내용(WAS 어플리케이션 로그)**

![Logstash%20Filebeat/Untitled%205.png](Logstash%20Filebeat/Untitled%205.png)

Pod 특성상 삭제 후 재생성 되면 기존의 설치했던 것들이 날아가기 때문에 Pod가 기동될 때 쉘 스크립트를 이용하여 Filebeat를 자동설치 , 설정, 실행을 시킬 것입니다.

### Shell Script (filebeat_init.sh)

```bash
#/bin/bash
dpkg -i /config/log_setting/filebeat-7.9.3.deb;
cp /config/log_setting/filebeat.yml /etc/filebeat/;
service filebeat start;
```

### Yaml 수정 (Command 추가)

```yaml
command:
            - /bin/sh
            - '-c'
            - >
              /config/log_setting/filebeat_init.sh;
```

이제 Pod가 재기동 되어도 Filebeat 설치, 설정, 로그수집까지 진행되도록 세팅이 되었습니다.

## Logstash

Filebeat와 마찬가지로 Logstash 또한 우분투 Pod를 이용하여 생성합니다.

공통 config PVC를 마운트 하여 생성합니다.

**PVC Mount Path**

- Config → /config

### Service 생성

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
      port: 6044
      targetPort: 6044
selector:
    app: logstash
  type: ClusterIP
```

기본포트는 5044이지만 저는 6044를 사용하겠습니다.

### Logstash 설정

wget을 이용하여 [https://artifacts.elastic.co/downloads/logstash/logstash-7.9.3.deb](https://artifacts.elastic.co/downloads/logstash/logstash-7.9.3.deb) 설치 파일을 `/config/log_setting/` 디렉토리에 다운로드합니다.

logstash를 설치합니다.

`dpkg -i /config/log_setting/logstash-7.9.3.deb;`

logstash 설정 파일을 수정하기 위해 PVC영역으로 복사합니다.

`cp /etc/logstash/logstash.yml /config/log_setting/logstash.yml`

Logstash 설정 파일을 열고 수정합니다.

`vi /config/log_setting/logstash.yml`

![Logstash%20Filebeat/Untitled%206.png](Logstash%20Filebeat/Untitled%206.png)

`pipeline.workers` 부분에 Logstash Pod 설정 CPU를 적어주고 저장합니다.

Filebeat에서 Logstash로 전송한 로그에 대해 처리하기 위해 config 파일을 작성합니다.

### conf 양식

```json
input {

}

filter {

}

output {

}
```

**input >> filter >> output** 형식의 **pipeline**으로 구성이 되어 있습니다.

자세한 사용법은 [https://brownbears.tistory.com/67](https://brownbears.tistory.com/67) 에서 확인하시면 됩니다

### test.conf

```json
input {
        beats {
             port => "6044"
        }
        http { # Post 형식으로 받을수도 있음
             port => "5044"
        }
}

filter {
  if "wtb" in [tags] { # Filebeat에서 가져온 웹서버 로그중 필요한 정보를 필터링 함
    grok {
      match => { "message" => "%{IPORHOST:clientip} \[%{HTTPDATE:timestamp}\] \"%{WORD:method} %{DATA:request} HTTP/%{NUMBER:httpversion}\" %{NUMBER:response} %{NUMBER:responsesize:int} %{NUMBER:responsetime:int}"}
    }
    date {
      match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
    }
  }
  else if "g02_po" in [tags] { # Filebeat에서 가져온 어플리케이션 로그중 필요한 정보를 필터링 함
    grok {
      match => { "message" => "\[%{YEAR:year}.%{MONTHNUM:month}.%{MONTHDAY:day} %{HOUR:hour}:%{MINUTE:minutes}:%{SECOND:second}\]"}
    }
    mutate {
      add_field => { "timestamp" => "%{year}.%{month}.%{day} %{hour}:%{minutes}:%{second}" }
    }
    date {
      match => [ "timestamp" , "yyyy.MM.dd HH:mm:ss" ]
    }
    mutate {
      remove_field => [ "year" ,"month","day","hour","minutes","second"]
    }
  }

}

output {
 if "wtb" in [tags] {
        elasticsearch {
                hosts => ["10.4.189.176:9200"]
                index => "g00-wtb--%{+YYYY.MM.dd}"
        }
 }
 else if "g02_po" in [tags] {
        elasticsearch {
                hosts => ["10.4.189.176:9200"]
                index => "g02-po--%{+YYYY.MM.dd}"
        }
 }
}
```

Logstash config 파일을 설정하고 쉘스크립트 작성 후 Yaml 파일을 수정해줍니다.

### Shell Script (logstash_init.sh)

```bash
#/bin/bash
sleep 15
dpkg -i /config/log_setting/logstash-7.9.3.deb;
sleep 45
cp /config/log_setting/test.conf /etc/logstash/conf.d/
cp /config/log_setting/logstash.yml /etc/logstash/
cp -r /etc/logstash /usr/share/logstash/config;

nohup bash /usr/share/logstash/bin/logstash &
```

### Yaml 수정 (Command 추가)

```yaml
command:
            - /bin/sh
            - '-c'
            - >
              /config/log_setting/logstash_init.sh;
```

이제 Pod가 재기동 되어도 Logstash 설치, 설정, 로그수집까지 진행되도록 세팅이 되었습니다.

## 로그 수집 확인

웹브라우저에서 **LoadBalancer(Kibana) IP:5601** 입력하여 접속합니다.

좌측 메뉴 **Management → Stack Managerment → Index Management**에서 수집이 되고있는지 확인합니다.

![Logstash%20Filebeat/Untitled%207.png](Logstash%20Filebeat/Untitled%207.png)

![Logstash%20Filebeat/Untitled%208.png](Logstash%20Filebeat/Untitled%208.png)

정상적으로 수집이 되었다면 **Management → Stack Managerment → Kibana → Index Patterns**로 이동하여 **Index Patterns**을 만들어줍니다.

만든 이후 좌측 메뉴 **Kibana → Discover** 탭으로 이동하여 자신이 만든 **Index Pattern**을 선택하면, 로그가 수집되는 것을 확인 할 수 있습니다.

![Logstash%20Filebeat/Untitled%209.png](Logstash%20Filebeat/Untitled%209.png)

정상적으로 로그 수집이 되고 있다면, 언제든지 Kibana에서 시각화하여 대시보드를 구성 할 수 있습니다. ELK Stack을 구성해보시고 수집된 데이터를 가지고 시각화를 해보시기 바랍니다.

![Logstash%20Filebeat/Untitled%2010.png](Logstash%20Filebeat/Untitled%2010.png)