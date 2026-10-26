# AImmunity-Snort

Snort 3 기반 AI Web Agent 접근 탐지 및 차단 규칙

## Overview

AImmunity-Snort는 AI Web Agent의 웹사이트 접근 과정에서 발생하는 네트워크 및 HTTP 요청 특징을 기반으로 탐지 규칙을 설계하고 관리하는 프로젝트입니다.

User-Agent, IP 주소와 같은 정적 식별 정보뿐만 아니라 요청 빈도, URI 접근 패턴 등의 행위 기반 특징을 활용하여 AI Agent의 접근을 탐지하고, 기존 접근 통제 기술의 우회 가능성을 분석하는 것을 목표로 합니다.

본 저장소는 Snort 3 설정 파일과 사용자 정의 탐지 규칙을 제공하며, Snort 3가 설치된 Linux 환경에서 사용할 수 있습니다.

## Repository Structure

```text
AImmunity-Snort/
├── rules/
│   ├── ai_ip.rules
│   ├── ai_user_agent.rules
│   └── codex_automation.rules
├── LICENSE
├── README.md
└── snort.lua
```

| 파일 | 설명 |
|---|---|
| `snort.lua` | Snort 3 설정 및 탐지 규칙 로딩 |
| `ai_ip.rules` | AI 서비스 제공자의 IP 기반 탐지 및 차단 규칙 |
| `ai_user_agent.rules` | AI Bot 및 AI Agent의 User-Agent 기반 탐지 및 차단 규칙 |
| `codex_automation.rules` | Codex 실험에서 관찰한 HTTP 요청 및 행동 패턴 기반 탐지 규칙 |
| `LICENSE` | 프로젝트 라이선스 |

## Requirements

- Linux (Ubuntu 권장)
- Snort 3
- Git
- 관리자 권한 (`sudo`)
- 패킷을 관찰할 수 있는 네트워크 인터페이스

Snort 3가 설치되어 있어야 하며, 다음 기본 설정 파일이 필요합니다.

- `snort_defaults.lua`
- `file_magic.rules`

본 README는 Snort 설정 디렉터리가 `/usr/local/etc/snort/`인 환경을 기준으로 작성되었습니다.

## Installation

### 1. Clone Repository

```bash
git clone https://github.com/USERNAME/AImmunity-Snort.git
cd AImmunity-Snort
```

`USERNAME`을 실제 GitHub 사용자명으로 변경합니다.

### 2. Verify Snort Installation

```bash
snort --version
```

필수 설정 파일을 확인합니다.

```bash
ls /usr/local/etc/snort/snort_defaults.lua
ls /usr/local/etc/snort/file_magic.rules
```

해당 파일이 없다면 Snort 설치 경로 및 기본 설정 파일 구성을 확인해야 합니다.

### 3. Backup Existing Configuration

```bash
sudo cp /usr/local/etc/snort/snort.lua \
  /usr/local/etc/snort/snort.lua.bak
```

### 4. Apply Configuration and Rules

저장소의 설정 파일과 탐지 규칙을 Snort 설정 디렉터리에 복사합니다.

```bash
# Snort 설정 적용
sudo cp snort.lua /usr/local/etc/snort/

# 규칙 디렉터리 생성
sudo mkdir -p /usr/local/etc/snort/rules

# 탐지 규칙 복사
sudo cp rules/*.rules /usr/local/etc/snort/rules/
```

규칙이 정상적으로 복사되었는지 확인합니다.

```bash
ls -l /usr/local/etc/snort/rules/
```

### 5. Validate Configuration

```bash
sudo snort \
  -c /usr/local/etc/snort/snort.lua \
  -T
```

설정 검증에 성공하면 `Snort successfully validated the configuration` 메시지가 출력됩니다.

오류가 발생하면 Snort 버전, 설정 파일 경로 및 규칙 문법을 확인합니다.

## Usage

### 1. Check Network Interface

```bash
ip -br link
```

실행 환경에서 패킷을 수집할 네트워크 인터페이스를 확인합니다.

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
| `-i` | 패킷 수집 인터페이스 지정 |
| `-A alert_fast` | 탐지 결과를 간략한 형식으로 출력 |
| `-T` | 설정 및 규칙 유효성 검증 |

