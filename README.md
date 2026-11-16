# AImmunity-Snort

Snort 3 기반 AI Web Agent 접근 탐지 및 차단 규칙 연구

## Overview

**AImmunity-Snort**는 AI Web Agent 및 AI 기반 보안 자동화 도구가 웹사이트에 접근할 때 발생하는 네트워크·HTTP 트래픽의 특징을 분석하여 탐지 규칙을 설계하고 검증하는 프로젝트입니다.

User-Agent와 IP 주소 등의 **정적 식별 정보**를 비롯하여 요청 빈도, HTTP 오류 응답, URI 접근 및 자동화 도구의 사용 흔적 등 **행위·환경 기반 특징**을 활용합니다. 이를 통해 기존 접근 통제 기술의 우회 가능성을 분석하고, 복수의 특징을 결합한 탐지 방식의 효과와 한계를 평가하는 것을 목표로 합니다.

본 저장소는 Snort 3 설정 파일과 사용자 정의 규칙을 제공합니다. 규칙에 따라 HTTP 요청·응답과 특정 AI 침투 테스트 도구의 식별 문자열을 탐지할 수 있으나, **규칙 매칭만으로 AI 사용 여부, 악의적인 의도 또는 공격 성공을 확정할 수는 없습니다.**

## Repository Structure

```text
AImmunity-Snort/
├── rules/
│   ├── ai_ip.rules
│   ├── ai_user_agent.rules
│   ├── aipt-detections.rules
│   ├── codex_automation.rules
│   └── http_4xx_frequency.rules
├── LICENSE
├── README.md
└── snort.lua
```

| 파일 | 설명 |
|---|---|
| `snort.lua` | Snort 3 기본 설정, 네트워크 변수 및 탐지 규칙 로딩 |
| `rules/ai_ip.rules` | AI 서비스 제공자의 알려진 IP 주소 대역을 활용한 정적 식별 규칙 |
| `rules/ai_user_agent.rules` | AI Bot·Agent 및 관련 HTTP 클라이언트의 User-Agent 문자열 기반 탐지 규칙 |
| `rules/codex_automation.rules` | Codex 실험에서 관찰한 HTTP 클라이언트 식별정보와 반복 요청 특성을 활용한 탐지 규칙 |
| `rules/http_4xx_frequency.rules` | HTTP 4xx 계열 응답의 반복 발생을 이용하여 비정상적인 경로 탐색·자동화 요청의 가능성을 관찰하는 규칙 |
| `rules/aipt-detections.rules` | Theori의 AI 침투 테스트 도구 탐지용 Suricata **target-facing** 규칙 19개를 Snort 3용으로 변환한 규칙. 도구별 User-Agent, 일반 자동화·스캐너 특징, 404 응답 등을 포함 |
| `LICENSE` | 본 프로젝트의 라이선스. 외부 규칙의 별도 라이선스는 아래 License 항목 참고 |

