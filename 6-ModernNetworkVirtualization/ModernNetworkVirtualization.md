# 클라우드 가상화 기술

## 네트워크 가상화 기술 실습 (현대)

## 1. Mininet

`Mininet`은 가상화된 환경에서 가상의 스위치, 호스트, 링크를 생성하여 실제 네트워크 동작을 시뮬레이션하는 도구

- Linux 기반 Network Namespace 활용
- 빠르고 가벼운 네트워크 토폴로지 구성 가능

---

## 2. Mininet 설치

```bash
# 패키지 업데이트
sudo apt-get update

# 필수 패키지 설치
sudo apt-get install -y git python3-pip openvswitch-switch

# Mininet 소스코드 클론
git clone https://github.com/mininet/mininet

# 설치 스크립트 실행
cd mininet

# Mininet 설치
sudo ./util/install.sh -a

# 설치 확인
mn --version
```

![figure1](./images/figure1.png)

---

## 3. POX 컨트롤러

`POX`는 Python 기반의 `OpenFlow 컨트롤러`로 학습 및 프로토타이핑에 적합

- `OpenFlow 기반 SDN 컨트롤러`
- 가볍고 사용이 간단

---

## 4. POX 설치

```bash
# 홈 디렉토리에서 POX 클론
cd ~
git clone https://github.com/noxrepo/pox
# Mininet 설치 시 이미 포함되어 있어 별도로 하지 않아도 됨

# 실행 테스트
cd ~/pox
./pox.py -h
```

![figure2](./images/figure2.png)

---

## 5. Mininet 기본 실행

```bash
# 최소 토폴로지 실행
sudo mn --topo minimal
```

![figure3](./images/figure3.png)  

---

## 6. Tree 토폴로지 생성

```bash
# Tree 구조 토폴로지 생성
# depth: 트리의 깊이
# fanout: 각 스위치의 하위 호스트 수
sudo mn --topo tree,depth=2,fanout=2
```

![figure4](./images/figure4.png)

---

## 7. Mininet CLI 명령어

```bash
# Mininet CLI에서 실행
mininet> h1 ping -c 1 h2
mininet> h2 ping -c 3 h4
mininet> h3 ip a
mininet> h4 ip route
```

![figure5](./images/figure5.png)
![figure6](./images/figure6.png)

---

## 8. 호스트에서 Mininet 내부 네트워크 정보 확인

```bash
# 호스트에서 실행
ps -ef | grep mininet

# mininet의 PID를 통해 h3 네트워크 네임스페이스 네트워크 정보 확인
sudo nsenter -t <h3 PID> -n ip a
```

![figure7](./images/figure7.png)

### s1, s2, s3 간 연결 관계 확인

```bash
# OVS 스위치 정보 확인
sudo ovs-vsctl show

# 인터페이스 정보 확인
ip a | grep s
```
`ovs-vsctl show와 ip a 결과를 비교하면 스위치 간 인터페이스 연결 관계를 확인할 수 있음`

---

## 9. POX 실행

```bash
cd ~/pox

# L2 Learning 스위치 실행
./pox.py log.level --DEBUG forwarding.l2_learning
```

![figure8](./images/figure8.png)

---

## 10. Mininet에서 컨트롤러 연결

```bash
sudo mn --topo single,3 \
--controller=remote,ip=127.0.0.1,port=6633
```

![figure9](./images/figure9.png)

---

## 11. L2 Learning 모듈 동작

경로: `~/pox/pox/forwarding/l2_learning.py`

### 주요 동작

- PacketIn 이벤트 발생 시 처리
- MAC 주소 학습
- 목적지 MAC 확인 후 forwarding 수행

```python
# MAC 학습
self.macToPort[packet.src] = event.port
```

### 동작 흐름

- 목적지 MAC 존재 → 유니캐스트 전달 (FlowMod)
- 목적지 MAC 없음 → Flood 처리 (PacketOut)
- Flow 설치 후 스위치에서 직접 처리 가능

### h1에서 h3로 ping 테스트

