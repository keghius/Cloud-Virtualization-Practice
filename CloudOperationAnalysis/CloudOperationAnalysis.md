# 클라우드 가상화 기술

## 클라우드 운영 및 분석 기술 실습

본 실습에서는 클라우드 네이티브 환경의 **관측 가능성(Observability)** 을 직접 구축하는 과정을 다룸

이론에서 학습한 **Metrics, Logs, Traces** 의 세 가지 축을 실제 도구로 구현하고, 수집 → 저장 → 시각화 → 알림 → 분산 추적까지의 전체 파이프라인을 단일 호스트(Ubuntu 24.04) 위에 네이티브로 배포함

| 구분 | 사용 도구 | 포트 |
| --- | --- | --- |
| 메트릭 수집 | Prometheus, Node Exporter, Custom Exporter | 9090, 9100, 8000 |
| 알림 | Alertmanager | 9093 |
| 시각화 | Grafana | 3000 |
| 로그 수집/저장 | Filebeat, Logstash, Elasticsearch | 5044, 9200 |
| 로그 시각화 | Kibana | 5601 |
| 분산 추적 수집 | OpenTelemetry Collector | 4317, 4318 |
| 분산 추적 시각화 | Jaeger | 16686 |

[figure1]

---

## 1. 실습 환경 사전 준비

### 환경 요구사항

- Ubuntu 24.04 LTS
- 최소 RAM 6GB (Elasticsearch + Logstash + Kibana 동시 구동 시 필요)
- 디스크 여유 공간 10GB 이상
- 인터넷 연결 (apt 저장소 및 바이너리 다운로드)

### 필수 패키지 설치

```
# 패키지 인덱스 갱신
sudo apt update

# 실습 전반에서 사용할 도구 설치
sudo apt install -y curl wget tar gnupg apt-transport-https \
  software-properties-common ca-certificates python3 python3-pip \
  python3-venv stress-ng net-tools
```

### 작업 디렉토리 생성

```
# 다운로드 및 설정 파일 저장용 디렉토리
mkdir -p ~/observability
cd ~/observability
```

### 참고

- 본 실습의 모든 컴포넌트는 `systemd` 서비스로 등록하여 영구 실행되도록 구성함
- 각 컴포넌트는 전용 시스템 사용자 계정으로 실행하여 권한을 최소화함
- 메모리가 부족한 경우 ELK 스택(섹션 12~16)은 다른 컴포넌트를 중지한 후 진행 가능

---

## 2. Prometheus 설치

`Prometheus`는 Pull 방식으로 메트릭을 수집하는 시계열 데이터베이스 및 모니터링 시스템

### Prometheus 전용 사용자 생성

```
# 시스템 사용자 생성 (로그인 불가, 홈 디렉토리 없음)
sudo useradd --no-create-home --shell /bin/false prometheus

# 데이터 및 설정 디렉토리 생성
sudo mkdir -p /etc/prometheus /var/lib/prometheus
```

### 바이너리 다운로드 및 설치

```
# Prometheus 릴리즈 다운로드 (버전은 최신으로 변경 가능)
cd ~/observability
wget https://github.com/prometheus/prometheus/releases/download/v2.55.1/prometheus-2.55.1.linux-amd64.tar.gz

# 압축 해제
tar xvf prometheus-2.55.1.linux-amd64.tar.gz

# 바이너리를 /usr/local/bin 으로 이동
sudo cp prometheus-2.55.1.linux-amd64/prometheus /usr/local/bin/
sudo cp prometheus-2.55.1.linux-amd64/promtool /usr/local/bin/

# 설정 및 콘솔 파일 복사
sudo cp -r prometheus-2.55.1.linux-amd64/consoles /etc/prometheus
sudo cp -r prometheus-2.55.1.linux-amd64/console_libraries /etc/prometheus

# 소유권 변경
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

### 설치 확인

```
# 버전 확인
prometheus --version
```

[figure2]

---

## 3. Prometheus 기본 설정

### prometheus.yml 작성

```
# 설정 파일 생성
sudo tee /etc/prometheus/prometheus.yml > /dev/null <<EOF
global:
  scrape_interval: 15s          # 전역 수집 주기
  evaluation_interval: 15s      # 알림 규칙 평가 주기

scrape_configs:
  - job_name: 'prometheus'      # 자기 자신 모니터링
    static_configs:
      - targets: ['localhost:9090']
EOF

# 소유권 적용
sudo chown prometheus:prometheus /etc/prometheus/prometheus.yml
```

### systemd 서비스 등록

```
sudo tee /etc/systemd/system/prometheus.service > /dev/null <<EOF
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus/ \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
EOF
```

### 서비스 시작

```
# systemd 데몬 재로드
sudo systemctl daemon-reload

# Prometheus 시작 및 부팅 시 자동 시작 설정
sudo systemctl start prometheus
sudo systemctl enable prometheus

# 상태 확인
sudo systemctl status prometheus
```

### 웹 UI 접속 확인

브라우저에서 `http://<호스트 IP>:9090` 접속

상단 메뉴 `Status > Targets` 에서 `prometheus` 타겟이 `UP` 상태인지 확인

[figure3]

### 참고

- `--web.enable-lifecycle` 옵션은 `curl -X POST http://localhost:9090/-/reload` 로 무중단 설정 재로드를 가능하게 함
- TSDB 저장 경로(`/var/lib/prometheus`)는 시계열 데이터가 누적되는 위치이며 디스크 용량 관리 대상
- 기본 보존 기간은 15일이며 `--storage.tsdb.retention.time=30d` 옵션으로 변경 가능

---

## 4. Node Exporter 설치

`Node Exporter`는 호스트의 CPU, Memory, Disk, Network 등 OS 및 하드웨어 메트릭을 수집하여 Prometheus 가 읽을 수 있는 포맷으로 노출하는 Exporter

### 전용 사용자 및 바이너리 설치

```
# 시스템 사용자 생성
sudo useradd --no-create-home --shell /bin/false node_exporter

# 바이너리 다운로드
cd ~/observability
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz

# 압축 해제 및 설치
tar xvf node_exporter-1.8.2.linux-amd64.tar.gz
sudo cp node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

### systemd 서비스 등록

```
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter --web.listen-address=:9100

[Install]
WantedBy=multi-user.target
EOF