`ens4`는 AImmunity 실험 환경에서 사용하는 네트워크 인터페이스입니다. 다른 환경에서는 실제 인터페이스 이름으로 변경해야 합니다.

위 명령어는 IDS 모드로 실행되며, 패킷을 감시하고 탐지 이벤트를 생성합니다.

실제 패킷 차단을 위해서는 별도의 Inline IPS 환경 구성이 필요합니다.

### 3. Enable Detection Rules

현재 `snort.lua`에서는 다음 두 규칙 파일이 기본적으로 활성화되어 있습니다.

```lua
ips =
{
    variables = default_variables,

    rules = [[
        include /usr/local/etc/snort/rules/ai_ip.rules
        include /usr/local/etc/snort/rules/ai_user_agent.rules
    ]]
}
```

`codex_automation.rules`는 저장소에 포함되어 있지만 기본 설정에서는 로드되지 않습니다.

Codex 행동 기반 탐지 규칙을 활성화하려면 `snort.lua`의 `ips` 설정을 다음과 같이 수정합니다.

```lua
ips =
{
    variables = default_variables,

    rules = [[
        include /usr/local/etc/snort/rules/ai_ip.rules
        include /usr/local/etc/snort/rules/ai_user_agent.rules
        include /usr/local/etc/snort/rules/codex_automation.rules
    ]]
}
```

규칙을 추가한 후 설정을 검증합니다.

```bash
sudo snort \
  -c /usr/local/etc/snort/snort.lua \
  -T
```

## Updating Rules

GitHub 저장소에서 최신 규칙을 가져와 적용합니다.

```bash
# 저장소 이동
cd AImmunity-Snort

# 최신 파일 가져오기
git pull

# 규칙 업데이트
sudo cp rules/*.rules /usr/local/etc/snort/rules/

# 설정 검증
sudo snort \
  -c /usr/local/etc/snort/snort.lua \
  -T
```

`snort.lua`도 변경된 경우에는 기존 설정을 백업하고 새 설정 파일을 별도로 적용해야 합니다.

실행 중인 Snort는 규칙 파일을 복사하는 것만으로 변경 사항이 즉시 반영되지 않을 수 있으므로, 설정 검증 후 Snort 프로세스를 재시작합니다.

## Detection Methods

| Detection Method | Description |
|---|---|
| IP-based Detection | AI 서비스 제공자의 IP 대역 기반 탐지 |
| User-Agent Detection | HTTP User-Agent 헤더 기반 탐지 |
| Request Rate Detection | 특정 시간 내 반복적인 요청 탐지 |
| URI Pattern Detection | 특정 URI에 대한 반복 접근 및 탐색 패턴 분석 |
| Behavioral Detection | 자동화 요청에서 관찰된 복합적인 행동 특징 분석 |

각 탐지 방식은 해당 규칙 파일의 활성화 여부 및 실험 환경에 따라 적용됩니다.

## Limitations

- User-Agent 및 IP 기반 탐지는 해당 정보가 변경되면 우회될 수 있습니다.
- 요청 빈도 및 행동 패턴은 정상적인 자동화 트래픽에서도 나타날 수 있습니다.
- 특정 AI Agent에서 관찰된 특징이 모든 AI Agent의 공통 특징을 의미하지는 않습니다.
- 암호화된 HTTPS 트래픽의 HTTP 헤더와 URI는 복호화 없이 직접 분석할 수 없습니다.
- `HOME_NET = 'any'`로 설정되어 있으므로 실제 운영 환경에서는 보호 대상 네트워크 범위를 지정하는 것이 권장됩니다.
- 탐지 규칙을 실제 서비스에 적용하기 전 탐지율 및 오탐률 검증이 필요합니다.

## References

- [Snort 3 Official Documentation](https://docs.snort.org/)
- [Snort 3 GitHub](https://github.com/snort3/snort3)
- [Snort 3 Rule Writing Guide](https://docs.snort.org/rules/)

## License

본 프로젝트의 라이선스는 저장소의 [LICENSE](LICENSE) 파일을 참고하세요.

## Disclaimer

본 프로젝트는 AI Web Agent의 접근 통제 및 네트워크 보안 연구를 목적으로 합니다. 제공되는 탐지 규칙은 실험 환경을 기반으로 작성되었으며, 실제 운영 환경에서의 탐지 성능과 차단 효과를 보장하지 않습니다.