> `aipt-detections.rules`는 [Theori aipt-detections](https://github.com/theori-io/aipt-detections)의 [`suricata/aipt-target-facing.rules`](https://github.com/theori-io/aipt-detections/blob/main/suricata/aipt-target-facing.rules)를 기반으로 변환한 파생 규칙입니다. 원본의 모든 Suricata 탐지 규칙이 아닌 **웹 서버 관측 위치에 해당하는 19개 규칙**만 포함합니다.

## Requirements

- Linux (Ubuntu 권장)
- Snort 3 (본 프로젝트 실험 환경: **3.12.2.0**)
- Git
- 관리자 권한 (`sudo`)
- 보호 대상 트래픽을 관찰할 수 있는 네트워크 인터페이스
- Snort 기본 설정에 필요한 `snort_defaults.lua`, `file_magic.rules` 등

이 문서의 경로는 `/usr/local/etc/snort/`을 기준으로 작성했습니다. 설치 환경에 맞게 변경할 수 있습니다.

## Installation

### 1. Clone Repository

```bash
git clone https://github.com/USERNAME/AImmunity-Snort.git
cd AImmunity-Snort
```

`USERNAME`을 실제 GitHub 계정 또는 조직 이름으로 변경합니다.

### 2. Verify Snort Installation

```bash
snort --version
ls /usr/local/etc/snort/snort_defaults.lua
ls /usr/local/etc/snort/file_magic.rules
```

기본 파일의 위치가 다른 경우 Snort 설치 경로를 확인합니다.

### 3. Backup Existing Configuration

```bash
sudo cp /usr/local/etc/snort/snort.lua \
  /usr/local/etc/snort/snort.lua.bak
```

### 4. Apply Configuration and Rules

```bash
sudo mkdir -p /usr/local/etc/snort/rules
sudo cp snort.lua /usr/local/etc/snort/
sudo cp rules/*.rules /usr/local/etc/snort/rules/
ls -l /usr/local/etc/snort/rules/
```

기존 `snort.lua`에 별도 설정이 있다면 덮어쓰기 전에 차이를 확인해야 합니다. 외부에서 가져온 규칙 파일은 검증하기 전까지 운영 환경에서 차단 동작으로 활성화하지 않는 것을 권장합니다.

### 5. Validate Configuration

```bash
sudo snort \
  -c /usr/local/etc/snort/snort.lua \
  -T
```

설정 검증에 실패하면 출력에 표시된 파일명·줄 번호를 기준으로 문법을 확인합니다. **`-T` 통과는 규칙 파싱 및 설정 검증을 의미하며 탐지 정확성 검증은 아닙니다.**

## Usage

### 1. Check Network Interface

```bash
ip -br link
```

### 2. Run Snort (IDS Mode)

```bash
sudo snort \
  -c /usr/local/etc/snort/snort.lua \
  -i ens4 \
  -A alert_fast
```

| 옵션 | 설명 |
|---|---|
| `-c` | Snort 설정 파일 지정 |
| `-i` | 트래픽을 수집할 네트워크 인터페이스 지정 |
| `-A alert_fast` | 탐지 이벤트를 간략히 출력 |
| `-T` | 설정·규칙 유효성 검증 |
| `-r` | 저장된 PCAP 파일 분석 |

`ens4`는 실험 환경에서 사용하는 인터페이스 예시이며, 실제 환경에 맞게 변경해야 합니다. 위 실행은 **IDS(탐지) 모드**이므로 `alert` 규칙이 패킷을 직접 차단하지 않습니다. 실제 차단에는 Inline IPS 환경 구성과 충분한 오탐 검증이 필요합니다.

### 3. Enable Detection Rules

`snort.lua`의 `ips` 설정에서 다음 **5개 규칙 파일**이 포함되어 있는지 확인합니다.

```lua
ips =
{
    variables = default_variables,

    rules = [[
        include /usr/local/etc/snort/rules/ai_ip.rules
        include /usr/local/etc/snort/rules/ai_user_agent.rules
        include /usr/local/etc/snort/rules/aipt-detections.rules
        include /usr/local/etc/snort/rules/codex_automation.rules
        include /usr/local/etc/snort/rules/http_4xx_frequency.rules
    ]]
}
```

규칙 파일을 복사하는 것만으로 자동 활성화되지는 않습니다. `snort.lua`에 `include`되어 있어야 합니다. **이미 `snort.lua`로 로드하는 규칙을 `-R`로 다시 지정하지 마세요.** 중복 로딩 문제가 발생할 수 있습니다.

규칙 목록을 수정한 뒤에는 `sudo snort -c /usr/local/etc/snort/snort.lua -T`로 재검증합니다.

### 4. AIPT Rule Event Filters (Optional but Important)

`aipt-detections.rules`는 Theori의 target-facing Suricata 규칙 **19개**를 변환한 파일입니다. 이 중 **13개**의 원본 빈도·경고 제한 조건은 `.rules`에 포함되지 않으며, Snort 3에서는 별도 `event_filter` 설정이 필요합니다.

특히 다음 규칙은 필터 적용 여부에 따라 경보 의미가 달라집니다.

| SID | 규칙 | 원본 의도에 가까운 필터 설정 |
|---|---|---|
| `9500008` | HTTP 404 집중 발생 | 응답 목적지(클라이언트) IP별 60초 동안 30회 |
| `9500297` | 빈 User-Agent 요청 반복 | 요청 출발지 IP별 60초 동안 20회 |
| `9500299` | User-Agent 누락 요청 반복 | 요청 출발지 IP별 60초 동안 20회 |

별도 `.lua` 파일을 저장소에 추가하지 않고 관리하려면 `snort.lua`의 기존 `event_filter` 선언 **다음 위치**에 아래 코드를 추가할 수 있습니다. 기존 필터가 있다면 덮어쓰거나 같은 SID의 필터를 중복 등록하지 마세요.

```lua
-- Theori AIPT 변환 규칙용 이벤트 필터
-- 기존 event_filter 설정 다음 위치에 추가
-- 이미 같은 SID의 필터가 있다면 병합/중복 제거 필요
event_filter = event_filter or {}
table.insert(event_filter, {gid=1, sid=9500006, type='both',      track='by_src', count=2,  seconds=60})
table.insert(event_filter, {gid=1, sid=9500007, type='limit',     track='by_src', count=1,  seconds=300})
table.insert(event_filter, {gid=1, sid=9500008, type='both',      track='by_dst', count=30, seconds=60})
table.insert(event_filter, {gid=1, sid=9500010, type='limit',     track='by_src', count=1,  seconds=600})
table.insert(event_filter, {gid=1, sid=9500110, type='limit',     track='by_src', count=1,  seconds=300})
table.insert(event_filter, {gid=1, sid=9500290, type='limit',     track='by_src', count=1,  seconds=300})
table.insert(event_filter, {gid=1, sid=9500291, type='limit',     track='by_src', count=1,  seconds=300})
table.insert(event_filter, {gid=1, sid=9500292, type='limit',     track='by_src', count=1,  seconds=300})
table.insert(event_filter, {gid=1, sid=9500293, type='limit',     track='by_src', count=1,  seconds=300})
table.insert(event_filter, {gid=1, sid=9500294, type='limit',     track='by_src', count=1,  seconds=300})
table.insert(event_filter, {gid=1, sid=9500295, type='limit',     track='by_src', count=1,  seconds=300})
table.insert(event_filter, {gid=1, sid=9500297, type='both',      track='by_src', count=20, seconds=60})
table.insert(event_filter, {gid=1, sid=9500299, type='both',      track='by_src', count=20, seconds=60})
```

`event_filter`는 기본적으로 **경보 출력 빈도를 제어하는 기능**이며, 특정 횟수 이상을 검출하는 `detection_filter`와 목적·동작이 다릅니다. 따라서 변환된 빈도 규칙은 원본과 탐지 이벤트 수가 같다고 단정할 수 없으며 PCAP으로 검증해야 합니다.

`aipt-detections.rules`의 모든 동작은 기본적으로 `alert`입니다. 원본의 `block-eligible` 분류는 차단 여부를 검토할 수 있는 정책 분류이지 Snort의 `drop` 액션이 아닙니다.

### 5. Analyze a PCAP (Optional)

```bash
sudo snort \
  -c /usr/local/etc/snort/snort.lua \
  -r /path/to/test.pcap \
  -A alert_fast
```

정상 브라우저, 허가된 자동화 도구 및 AI Agent의 PCAP을 각각 분석해 탐지율과 오탐률을 비교할 수 있습니다. 분석 대상이 암호화된 HTTPS 패킷뿐이라면 HTTP 헤더·URI·상태 코드를 복호화 없이 확인할 수 없습니다.

## Updating Rules

```bash
cd AImmunity-Snort
git pull
sudo cp rules/*.rules /usr/local/etc/snort/rules/
sudo snort -c /usr/local/etc/snort/snort.lua -T
```

`snort.lua`가 변경된 경우에는 백업 후 차이를 검토하여 별도로 반영합니다. 규칙 파일만 복사해도 실행 중인 프로세스에 즉시 반영되지 않을 수 있으므로 검증 후 Snort를 재시작하거나 서비스 재시작 절차를 수행합니다.

## Detection Methods

| Detection Method | 관련 규칙 | 탐지 내용 |
|---|---|---|
| IP-based Detection | `ai_ip.rules` | 알려진 AI 서비스 제공자 IP 대역 기반 정적 식별 |
| User-Agent Detection | `ai_user_agent.rules`, `codex_automation.rules`, `aipt-detections.rules` | AI Bot, 자동화 HTTP 클라이언트, AI 침투 테스트 도구의 UA 문자열 식별 |
| Request Rate Detection | `codex_automation.rules`, `http_4xx_frequency.rules` 등 | 반복 요청 또는 빈도 기반 징후 관찰. 실제 시간·횟수 조건은 개별 규칙을 기준으로 확인 |
| HTTP Error Response Detection | `http_4xx_frequency.rules`, `aipt-detections.rules` | 4xx 오류 응답의 증가 또는 반복 발생을 통한 경로 탐색·스캐닝 징후 관찰 |
| URI/Reconnaissance Detection | `aipt-detections.rules` | `manifest.json` 계열 경로 탐색처럼 특정 HTTP 요청 조건에 부합하는 정찰 징후 탐지 |
| Tool Fingerprint Detection | `aipt-detections.rules` | ARTEX·CyberStrikeAI·HexStrike-AI, 헤드리스 브라우저 및 범용 스캐너의 식별 문자열 탐지 |

`aipt-detections.rules`의 19개 규칙은 ARTEX 3개, CyberStrikeAI 1개, HexStrike-AI 2개, 일반 자동화 관련 4개, 범용 스캐너·퍼징 관련 9개로 분류됩니다. **범용 도구·클라이언트 지표를 AI Agent 고유 특징으로 간주하지 않습니다.**

또한 `http_4xx_frequency.rules`와 AIPT의 SID `9500008`은 HTTP 오류 응답을 근거로 경보가 겹칠 수 있습니다. 분석 결과를 집계할 때 동일 이벤트의 중복 경보를 구분해야 합니다.

## Limitations

- **우회 가능성:** User-Agent는 위조할 수 있으며 IP·Proxy·HTTP 클라이언트 변경으로 정적 지표의 유효성이 감소할 수 있습니다.
- **오탐 가능성:** 정상 브라우저 자동화, 모니터링 시스템, 크롤러, 승인된 취약점 점검 도구도 자동화·스캔 특성을 보일 수 있습니다.
- **4xx 응답의 비특이성:** 400·401·403·404 등의 응답은 일반 사용자·클라이언트 오류에서도 발생합니다. HTTP 상태 코드의 반복만으로 공격이나 AI Agent 사용을 확정할 수 없습니다.
- **구현 환경 종속성:** 특정 도구의 HTTP User-Agent는 해당 도구·라이브러리·버전의 특징일 수 있으며 LLM 자체의 고유 특징은 아닙니다.
- **빈도 필터의 분리:** AIPT 변환 규칙의 원본 빈도 조건 일부는 별도 `event_filter` 적용이 필요하며, 적용 전후 경고 발생 수가 달라질 수 있습니다.
- **HTTPS 관찰 위치:** 암호화된 HTTPS 구간을 관찰하는 Snort는 HTTP 헤더·URI·응답 코드를 직접 볼 수 없습니다. TLS 종료 지점 이후의 평문 트래픽 관찰 여부를 확인해야 합니다.
- **네트워크 범위:** `HOME_NET = 'any'`인 경우 실제 보호 대상에 한정되지 않을 수 있으므로 운영 환경에서는 보호할 IP/CIDR로 지정하는 것이 권장됩니다.
- **탐지와 차단의 구분:** `alert`는 탐지이며, 실제 차단을 보장하지 않습니다. Snort 규칙을 `drop`으로 전환하기 전에는 Inline IPS 구성과 오탐·서비스 영향 평가가 필요합니다.
- **검증 범위:** Snort의 `-T` 검사는 문법·설정 점검일 뿐이며, 실제 탐지율(TPR), 오탐률(FPR), 우회 내성을 보장하지 않습니다. Suricata 원본과 Snort 변환본의 동작도 일치 여부를 별도로 검증해야 합니다.

## References

- [Snort 3 Official Documentation](https://docs.snort.org/)
- [Snort 3 Rule Writing Guide](https://docs.snort.org/start/rules)
- [Snort 3 GitHub](https://github.com/snort3/snort3)
- [Snort 3 `event_filter` Configuration Example](https://github.com/snort3/snort3/blob/master/lua/snort.lua)
- [Theori — AIPT Detection Pack](https://github.com/theori-io/aipt-detections)
- [Theori — Suricata Target-facing Rules](https://github.com/theori-io/aipt-detections/blob/main/suricata/aipt-target-facing.rules)
- [Detection Rule License 1.1 (DRL-1.1)](https://github.com/theori-io/aipt-detections/blob/main/LICENSE)

## License

본 프로젝트에 직접 작성한 코드 및 규칙의 라이선스는 저장소의 [LICENSE](LICENSE)를 참고하세요.

`aipt-detections.rules`는 **Theori의 `aipt-detections` 규칙을 Snort 3용으로 변환한 파생물**로, 원본의 **Detection Rule License (DRL) 1.1** 및 저작자 표시 의무가 적용됩니다. 원본 출처·라이선스 링크와 변환 사실을 유지해야 하며, 해당 규칙의 탐지 결과를 외부에 제공할 때에도 원본에 명시된 저작자 표시 요구사항을 준수해야 합니다. 본 프로젝트의 `LICENSE`가 외부 규칙의 라이선스를 대체하지는 않습니다.

- Original project: [theori-io/aipt-detections](https://github.com/theori-io/aipt-detections)
- Original rules: [`suricata/aipt-target-facing.rules`](https://github.com/theori-io/aipt-detections/blob/main/suricata/aipt-target-facing.rules)
- License: [Detection Rule License 1.1](https://github.com/theori-io/aipt-detections/blob/main/LICENSE)

## Disclaimer

본 프로젝트는 AI Web Agent의 접근 통제 및 네트워크 보안 연구 목적으로 개발되었습니다. 포함된 규칙은 실험 또는 공개된 탐지 지표를 기반으로 하며, AI 사용 여부의 확정, 공격 성공 판별, 운영 환경의 탐지 정확도 및 차단 효과를 보장하지 않습니다. 실제 운영 적용 전 정상 트래픽과 허가된 보안 테스트 트래픽을 이용한 탐지·오탐·성능 검증이 필요합니다.