# 서비스 시작
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter
```

### 메트릭 노출 확인

```
# /metrics 엔드포인트 직접 호출 (Pull 방식의 실체 확인)
curl http://localhost:9100/metrics | head -30
```

[figure4]

### 참고

- 출력되는 `# HELP`, `# TYPE` 주석은 Prometheus 표준 노출 포맷의 메타데이터임 (이론 슬라이드 39 참고)
- `node_cpu_seconds_total` 과 같이 `_total` 접미사가 붙은 메트릭은 Counter 타입임
- `node_memory_MemAvailable_bytes` 와 같이 현재 상태를 나타내는 메트릭은 Gauge 타입임

---

## 5. Prometheus 와 Node Exporter 연동

### 수집 대상 추가

```
# prometheus.yml 수정
sudo tee /etc/prometheus/prometheus.yml > /dev/null <<EOF
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'                              # Node Exporter Job 추가
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance: 'lab-server'                  # 사용자 정의 레이블 부여
EOF
```

### 무중단 설정 재로드

```
# 서비스 재시작 없이 설정 반영
curl -X POST http://localhost:9090/-/reload
```

### Targets 확인

브라우저에서 `http://<호스트 IP>:9090/targets` 접속

`node` Job 하위에 `lab-server` 인스턴스가 `UP` 상태인지 확인

[figure5]

### 참고

- 레이블(`instance`, `job` 등)은 동일한 메트릭 이름을 가진 여러 시계열을 구분하는 핵심 식별자임
- PromQL 의 모든 필터링과 그룹화는 레이블 기반으로 동작함
- 동적 환경에서는 `static_configs` 대신 `kubernetes_sd_configs`, `consul_sd_configs` 등의 서비스 디스커버리를 사용함

---

## 6. PromQL 기초 실습

`PromQL(Prometheus Query Language)` 은 시계열 데이터를 선택하고 집계하기 위한 함수형 쿼리 언어

Prometheus UI 의 `Graph` 탭에서 다음 쿼리들을 차례로 실행해보고 결과 확인

### Instant Vector 와 Range Vector

```
# Instant Vector: 현재 시점의 단일 값 집합
up

# Range Vector: 최근 5분간의 값 집합 (시계열 구간)
node_cpu_seconds_total[5m]
```

### Counter 메트릭과 rate() 함수

```
# 누적 카운터 자체를 보면 단조 증가 그래프만 나옴
node_cpu_seconds_total{mode="user"}

# rate() 로 초당 증가율을 계산해야 의미 있는 값이 나옴
rate(node_cpu_seconds_total{mode="user"}[5m])
```

### 집계 연산 (Aggregation)

```
# CPU mode 별로 합산하여 전체 CPU 사용 분포 확인
sum by (mode) (rate(node_cpu_seconds_total[5m]))

# CPU 사용률(%) 계산 - idle 시간의 비율을 100에서 빼는 방식
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### 시스템 리소스 쿼리 예시

```
# 가용 메모리 비율(%)
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100

# 디스크 사용량(%) - 루트 파일시스템 기준
100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)

# 네트워크 수신 트래픽(bytes/sec)
rate(node_network_receive_bytes_total{device!="lo"}[5m])
```

[figure6]

### 부하 생성을 통한 그래프 변화 관찰

별도 터미널에서 CPU 부하 발생 후, Prometheus Graph 탭에서 CPU 사용률 쿼리가 변하는 것을 확인

```
# 4개 코어에 60초간 부하 생성
stress-ng --cpu 4 --timeout 60s
```

### 참고

- `rate()` 는 평균 증가율(범위 내 첫/끝 데이터 포인트 기반)
- `irate()` 는 가장 최근 두 데이터 포인트만으로 계산한 순간 증가율 (변동성 크지만 즉시성 높음)
- Counter 가 재시작되어 값이 초기화되더라도 `rate()` 는 이를 자동으로 보정함
- `histogram_quantile()` 같은 분포 함수는 Histogram 타입 메트릭에만 적용 가능

---

## 7. Grafana 설치

`Grafana` 는 시계열 데이터 및 로그를 시각화하는 오픈소스 대시보드 도구

### 공식 APT 저장소 등록

```
# GPG 키 추가
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null

# 저장소 등록
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
```

### 설치 및 시작

```
# 패키지 인덱스 갱신 후 설치
sudo apt update
sudo apt install -y grafana

# 서비스 시작
sudo systemctl daemon-reload
sudo systemctl start grafana-server
sudo systemctl enable grafana-server

# 상태 확인
sudo systemctl status grafana-server
```

### 웹 UI 접속

브라우저에서 `http://<호스트 IP>:3000` 접속

초기 로그인 정보:
- 사용자명: `admin`
- 비밀번호: `admin` (최초 로그인 시 변경 요구됨)

[figure7]

---

## 8. Grafana 데이터소스 및 대시보드 구성

### Prometheus 데이터소스 연결

1. 좌측 메뉴 `Connections > Data sources` 진입
2. `Add new data source` 클릭 후 `Prometheus` 선택
3. `Connection > Prometheus server URL` 에 `http://localhost:9090` 입력
4. 하단 `Save & test` 클릭하여 연결 확인

[figure8]

### 새 대시보드 생성

1. 좌측 메뉴 `Dashboards > New > New dashboard` 클릭
2. `Add visualization` 클릭 후 데이터소스로 Prometheus 선택

### 패널 1: CPU 사용률 (Time series)

