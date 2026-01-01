# GPU 클러스터 모니터링 Ansible Playbook

GPU 노드 모니터링을 위한 Prometheus 스택 설치 자동화 Ansible Playbook입니다.

## 구성 요소

| 컴포넌트 | 설치 위치 | 역할 |
|----------|-----------|------|
| Prometheus | Control Node | 메트릭 수집 및 저장 |
| Grafana | Control Node | 시각화 대시보드 |
| Alertmanager | Control Node | 알람 관리 및 발송 |
| Node Exporter | 모든 노드 | 시스템 메트릭 수집 |
| DCGM Exporter | GPU 노드 | GPU 메트릭 수집 |

## 아키텍처

```
┌─────────────────────────────────────────────────────────────┐
│                     Control Node                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ Prometheus  │  │   Grafana   │  │   Alertmanager      │  │
│  │   :9090     │  │    :3000    │  │      :9093          │  │
│  └──────┬──────┘  └─────────────┘  └─────────────────────┘  │
│         │                                                    │
│         │ scrape (pull)                                      │
└─────────┼────────────────────────────────────────────────────┘
          │
          ├──────────────────┬──────────────────┐
          ▼                  ▼                  ▼
   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
   │  GPU Node 1 │    │  GPU Node 2 │    │  GPU Node N │
   │ ┌─────────┐ │    │ ┌─────────┐ │    │ ┌─────────┐ │
   │ │ node_exp│ │    │ │ node_exp│ │    │ │ node_exp│ │
   │ │  :9100  │ │    │ │  :9100  │ │    │ │  :9100  │ │
   │ ├─────────┤ │    │ ├─────────┤ │    │ ├─────────┤ │
   │ │ dcgm_exp│ │    │ │ dcgm_exp│ │    │ │ dcgm_exp│ │
   │ │  :9400  │ │    │ │  :9400  │ │    │ │  :9400  │ │
   │ └─────────┘ │    │ └─────────┘ │    │ └─────────┘ │
   └─────────────┘    └─────────────┘    └─────────────┘
```

---

## Ansible 기초 개념

### Ansible이란?

Ansible은 서버 구성 관리 및 자동화 도구입니다. SSH를 통해 원격 서버에 접속하여 명령을 실행합니다.

### 핵심 용어

| 용어 | 설명 | 예시 |
|------|------|------|
| **Inventory** | 관리할 서버 목록 | `inventory/hosts.yaml` |
| **Playbook** | 실행할 작업들의 정의 | `playbooks/site.yaml` |
| **Role** | 재사용 가능한 작업 모음 | `roles/prometheus/` |
| **Task** | 개별 작업 단위 | 패키지 설치, 파일 복사 등 |
| **Handler** | 특정 조건에서 실행되는 작업 | 서비스 재시작 |
| **Template** | 변수가 포함된 설정 파일 | `prometheus.yaml.j2` |
| **Variable** | 설정값을 저장하는 변수 | `prometheus_version: "2.47.0"` |

### 디렉토리 구조 설명

```
ansible-prometheus-stack/
├── inventory/
│   └── hosts.yaml           # 서버 목록 (어디에 설치할지)
│
├── group_vars/
│   └── all.yaml             # 모든 서버에 적용할 변수
│
├── roles/                  # 역할별 작업 모음
│   ├── prometheus/
│   │   ├── tasks/          # 실행할 작업들
│   │   │   └── main.yaml
│   │   ├── templates/      # 설정 파일 템플릿
│   │   │   └── prometheus.yaml.j2
│   │   ├── handlers/       # 서비스 재시작 등
│   │   │   └── main.yaml
│   │   └── defaults/       # 기본 변수값
│   │       └── main.yaml
│   └── ...
│
├── playbooks/
│   └── site.yaml            # 메인 실행 파일
│
└── files/                  # 정적 파일 (대시보드 JSON 등)
```

### Jinja2 템플릿 문법

템플릿 파일(`.j2`)에서 사용되는 문법입니다:

```yaml
# 변수 출력
server_name: {{ ansible_hostname }}

# 조건문
{% if alertmanager_slack_webhook is defined %}
slack_configs:
  - channel: '#alerts'
{% endif %}

# 반복문
{% for host in groups['gpu_nodes'] %}
  - targets: ['{{ host }}:9100']
{% endfor %}
```