```bash
h1 ping -c 3 h3

# Flow Table 확인
sudo ovs-ofctl dump-flows s1
```

![figure10](./images/figure10.png)  
![figure11](./images/figure11.png)
![figure12](./images/figure12.png)
`어떻게 학습되는지 확인해보기`

---

## 12. OVS 스위치 정보 확인

```bash
# OVS 스위치 정보 확인

# s1 스위치의 포트와 연결 상태 확인
sudo ovs-ofctl show s1

# Flow Table 확인
# 트래픽이 없으면 빈 테이블이지만, ping 테스트 후에는 학습된 Flow가 보임
sudo ovs-ofctl dump-flows s1

# Port 정보 확인
sudo ovs-ofctl dump-ports s1
```

---

## 13. 방화벽 구현

`l2_learning` 코드를 기반으로 특정 트래픽 차단 구현
- 특정 트래픽(출발지 MAC/IP, 목적지 MAC/IP, 포트 등)에 대한 차단 규칙을 적용
- 편의상, 코드에 차단 규칙을 하드 코딩해도 됨
- ~/pox/pox/forwarding/l2_learning.py 파일에서 PacketIn 이벤트 처리 부분 수정


### 방법 1

- 특정 트래픽이 들어오면 `PacketOut`, `FlowMod` 메시지 없이 패킷 드랍 
- [예시코드](./codes/l2_learning_1.py) s1 - h1, h2, h3, h4 구조일 때

![figure13](./images/figure13.png)
![figure14](./images/figure14.png)

### 방법 2

- 특정 트래픽이 들어오면 패킷 드랍과 동시에 차단을 위한 `FlowMod` 메시지 생성하여 설치하기
- [예시코드](./codes/l2_learning_2.py) s1 - h1, h2, h3, h4 구조일 때

![figure15](./images/figure15.png)
![figure16](./images/figure16.png)
![figure17](./images/figure17.png)
![figure18](./images/figure18.png)

---

## 14. ONOS 컨트롤러

`ONOS`는 오픈소스 SDN 컨트롤러로 대규모 네트워크 환경에 적합

- OpenFlow 기반 SDN 컨트롤러
- Web UI 및 REST API 제공

---

## 15. Docker 설치 및 ONOS 실행

```bash
# 필수 패키지 설치
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# Docker GPG 키 추가
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Docker 저장소 추가
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker 설치
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

sudo systemctl start docker
sudo systemctl enable docker

# sudo 없이 docker 사용 (선택사항)
sudo usermod -aG docker $USER
newgrp docker

# 도커 설치 확인
docker --version
```

![figure19](./images/figure19.png)

```bash
# ONOS 실행
docker run -d --net=host --name onos onosproject/onos:2.1.0

# 로그 확인 (ONOS 완전히 뜰 때까지 대기)
docker logs -f onos
```

![figure20](./images/figure20.png)
![figure21](./images/figure21.png)

### 참고

`--net=host` 옵션으로 컨테이너를 호스트 네트워크 스택에 직접 연결하므로 별도 포트 매핑 없이 ONOS가 사용하는 포트가 그대로 열림

```bash
docker inspect --format='{{range $p, $conf := .Config.ExposedPorts}}{{$p}}{{end}}' onos
```

- `8181` - Web UI
- `6653` - OpenFlow
- `8101` - ONOS CLI

![figure22](./images/figure22.png)

---

## 16. Mininet에서 ONOS 연결

브라우저에서 접속:
```
http://<IP주소>:8181/onos/ui/login.html
```
계정: `onos / rocks`

**앱 활성화 (Applications 탭에서)**
- `OpenFlow Provider Suite` → Activate
- `Reactive Forwarding` → Activate

![figure23](./images/figure23.png)
![figure24](./images/figure24.png)


```bash
# Mininet 실행 (ONOS 컨트롤러와 연결)
# Tree 토폴로지: 깊이 2, 스위치당 하위 노드 3개 (스위치 4개, 호스트 9개)
# hostname -I : 호스트의 IP 주소 확인 | awk '{print $1}' : 첫 번째 IP 주소만 추출(ens3 인터페이스의 IP)
sudo mn --topo tree,depth=2,fanout=3 \
  --controller=remote,ip=$(hostname -I | awk '{print $1}'),port=6653 \
  --switch ovs,protocols=OpenFlow13
```