쿼리:
```
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

우측 패널 옵션:
- `Title`: CPU Usage (%)
- `Standard options > Unit`: Percent (0-100)
- `Visualizations`: Time series

### 패널 2: 메모리 사용량 (Gauge)

쿼리:
```
100 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100)
```

패널 옵션:
- `Title`: Memory Usage (%)
- `Visualizations`: Gauge
- `Standard options > Unit`: Percent (0-100)
- `Thresholds`: 70 (orange), 90 (red)

### 패널 3: 디스크 사용량 (Stat)

쿼리:
```
100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)
```

패널 옵션:
- `Title`: Disk Usage (%)
- `Visualizations`: Stat

### 패널 4: 네트워크 트래픽 (Time series)

쿼리:
```
rate(node_network_receive_bytes_total{device!="lo"}[5m])
rate(node_network_transmit_bytes_total{device!="lo"}[5m])
```

패널 옵션:
- `Title`: Network Traffic
- `Standard options > Unit`: bytes/sec

### 대시보드 저장

우측 상단 `Save dashboard` 아이콘 클릭 후 이름 입력 (예: `Node Overview`)

[figure9]

### 참고

- 각 패널 우상단 메뉴에서 `Inspect > JSON` 을 통해 패널 정의를 JSON 으로 확인 가능
- 대시보드 전체는 `Share > Export` 에서 JSON 으로 export 하여 버전 관리 가능 (이론 슬라이드 37 참고)
- 동일한 JSON 을 다른 Grafana 인스턴스에 import 하면 동일 대시보드 재현 가능

---

## 9. Grafana 변수 (Templating)

대시보드에 변수를 추가하여 동적으로 필터링되는 대시보드 구성

### 변수 추가

1. 대시보드 우상단 `Dashboard settings (톱니바퀴)` > `Variables` 진입
2. `Add variable` 클릭

### Job 변수 생성

| 항목 | 값 |
| --- | --- |
| Variable type | Query |
| Name | `job` |
| Data source | Prometheus |
| Query | `label_values(job)` |
| Refresh | On dashboard load |

### Instance 변수 생성 (Job 에 의존)

| 항목 | 값 |
| --- | --- |
| Variable type | Query |
| Name | `instance` |
| Query | `label_values(up{job="$job"}, instance)` |
| Include All option | 활성화 |

### 패널 쿼리에 변수 적용

기존 CPU 패널 쿼리를 아래와 같이 수정:

```
100 - (avg by (instance) (rate(node_cpu_seconds_total{job="$job",instance="$instance",mode="idle"}[5m])) * 100)
```

대시보드 상단에 `job`, `instance` 드롭다운이 생기며 선택값에 따라 패널이 자동 필터링됨

[figure10]

### 참고

- 변수 간 의존성(Chained Variables)을 사용하면 상위 변수 선택에 따라 하위 변수의 옵션이 자동으로 갱신됨
- `$variable_name` 외에도 `${variable_name}`, `[[variable_name]]` 형태로 참조 가능
- `Include All option` 활성화 시 `.*` regex 가 자동 적용되어 전체 데이터 조회 가능

---

## 10. Alertmanager 설치

`Alertmanager` 는 Prometheus 가 발생시킨 알림을 받아 중복 제거, 그룹화, 라우팅, 채널 전송을 담당하는 별도 컴포넌트

### 전용 사용자 및 디렉토리 생성

```
sudo useradd --no-create-home --shell /bin/false alertmanager
sudo mkdir -p /etc/alertmanager /var/lib/alertmanager
```

### 바이너리 설치

```
cd ~/observability
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz

tar xvf alertmanager-0.27.0.linux-amd64.tar.gz
sudo cp alertmanager-0.27.0.linux-amd64/alertmanager /usr/local/bin/
sudo cp alertmanager-0.27.0.linux-amd64/amtool /usr/local/bin/

sudo chown alertmanager:alertmanager /usr/local/bin/alertmanager /usr/local/bin/amtool
sudo chown -R alertmanager:alertmanager /etc/alertmanager /var/lib/alertmanager
```

### Alertmanager 설정 파일 작성

테스트용 알림 채널로 [webhook.site](https://webhook.site) 를 사용하면 별도 Slack/이메일 설정 없이 알림 전송 확인 가능

먼저 webhook.site 에 접속하여 본인의 고유 URL 을 발급받음

```
sudo tee /etc/alertmanager/alertmanager.yml > /dev/null <<EOF
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']         # 동일한 alertname 의 알림을 그룹화
  group_wait: 10s                 # 그룹의 첫 알림 대기 시간
  group_interval: 10s             # 동일 그룹 내 추가 알림 대기 시간
  repeat_interval: 1h             # 동일 알림 재전송 주기
  receiver: 'webhook-receiver'

receivers:
  - name: 'webhook-receiver'
    webhook_configs:
      - url: 'https://webhook.site/<발급받은-UUID>'  # 본인 URL 로 교체
        send_resolved: true       # 알림 해제 시에도 전송
EOF

sudo chown alertmanager:alertmanager /etc/alertmanager/alertmanager.yml
```

### systemd 서비스 등록 및 시작

```
sudo tee /etc/systemd/system/alertmanager.service > /dev/null <<EOF
[Unit]
Description=Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager \
  --web.listen-address=:9093

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start alertmanager
sudo systemctl enable alertmanager
```

### 웹 UI 접속

`http://<호스트 IP>:9093` 에서 Alertmanager UI 확인

[figure11]

---

## 11. Alert Rule 작성 및 Prometheus 연동

### Prometheus 가 Alertmanager 를 인식하도록 설정 추가

```
sudo tee /etc/prometheus/prometheus.yml > /dev/null <<EOF
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']

rule_files:
  - "/etc/prometheus/rules/*.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
        labels:
          instance: 'lab-server'
EOF
```

### Alert Rule 파일 작성

```
# 규칙 디렉토리 생성
sudo mkdir -p /etc/prometheus/rules

# CPU 및 메모리 관련 알림 규칙
sudo tee /etc/prometheus/rules/node-alerts.yml > /dev/null <<EOF
groups:
  - name: node-alerts
    interval: 30s
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 80
        for: 1m                   # 1분 이상 지속 시 발송
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ \$labels.instance }}"
          description: "CPU usage is {{ \$value | printf \"%.2f\" }}% on {{ \$labels.instance }}"

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ \$labels.instance }}"
          description: "Memory usage is {{ \$value | printf \"%.2f\" }}%"

      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ \$labels.instance }} is down"
EOF

sudo chown -R prometheus:prometheus /etc/prometheus/rules
```

### 규칙 문법 검증

```
# promtool 로 사전 검증
promtool check rules /etc/prometheus/rules/node-alerts.yml
```

### 설정 재로드 및 규칙 확인

```
# Prometheus 설정 재로드
curl -X POST http://localhost:9090/-/reload
```

Prometheus UI `Alerts` 탭에서 정의된 규칙 3개가 표시되는지 확인

[figure12]

### 알림 발생 테스트

CPU 부하를 80% 이상으로 유지하여 `HighCPUUsage` 알림 트리거

```
# 모든 코어에 90초간 부하 생성
stress-ng --cpu $(nproc) --timeout 90s
```