---

## 사전 요구사항

### Control Node (Ansible 실행 서버)

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install -y ansible python3-pip

# RHEL/CentOS
sudo dnf install -y ansible python3-pip

# 버전 확인
ansible --version
```

### 모든 대상 서버

1. **SSH 접속 가능**: Control Node에서 SSH 키 기반 로그인
2. **Python 3 설치됨**: Ansible 모듈 실행에 필요
3. **sudo 권한**: 패키지 설치 및 서비스 관리에 필요

### SSH 키 설정 (아직 안 되어 있다면)

```bash
# Control Node에서 SSH 키 생성
ssh-keygen -t ed25519 -C "ansible"

# 각 대상 서버에 키 복사
ssh-copy-id root@gpu-node-01
ssh-copy-id root@gpu-node-02
# ...
```

---

## 설치 방법

### 1단계: 인벤토리 수정

`inventory/hosts.yaml` 파일을 실제 환경에 맞게 수정합니다:

```yaml
all:
  children:
    control:
      hosts:
        control-node:
          ansible_host: 192.168.1.100  # Control Node IP

    gpu_nodes:
      hosts:
        gpu-node-01:
          ansible_host: 192.168.1.101  # 실제 IP로 변경
        gpu-node-02:
          ansible_host: 192.168.1.102
        # 추가 노드...
```

### 2단계: 변수 설정

`group_vars/all.yaml` 파일에서 환경에 맞게 변수를 수정합니다:

```yaml
# 알람 수신 설정 (필수)
alertmanager_slack_webhook: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
alertmanager_slack_channel: "#gpu-alerts"

# 또는 이메일 설정
alertmanager_smtp_host: "smtp.company.com"
alertmanager_email_to: "infra-team@company.com"

# Grafana 비밀번호
grafana_admin_password: "your-secure-password"
```

### 3단계: 연결 테스트

```bash
cd examples/ansible-prometheus-stack

# 모든 서버에 ping 테스트
ansible -i inventory/hosts.yaml all -m ping

# 예상 출력:
# gpu-node-01 | SUCCESS => {"ping": "pong"}
# gpu-node-02 | SUCCESS => {"ping": "pong"}
```

### 4단계: Playbook 실행

```bash
# 전체 설치 (권장)
ansible-playbook -i inventory/hosts.yaml playbooks/site.yaml

# 특정 역할만 실행 (태그 사용)
ansible-playbook -i inventory/hosts.yaml playbooks/site.yaml --tags "prometheus"
ansible-playbook -i inventory/hosts.yaml playbooks/site.yaml --tags "gpu_exporters"

# Dry-run (실제 실행 없이 확인)
ansible-playbook -i inventory/hosts.yaml playbooks/site.yaml --check

# 상세 출력
ansible-playbook -i inventory/hosts.yaml playbooks/site.yaml -v
```

### 5단계: 확인

설치 완료 후 브라우저에서 접속:

| 서비스 | URL | 기본 계정 |
|--------|-----|-----------|
| Grafana | http://control-node:3000 | admin / (설정한 비밀번호) |
| Prometheus | http://control-node:9090 | - |
| Alertmanager | http://control-node:9093 | - |

---

## 주요 파일 설명

### inventory/hosts.yaml

관리할 서버 목록입니다. 그룹으로 분류하여 역할별 설치를 지정합니다.

```yaml
all:
  children:
    control:        # Prometheus, Grafana, Alertmanager 설치
      hosts:
        control-node:
          ansible_host: 192.168.1.100

    gpu_nodes:      # Node Exporter, DCGM Exporter 설치
      hosts:
        gpu-node-01:
          ansible_host: 192.168.1.101
```

### group_vars/all.yaml

모든 서버에 적용되는 변수입니다. 버전, 포트, 알람 설정 등을 정의합니다.

### roles/*/tasks/main.yaml

각 컴포넌트의 설치 작업을 정의합니다. 예시:

```yaml
# roles/prometheus/tasks/main.yaml
- name: Download Prometheus
  get_url:
    url: "https://github.com/prometheus/prometheus/releases/..."
    dest: /tmp/prometheus.tar.gz

