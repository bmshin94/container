# Apple `container` 전수조사 & 활용 가이드 (한국어)

> 이 문서는 Claude Code 세션에서 `bmshin94/container` 레포지토리를 전수조사하고
> 나눈 대화를 정리한 자료입니다.

- **이 레포지토리:** https://github.com/bmshin94/container
- **원본(Upstream):** https://github.com/apple/container
- **릴리스 다운로드:** https://github.com/apple/container/releases
- **의존 패키지:** https://github.com/apple/containerization
- **API 문서:** https://apple.github.io/container/documentation/
- **라이선스:** Apache-2.0
- **작성일:** 2026-10-01

---

## 목차

1. [한 줄 요약](#1-한-줄-요약)
2. [레포지토리 전수조사](#2-레포지토리-전수조사)
3. [쉽게 이해하기](#3-쉽게-이해하기)
4. [설치 및 사용법](#4-설치-및-사용법)
5. [플러그인 vs 스킬 vs MCP](#5-플러그인-vs-스킬-vs-mcp)
6. [API 토큰 필요 여부](#6-api-토큰-필요-여부)
7. [AI 에이전트 구축에 도움이 되는가](#7-ai-에이전트-구축에-도움이-되는가)
8. [React / PHP 로 만들 수 있는가](#8-react--php-로-만들-수-있는가)
9. [유튜브 강의 제작 가능성](#9-유튜브-강의-제작-가능성)
10. [수익화 아이디어 전체](#10-수익화-아이디어-전체)
11. [리스크와 대응](#11-리스크와-대응)
12. [최종 추천 전략](#12-최종-추천-전략)

---

## 1. 한 줄 요약

**Apple이 만든 공식 오픈소스 `container`** — macOS(Apple Silicon)에서 리눅스 컨테이너를
**컨테이너 1개당 경량 VM 1개** 방식으로 실행하는 도구. 사실상 **Docker Desktop의 애플 공식 대체제**.

이 레포는 `apple/container`를 포크한 것이며, 원본 대비 추가된 것은 `CLAUDE.md`(페르소나 가이드) 하나뿐.

```
ca9dbd0  Merge pull request #1 from bmshin94/feat/claude-guide
ba56342  docs: created CLAUDE.md persona guide        <- 포크에서 추가
57f0b93  K8s plugin: Support custom CNI manifest       <- 여기부터 apple/container 원본
```

---

## 2. 레포지토리 전수조사

### 2.1 규모

| 항목 | 수치 |
|---|---|
| Swift 소스 파일 | 447개 |
| 소스 코드 라인 수 | 약 45,700줄 |
| 문서 | 21개 (command-reference.md 만 1,700줄+) |
| 테스트 스위트 | 17개 |
| CI 워크플로 | 7개 |

### 2.2 디렉토리 구조

| 폴더 | 역할 |
|---|---|
| `Sources/` | 본체 Swift 코드 (20개 모듈) |
| `Tests/` | 유닛 테스트 + 통합 테스트 |
| `docs/` | 전체 문서 (튜토리얼, 커맨드 레퍼런스, 네트워킹, k8s 등) |
| `skills/` | **Claude Code용 에이전트 스킬** |
| `.claude-plugin/` | **Claude Code 플러그인 매니페스트** |
| `examples/` | VSCode + 컨테이너 머신 연동 예제 |
| `scripts/` | 설치 / 제거 / 업데이트 / 포맷 스크립트 |
| `signing/` | 코드 서명 entitlements |
| `.github/` | 이슈 템플릿, PR 템플릿, CI 워크플로 |
| `Package.swift` | Swift Package Manager 빌드 정의 (30KB) |
| `Makefile` | 빌드 / 테스트 / 설치 자동화 (20KB) |

### 2.3 아키텍처 — XPC 기반 마이크로서비스

```
container (CLI)
      |  XPC
container-apiserver            <- launchd 가 관리하는 중앙 데몬
      |
      +-- container-core-images          이미지 관리 + 콘텐츠 스토어
      +-- container-network-vmnet        가상 네트워크 (vmnet 프레임워크)
      +-- container-runtime-linux        컨테이너 1개당 1 프로세스
      +-- container-machine-apiserver    컨테이너 머신
```

주요 모듈:

| 모듈 | 역할 |
|---|---|
| `CLI/`, `ContainerCommands/` | 명령어 전체 (run, build, image, volume, network, machine, system, k8s) |
| `APIServer/` | 중앙 데몬 + DNS 핸들러 |
| `Services/` | 컨테이너 / 이미지 / 네트워크 / 머신 API 서비스 |
| `ContainerBuild/` | Dockerfile 빌드 (gRPC 로 빌더 VM과 통신) |
| `ContainerXPC/` | 애플 XPC IPC 래퍼 |
| `DNSServer/` | 컨테이너 이름 해석용 자체 DNS 서버 |
| `SocketForwarder/` | TCP/UDP 포트 포워딩 |
| `ContainerK8s/`, `Plugins/K8s/` | 로컬 쿠버네티스 클러스터 (실험적) |
| `TerminalProgress/` | 터미널 진행률 바 |
| `ContainerPlugin/` | 내부 플러그인 로더 |

### 2.4 Docker Desktop 과의 차이

| 항목 | Docker Desktop | Apple `container` |
|---|---|---|
| 구조 | 리눅스 VM 1개 안에 컨테이너 전부 | **컨테이너 1개당 경량 VM 1개** |
| 보안 | 컨테이너끼리 커널 공유 | 각자 완전한 VM 격리 |
| 프라이버시 | VM에 광범위하게 마운트 | 각 VM에 필요한 것만 마운트 |
| 네트워크 | 포트 매핑 필요 | **컨테이너마다 고유 IP** |
| 비용 | 기업 사용 유료 | Apache-2.0 무료 |
| 요구사항 | 인텔/M 모두 | **Apple Silicon + macOS 26** |

사용하는 macOS 기술: Virtualization.framework, vmnet, XPC, launchd, Keychain, unified logging.

### 2.5 `skills/` — 애플이 직접 쓴 Claude Code 스킬

```
.claude-plugin/
  plugin.json        <- 플러그인 정의 (skills: ["./skills/container"])
  marketplace.json   <- 마켓플레이스 정의 (name: "apple-container")
skills/
  README.md
  container/
    SKILL.md                            <- 항상 로드되는 요약 (짧게 유지)
    references/
      docker-migration.md               <- Docker -> container 전체 매핑
      container-machines.md             <- 컨테이너 머신 사용법
```

스킬이 가르치는 핵심:

1. 명령어 그룹은 전부 **단수형** (`container images` X / `container image ls` O)
2. Docker -> container 명령어 매핑표
3. 함정(Gotchas)
   - `container build` 에 `-t` 없으면 태그가 랜덤 UUID
   - `container ls` 는 멈춘 컨테이너를 숨김 (`-a` 필요)
   - DNS 이름 해석은 **3단계** 설정 필요
   - 커스텀 네트워크에서는 이름 해석 불가 (이슈 #1809) — IP 로 접근
   - `container images` 는 "Plugins are unavailable" 라는 **오해를 유발하는 에러**를 냄
     (서비스 문제가 아니라 명령어 이름 문제)
4. docker-compose 를 셸 스크립트로 번역하는 완전한 예제

### 2.6 컨테이너 머신 (Docker 에 없는 기능)

- `container run` = **애플리케이션** 실행
- `container machine` = **리눅스 환경** 자체 (init 시스템 부팅, systemd 동작)

핵심: **macOS 사용자명과 홈 디렉토리가 리눅스 안에 그대로 매핑됨.**
맥에서 편집 -> 리눅스에서 컴파일/실행, 복사 단계 없음. Lima / Colima 대체제.

---

## 3. 쉽게 이해하기

### 컨테이너란?

프로그램 + 실행에 필요한 모든 것을 하나로 밀봉한 "도시락". 어디서 열어도 동일하게 동작.

### 왜 맥 전용 도구가 필요한가?

컨테이너는 원래 리눅스 커널 기술(namespace, cgroup)이라 맥에서 직접 실행 불가.
따라서 맥에서는 리눅스 VM 을 띄워야 함.

- **Docker Desktop:** 리눅스 "원룸" 하나를 잡고 컨테이너를 전부 거기 넣음
- **Apple container:** 컨테이너마다 "독채"(경량 VM)를 하나씩 줌

무거워 보이지만, 애플이 자기 칩(하드웨어 가상화) + 자기 OS + 자기 프레임워크를
모두 통제하기 때문에 VM 을 거의 앱 수준으로 가볍게 만들 수 있었음.

### 호텔 비유

```
사용자 명령          ->  container CLI          (프런트 데스크)
                     ->  container-apiserver    (지배인, 항상 상주)
                     ->  core-images            (창고 담당)
                         network-vmnet          (통신 담당)
                         runtime-linux          (객실 담당, 컨테이너당 1명)
```

부서를 나눈 이유: 장애 격리 + 최소 권한 원칙.

### 스킬이 왜 필요한가

AI 는 Docker 를 수없이 학습했기 때문에 반사적으로 `container images` 를 입력함.
그런데 에러 메시지가 "Plugins are unavailable. Start the container system services" 라서
AI 가 "서비스가 꺼졌구나" 하고 엉뚱하게 서비스를 재시작하는 삽질을 함.

애플은 이런 **AI 의 예상 실패 지점**을 미리 문서로 적어둠. 이것이 스킬의 본질.

---

## 4. 설치 및 사용법

### 4.1 요구사항

```
- Apple Silicon Mac (M1 이상). 인텔 맥 불가
- macOS 26 권장 (15 에서는 네트워크 기능 대폭 제한)
- 소스 빌드 시 Xcode 26
```

macOS 15 제약: 컨테이너 간 통신 불가, `container network` 명령어 자체 부재.

### 4.2 설치 (권장: 설치 파일)

```
1. https://github.com/apple/container/releases 접속
2. 최신 .pkg 다운로드 (애플 서명됨)
3. 더블클릭 -> 관리자 비밀번호 입력 (/usr/local 아래 설치)
```

```bash
container system start     # 시스템 서비스 시작
container system status    # 확인
container --version
```

### 4.3 소스 빌드

```bash
git clone https://github.com/bmshin94/container
cd container

rm -rf test-data
make APP_ROOT=test-data all test integration
make install

# 릴리스 빌드 (더 빠름)
BUILD_CONFIGURATION=release make all
BUILD_CONFIGURATION=release make install
```

> 주의: macOS 26 vmnet 버그로 인해 프로젝트를 `Documents` 또는 `Desktop` 아래에 두면
> 네트워크 생성이 실패함. `~/projects/container` 같은 경로를 사용할 것.

### 4.4 업그레이드 / 다운그레이드 / 제거

```bash
container system stop                            # 반드시 먼저 중지

/usr/local/bin/update-container.sh               # 최신 버전으로
/usr/local/bin/update-container.sh -v 0.3.0      # 특정 버전으로

/usr/local/bin/uninstall-container.sh -k         # 제거 (데이터 유지)
/usr/local/bin/uninstall-container.sh -d         # 제거 (데이터 포함)
```

### 4.5 기본 명령어

```bash
container run --rm alpine echo hello

container ls                 # 실행 중 (= docker ps)
container ls -a              # 멈춘 것 포함 (-a 없으면 안 보임)
container image ls           # (= docker images)  * images 아님, image 임
container image pull nginx
container image rm nginx
container delete mybox       # (= docker rm), 별칭 rm
container system status      # (= docker info)
container system df
container system logs
```

### 4.6 빌드

```bash
container build -t myapp:latest .     # -t 사실상 필수 (없으면 태그가 랜덤 UUID)

container build -t myapp --platform linux/arm64,linux/amd64 .
container build -t myapp --build-arg VERSION=1.0 --no-cache .
container build -t myapp --secret id=mykey,src=./key.txt .
container build -o type=oci,dest=./out.tar .

container builder status                       # 빌드 실패 시 확인
container builder start --cpus 8 --memory 16g
```

### 4.7 실행과 네트워크

```bash
container run -d --name web -p 8080:80 nginx
container run -it --rm ubuntu bash
container run -d --name db -e POSTGRES_PASSWORD=secret postgres:16
container run -v $(pwd):/app -w /app node:22 npm test

container inspect web      # 컨테이너의 실제 IP 확인
container logs -f web
container exec -it web sh
container stats
```

컨테이너마다 고유 IP 가 부여되므로, 호스트에서 그 IP 로 바로 접근 가능.
`-p` 는 "맥의 특정 포트에 바인딩하고 싶을 때"만 필요.

### 4.8 이름 기반 통신 (DNS 3단계 설정)

```bash
# 1단계: 설정 파일 생성 (처음엔 파일 자체가 없음)
mkdir -p ~/.config/container
cat > ~/.config/container/config.toml <<'EOF'
[dns]
domain = "test"
EOF

# 2단계: 설정 재적용
container system stop && container system start

# 3단계: macOS 가 container 의 리졸버를 쓰도록 등록
sudo container system dns create test

container system property ls    # 확인
```

- `db.test` 로 접근해야 함. **`db` 만으로는 해석되지 않음.**
- `container network create` 로 만든 커스텀 네트워크에서는 이름 해석이 **아예 동작하지 않음**
  (https://github.com/apple/container/issues/1809). 커스텀 네트워크는 격리 목적이며,
  그 안에서는 `container inspect` 로 얻은 IP 로 연결할 것.

### 4.9 docker-compose 대체

`container compose` 는 존재하지 않음. 셸 스크립트로 번역.

```bash
#!/bin/bash
set -euo pipefail

# --network 를 쓰지 않음 -> 둘 다 default 네트워크 -> 이름 해석 동작
container run -d --name db -e POSTGRES_PASSWORD=secret postgres:16

# depends_on -> 명시적 준비 대기 (compose 는 start 만 기다리므로 이게 더 정확)
until container exec db pg_isready -q; do sleep 1; done

container run -d --name web -p 8080:80 \
  -e DATABASE_URL=postgres://postgres:secret@db.test:5432/postgres \
  myapp:latest

# 정리
# container stop web db && container delete web db
```

### 4.10 컨테이너 머신

```bash
container machine create ubuntu:24.04 --name dev --cpus 8 --memory 16G --set-default
container machine run                               # 기본 머신 셸 진입
container machine run -n dev uname -a               # 단일 명령
container machine run -n dev -- cat /proc/cpuinfo   # 플래그가 있으면 --

container machine ls                 # 별칭: m ls
container machine inspect dev
container machine logs dev
container machine set -n dev cpus=4 memory=8G
container machine stop dev           # set 은 재부팅 후 적용
container machine rm dev
```

컨테이너 머신 안에서 `whoami` 는 맥 사용자명, `pwd` 는 맥 홈 디렉토리.
systemd 이미지라면 `sudo systemctl start postgresql` 같은 서비스 관리도 가능.

### 4.11 로컬 쿠버네티스 (실험적)

```bash
container k8s create            # 기본 이름 k8s-dev
container k8s list
kubectl get nodes               # ~/.kube/config 에 자동 등록
container k8s load-image myapp:latest
container k8s write-config
container k8s delete
```

### 4.12 정리 / 자동완성

```bash
container prune           # 멈춘 컨테이너
container image prune
container volume prune
container network prune

container --generate-completion-script zsh > ~/.container-completion.zsh
```

### 4.13 Docker 와 다른 점 요약

| 항목 | 주의 |
|---|---|
| `container images` | 없음 -> `container image ls` |
| `container build` | `-t` 사실상 필수 |
| `container ls` | 멈춘 것 안 보임 -> `-a` |
| `container restart` | 없음 -> `stop && start` |
| `container attach` | 없음 -> `exec -it <id> sh` |
| `container commit` | 없음 -> Dockerfile 빌드 |
| `container top` | 없음 -> `exec <id> ps aux` |
| `container port` | 없음 -> `inspect` |
| `--restart` 정책 | 없음 -> launchd 또는 컨테이너 머신 |
| `docker events` / `docker history` | 없음 |
| `rename` / `pause` / `wait` / `diff` / `update` | 없음 |

### 4.14 container 에만 있는 기능

```bash
container run --publish-socket /host/path:/container/path   # Unix 소켓 발행
container run --ssh                     # SSH 에이전트 포워딩
container run --rosetta                 # x86-64 바이너리 실행
container run --virtualization          # 중첩 가상화
container system kernel set             # 커널 교체
container machine ...                   # 컨테이너 머신
```

---

## 5. 플러그인 vs 스킬 vs MCP

### 결론: 이 레포는 **"스킬을 담은 플러그인"**. MCP 가 아님.

```
Claude Code 플러그인 ("container")      <- 배포 단위 (포장 박스)
    └── 스킬 ("container")              <- 내용물 (지식 문서)
            ├── SKILL.md
            └── references/*.md
```

근거:

```json
// .claude-plugin/plugin.json
{ "name": "container", "skills": ["./skills/container"] }

// .claude-plugin/marketplace.json
{ "name": "apple-container", "plugins": [{ "name": "container", "source": "./" }] }
```

### 셋의 차이

| | 플러그인 | 스킬 | MCP |
|---|---|---|---|
| 정체 | 배포 포장지 | 지식 / 문서 | 외부 연결 서버 |
| 담는 것 | 스킬 + 명령어 + 에이전트 + MCP설정 + 훅 | 마크다운 | 실행 가능한 도구 |
| 동작 | 설치 단위 | 읽혀서 AI를 똑똑하게 | 호출되어 AI가 행동 |
| 핵심 파일 | `plugin.json` | `SKILL.md` | 별도 서버 프로세스 |
| 별도 프로세스 | 없음 | 없음 | 있음 |
| 토큰 | 불필요 | 불필요 | 보통 필요 |

**핵심 구분:** 스킬 = "어떻게 하는지 아는 것"(지식), MCP = "실제로 할 수 있는 것"(손발),
플러그인 = "그걸 담아 배포하는 상자".

### 동작 방식

```
사용자: "nginx 컨테이너 띄워줘"
 -> Claude 가 SKILL.md 의 description 을 보고 관련 있다고 판단
 -> SKILL.md 를 읽음 (명령어 단수형, -t 필요 등)
 -> 기존 Bash 도구로 실행: container run -d --name web -p 8080:80 nginx
```

새 도구가 추가되는 게 아니라, 기존 Bash 를 **정확하게** 쓰게 되는 것.
그래서 MCP 서버가 필요 없음.

### 주의: 이름이 같은 다른 것

- `.claude-plugin/` — Claude Code 플러그인
- `Sources/Plugins/` — container 자체의 내부 XPC 헬퍼 플러그인 (K8s, NetworkVmnet 등)

둘은 완전히 무관함.

### 설치

```bash
# GitHub 에서 바로 (클론 불필요)
/plugin marketplace add apple/container
/plugin install container@apple-container

# 이 포크에서
/plugin marketplace add bmshin94/container
/plugin install container@apple-container

# 로컬 클론에서
/plugin marketplace add /path/to/container
/plugin install container@apple-container
```

---

## 6. API 토큰 필요 여부

| 대상 | 토큰 | 비고 |
|---|---|---|
| `container` CLI 설치 / 실행 | 불필요 | 100% 로컬 |
| Claude Code 스킬 사용 | 불필요 | 마크다운을 읽을 뿐 |
| 플러그인 설치 | 불필요 | 공개 레포 |
| 퍼블릭 이미지 pull | 불필요 | 익명 접근 |
| 프라이빗 레지스트리 / 이미지 push | **필요** | 아래 참고 |

```bash
container registry login ghcr.io
# Username: <id>
# Password: <Personal Access Token>

container registry list
container registry logout ghcr.io
```

자격증명은 **macOS 키체인**에 저장됨 (Docker 의 평문 `~/.docker/config.json` 과 다름).

AI 에이전트를 직접 개발할 경우에는 별개로 Anthropic API 키가 필요하지만,
이는 `container` 와 무관한 사항임.

---

## 7. AI 에이전트 구축에 도움이 되는가

### 결론: 세 가지 레벨로 도움이 됨.

### 레벨 1 — AI 에이전트 샌드박스 (가장 실전적)

AI 에이전트의 핵심 난제: **AI 가 생성한 코드를 어디서 안전하게 실행할 것인가.**
AI 가 작성한 코드는 "신뢰할 수 없는 코드"로 다뤄야 함 (프롬프트 인젝션 위험).

| 요구사항 | container 의 강점 |
|---|---|
| 강한 격리 | **컨테이너당 독립 VM** — 커널 공유 없음. 컨테이너 탈출 방어력이 근본적으로 높음 |
| 빠른 생성/파기 | 부팅이 컨테이너급 |
| 최소 데이터 노출 | 필요한 폴더만 마운트 |
| 네트워크 제어 | 컨테이너별 IP + 격리 네트워크 |
| 스크립팅 | `inspect` / `list` 가 JSON 출력 지원 |

```python
import subprocess, uuid

def run_ai_code_safely(code: str, timeout=30):
    name = f"sandbox-{uuid.uuid4().hex[:8]}"
    try:
        r = subprocess.run([
            "container", "run", "--rm", "--name", name,
            "--memory", "512m", "--cpus", "1",
            "--network", "isolated",
            "python:3.12-alpine", "python", "-c", code
        ], capture_output=True, text=True, timeout=timeout)
        return {"stdout": r.stdout, "stderr": r.stderr, "code": r.returncode}
    except subprocess.TimeoutExpired:
        subprocess.run(["container", "kill", name])
        return {"error": "timeout"}
```

### 레벨 2 — 스킬 작성법 교과서

애플이 실전에서 검증한 패턴 5가지:

1. **점진적 공개(Progressive Disclosure)** — `SKILL.md` 는 짧게(항상 로드),
   상세는 `references/` 로 (필요할 때만 로드)
2. **AI 의 실패 모드를 문서화** — "여기서 틀린 명령어는 오타가 아니라 그럴듯한 창작이다"
3. **낡을 정보 대신 확인 방법을 적기** — "플래그 목록을 붙여넣지 마라. `--help` 가 권위 있다"
4. **오해를 유발하는 에러 메시지를 해설** — "'Plugins are unavailable' 가 떠도 서비스 문제가
   아닐 수 있다. 이름부터 확인해라"
5. **description 에 트리거 키워드를 촘촘히** — Docker, Colima, Lima, Podman, Dockerfile,
   OCI, Apple silicon 등 사용자가 쓸 법한 단어를 모두 포함

### 레벨 3 — 에이전트 아키텍처 참고서

| container 구조 | AI 에이전트 시스템 |
|---|---|
| `container` CLI | 사용자 인터페이스 |
| `container-apiserver` | 오케스트레이터 에이전트 |
| `core-images`, `network-vmnet`, `runtime-linux` | 전문 서브에이전트 |
| XPC 메시지 | 에이전트 간 메시지 패싱 |
| 플러그인 로더 (`config.toml`) | 동적 도구 등록 |
| 컨테이너당 런타임 1개 | 작업당 워커 1개 |

### 제약

맥 전용이므로 프로덕션 에이전트 서버(보통 리눅스)에는 직접 쓸 수 없음.
**로컬 개발/테스트 단계**에 사용하고, 배포는 Docker / Firecracker / gVisor 로 가는 것이 현실적.
OCI 표준이라 이미지는 그대로 호환되므로 전환은 매끄러움.

---

## 8. React / PHP 로 만들 수 있는가

### 결론: **엔진 자체는 불가능, 그 위의 레이어는 전부 가능.**

### 불가능한 이유

`container` 는 `Virtualization.framework`, `vmnet`, `XPC`, `Security(Keychain)` 등
**macOS 시스템 프레임워크**를 직접 호출하며, 코드 서명과 entitlements 가 필요함.
Node.js / PHP 에는 이 바인딩이 없음.

비유: 엔진은 JS 로 만들 수 없지만, **엔진을 조종하는 대시보드는 얼마든지 만들 수 있음.**

### 만들 수 있는 것

#### (1) container Desktop — GUI 앱 (가장 큰 기회)

현재 `container` 는 CLI 만 있고 GUI 가 없음. 경쟁 제품 사실상 없음.

```
Tauri 2.0 (Rust 셸, 번들 ~10MB)
  + React 19 + TypeScript + TailwindCSS + shadcn/ui
  + xterm.js (터미널) + Recharts (대시보드)
  -> container CLI 호출 (--format json)
```

```typescript
import { exec } from 'child_process';
import { promisify } from 'util';
const run = promisify(exec);

export async function listContainers() {
  const { stdout } = await run('container ls -a --format json');
  return JSON.parse(stdout);
}
export async function getStats() {
  const { stdout } = await run('container stats --format json');
  return JSON.parse(stdout);
}
export async function startContainer(id: string) {
  await run(`container start ${id}`);
}
```

담을 기능: 실시간 대시보드, 컨테이너/이미지/볼륨/네트워크 관리, 로그 뷰어,
내장 터미널, **compose.yml 변환기**, **DNS 세팅 마법사**, Dockerfile 마법사.

#### (2) PHP 웹 관리 패널

```php
<?php
class ContainerService {
    public function list(bool $all = true): array {
        $cmd = 'container ls' . ($all ? ' -a' : '') . ' --format json';
        exec($cmd, $out, $code);
        if ($code !== 0) throw new RuntimeException("실패: $code");
        return json_decode(implode("\n", $out), true) ?? [];
    }

    public function run(string $image, array $opts = []): string {
        $args = ['container', 'run', '-d'];
        if (!empty($opts['name']))  { $args[] = '--name'; $args[] = $opts['name']; }
        if (!empty($opts['ports'])) { foreach ($opts['ports'] as $p) { $args[]='-p'; $args[]=$p; } }
        $args[] = $image;
        $cmd = implode(' ', array_map('escapeshellarg', $args));   // 필수
        exec($cmd, $out, $code);
        return trim(implode('', $out));
    }
}
```

> 보안 경고: 사용자 입력을 셸에 넘기면 커맨드 인젝션 위험. 반드시 `escapeshellarg()` 사용,
> 화이트리스트 검증, 그리고 **127.0.0.1 에만 바인딩**할 것.

#### (3) REST API 래퍼 — React / Vue / PHP / 모바일 어디서든 붙일 수 있음

#### (4) container MCP 서버 — 애플은 스킬(지식)만 제공했고 MCP(행동)는 공백. 선점 기회.

### 정리

| 만들 것 | React | PHP | 난이도 | 시장성 |
|---|:---:|:---:|---|---|
| container 엔진 자체 | X | X | 불가능 | - |
| 데스크톱 GUI (Tauri/Electron) | O | X | 중간 | 최상 |
| 웹 관리 패널 | O | O | 쉬움 | 중 |
| REST API 래퍼 | O | O | 쉬움 | 상 |
| MCP 서버 | O | 세모 | 중간 | 최상 |
| compose 변환기 | O | O | 쉬움 | 상 |
| VSCode 확장 | O | X | 중간 | 상 |

**추천: Tauri + React 로 container Desktop.**

---

## 9. 유튜브 강의 제작 가능성

### 결론: 가능하며, 지금이 적기.

| 요인 | 내용 |
|---|---|
| 신선도 | v1.0 도달 직후. **한국어 콘텐츠 거의 없음** |
| 브랜드 | "애플이 만든 Docker 대체제" = 높은 클릭률 |
| 페인포인트 | Docker Desktop 유료화 불만 |
| 타겟 | M칩 맥 개발자 — 한국 개발자의 다수 |
| AI 결합 | "AI 스킬" 주제까지 엮으면 차별화 |
| 라이선스 | Apache-2.0 — 상업적 강의 제작 자유 |

### 리스크

| 리스크 | 대응 |
|---|---|
| macOS 26 + Apple Silicon 필수 | 제목/썸네일에 미리 명시 |
| 업데이트가 빨라 영상이 금방 낡음 | 버전 명시 + 업데이트 영상을 정기 콘텐츠화 |
| 기능 부족 (compose 없음 등) | 솔직하게 다루는 것이 오히려 신뢰 상승 |

### 트랙 A — 입문 시리즈 (8편, 각 10~15분, 무료)

| # | 제목 | 핵심 |
|---|---|---|
| 1 | Docker Desktop 지웠습니다. 애플이 대신 만들어줬거든요 | 훅 + 배경 + 설치 |
| 2 | 컨테이너 5분 만에 이해하기 | 개념 + 첫 실행 |
| 3 | Docker 명령어 그대로 쓰면 안 되는 이유 | 매핑표 + 함정 |
| 4 | 내 첫 이미지 빌드 & 배포 | build -> push |
| 5 | 컨테이너끼리 이름으로 통신하기 (3단계 DNS) | 최다 질문 구간 |
| 6 | docker-compose 가 없다고? 그럼 이렇게 하세요 | 셸 스크립트 변환 |
| 7 | 맥에서 진짜 리눅스 쓰기 — 컨테이너 머신 | 홈 디렉토리 매핑 |
| 8 | 명령어 한 줄로 쿠버네티스 클러스터 | k8s |

### 트랙 B — AI 결합 시리즈 (5편, 차별화)

| # | 제목 | 핵심 |
|---|---|---|
| 1 | 애플이 AI한테 쓴 쪽지를 발견했습니다 | `skills/` 해부 |
| 2 | Claude Code 스킬 만드는 법 — 애플 코드로 배우기 | 5가지 패턴 |
| 3 | 플러그인 vs 스킬 vs MCP, 10분 정리 | 개념 정리 |
| 4 | AI가 짠 코드, 안전하게 실행하는 법 | 샌드박스 + 보안 |
| 5 | container MCP 서버 직접 만들기 | 라이브 코딩 |

### 트랙 C — 실전 프로젝트 (유료 강의, 10~15시간)

```
Part 1. 환경 구축 + container CLI 완전정복      (2.0h)
Part 2. Tauri + React 프로젝트 셋업             (1.5h)
Part 3. CLI 래핑 레이어 + JSON 파싱             (2.0h)
Part 4. 컨테이너/이미지 대시보드 UI             (3.0h)
Part 5. 실시간 로그 + xterm.js 터미널           (2.0h)
Part 6. compose.yml 변환기 구현                 (2.0h)
Part 7. DNS 세팅 마법사                         (1.0h)
Part 8. 패키징, 코드 서명, 배포                 (1.5h)
```

완성작을 오픈소스로 공개 -> 포트폴리오 + 강의 + 스폰서십으로 연결.

### 제작 팁

1. OBS + 터미널 폰트 18pt 이상 (모바일 시청자 배려)
2. 첫 15초에 결과물부터 보여주기
3. 챕터 타임스탬프 필수 (검색 유입)
4. 설명란에 GitHub 링크 + 명령어 전문
5. 영상 초반에 요구사항 자막 (Apple Silicon + macOS 26)
6. 설치/다운로드 구간은 배속 처리
7. **실패 장면을 일부러 보여주기** (`container images` 의 오해 유발 에러) — 공감 + 교육 효과
8. Docker 와 메모리 사용량 비교 벤치마크
9. 영어 자막 추가 (영어권에도 콘텐츠가 거의 없음)

---

## 10. 수익화 아이디어 전체

### Tier S

#### S-1. container Desktop (GUI 앱)

- **문제:** CLI 만 있고 GUI 없음 / **경쟁:** 사실상 없음 / **진입장벽:** 중간

```
Free          : 컨테이너 조회/시작/중지, 로그, 기본 대시보드
Pro  $5~9/월  : compose 변환기, DNS 마법사, 다중 프로파일, 리소스 알림,
                내장 터미널, 템플릿 라이브러리
Team $20/석/월: 설정 동기화, 공유 템플릿, 감사 로그, 우선 지원
```

킬러 기능:
1. **compose.yml -> 셸 스크립트 자동 변환기** (수동 번역을 자동화)
2. **DNS 3단계 세팅 마법사** (최대 좌절 지점을 버튼 하나로)
3. 컨테이너 머신 매니저 (Lima / Colima 사용자 흡수)
4. Docker 대비 리소스 절감 리포트 (전환 설득 자료)
5. "Docker 에서 이주하기" 마법사

수익 시뮬레이션:
```
보수적: 유료 500명   x $7/월 = $3,500/월
중간  : 유료 2,000명 x $7/월 = $14,000/월
```

로드맵:
```
Week 1-2   CLI 래핑 레이어 + JSON 파싱
Week 3-4   컨테이너/이미지 리스트 UI
Week 5-6   로그 뷰어 + 터미널
Week 7-8   compose 변환기
Week 9-10  DNS 마법사
Week 11-12 코드 서명 + 배포 + 결제 연동
```

#### S-2. AI 스킬/플러그인 제작 교육

애플이 직접 쓴 고품질 스킬을 교재로 사용. AI 에이전트 교육 수요는 폭증 중.

```
1. 유튜브 무료 시리즈           -> 트래픽 + 신뢰
2. 인프런/유데미 강의 ₩99,000   -> 1,000명이면 9,900만원
3. 전자책/노션 템플릿 ₩29,000   -> 부수익 + 리드
4. 기업 사내교육 (1일)          -> 300~800만원/회
5. 스킬 제작 외주               -> 건당 300~1,500만원
6. 구독형 스킬 라이브러리 ₩19,000/월 -> 반복 수익
```

커리큘럼:
```
Ch.1  스킬 vs 플러그인 vs MCP
Ch.2  애플 SKILL.md 완전 해부
Ch.3  점진적 공개 설계법
Ch.4  AI 실패 모드를 예측해서 문서화하기
Ch.5  description 작성법 — 트리거 키워드 전략
Ch.6  references/ 분리 기준
Ch.7  사내 도구용 스킬 만들기 (실습)
Ch.8  MCP 서버 직접 구현
Ch.9  플러그인 패키징 & 마켓플레이스 배포
Ch.10 평가(eval)로 스킬 품질 측정
```

#### S-3. 기업 Docker -> container 마이그레이션 컨설팅

```
Docker Desktop Business = $24/사용자/월

맥 개발자 50명  -> 연 $14,400   (약 1,900만원)
맥 개발자 200명 -> 연 $57,600   (약 7,600만원)
맥 개발자 500명 -> 연 $144,000  (약 1억 9천만원)
```

| 패키지 | 내용 | 가격 |
|---|---|---|
| 진단 | 현황 분석, 전환 가능성 평가, ROI 리포트 | 500~1,000만원 |
| 파일럿 | 팀 1개 전환 + 검증 + 가이드 | 2,000~3,000만원 |
| 전사 전환 | 전부서 마이그레이션 + 교육 + 도구 | 5,000만원~2억 |
| 유지보수 | 기술지원 + 업데이트 대응 | 월 300~800만원 |

재사용 자산: compose 변환 CLI, CI/CD 전환 템플릿, 교육 자료,
트러블슈팅 플레이북, 사내 도구용 Claude 스킬 커스터마이징.

적합성 진단(macOS 26 필수, compose 부재, x86 맥 불가 등)을 먼저 파는 것이 핵심.

### Tier A

- **A-1. 유튜브 + 온라인 강의** — 유튜브는 수익이 아니라 **리드 수집 채널**로 볼 것
- **A-2. container MCP 서버** — 기본은 오픈소스, Pro(멀티 머신, 쿼터/정책, 팀 템플릿,
  감사 로그, 비용 추적)로 수익화
- **A-3. 맥 전용 CI/CD 서비스** — Starter $49/월, Pro $199/월, Enterprise 협의.
  경량 VM 격리로 빌드마다 깨끗한 환경 제공
- **A-4. 개발환경 템플릿 마켓플레이스** — compose 가 없으므로 "잘 만든 셸 스크립트 세트"의
  가치가 오히려 높음. 유료 팩 $19~49, 구독 $9/월

### Tier B

- VSCode / JetBrains 확장 (Free + Pro $3/월)
- 전자책 ₩29,000 / 유료 뉴스레터 ₩9,900월
- 오픈소스 -> 스폰서십: `compose2container`, `container-tui`, `container-compose`
- 컨퍼런스 발표 / 기술 블로그 (직접 수익은 작지만 고단가 리드로 전환)

---

## 11. 리스크와 대응

| 리스크 | 심각도 | 대응 |
|---|---|---|
| 애플이 공식 GUI 를 출시 | 높음 | Docker Desktop 도 서드파티와 공존. **compose 변환, 팀 기능 등 애플이 하지 않을 영역**에 집중 |
| 플랫폼 제한 (M칩 + macOS 26) | 중간 | 타겟을 "맥 개발자"로 좁혀 니치 지배 |
| 빠른 버전 변화 | 중간 | 버전 명시 + 업데이트를 정기 콘텐츠로 (오히려 구독 유지 효과) |
| 기능 부족 (compose 등) | 낮음 | 그것이 곧 제품의 존재 이유 |
| 시장 규모 불확실 | 중간 | 1단계 반응을 보고 투자 결정 (린 접근) |

---

## 12. 최종 추천 전략

### 3단 로켓

```
1단계 (0~3개월)  인지도 구축
  - 유튜브 트랙 A 8편 공개 (무료)
  - 기술 블로그 연재 (한글 + 영어)
  - compose2container 오픈소스 공개
  목표: 구독 5,000 / GitHub 스타 500
  투자: 시간. 금전 비용 거의 0

2단계 (3~9개월)  제품화
  - container Desktop MVP 출시 (Tauri + React)
  - 인프런 강의 런칭 (트랙 A + B)
  - AI 스킬 작성법 전자책
  목표: 월 500~1,500만원

3단계 (9~18개월) 고수익 전환
  - 기업 컨설팅 / 마이그레이션 수주
  - Desktop Pro/Team 구독 안정화
  - 기업 사내교육 정기 계약
  목표: 월 3,000만원+
```

### 하나만 고른다면

> **`compose2container` 오픈소스 CLI -> container Desktop 유료화**

이유:
1. 가장 명확한 페인포인트 (compose 부재 = 전환 최대 장벽)
2. 진입장벽이 낮음 (Node/TS 로 2주면 MVP)
3. 오픈소스 -> 신뢰 -> 유료 제품의 교과서적 경로
4. GUI 제품의 킬러 기능으로 자연스럽게 확장
5. 컨설팅 영업 자산으로도 그대로 사용
6. 실패해도 포트폴리오 + 콘텐츠 소재로 남아 리스크가 최소

```
$ npx compose2container ./docker-compose.yml -o ./up.sh
3 services converted
'restart: always' has no equivalent — launchd snippet generated
depends_on -> readiness loops added
service names -> db.test, cache.test
Written to ./up.sh
```

---

## 참고 링크

| 항목 | URL |
|---|---|
| 이 레포지토리 | https://github.com/bmshin94/container |
| 원본 프로젝트 | https://github.com/apple/container |
| 릴리스 (설치 파일) | https://github.com/apple/container/releases |
| Containerization 패키지 | https://github.com/apple/containerization |
| API 문서 | https://apple.github.io/container/documentation/ |
| 커스텀 네트워크 DNS 이슈 | https://github.com/apple/container/issues/1809 |
| 기여 가이드 | https://github.com/apple/containerization/blob/main/CONTRIBUTING.md |

### 레포 내부 문서

| 파일 | 내용 |
|---|---|
| `README.md` | 개요, 설치, 업그레이드 |
| `BUILDING.md` | 소스 빌드 방법 |
| `docs/how-to.md` | 전체 How-to 목차 |
| `docs/command-reference.md` | 전체 명령어 레퍼런스 |
| `docs/technical-overview.md` | 아키텍처 및 설계 |
| `docs/networking.md` | 네트워킹 / DNS |
| `docs/container-machine.md` | 컨테이너 머신 |
| `docs/kubernetes.md` | 로컬 쿠버네티스 |
| `docs/container-system-config.md` | `config.toml` 전체 레퍼런스 |
| `skills/container/SKILL.md` | Claude Code 스킬 본문 |
| `skills/container/references/docker-migration.md` | Docker 마이그레이션 전체 매핑 |
| `examples/container-machine-vscode/README.md` | VSCode 연동 예제 |

---

*문서 작성: Claude Code 세션 (2026-10-01)*
