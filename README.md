## [EMK Stack을 이용한 Apache Kafka Monitoring 방법]

<br>

### [Usage & Advantage]

<br>

> #### Metricbeat를 이용하면 복잡한 설정 및 시각화 작업 필요 없이, kafka 주요 metric을 수집 및 시각화가 가능함
> 
- 쉬운 Metric 수집 : Kafka Broker에 JXM 설정 필요없음, Metricbeat 설정 단순함
- JMX 직접 수집 가능 : Jolokia module을 통해 jmx metric 직접 수집
- Kafka Dashboard 제공 : 사전에 정의된 kafka monitoring 용 dashboard 제공

<br>

### [환경 구성도]

![image](https://user-images.githubusercontent.com/30817824/172817156-666da4aa-fbf9-431f-8fef-5379677602af.png)


(refer: https://github.com/freepsw/kafka-metrics-monitoring)

<br><br>

#### [사전 준비 사항]
- Elasticsearch 및 Kibana가 설치되어 있어야 함
  (refer: https://github.com/maisonde3cochons/kafka-monitoring-elasticstack)

<br>

----------------------------------------------------

<br><br>

> ##  STG.01 Zookeeper, Kafka, ElasticSearch, Kibana 기동

<br>

#### 1) broker-01 서버에 접속하여 zookeeper 및 kafka를 기동한다


```
./start_zk.sh
./start_kafka.sh
```

<br>

#### 2) Monitoring 서버에 접속하여 ElasticSearch 및 Kibana를 기동한다

```
# elasticsearch 기동 및 확인
cd ~/elasticsearch-7.10.2/
./bin/elasticsearch -d -p elastic_pid
curl -X GET "localhost:9200/?pretty"

# kibana 기동
cd ~/kibana-7.10.2-linux-x86_64
nohup bin/kibana &


# extra. kibana process kill 필요 시 :
## 마지막 칼럼의 16998이 process id (process id는 개인별로 다르게 표시됨)
netstat -anp | grep 5601
tcp        0      0 0.0.0.0:5601            0.0.0.0:*               LISTEN      1161/bin/../node/b

## process 종료
kill -9 1161
```

<br><br>

> ##  STG.02 MetricBeat 설치 및 설정

<br>

#### 1) MetricBeat 설치

```
cd ~
curl -OL https://artifacts.elastic.co/downloads/beats/metricbeat/metricbeat-oss-7.10.2-linux-x86_64.tar.gz
tar xvf metricbeat-oss-7.10.2-linux-x86_64.tar.gz
rm -rf metricbeat-oss-7.10.2-linux-x86_64.tar.gz
cd ~/metricbeat-7.10.2-linux-x86_64
```

<br>

#### 2) MetricBeat 설정

- metricbeat은 각 모듈별로 다양한 지원(시각화, 설정 등)을 한다. 
- 이를 위해서는 각 모듈을 사용할 수 있도록 아래 설정이 필요

```
## 현재 활성돠된(사용 가능한) module을 확인한다. 
## kafka는 기본으로 비활성화되어 있음. 
cd ~/metricbeat-7.10.2-linux-x86_64
./metricbeat modules list
Enabled:
system

Disabled:
apache
docker
http
kafka
....

## kafak module 활성화
./metricbeat modules enable kafka
Enabled kafka

## kafka module이 활성화 되었는지 확인 
./metricbeat modules list
Enabled:
kafka
system


## setup 명령어를 실행하면, metricbeat에서 수집한 데이터를 시각화할 화면과 index를 자동을 로딩한다. 
./metricbeat setup -e kafka
........
Index setup finished.
Loading dashboards (Kibana must be running and reachable)
2021-12-09T06:57:46.315Z	INFO	kibana/client.go:119	Kibana url: http://localhost:5601
2021-12-09T06:57:46.658Z	INFO	kibana/client.go:119	Kibana url: http://localhost:5601

2021-12-09T06:58:26.564Z	INFO	instance/beat.go:815	Kibana dashboards successfully loaded.
Loaded dashboards

```


<br><br>

> ##  STG.03 MetricBeat 실행

<br>

#### 1) kafka module에서 사용할 config를 수정

- metric을 수집할 broker의 ip:port를 수정 
```
## metric
cd ~/metricbeat-7.10.2-linux-x86_64
vi modules.d/kafka.yml
- module: kafka
  #metricsets:
  #  - partition
  #  - consumergroup
  period: 3s
  hosts: ["broker-01:9092"]
```

<br>

#### 2) MetricBeat 실행

```
cd ~/metricbeat-7.10.2-linux-x86_64
./metricbeat -e
```

<br>

#### 3) 생성된 index (metricbeat-7.10.2-2021.12.10) 확인 
- 자동으로 kibana에서 시각화할 metric을 저장할 index가 생성된 것을 확인

```
curl -X GET "localhost:9200/_cat/indices/"
yellow open metricbeat-7.10.2-2021.12.10 7L6dT3IMQu6TgdMYSBufvg 1 1   0  0    208b    208b
```


<br><br>

> ##  STG.04 Kibana에서 자동으로 생성된 Dashboard 확인

<br>

#### 01) GCP VPC Network Firewall 설정 (5601 port 허용)

<br>

![image](https://user-images.githubusercontent.com/30817824/172518925-0a960d8a-8a56-4d92-9af6-bf24231fcf53.png)

<br>

![image](https://user-images.githubusercontent.com/30817824/172541307-5011100c-2428-4d12-aa07-93d52feeb89d.png)


<br>

#### 02) Kibana 접속

```
http://<External IP>:5601
```

<br>

#### 03) 자동 생성된 Dashboard 확인

![image](https://user-images.githubusercontent.com/30817824/172825176-b7de3bb5-ed33-41d2-9985-356cfba5e20f.png)

<br><br>

![image](https://user-images.githubusercontent.com/30817824/172825203-d37fb2ae-f0db-4f4a-9a77-f15d8bfb68eb.png)

<br><br>

![image](https://user-images.githubusercontent.com/30817824/172825227-f9543414-c918-4283-928c-9f04e5d07635.png)