- name: Extract Prometheus
  unarchive:
    src: /tmp/prometheus.tar.gz
    dest: /opt/
    remote_src: yes

- name: Create systemd service
  template:
    src: prometheus.service.j2
    dest: /etc/systemd/system/prometheus.service
  notify: Restart Prometheus
```

---

## 운영 가이드

### 노드 추가

```bash
# 1. inventory/hosts.yaml에 노드 추가
# 2. 해당 노드에만 exporter 설치
ansible-playbook -i inventory/hosts.yaml playbooks/site.yaml \
  --tags "exporters" \
  --limit "gpu-node-new"

# 3. Prometheus 설정 갱신 (targets 추가)
ansible-playbook -i inventory/hosts.yaml playbooks/site.yaml \
  --tags "prometheus_config"
```

### 노드 제거

```bash
# 1. inventory/hosts.yaml에서 노드 제거
# 2. Prometheus 설정 갱신
ansible-playbook -i inventory/hosts.yaml playbooks/site.yaml \
  --tags "prometheus_config"

# 3. (선택) 해당 노드에서 exporter 제거
ansible -i inventory/hosts.yaml gpu-node-old -m shell \
  -a "systemctl stop node_exporter dcgm-exporter && rm -rf /opt/exporters"
```

### 알람 룰 수정

```bash
# 1. group_vars/all.yaml 또는 files/alert-rules.yaml 수정
# 2. 알람 룰만 업데이트
ansible-playbook -i inventory/hosts.yaml playbooks/site.yaml \
  --tags "alert_rules"
```

### 서비스 재시작

```bash
# Prometheus 재시작
ansible -i inventory/hosts.yaml control -m systemd \
  -a "name=prometheus state=restarted"

# 모든 GPU 노드의 DCGM exporter 재시작
ansible -i inventory/hosts.yaml gpu_nodes -m systemd \
  -a "name=dcgm-exporter state=restarted"
```

---

## 트러블슈팅

### SSH 연결 실패

```bash
# 수동으로 SSH 연결 테스트
ssh -v root@gpu-node-01

# SSH 키 권한 확인
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

### Python 없음 오류

```bash
# 대상 서버에 Python 설치 (raw 모듈 사용)
ansible -i inventory/hosts.yaml gpu_nodes -m raw \
  -a "apt install -y python3 || yum install -y python3"
```

### DCGM Exporter 시작 실패

```bash
# NVIDIA 드라이버 확인
nvidia-smi

# DCGM 설치 확인
dcgmi discovery -l

# 로그 확인
journalctl -u dcgm-exporter -f
```

### Prometheus에서 Target이 down 표시

```bash
# 방화벽 확인
sudo firewall-cmd --list-all

# 포트 열기
sudo firewall-cmd --permanent --add-port=9100/tcp
sudo firewall-cmd --permanent --add-port=9400/tcp
sudo firewall-cmd --reload

# 또는 iptables
sudo iptables -A INPUT -p tcp --dport 9100 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 9400 -j ACCEPT
```

---

## 자주 사용하는 Ansible 명령어

```bash
# 인벤토리의 모든 호스트 나열
ansible-inventory -i inventory/hosts.yaml --list

# 특정 그룹의 호스트만 나열
ansible-inventory -i inventory/hosts.yaml --graph gpu_nodes

# Ad-hoc 명령 실행 (모든 GPU 노드에서)
ansible -i inventory/hosts.yaml gpu_nodes -m shell -a "nvidia-smi"

# 파일 복사
ansible -i inventory/hosts.yaml gpu_nodes -m copy \
  -a "src=./files/config.yaml dest=/etc/app/config.yaml"

# 서비스 상태 확인
ansible -i inventory/hosts.yaml gpu_nodes -m systemd \
  -a "name=dcgm-exporter" | grep ActiveState
```

---

## 참고 자료

- [Ansible 공식 문서](https://docs.ansible.com/)
- [Prometheus 공식 문서](https://prometheus.io/docs/)
- [DCGM Exporter GitHub](https://github.com/NVIDIA/dcgm-exporter)
- [Grafana GPU 대시보드](https://grafana.com/grafana/dashboards/12239)