![figure25](./images/figure25.png)

---

## 17. ONOS 동작 확인

### 토폴로지 확인

- ONOS UI 좌측 메뉴 → **Topology** 에서 스위치/호스트 연결 확인

![figure26](./images/figure26.png)

```bash
# pingall 명령어를 통해 호스트 간 연결 확인
# 호스트 간 연결이 성공하면 ONOS UI에서 Flow가 설치되는 것을 확인할 수 있음(웹UI에서 H 입력 시 호스트 출력)
mininet> pingall
```

![figure27](./images/figure27.png)

### Flow Table 확인

```bash
# OVS에서 Flow 확인
sudo ovs-ofctl -O OpenFlow13 dump-flows s1
```

![figure28](./images/figure28.png)

```bash
# ONOS CLI 접속
docker exec -it onos /root/onos/apache-karaf-4.2.3/bin/client -h localhost

# karaf@root에서
# ONOS에 연결된 스위치 목록 확인
flows

# 학습된 호스트(MAC/IP) 목록 확인
hosts

# 스위치에 설치된 Flow 규칙 목록 확인
devices
```

![figure29](./images/figure29.png)
![figure30](./images/figure30.png)
![figure31](./images/figure31.png)
![figure32](./images/figure32.png)

```bash
# REST API로 Flow 조회
curl -u onos:rocks \
  http://localhost:8181/onos/v1/flows | python3 -m json.tool
```

![figure33](./images/figure33.png)

---

## 18. ONOS 방화벽 구현

REST API로 특정 트래픽 차단 Flow 직접 설치

```bash
# h1(10.0.0.1) → h3(10.0.0.3) 차단 Flow 설치
# h1, h3가 연결된 스위치에 Flow 규칙 설치 (hosts 명령어로 확인, 예시에서는 s2) | deviceId 및 주소의 마지막 of 아래에 스위치 ID 입력
curl -v -u onos:rocks -X POST \
  -H 'Content-Type: application/json' \
  -d '{
    "priority": 50000,
    "timeout": 0,
    "isPermanent": true,
    "deviceId": "of:0000000000000002",
    "treatment": {"instructions": []},
    "selector": {
      "criteria": [
        {"type": "ETH_TYPE", "ethType": "0x800"},
        {"type": "IPV4_SRC", "ip": "10.0.0.1/32"},
        {"type": "IPV4_DST", "ip": "10.0.0.3/32"}
      ]
    }
  }' \
  http://localhost:8181/onos/v1/flows/of:0000000000000002

# 차단 확인
mininet> h1 ping -c 3 h3
```

![figure34](./images/figure34.png)
![figure35](./images/figure35.png)

```bash
# 설치된 Flow 확인
# of 아래에 스위치 ID 입력(없어도 전체 Flow 조회는 가능)
curl -u onos:rocks \
  http://localhost:8181/onos/v1/flows/of:0000000000000002 | python3 -m json.tool
# http://localhost:8181/onos/v1/flows/
``` 


### 참고

Mininet 호스트는 Linux Network Namespace로 격리되어 있어 트래픽이 외부로 나가지 않음. ONOS가 설치하는 Flow는 Mininet 내부 가상 OVS 스위치 안에서만 처리됨

---

## 19. 기타 SDN 컨트롤러

### Ryu

- https://github.com/faucetsdn/ryu

### OpenDayLight

- https://docs.opendaylight.org/en/latest/getting-started-guide/installing_opendaylight.html

---

## 20. 심화 과제 - 로드 밸런서 구현

`라운드 로빈 방식으로 서버 선택하는 로드밸런서 구현해보기`


---

## Q & A

박찬욱  
cupark@dankook.ac.kr

남재현  
namjh@dankook.ac.kr  

## Networked Systems and Security Lab (BoanLab) @ DKU
<img src="../images/boanlab_logo.svg" width="25%"/>