진행 과정:
1. Prometheus UI `Alerts` 탭에서 `PENDING` (조건 만족, `for` 대기 중) 상태 확인
2. 1분 경과 후 `FIRING` 상태로 전환되는지 확인
3. Alertmanager UI (`:9093`) 에서 활성 알림 수신 확인
4. webhook.site 페이지에서 webhook 호출 내역 확인

[figure13]

### 참고

- `for: 1m` 은 조건이 1분 이상 지속될 때만 알림을 전송하여 일시적 스파이크로 인한 오탐을 방지함
- Alert 의 라이프사이클은 `Inactive → Pending → Firing` 으로 전이됨
- `{{ $labels.instance }}` 와 같은 Go 템플릿 문법으로 알림 메시지에 동적 정보 삽입 가능
- 운영 환경에서는 webhook 대신 Slack, PagerDuty, 이메일 등 실제 채널을 사용함

---

## 12. Custom Exporter 개발

표준 Exporter 가 제공하지 않는 애플리케이션 비즈니스 메트릭을 노출하기 위해 Python 으로 Custom Exporter 직접 작성

### Python 가상환경 및 라이브러리 설치

```
cd ~/observability
python3 -m venv exporter-venv
source exporter-venv/bin/activate

# Prometheus Python 클라이언트 라이브러리
pip install prometheus_client
```

### Custom Exporter 코드 작성

```
cat > ~/observability/custom_exporter.py <<'EOF'
from prometheus_client import Counter, Gauge, Histogram, start_http_server
import random
import time

# Counter: 누적 증가만 가능한 메트릭 (요청 수, 에러 수 등)
REQUESTS = Counter(
    'myapp_requests_total',
    'Total HTTP requests processed',
    ['endpoint', 'status']         # 레이블 정의
)

# Gauge: 증감 가능한 현재 상태 메트릭 (활성 사용자, 큐 크기 등)
ACTIVE_USERS = Gauge(
    'myapp_active_users',
    'Currently active users'
)

# Histogram: 값의 분포를 버킷 단위로 집계 (응답 시간 분포 등)
LATENCY = Histogram(
    'myapp_request_duration_seconds',
    'Request latency distribution',
    buckets=[0.1, 0.25, 0.5, 1.0, 2.5, 5.0]
)

if __name__ == '__main__':
    # 8000번 포트에서 /metrics 엔드포인트 노출
    start_http_server(8000)
    print("Custom exporter started on :8000")

    # 비즈니스 로직 시뮬레이션 - 가상의 요청을 지속 생성
    endpoints = ['/api/login', '/api/checkout', '/api/search']
    while True:
        endpoint = random.choice(endpoints)
        # 90% 성공, 10% 에러를 가정
        status = '200' if random.random() < 0.9 else '500'

        REQUESTS.labels(endpoint=endpoint, status=status).inc()
        ACTIVE_USERS.set(random.randint(10, 100))
        LATENCY.observe(random.expovariate(2))   # 평균 0.5초 응답시간 분포

        time.sleep(0.5)
EOF
```

### 직접 실행 및 메트릭 확인

```
# 가상환경 활성화 상태에서 실행
python3 ~/observability/custom_exporter.py
```

별도 터미널에서 메트릭 노출 확인:

```
curl http://localhost:8000/metrics | grep myapp
```

3종 메트릭(Counter, Gauge, Histogram)이 모두 노출되는 것 확인

[figure14]

### systemd 서비스 등록

운영을 위해 서비스로 등록 (Ctrl+C 로 종료 후)

```
sudo tee /etc/systemd/system/custom-exporter.service > /dev/null <<EOF
[Unit]
Description=Custom Prometheus Exporter
After=network.target

[Service]
Type=simple
User=$USER
WorkingDirectory=$HOME/observability
ExecStart=$HOME/observability/exporter-venv/bin/python3 $HOME/observability/custom_exporter.py
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start custom-exporter
sudo systemctl enable custom-exporter
```

### Prometheus 수집 대상에 추가

```
# prometheus.yml 의 scrape_configs 에 추가
sudo sed -i '/- job_name: '\''node'\''/i\  - job_name: '\''custom-app'\''\n    static_configs:\n      - targets: ['\''localhost:8000'\'']\n' /etc/prometheus/prometheus.yml

# 설정 재로드
curl -X POST http://localhost:9090/-/reload
```

### PromQL 로 Custom 메트릭 조회

Prometheus UI 에서 다음 쿼리 실행:

```
# 엔드포인트별 초당 요청 수
sum by (endpoint) (rate(myapp_requests_total[1m]))

# 에러율(%) - 500 응답 비율
sum(rate(myapp_requests_total{status="500"}[1m])) / sum(rate(myapp_requests_total[1m])) * 100

# 응답 시간 P99 (Histogram 메트릭에 histogram_quantile 적용)
histogram_quantile(0.99, sum by (le) (rate(myapp_request_duration_seconds_bucket[1m])))

# 현재 활성 사용자 수
myapp_active_users
```

[figure15]

### 참고

- Counter 는 단조 증가만 가능하므로 `_total` 접미사를 붙이는 것이 관례
- Gauge 는 `set()`, `inc()`, `dec()` 모두 호출 가능
- Histogram 은 내부적으로 `_bucket`, `_sum`, `_count` 3개의 시계열을 생성하며 `histogram_quantile()` 함수와 함께 사용함
- Summary 타입은 클라이언트 측에서 분위수를 계산하므로 정확하지만 집계가 불가능함

---

## 13. Elasticsearch 설치

`Elasticsearch` 는 JSON 기반의 분산형 RESTful 검색 엔진으로, ELK 스택의 저장소 역할 담당

### Elastic 공식 APT 저장소 등록

```
# GPG 키 추가
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

# 저장소 등록 (8.x 버전)
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

### 설치

```
sudo apt update
sudo apt install -y elasticsearch
```

설치 완료 시 출력되는 초기 비밀번호(`elastic` 사용자)를 별도 저장

만약 놓쳤다면 아래 명령어로 재설정:

```
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

### 메모리 설정 (실습용 축소)

기본값은 ES 가 시스템 메모리의 절반을 사용하려 하므로, 실습 환경에 맞게 축소

```
sudo tee /etc/elasticsearch/jvm.options.d/heap.options > /dev/null <<EOF
-Xms1g
-Xmx1g
EOF
```

### 단일 노드 모드 설정

