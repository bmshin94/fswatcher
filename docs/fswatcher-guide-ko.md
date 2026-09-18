# fswatcher 완전 분석 가이드 (한국어)

> 이 문서는 fswatcher 저장소를 처음 접한 개발자를 위해, 저장소 전체를 직접 읽고
> 정리한 분석 노트입니다. "이게 뭔지 / 언제 쓰는지 / 어떻게 쓰는지 / 어디에
> 활용할 수 있는지"를 한 번에 파악할 수 있도록 구성했습니다.

## 📌 관련 링크

| 구분 | 주소 |
|------|------|
| 이 저장소 | https://github.com/bmshin94/fswatcher |
| 업스트림 원본 | https://github.com/fswatcher/fswatcher |
| Go 패키지 문서 | https://pkg.go.dev/github.com/fswatcher/fswatcher |
| 이전 경로 (deprecated) | `github.com/gofsnotify/fsnotify` → 현재 경로로 이전됨 (#27) |
| 참고: 원조 라이브러리 | https://github.com/fsnotify/fsnotify |
| purego (macOS FFI) | https://github.com/ebitengine/purego |

---

## 1. 한 줄 정의

**파일/폴더가 바뀌면 즉시 알려주는 Go용 크로스 플랫폼 파일시스템 감시 라이브러리 + CLI 도구.**

- 라이선스: **MIT**
- 모듈 경로: `github.com/fswatcher/fswatcher`
- Go 버전: `1.25.0`
- 현재 버전: `v0.1.0` (+ Unreleased 변경분)

---

## 2. 왜 필요한가

파일 변경을 감지하는 두 가지 방법:

| 방식 | 동작 | 문제점 |
|------|------|--------|
| 폴링(polling) | 주기적으로 디렉터리를 반복 스캔 | CPU 낭비, 감지 지연 |
| OS 네이티브 알림 | 커널이 변경 발생 시 직접 통지 | **OS마다 API가 전부 다름** |

fswatcher는 두 번째 방식을 쓰되, OS별 차이를 하나의 동일한 Go API로 흡수합니다.

| OS | 백엔드 | 특징 |
|----|--------|------|
| Linux | `inotify` | 디렉터리 단위 등록. 하위 디렉터리는 자동 감시되지 않아 직접 walk 필요 |
| Windows | `ReadDirectoryChangesW` (IOCP) | `bWatchSubtree`로 하위 트리 자동 감시 |
| macOS | `FSEvents` (purego 경유, **cgo 불필요**) | 볼륨 단위 감시, 재귀 기본 지원, fd 소비 없음 |
| FreeBSD / OpenBSD / NetBSD | `kqueue` | 감시 대상마다 fd 1개 → fd 예산 관리 필요 |
| 그 외 | 스텁 | `ErrUnsupported` 반환 |

---

## 3. 저장소 구조 (실측)

총 **3,511줄**의 Go 코드.

| 파일 | 줄 수 | 역할 |
|------|------:|------|
| `fswatcher.go` | 104 | 공용 API — `Op`, `Event`, 센티널 에러 |
| `watcher_linux.go` | 397 | inotify 백엔드 |
| `watcher_windows.go` | 373 | ReadDirectoryChangesW 백엔드 |
| `watcher_darwin.go` | 571 | FSEvents 백엔드 |
| `watcher_bsd.go` | 397 | kqueue 공통 백엔드 |
| `watcher_freebsd.go` / `watcher_otherbsd.go` | 99 / 105 | BSD 계열 분기 |
| `watcher_other.go` | 26 | 미지원 플랫폼 스텁 |
| `path.go` / `path_darwin.go` / `path_windows.go` / `path_other.go` | 31 / 23 / 54 / 13 | 경로 정규화 |
| `cmd/fswatcher/main.go` | 162 | CLI 실행 파일 |
| `watcher_test.go` 외 | 900+ | 통합 테스트 34개 + 벤치마크 3개 |

문서 파일:

- `README.md` — 설치/사용법/API/플랫폼 지원표
- `DESIGN.md` — **설계 의사결정 근거** (왜 이렇게 만들었는지)
- `CHANGELOG.md` — Keep a Changelog 규격
- `AGENTS.md` — 커밋/이슈/PR 컨벤션 (영어 작성, `Co-Authored-By` 트레일러 필수 등)
- `CLAUDE.md` — AI 코딩 에이전트용 페르소나 지침

---

## 4. 공개 API

```go
w, err := fswatcher.NewWatcher()
```

| 함수 | 설명 |
|------|------|
| `NewWatcher() (*Watcher, error)` | 감시자 생성 |
| `(*Watcher).Add(path, op) error` | 경로 1개 등록. 중복 시 `ErrAlreadyAdded` |
| `(*Watcher).AddRecursive(path, op) error` | 경로 + 하위 전체 등록. 새 하위 디렉터리 자동 추가, 삭제 시 자동 해제 |
| `(*Watcher).Remove(path) error` | 등록 해제. 재귀 등록은 루트에서만 가능 (하위는 `ErrNotAdded`) |
| `(*Watcher).Close() error` | 종료 및 채널 닫기 |
| `(*Watcher).Events <-chan Event` | 변경 알림 (버퍼 64) |
| `(*Watcher).Errors <-chan error` | 비치명적 에러 (버퍼 8) |
| `Canonicalize(path) (string, error)` | 내부에서 쓰는 정규화된 경로 반환 |

### 이벤트 종류 (`Op` 비트마스크)

| Op | 의미 |
|----|------|
| `Create` | 파일/디렉터리 생성 |
| `Write` | 파일 내용 수정 |
| `Remove` | 파일/디렉터리 삭제 |
| `Rename` | 이름 변경 또는 이동 |
| `Chmod` | 권한/속성 변경 |
| `All` | 위 전체의 합집합 |

`op.Has(target)`는 **"target 비트 중 하나라도 포함하는가"**를 반환합니다 (v0.0.5에서 의미 변경, #21).
따라서 `ev.Op.Has(Create | Write)`는 "Create **또는** Write"로 읽힙니다.

### 센티널 에러

`ErrAlreadyAdded` / `ErrNotAdded` / `ErrClosed` / `ErrUnsupported`

---

## 5. 설치 및 사용법

### 5.1 라이브러리로 사용

```bash
go get github.com/fswatcher/fswatcher
```

```go
package main

import (
    "log"

    "github.com/fswatcher/fswatcher"
)

func main() {
    w, err := fswatcher.NewWatcher()
    if err != nil {
        log.Fatal(err)
    }
    defer w.Close()

    if err := w.AddRecursive("./src", fswatcher.Create|fswatcher.Write|fswatcher.Remove); err != nil {
        log.Fatal(err)
    }

    for {
        select {
        case ev, ok := <-w.Events:
            if !ok {
                return // Close()로 채널이 닫힘
            }
            if ev.Op.Has(fswatcher.Write) {
                log.Println("수정됨:", ev.Name)
            }
        case err, ok := <-w.Errors:
            if !ok {
                return
            }
            log.Println("error:", err)
        }
    }
}
```

### 5.2 CLI로 사용

```bash
go install github.com/fswatcher/fswatcher/cmd/fswatcher@latest
```

```bash
fswatcher ./src                            # 기본: "OP<TAB>PATH" 출력
fswatcher -r ./src                         # 재귀 감시
fswatcher -r -e 'go test ./...' ./src      # 변경 시 명령 실행
fswatcher -r -json ./src | jq -r .path     # NDJSON 스트림
```

| 플래그 | 의미 |
|--------|------|
| `-r` | 재귀 감시 |
| `-V` | 상세 로그(stderr) |
| `-e CMD` | 이벤트마다 셸 실행 (`sh -c`, Windows는 `cmd /C`) |
| `-json` | NDJSON 출력. `-e`와 동시 사용 불가 |

`-e`로 실행되는 자식 프로세스에는 `FSWATCHER_PATH`, `FSWATCHER_OP` 환경변수가 주입됩니다.

### 5.3 소스에서 직접 검증

```bash
git clone https://github.com/bmshin94/fswatcher
cd fswatcher
go vet ./...
go test -race ./...
go test -bench=. -benchmem
```

---

## 6. 실전 사용처

1. 핫 리로드 / 라이브 개발 서버
2. 빌드 워처 (SCSS/TS 저장 시 자동 컴파일)
3. 테스트 자동 실행
4. 파일 동기화 / 실시간 백업
5. 로그 파일 감시 및 파싱
6. 보안 감사 — 중요 디렉터리 변조 탐지
7. 설정 파일 변경 시 무중단 리로드
8. AI 에이전트 워크스페이스 변경 추적

---

## 7. 설계 포인트 (DESIGN.md 요약)

### 7.1 경로 정규화

`Add`/`AddRecursive`/`Remove`에 들어오는 모든 경로는 동일한 파이프라인을 거칩니다.

- `filepath.Abs` + `filepath.Clean` — 상대 경로와 `.` / `..` 정리
- `filepath.EvalSymlinks` — 대상이 존재하면 심볼릭 링크 해석
- Windows — `GetLongPathName`으로 8.3 단축형 확장(`C:\PROGRA~1` → `C:\Program Files`) + 맵 키 소문자 폴딩
- macOS — APFS 특성에 맞춰 대소문자 및 유니코드 정규화 형태 차이를 무시하고 매칭 (#28)

덕분에 같은 위치를 가리키는 서로 다른 표기가 중복 등록되지 않고, `Event.Name`이 항상 일관된 형태로 옵니다.

### 7.2 재귀를 별도 메서드로 분리한 이유

옵션이나 가변 인자가 아니라 `AddRecursive`라는 별도 메서드로 둔 것은 의도된 설계입니다.
재귀 감시는 하위 디렉터리 생명주기, walk와 이벤트 사이의 경합, fd 예산 문제로 버그가 나기 쉬워서
**호출부가 명시적으로 선택**하게 만들었습니다.

### 7.3 플랫폼 차이 보정 사례 (CHANGELOG 발췌)

- `mkdir -p a/b/c` 처럼 아직 감시 대상이 아니던 중첩 디렉터리에 대해 커널이 `Create`를 주지 않는 문제 → 워처가 직접 합성 (#29, #30)
- Windows에서 감시 루트가 삭제/연결 해제될 때 핸들 누수 → 엔트리 정리 및 다른 백엔드와 동일하게 `Remove` 이벤트 발행 (#19)
- BSD 계열에서 시그널로 중단된 syscall(`EINTR`) → 재시도 처리 (#50)
- `Events` / `Errors`를 수신 전용(`<-chan`)으로 변경 (#26)
- `Close` 시 `close(w.error)`와 `w.error <- err` 사이의 레이스 수정 (#26)

### 7.4 테스트 정책

- mock 없이 **실제 파일시스템 이벤트**로 통합 테스트
- CI는 Linux / macOS / Windows + FreeBSD / OpenBSD / NetBSD(VM)에서 실행
- `-race`는 항상 켬. 특정 백엔드가 불안정하면 `-race`를 빼는 대신 **타임아웃을 늘림**
  (OpenBSD/amd64는 `-race` 미지원이라 예외)

---

## 8. 자주 하는 오해 정리

| 질문 | 답 |
|------|-----|
| 플러그인인가? | 아니다. 호스트 앱에 붙는 확장이 아니라 독립 라이브러리/CLI다. |
| 스킬(Skill)인가? | 아니다. 저장소 안의 `CLAUDE.md`/`AGENTS.md`는 이 저장소에서 작업하는 에이전트를 위한 규칙 문서일 뿐, 라이브러리 기능이 아니다. |
| MCP 서버인가? | 아니다. 다만 **이걸 감싸서 MCP 서버를 만들 수는 있다**(9장 참고). |
| API 토큰이 필요한가? | **필요 없다.** 네트워크 통신, 인증, 계정, 클라우드 전송이 전혀 없다. 전부 로컬 syscall이다. |
| 비용이 드는가? | 0원. 폐쇄망/에어갭 환경에서도 동작한다. |

의존성은 세 개뿐입니다.

```
github.com/ebitengine/purego v0.10.1   # macOS FSEvents를 cgo 없이 호출
golang.org/x/sys            v0.46.0   # OS syscall 바인딩
golang.org/x/text           v0.39.0   # 유니코드 정규화(macOS 파일명)
```

---

## 9. AI / 로컬 에이전트 활용

로컬 에이전트의 약점은 "무엇이 바뀌었는지 모른다"는 점입니다. 매번 전체를 다시 읽으면
느리고 컨텍스트도 낭비됩니다. fswatcher는 이를 **변경분 이벤트 스트림**으로 바꿔줍니다.

### 활용 패턴

1. **증분 RAG 인덱싱** — 변경된 파일만 재임베딩. 전체 재인덱싱 대비 비용/시간 대폭 절감
2. **에이전트 작업 감사 로그** — 에이전트의 보고와 무관하게 실제 파일 변경을 독립 기록
3. **자동 검증 루프** — 코드 수정 감지 → 테스트 자동 실행 → 실패 로그를 다시 컨텍스트로 피드백
4. **세션 컨텍스트 주입** — "마지막 대화 이후 바뀐 파일"만 프롬프트에 포함
5. **멀티 에이전트 충돌 감지** — 동일 파일 동시 수정 경고

### MCP 서버로 감쌀 때의 툴 설계 예

```
watch_start(path, recursive)  # 감시 시작
get_changes(since)            # 마지막 조회 이후 변경 목록
watch_stop(path)              # 감시 중지
```

### 구현 시 주의할 점

| 함정 | 대응 |
|------|------|
| `node_modules`, `.git` 감시 → 이벤트 폭주 | ignore 리스트 필수. 비교 전에 `Canonicalize()`로 경로 형태를 맞출 것 |
| 에디터 저장 1회에 이벤트 여러 개 (임시파일 → rename 패턴) | 200~500ms 디바운스 |
| `Events` 버퍼(64) 초과 시 유실 가능 | 수신 즉시 내부 큐에 적재하고 처리는 별도 고루틴 |
| Linux inotify watch 개수 한계 | `/proc/sys/fs/inotify/max_user_watches` 확인 (기본 8192) |
| BSD는 대상당 fd 1개 | `ulimit -n` 확인 |

---

## 10. 다른 언어/스택으로 만들 수 있는가

| 스택 | 가능 여부 | 비고 |
|------|----------|------|
| React (브라우저) | ❌ | 브라우저 샌드박스 제약. File System Access API도 이벤트 푸시가 없어 폴링해야 함 |
| React + Electron / Tauri | ✅ | 감시는 네이티브 계층(Node `chokidar` / Rust `notify`)이 담당하고 React는 UI만 |
| PHP (웹서버) | ❌ | 요청-응답 후 프로세스 종료 모델이라 장시간 상주 감시에 부적합 |
| PHP CLI + PECL `inotify` | ⚠️ | 리눅스 전용. 크로스 플랫폼 포기 |
| Node.js | ✅ | `chokidar`. 대형 트리에서 폴링 폴백으로 CPU 부담이 생길 수 있음 |
| Python | ✅ | `watchdog` |
| Rust | ✅ | `notify` 크레이트 |
| Go | ✅ 최적 | 단일 바이너리, 고루틴/채널이 이벤트 스트림과 잘 맞음, 크로스 컴파일 용이 |

### 권장 하이브리드 구조

```
React (UI: 대시보드 / 설정 화면)
        ↕ WebSocket / SSE / HTTP
Go 백엔드 (fswatcher 내장, 이벤트 푸시)
```

Go의 `embed` 패키지로 React 빌드 산출물을 바이너리에 포함시키면
**설치도 런타임도 필요 없는 실행 파일 하나(약 10~20MB)** 로 배포할 수 있습니다.

```go
//go:embed dist/*
var uiFiles embed.FS
```

PHP에 익숙하다면 **감시는 Go 에이전트, 관리 콘솔/과금은 PHP(Laravel)** 로 역할을 나누면 됩니다.

---

## 11. 수익화 아이디어

MIT 라이선스이므로 **상업적 이용, 수정, 재배포, 클로즈드소스화가 모두 허용**됩니다.
조건은 저작권 표시와 라이선스 전문 포함뿐입니다.

### ① AI 에이전트용 파일 감시 MCP 서버

| 항목 | 내용 |
|------|------|
| 타깃 | Claude Code / Cursor / Windsurf 사용자 |
| 모델 | 무료 OSS + Pro 구독(월 $9 수준) |
| 난이도 | 중 |
| 수익성 | 중상 |
| 타이밍 | **가장 좋음** |

유료 차별화 포인트: `.gitignore` 기반 스마트 필터, 변경을 의미 단위로 요약,
스냅샷/롤백, 멀티 에이전트 충돌 조정.

### ② 파일 무결성 모니터링(FIM) SaaS

| 항목 | 내용 |
|------|------|
| 타깃 | PCI-DSS, ISO 27001, ISMS-P 인증이 필요한 기업 |
| 모델 | 서버당 월 $15~30 B2B 구독 |
| 난이도 | 중상 |
| 수익성 | **최상** |

PCI-DSS 요구사항 11.5가 FIM을 사실상 강제하므로 "필수 구매" 성격이 있습니다.
기존 솔루션(Tripwire, OSSEC 등)은 비싸거나 설정이 복잡합니다.

구조 예: `Go 에이전트(고객 서버)` → `PHP/Laravel 백엔드` → `React 대시보드 + Slack/이메일 알림`.
핵심 지불 포인트는 **감사 보고서 자동 생성**입니다.

### ③ 개발자/크리에이터용 GUI 워처 앱

| 항목 | 내용 |
|------|------|
| 타깃 | 사진가, 영상 편집자, 비개발자 파워유저 |
| 모델 | $29 평생 라이선스 |
| 난이도 | **낮음** |
| 수익성 | 중 |

"폴더에 조건이 맞는 파일이 들어오면 자동으로 동작 실행"을 코딩 없이 GUI로 제공.
Go + React + `embed`로 단일 실행 파일(약 20MB) — Electron 대비 경량성이 그 자체로 셀링 포인트.

### ④ 셀프호스팅 실시간 동기화 도구

Syncthing/Resilio 경쟁군. 동기화 충돌 해결 난이도가 매우 높아 **난이도 대비 수익성이 낮음**.
우선순위는 가장 낮게 두는 것을 권합니다.

### ⑤ 교육 콘텐츠

저장소 자체가 교재가 됩니다(3,500줄, 강의 8~10시간 분량).

1. inotify로 리눅스 워처 만들기
2. Windows IOCP + `ReadDirectoryChangesW`
3. **purego로 cgo 없이 macOS FSEvents 호출** — 한국어 자료가 거의 없는 영역
4. kqueue와 fd 예산 관리
5. 크로스 플랫폼 추상화 설계
6. race 감지 통합 테스트와 멀티 OS CI

### 권장 실행 순서

```
1단계 (1~2개월)  ①을 OSS로 출시 → 스타/피드백 확보
2단계 (3~4개월)  ①의 Pro 버전 출시 (React 대시보드 추가)
3단계 (6개월~)   ②로 B2B 진입 (①의 Go 에이전트 코드 70% 이상 재활용)
병행             ⑤ 교육 콘텐츠 — 개발 과정의 부산물로 제작
```

①→②는 코어 코드를 공유하고, OSS로 쌓은 신뢰가 B2B 영업 자산이 됩니다.
API 비용이 0원이라 한계원가가 사실상 없다는 점도 구독 모델에 유리합니다.

---

## 12. 요약

- fswatcher는 **OS 네이티브 파일 알림 API를 하나의 Go API로 통합한 라이브러리 + CLI**다.
- 플러그인도, 스킬도, MCP도 아니다. **API 토큰도 비용도 필요 없다.**
- macOS를 **cgo 없이(purego)** 지원하고, **BSD 3종을 실제 VM CI로 검증**하며,
  **`AddRecursive`로 재귀 감시를 내장**한 점이 차별점이다.
- 로컬 AI 에이전트에게는 "무엇이 바뀌었는가"를 알려주는 **센서** 역할을 한다.
- 브라우저 React나 웹 PHP로는 대체할 수 없고, **Go 코어 + React UI 하이브리드**가 현실적이다.
- 수익화는 **MCP 서버 → Pro 구독 → FIM SaaS** 순서가 코드 재활용과 신뢰 축적 면에서 유리하다.
