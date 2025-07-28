# Домашнее задание к занятию 15 «Система сбора логов Elastic Stack»

> Репозиторий: hw-30\
> Выполнил: Асадбек Асадбеков\
> Дата: июль 2025

## Описание

В рамках данного задания был развёрнут стек Elastic Stack (ELK) с помощью Docker Compose:  
- **Elasticsearch (hot и warm-ноды)**
- **Logstash**
- **Kibana**
- **Filebeat**

Цель: автоматизированно собирать логи Docker-контейнеров средствами Filebeat и Logstash, хранить и визуализировать их в Elasticsearch и Kibana.

---

## Скриншоты и описания

### 1. Список контейнеров через 5 минут после старта (должно быть 5):

![docker ps](https://github.com/asad-bekov/hw-30/blob/main/img/2.PNG)

*На скриншоте видно, что все сервисы (filebeat, kibana, logstash, es-hot, es-warm) успешно подняты и работают.*

---

### 2. Скриншот интерфейса Kibana

![Kibana Home](https://github.com/asad-bekov/hw-30/blob/main/img/1.PNG)

*Kibana успешно подключена к elasticsearch, открывается главная страница, доступен весь функционал для дальнейшей настройки.*

---

### 3. Скриншот Dashboard (Discover) с логами

![Kibana Discover](https://github.com/asad-bekov/hw-30/blob/main/img/3.PNG)

*В разделе Discover видны события, полученные через Filebeat → Logstash → Elasticsearch, отображаются поля и таймлайны.*

---

## Конфигурационные файлы стека

### docker-compose.yml

<details>
<summary>Показать docker-compose.yml</summary>

```yaml
version: '3.7'
services:
  es-hot:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.2
    container_name: es-hot
    environment:
      - node.name=es-hot
      - cluster.name=elk-cluster
      - node.roles=master,data,ingest
      - bootstrap.memory_lock=true
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - discovery.seed_hosts=es-hot,es-warm
      - cluster.initial_master_nodes=es-hot
      - xpack.security.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    ports:
      - 9200:9200
    networks:
      - elk

  es-warm:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.13.2
    container_name: es-warm
    environment:
      - node.name=es-warm
      - cluster.name=elk-cluster
      - node.roles=data
      - bootstrap.memory_lock=true
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - discovery.seed_hosts=es-hot,es-warm
      - cluster.initial_master_nodes=es-hot
      - xpack.security.enabled=false
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      - elk

  logstash:
    image: docker.elastic.co/logstash/logstash:8.13.2
    container_name: logstash
    environment:
      - LS_JAVA_OPTS=-Xmx256m -Xms256m
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - 5044:5044
      - 5000:5000
    networks:
      - elk
    depends_on:
      - es-hot

  kibana:
    image: docker.elastic.co/kibana/kibana:8.13.2
    container_name: kibana
    ports:
      - 5601:5601
    environment:
      - ELASTICSEARCH_HOSTS=http://es-hot:9200
    networks:
      - elk
    depends_on:
      - es-hot

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.13.2
    container_name: filebeat
    user: root
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
    networks:
      - elk
    depends_on:
      - logstash

networks:
  elk:
    driver: bridge
```
</details>

---

### logstash.conf

<details>
<summary>Показать logstash.conf</summary>

```conf
input {
  tcp {
    port => 5000
    codec => json
  }
  beats {
    port => 5044
  }
}
output {
  elasticsearch {
    hosts => ["http://es-hot:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
  }
  stdout { codec => rubydebug }
}
```
</details>

---

### filebeat.yml

<details>
<summary>Показать filebeat.yml</summary>

```yaml
filebeat.inputs:
  - type: container
    paths:
      - /var/lib/docker/containers/*/*.log

output.logstash:
  hosts: ["logstash:5044"]
```
</details>

---

## Проблемы и их решения

### Проблема 1: **Elasticsearch падал и не назначал шарды (RED/YELLOW health)**

- **Решение:**  

  Понял, что обе ноды elasticsearch имеют роль `data`, и только одна из них — мастер.  
  Удалил все старые битые индексы, сбросил volume:
  ```
  sudo docker-compose down -v
  sudo docker-compose up -d
  ```
  Использовал только 1 primary shard и 0 реплик для теста:
  ```
  curl -X PUT "localhost:9200/_template/logstash" -H 'Content-Type: application/json' -d '{
    "index_patterns": ["logstash-*"] ,
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0
    }
  }'
  ```

---

### Проблема 2: **Filebeat не стартовал из-за прав на filebeat.yml**
- **Решение:**  
  ```
  sudo chown root:root filebeat.yml
  sudo chmod 0644 filebeat.yml
  ```

---

### Проблема 3: **Kibana не могла подключиться к Elasticsearch**
- **Решение:**  
  Надо было подождать 3–5 минут после запуска стека (особенно после очистки томов).  

---

### Проблема 4: **Кластер блокировал запись из-за недостатка места**
- **Решение:**  
  Освободил на диске достаточное место (>10% для docker volume).

---

## ИТОГ

Все компоненты Elastic Stack были успешно развернуты и интегрированы друг с другом.  
Логи docker-контейнеров автоматически поступают в elasticsearch и визуализируются через Kibana.

---

**[Репозиторий и скриншоты](https://github.com/asad-bekov/hw-30/)**  
Скриншоты:  
- [img/1.PNG](https://github.com/asad-bekov/hw-30/blob/main/img/1.PNG) — docker ps  
- [img/2.PNG](https://github.com/asad-bekov/hw-30/blob/main/img/2.PNG) — Kibana Home  
- [img/3.PNG](https://github.com/asad-bekov/hw-30/blob/main/img/3.PNG) — Kibana Discover

---