```
# elasticsearch.yml 주요 항목 수정
sudo sed -i 's/#cluster.name:.*/cluster.name: lab-cluster/' /etc/elasticsearch/elasticsearch.yml
sudo sed -i 's/#node.name:.*/node.name: node-1/' /etc/elasticsearch/elasticsearch.yml
sudo sed -i 's/#network.host:.*/network.host: 0.0.0.0/' /etc/elasticsearch/elasticsearch.yml

# 실습 편의를 위해 보안 기능 비활성화 (운영 환경에서는 활성화 필수)
sudo sed -i 's/xpack.security.enabled: true/xpack.security.enabled: false/' /etc/elasticsearch/elasticsearch.yml
sudo sed -i 's/xpack.security.enrollment.enabled: true/xpack.security.enrollment.enabled: false/' /etc/elasticsearch/elasticsearch.yml

# HTTP/Transport SSL 비활성화
sudo sed -i '/xpack.security.http.ssl:/,/^[^ ]/{s/enabled: true/enabled: false/}' /etc/elasticsearch/elasticsearch.yml
sudo sed -i '/xpack.security.transport.ssl:/,/^[^ ]/{s/enabled: true/enabled: false/}' /etc/elasticsearch/elasticsearch.yml
```

### 서비스 시작

```
sudo systemctl daemon-reload
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch

# 시작에 1~2분 소요됨
sudo systemctl status elasticsearch
```

### 동작 확인

```
# 클러스터 정보 조회
curl http://localhost:9200

# 클러스터 헬스 확인 (yellow 또는 green 이면 정상)
curl http://localhost:9200/_cluster/health?pretty
```

[figure16]

### 참고

- 운영 환경에서는 반드시 `xpack.security.enabled: true` 로 유지하고 TLS 와 인증을 활성화해야 함
- 단일 노드 클러스터는 `status: yellow` 가 정상 상태임 (복제 샤드가 할당될 노드가 없기 때문)
- ES 데이터는 `/var/lib/elasticsearch/` 에 저장되며 디스크 사용량 주의

---

## 14. Kibana 설치 및 설정

`Kibana` 는 Elasticsearch 데이터를 검색, 시각화, 관리하는 웹 UI

### 설치

```
# Elasticsearch 저장소가 이미 등록되어 있으므로 바로 설치
sudo apt install -y kibana
```

### 설정

```
# Elasticsearch 연결 및 외부 접속 허용
sudo sed -i 's/#server.host:.*/server.host: "0.0.0.0"/' /etc/kibana/kibana.yml
sudo sed -i 's|#elasticsearch.hosts:.*|elasticsearch.hosts: ["http://localhost:9200"]|' /etc/kibana/kibana.yml
```

### 서비스 시작

```
sudo systemctl daemon-reload
sudo systemctl start kibana
sudo systemctl enable kibana

# 시작에 1분 정도 소요됨
sudo systemctl status kibana
```

### 웹 UI 접속

브라우저에서 `http://<호스트 IP>:5601` 접속

`Explore on my own` 선택 후 메인 화면 진입

[figure17]

---

## 15. Logstash 설치 및 파이프라인 구성

`Logstash` 는 다양한 소스로부터 데이터를 수집하여 변환 후 저장소로 전송하는 ETL 도구

### 설치

```
sudo apt install -y logstash
```

### 파이프라인 설정 작성

Filebeat 로부터 입력을 받아, Grok 패턴으로 파싱하여 Elasticsearch 로 전송하는 파이프라인 구성

```
sudo tee /etc/logstash/conf.d/main.conf > /dev/null <<'EOF'
input {
  beats {
    port => 5044                                    # Filebeat 수신 포트
  }
}

filter {
  # Nginx access log 형식 파싱
  if [fields][log_type] == "nginx" {
    grok {
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }
    # 응답 시간을 숫자형으로 변환
    mutate {
      convert => { "response" => "integer" }
      convert => { "bytes" => "integer" }
    }
    # @timestamp 를 로그 내부 타임스탬프로 교체
    date {
      match => [ "timestamp", "dd/MMM/yyyy:HH:mm:ss Z" ]
    }
  }

  # syslog 파싱
  if [fields][log_type] == "syslog" {
    grok {
      match => { "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_host} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}" }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "logs-%{[fields][log_type]}-%{+YYYY.MM.dd}"   # 일자별 인덱스
  }

  # 디버깅용 표준 출력 (운영시 제거 권장)
  stdout {
    codec => rubydebug
  }
}
EOF
```

### 설정 검증 및 시작

```
# 설정 문법 검증
sudo -u logstash /usr/share/logstash/bin/logstash --config.test_and_exit -f /etc/logstash/conf.d/main.conf

# 서비스 시작
sudo systemctl start logstash
sudo systemctl enable logstash

# 시작에 30초 정도 소요됨
sudo systemctl status logstash
```

### 동작 확인

```
# Logstash 가 5044 포트에서 수신 대기 중인지 확인
sudo ss -tlnp | grep 5044
```

[figure18]

### 참고

- Grok 패턴은 정규표현식의 추상화된 형태로 `%{PATTERN:field_name}` 문법 사용
- 내장 패턴 목록은 [Logstash grok-patterns 저장소](https://github.com/logstash-plugins/logstash-patterns-core/tree/main/patterns) 에서 확인 가능
- `stdout { codec => rubydebug }` 는 개발 단계에서 파싱 결과를 직접 확인할 때 유용함
- 인덱스 명에 날짜를 포함하면 일자별로 분리 저장되어 보존 정책 관리가 용이함

---

## 16. Filebeat 설치 및 로그 수집

`Filebeat` 는 서버에 설치되어 로그 파일을 읽고 중앙으로 전송하는 경량 Shipper

### 설치

```
sudo apt install -y filebeat
```

### 테스트용 로그 소스 준비

실제 nginx 가 없어도 동작 확인이 가능하도록 샘플 access log 생성

```
sudo mkdir -p /var/log/nginx-sample

# 임의의 nginx access log 형식 데이터 생성
sudo tee /var/log/nginx-sample/access.log > /dev/null <<'EOF'
192.168.1.10 - - [21/Mar/2024:14:23:01 +0900] "GET /api/login HTTP/1.1" 200 1024 "-" "Mozilla/5.0"
192.168.1.11 - - [21/Mar/2024:14:23:02 +0900] "POST /api/checkout HTTP/1.1" 500 512 "-" "Mozilla/5.0"
192.168.1.12 - - [21/Mar/2024:14:23:03 +0900] "GET /api/search?q=test HTTP/1.1" 200 2048 "-" "curl/7.81"
192.168.1.13 - - [21/Mar/2024:14:23:04 +0900] "GET /api/login HTTP/1.1" 401 256 "-" "Mozilla/5.0"
192.168.1.14 - - [21/Mar/2024:14:23:05 +0900] "POST /api/checkout HTTP/1.1" 200 768 "-" "Mozilla/5.0"
EOF
```

### Filebeat 설정

```
sudo tee /etc/filebeat/filebeat.yml > /dev/null <<'EOF'
filebeat.inputs:
  - type: filestream
    id: nginx-access
    enabled: true
    paths:
      - /var/log/nginx-sample/*.log
    fields:
      log_type: nginx                # Logstash 필터 조건에 사용
    fields_under_root: false

  - type: filestream
    id: system-syslog
    enabled: true
    paths:
      - /var/log/syslog
    fields:
      log_type: syslog

output.logstash:
  hosts: ["localhost:5044"]

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
EOF
```

### 권한 설정 및 시작

```
# syslog 읽기 권한 부여 (adm 그룹에 추가)
sudo usermod -a -G adm root

# 서비스 시작
sudo systemctl start filebeat
sudo systemctl enable filebeat

sudo systemctl status filebeat
```

### Elasticsearch 에 인덱스 생성 확인

```
# 생성된 인덱스 목록 조회
curl http://localhost:9200/_cat/indices?v
```

`logs-nginx-*`, `logs-syslog-*` 인덱스가 생성되어 있어야 함

```
# nginx 인덱스의 문서 수 확인
curl http://localhost:9200/logs-nginx-*/_count?pretty

# 문서 내용 샘플 조회
curl http://localhost:9200/logs-nginx-*/_search?pretty&size=2
```

[figure19]

### 새 로그 추가하여 실시간 수집 확인

```
# 새 로그 라인 추가
echo '192.168.1.99 - - [21/Mar/2024:14:25:00 +0900] "GET /api/test HTTP/1.1" 200 100 "-" "test-agent"' | sudo tee -a /var/log/nginx-sample/access.log

# 몇 초 대기 후 ES 에서 새 문서 확인
sleep 5
curl http://localhost:9200/logs-nginx-*/_count?pretty
```

### 참고

- `filestream` 입력 타입은 Filebeat 7.x 이후 도입된 신형이며, 구형 `log` 타입을 대체함
- 파일 회전(rotation) 추적은 inode 기반으로 처리되며 상태는 `/var/lib/filebeat/registry/` 에 저장됨
- Filebeat → ES 직접 전송도 가능하나, 파싱이 필요하면 Logstash 경유가 일반적임

---

## 17. Kibana 에서 로그 검색 및 시각화

### Data View 생성

1. Kibana 좌측 메뉴 `Management > Stack Management` 진입
2. `Kibana > Data Views` 선택 후 `Create data view` 클릭
3. 입력값:
   - Name: `Nginx Logs`
   - Index pattern: `logs-nginx-*`
   - Timestamp field: `@timestamp`
4. `Save data view to Kibana` 클릭

동일하게 `logs-syslog-*` 도 생성

[figure20]

### Discover 로 로그 검색

1. 좌측 메뉴 `Analytics > Discover` 진입
2. 상단 Data view 드롭다운에서 `Nginx Logs` 선택
3. 우측 상단 시간 범위를 `Last 1 hour` 또는 `Last 24 hours` 로 설정

### KQL 쿼리 예시

검색창에 다음 쿼리들을 차례로 입력:

```
# 500 에러만 필터링
response : 500

# 특정 엔드포인트 + 에러
request : "/api/checkout" and response >= 400

# 클라이언트 IP 범위
clientip : "192.168.1.*"

# AND/OR 조합
response : 500 or response : 401
```

[figure21]

### Lens 로 시각화 생성

1. 좌측 메뉴 `Analytics > Visualize Library` 진입
2. `Create visualization > Lens` 클릭
3. Data view 로 `Nginx Logs` 선택

### 시각화 예시 1: 응답 상태 코드 분포 (Pie chart)

- Visualization type: `Pie`
- Slice by: `response` (Top values)
- Size by: `Count of records`

### 시각화 예시 2: 시간대별 요청 추이 (Bar chart)

- Visualization type: `Bar vertical stacked`
- Horizontal axis: `@timestamp`
- Vertical axis: `Count of records`
- Break down by: `response`

### 시각화 예시 3: 엔드포인트별 평균 응답 크기 (Table)

- Visualization type: `Table`
- Rows: `request` (Top values)
- Metrics: `Average of bytes`

각 시각화는 `Save` 클릭하여 라이브러리에 저장

[figure22]

### 대시보드 구성

1. 좌측 메뉴 `Analytics > Dashboard` 진입
2. `Create dashboard` 클릭
3. `Add from library` 로 저장한 시각화들을 추가
4. 드래그로 배치 조정 후 `Save` 클릭

### 참고

- KQL(Kibana Query Language)은 Lucene 쿼리보다 간결하나, 복잡한 쿼리는 Lucene 모드로 전환 가능
- Data view 는 인덱스 패턴(여러 인덱스를 하나의 논리적 단위로 묶음) 위에 동작함
- 대시보드 전체는 `Share > Export` 로 NDJSON 형식으로 export 하여 다른 환경에 import 가능

---

## 18. OpenTelemetry Collector 설치

`OpenTelemetry(OTel)` 는 메트릭, 로그, 트레이스를 벤더 중립적으로 수집/전송하는 단일 프레임워크

`OTel Collector` 는 다양한 소스에서 텔레메트리 데이터를 받아 처리한 후 여러 백엔드로 분배하는 중앙 에이전트

### 설치

```
cd ~/observability
wget https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.110.0/otelcol-contrib_0.110.0_linux_amd64.deb

sudo dpkg -i otelcol-contrib_0.110.0_linux_amd64.deb
```

설치와 함께 `otelcol-contrib` 서비스가 자동 등록됨

### Collector 설정 작성

`Receivers → Processors → Exporters` 의 파이프라인 구조 (이론 슬라이드 41 참고)

```
sudo tee /etc/otelcol-contrib/config.yaml > /dev/null <<EOF
receivers:
  # 애플리케이션의 OTLP 데이터 수신 (gRPC + HTTP)
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  # 배치 처리로 전송 효율 향상
  batch:
    timeout: 5s
    send_batch_size: 512

  # 리소스 속성 추가
  resource:
    attributes:
      - key: deployment.environment
        value: lab
        action: insert

exporters:
  # 트레이스: Jaeger 로 전송 (OTLP gRPC)
  otlp/jaeger:
    endpoint: localhost:4327     # Jaeger OTLP 수신 포트
    tls:
      insecure: true

  # 메트릭: Prometheus 가 가져갈 수 있도록 노출
  prometheus:
    endpoint: 0.0.0.0:8889

  # 로그: Elasticsearch 로 전송
  elasticsearch:
    endpoints: ["http://localhost:9200"]
    logs_index: otel-logs

  # 디버깅용 표준 출력
  debug:
    verbosity: basic

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [otlp/jaeger, debug]

    metrics:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [prometheus, debug]

    logs:
      receivers: [otlp]
      processors: [batch, resource]
      exporters: [elasticsearch, debug]
EOF
```

### 서비스 시작

```
sudo systemctl restart otelcol-contrib
sudo systemctl enable otelcol-contrib

# 상태 확인
sudo systemctl status otelcol-contrib
```

### 동작 확인

```
# Collector 내부 메트릭 (자체 텔레메트리)
curl http://localhost:8888/metrics | head -20

# Prometheus exporter 엔드포인트 (Collector 가 받은 OTel 메트릭이 여기로 노출됨)
curl http://localhost:8889/metrics
```

[figure23]

### Prometheus 가 OTel Collector 의 메트릭을 수집하도록 설정

```
# prometheus.yml 의 scrape_configs 에 OTel Collector 추가
sudo tee -a /etc/prometheus/prometheus.yml > /dev/null <<EOF

  - job_name: 'otel-collector'
    static_configs:
      - targets: ['localhost:8889']
EOF

curl -X POST http://localhost:9090/-/reload
```

### 참고

- `otelcol-contrib` 는 다양한 vendor-specific exporter 가 포함된 배포판이며, 표준 `otelcol` 보다 무거우나 호환성이 넓음
- Collector 는 Agent (각 호스트) 또는 Gateway (중앙 집계) 형태로 배포 가능
- 동일한 receiver 의 데이터를 여러 exporter 로 분기 가능 (예: traces 를 Jaeger 와 Tempo 에 동시 전송)

---

## 19. Jaeger 설치

`Jaeger` 는 분산 추적 데이터를 저장, 검색, 시각화하는 도구

### All-in-one 바이너리 다운로드

실습 편의를 위해 모든 컴포넌트가 하나의 바이너리에 포함된 `all-in-one` 배포판 사용

```
cd ~/observability
wget https://github.com/jaegertracing/jaeger/releases/download/v1.62.0/jaeger-1.62.0-linux-amd64.tar.gz

tar xvf jaeger-1.62.0-linux-amd64.tar.gz
sudo cp jaeger-1.62.0-linux-amd64/jaeger-all-in-one /usr/local/bin/
```

### systemd 서비스 등록

```
sudo tee /etc/systemd/system/jaeger.service > /dev/null <<EOF
[Unit]
Description=Jaeger All-in-One
After=network.target

[Service]
Type=simple
Environment="COLLECTOR_OTLP_ENABLED=true"
Environment="COLLECTOR_OTLP_GRPC_HOST_PORT=:4327"
Environment="COLLECTOR_OTLP_HTTP_HOST_PORT=:4328"
ExecStart=/usr/local/bin/jaeger-all-in-one
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl start jaeger
sudo systemctl enable jaeger

sudo systemctl status jaeger
```

### 포트 구성

| 포트 | 용도 |
| --- | --- |
| 16686 | Jaeger Web UI |
| 4327 | OTLP gRPC 수신 (Collector 로부터) |
| 4328 | OTLP HTTP 수신 |

OTel Collector 기본 OTLP 포트(4317/4318)와 충돌을 피하기 위해 Jaeger 는 4327/4328 로 변경 사용

### 웹 UI 접속

브라우저에서 `http://<호스트 IP>:16686` 접속

`Service` 드롭다운에 아직 데이터가 없는 상태 확인

[figure24]

---

## 20. 분산 추적 데모 애플리케이션

OTel SDK 를 사용한 두 개의 마이크로서비스를 만들어 Jaeger 에 분산 추적이 기록되는 과정 확인

### Python 환경 준비

```
cd ~/observability
python3 -m venv otel-venv
source otel-venv/bin/activate

# OTel 자동 계측 패키지
pip install flask requests \
  opentelemetry-distro \
  opentelemetry-exporter-otlp \
  opentelemetry-instrumentation-flask \
  opentelemetry-instrumentation-requests

# 자동 계측에 필요한 추가 라이브러리 자동 설치
opentelemetry-bootstrap -a install
```

### Backend 서비스 작성

```
cat > ~/observability/backend.py <<'EOF'
from flask import Flask, jsonify
import time
import random

app = Flask(__name__)

@app.route('/data')
def get_data():
    # 의도적인 지연 발생 (병목 시뮬레이션)
    time.sleep(random.uniform(0.1, 0.5))
    return jsonify({"value": random.randint(1, 100)})

@app.route('/slow')
def slow_endpoint():
    # 큰 지연을 발생시켜 trace 에서 식별되도록 함
    time.sleep(2)
    return jsonify({"status": "slow response"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5001)
EOF
```

### Frontend 서비스 작성

```
cat > ~/observability/frontend.py <<'EOF'
from flask import Flask, jsonify
import requests

app = Flask(__name__)

@app.route('/')
def index():
    # Backend 호출 - 이 호출이 trace 에서 자식 span 으로 기록됨
    resp = requests.get('http://localhost:5001/data')
    return jsonify({"frontend": "ok", "backend": resp.json()})

@app.route('/checkout')
def checkout():
    # 정상 호출 + 느린 호출을 연쇄적으로 수행
    requests.get('http://localhost:5001/data')
    resp = requests.get('http://localhost:5001/slow')
    return jsonify({"checkout": "complete", "detail": resp.json()})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
```

### OTel 자동 계측으로 서비스 실행

별도 터미널 2개 사용

**터미널 1 (Backend):**

```
cd ~/observability
source otel-venv/bin/activate

OTEL_SERVICE_NAME=backend-service \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
OTEL_EXPORTER_OTLP_PROTOCOL=grpc \
OTEL_TRACES_EXPORTER=otlp \
OTEL_METRICS_EXPORTER=otlp \
OTEL_LOGS_EXPORTER=otlp \
opentelemetry-instrument python3 backend.py
```

**터미널 2 (Frontend):**

```
cd ~/observability
source otel-venv/bin/activate

OTEL_SERVICE_NAME=frontend-service \
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317 \
OTEL_EXPORTER_OTLP_PROTOCOL=grpc \
OTEL_TRACES_EXPORTER=otlp \
OTEL_METRICS_EXPORTER=otlp \
OTEL_LOGS_EXPORTER=otlp \
opentelemetry-instrument python3 frontend.py
```

### 트래픽 발생

별도 터미널 3에서 요청을 여러 번 보내 trace 생성

```
# 정상 요청 5회
for i in {1..5}; do curl http://localhost:5000/; echo; done

# 느린 요청(checkout) 3회
for i in {1..3}; do curl http://localhost:5000/checkout; echo; done
```

### Jaeger UI 에서 trace 확인

브라우저에서 `http://<호스트 IP>:16686` 접속

1. `Service` 드롭다운에서 `frontend-service` 선택
2. `Find Traces` 클릭
3. trace 목록이 표시됨 - 각 trace 의 총 소요시간 차이 관찰

[figure25]

### Trace 상세 분석

trace 하나 클릭하여 상세 화면 진입

다음 정보들을 확인:

- **Span 트리 구조**: `frontend-service` 의 HTTP 요청 → `backend-service` 호출이 자식 span 으로 표시
- **Gantt 차트**: 각 span 의 시작/종료 시점이 시간축에 가시화됨
- **Context Propagation**: 동일한 `traceID` 가 서비스 간에 전파된 것 확인
- **병목 식별**: `/checkout` trace 에서 `/slow` 호출이 2초 이상 차지하는 것을 시각적으로 확인 가능

[figure26]

### Span 상세 정보 확인

특정 span 클릭 시 다음 메타데이터 확인 가능:

- `http.method`, `http.url`, `http.status_code` 등 표준 속성
- `net.peer.name`, `net.peer.port` 등 네트워크 정보
- 서비스별 리소스 속성 (`service.name`, `deployment.environment`)

### 참고

- `opentelemetry-instrument` 명령어는 Python 애플리케이션을 자동으로 계측하여 코드 수정 없이 trace 를 생성함
- 환경 변수 `OTEL_*` 로 모든 동작을 제어할 수 있음 - 코드에 vendor lock-in 없음
- `requests` 라이브러리 자동 계측이 HTTP 헤더 `traceparent` 를 통해 trace ID 를 전파함 (W3C Trace Context 표준)
- 더 세밀한 제어가 필요하면 코드에서 `tracer.start_as_current_span()` 으로 수동 계측 가능

---

## 21. 통합 시나리오: API 응답 지연 진단

이론 슬라이드 12, 44~46 에서 다룬 **3단계 진단 프로세스**를 직접 수행

### 시나리오

`/checkout` API 의 P99 응답 시간이 평소보다 급증하였다는 사용자 리포트가 접수됨

### Step 1. Detection (Metrics) - "무엇이 발생했는가?"

Grafana 에서 Custom Exporter 메트릭 기반 응답시간 패널 추가

```
# /checkout 엔드포인트의 P99 응답시간
histogram_quantile(0.99, sum by (le) (rate(myapp_request_duration_seconds_bucket[1m])))
```

OTel 데모 트래픽을 발생시키면서 그래프 변화 관찰

### Step 2. Troubleshooting (Traces) - "어디서 발생했는가?"

Jaeger UI 에서 `frontend-service` 의 최근 trace 조회

- `Min Duration: 1s` 필터로 느린 요청만 추출
- 느린 trace 들이 공통적으로 `/slow` 엔드포인트 호출 구간에서 지연된다는 사실 확인
- 병목 서비스: `backend-service`, 병목 구간: `/slow` 핸들러

### Step 3. Pinpointing (Logs) - "왜 발생했는가?"

Kibana Discover 에서 동일 시간대 로그 검색

- 시간 범위를 trace 발생 시각으로 좁힘
- 검색어: `service.name : "backend-service" and message : *slow*`
- 로그에서 의도적 `time.sleep(2)` 호출이 원인임을 확인

### 해결 및 검증

`backend.py` 에서 `/slow` 의 sleep 시간을 제거하고 재시작 후:

- 새로운 trace 들의 duration 이 감소
- Grafana P99 그래프가 정상 범위로 복귀

### 참고

- 이 흐름이 슬라이드 12의 `Metrics → Traces → Logs` 드릴다운 패턴의 실체임
- 운영 환경에서는 trace ID 가 로그에도 함께 기록되어, trace 와 log 가 ID 로 직접 연결됨 (correlation)
- `MTTR(Mean Time To Resolve)` 단축의 핵심은 이 3단계를 신속하게 수행하는 능력임

---

## 22. 심화 과제

### 과제 1. Service Discovery 적용

현재 `static_configs` 로 하드코딩된 Prometheus 수집 대상을, 파일 기반 서비스 디스커버리(`file_sd_configs`)로 변경하여 타겟 추가 시 Prometheus 재로드 없이 자동 인식되도록 구성

### 과제 2. Recording Rules

자주 사용되는 복잡한 PromQL 쿼리(예: 에러율, P99)를 사전 계산하여 새 메트릭으로 저장하는 Recording Rule 을 작성하고, 대시보드에서 활용

### 과제 3. Filebeat → Elasticsearch 직접 전송

Logstash 를 거치지 않고 Filebeat 가 ES 로 직접 데이터를 전송하도록 구성하고, Logstash 경유와 비교한 장단점 분석

### 과제 4. OTel Collector 다중 분기

동일한 trace 데이터를 Jaeger 와 Elasticsearch (로그 형태로 변환) 양쪽에 동시 전송하도록 Collector 파이프라인 수정

### 과제 5. Alert 라우팅 분기

`severity: critical` 알림은 webhook 으로, `severity: warning` 알림은 로그 파일로 분리되도록 Alertmanager `route` 트리를 구성

---

## Q & A

박찬욱
<cupark@dankook.ac.kr>

남재현
<namjh@dankook.ac.kr>

## Networked Systems and Security Lab (BoanLab) @ DKU