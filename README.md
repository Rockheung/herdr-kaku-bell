# herdr-kaku-bell

에이전트가 손을 기다릴 때 [kaku](https://github.com/tw93/Kaku) 탭에 점을 켠다.

**[herdr](https://github.com/herdrdev/herdr) 플러그인이다.** kaku 에는 아무것도 설치하지
않는다. 다만 표식을 그리는 쪽이 kaku 이므로, kaku 를 바깥 터미널로 쓸 때만 의미가 있다.

## 무엇이 필요한가

| | | |
|---|---|---|
| herdr | 0.8.0 이상 | 이 플러그인을 설치하는 곳 |
| kaku | — | 표식을 그리는 곳. 바깥 터미널이어야 한다 |
| macOS | — | `ps` 출력과 `/dev/ttysNNN` 쓰기에 기댄다 |

kaku 쪽은 설치가 아니라 설정 두 가지를 확인하면 된다. 자세한 내용은 아래
[kaku 쪽 권장 설정](#kaku-쪽-권장-설정)에 있다.

- `bell_tab_indicator` — 탭 점. **기본으로 켜져 있다.** 명시적으로 `false` 로 두지
  않았다면 손댈 것이 없다.
- `bell_dock_badge` — Dock 배지. **기본으로 꺼져 있다.** kaku 를 다른 앱 뒤에 두고
  일한다면 켜는 편이 낫다.

herdr 의 알림까지 함께 쓰려면 [herdr 쪽 권장 설정](#herdr-쪽-권장-설정)도 본다. kaku 에서는
설정 한 줄로 끝나지 않고 herdr 에게 터미널 정체를 다르게 알려야 한다.

## 왜 필요한가

kaku 는 BEL 을 받은 탭에 주황 점을 그리고, 그 탭을 열면 지운다. 완료 신호로 쓰라고
만들어진 기능이다.

그런데 herdr 는 에이전트가 `blocked` 나 `done` 으로 바뀔 때 알림(toast)만 보내고 BEL 은
쏘지 않는다. herdr 가 바깥 터미널로 내보내는 `TerminalBell` 은 pane 안 프로그램이 실제로
BEL 을 출력했을 때만 생기고, 그것도 포커스된 pane 의 것만
나간다([herdrdev/herdr#3095](https://github.com/herdrdev/herdr/issues/3095)). 정작 표식이
필요한 배경 탭은 조용하다.

`herdr --remote` 로 여러 서버에 붙어 있으면 이 문제가 커진다. 어느 서버의 에이전트가
멈춰 서서 답을 기다리는지 탭만 봐서는 알 수 없다.

이 플러그인은 밖에서 BEL 을 쓴다. 각 herdr 클라이언트가 점유한 tty 를 `ps` 로 찾아
`\a` 를 직접 써 넣는다.

## 설치

```sh
herdr plugin link /path/to/herdr-kaku-bell
```

`[[startup]]` 훅이 herdr 서버 기동 시 감시자를 띄운다. 서버를 재시작하지 않고 지금
바로 쓰려면 직접 실행한다.

```sh
bin/kaku-bell watch --daemon
```

## 구성

- `bin/kaku-bell ring [--local|<ssh-target>]` — 해당 herdr 세션이 붙어 있는 kaku 탭에
  BEL 을 쓴다.
- `bin/kaku-bell watch [--daemon]` — 로컬과 원격 세션을 지켜보다가 `blocked`·`done`
  이 **새로** 생기면 그 탭에 BEL 을 보낸다.

설계에서 신경 쓴 것들이다.

- **대상 판별**: 자식 `ssh` 프로세스도 같은 tty 를 쓰고 명령줄에 `ssh://target` 을
  담고 있다. `argv[0]` 이 `herdr` 이고 `argv[1]` 이 `--remote` 인 것만 센다.
- **JSON**: `agent list` 응답을 파서로 읽는다. 에이전트 제목에는 사용자가 친 프롬프트가
  들어가므로 중괄호나 따옴표를 텍스트로 긁으면 깨진다.
- **조회 실패**: "아무것도 없음"으로 취급하지 않는다. 상태를 그대로 두어서, 네트워크가
  끊겼다 붙을 때 가짜 알림이 나가지 않는다.
- **첫 주기**: 상태만 기록하고 울리지 않는다. 감시자를 켤 때마다 이미 멈춰 있던
  에이전트들이 한꺼번에 울리지 않는다.
- **병렬 조회**: 세션마다 스레드를 쓴다. 한 대가 타임아웃에 걸려도 주기가 밀리지 않는다.
- 대상 목록은 매 주기 `ps` 에서 다시 만든다. 재연결로 tty 가 바뀌거나 세션이 늘고
  줄어도 따라간다. 원격 조회는 ssh `ControlMaster` 로 연결을 재사용한다.

## kaku 쪽 권장 설정

탭 점은 `bell_tab_indicator` 로 기본 활성이라 따로 켤 것이 없다. 다만 **Dock 배지는
기본이 꺼져 있다.** kaku 를 다른 앱 뒤로 보내 두었을 때도 몇 건이 밀려 있는지 보려면
`~/.config/kaku/kaku.lua` 에 한 줄을 넣는다.

```lua
config.bell_dock_badge = true
```

kaku 문서에는 나오지 않는 설정이라 모르고 지나치기 쉽다. 배지는 kaku 창이 포커스를
받거나 탭을 전환하면 지워진다.

## herdr 쪽 권장 설정

이 플러그인은 탭 표식만 담당한다. 표식은 어느 세션인지까지만 알려주므로, 무슨 일인지
알려면 herdr 의 알림을 함께 켠다. `blocked` 는 `claude needs attention`, `done` 은
`claude finished` 로 구분해서 띄운다.

```toml
[ui.toast]
delivery = "terminal"
```

**kaku 에서는 여기에 한 가지가 더 필요하다.** herdr 는 `TERM_PROGRAM` 과 `TERM` 으로
바깥 터미널을 판별해 알림 시퀀스를 고르는데, kaku 는 WezTerm 포크이면서도 자기 이름으로
정체를 알리기 때문에(`TERM_PROGRAM=Kaku`, `TERM=xterm-256color`) 어느 갈래에도 걸리지
않는다. 그러면 herdr 는 아무 시퀀스도 내보내지 않으면서 `shown: true` 를 돌려준다.
응답만 보고 동작한다고 판단하면 안 된다.
([herdrdev/herdr#2513](https://github.com/herdrdev/herdr/issues/2513))

kaku 는 OSC 9 를 읽으므로, herdr 에게 WezTerm 계열이라고 알리면 실제 능력과 일치한다.
herdr 를 띄울 때만 걸어서 영향을 그 프로세스에 가둔다.

```sh
# ~/.zshrc.local
herdr() { TERM_PROGRAM=WezTerm command herdr "$@"; }
```

환경변수는 프로세스가 시작할 때 읽히므로, 이미 떠 있는 클라이언트는 다시 띄워야 한다.
`herdr --remote` 로 여러 서버에 붙어 있다면 탭마다 해야 한다.

이 방법을 쓰면 알림이 kaku 아이콘으로 뜨는 대신, OSC 9 규격상 제목과 본문이 한 줄로
합쳐진다. 알림 제목 자리에는 `Kaku` 가 들어가고 `claude needs attention: ~ · 1` 이
본문이 된다. 제목에서 상태를 바로 읽고 싶으면 `delivery = "system"` 을 쓴다 — 그쪽은
`osascript` 를 거치므로 알림이 "스크립트 편집기" 이름으로 뜬다.

## 설정

| 환경변수 | 기본값 | 뜻 |
|---|---|---|
| `HERDR_KAKU_BELL_INTERVAL` | `5` | 폴링 주기(초) |

상태와 로그는 `/tmp/herdr-kaku-bell/` 에 있다. `watch.log` 에 ssh 오류가 쌓인다.

## 한계

- 폴링이다. herdr 0.8.2 는 플러그인 이벤트 훅(`pane.agent_status_changed`)을 실제로
  호출하지 않는다 — 매니페스트에 선언은 해 두었으니, 훅이 동작하는 버전에서는 그쪽이
  먼저 반응한다.
- 여러 herdr 세션(`--session`)은 지원하지 않는다. 기본 세션과 `--remote` 만 센다.
- `done` 은 "안 본 배경 작업이 끝난 상태"라, 탭을 열면 herdr 쪽에서 `idle` 로 바뀐다.
  감시 주기와 겹치면 한 박자 늦게 울릴 수 있다.
- macOS 전용이다. `ps -axo tty=` 출력과 `/dev/ttysNNN` 쓰기에 기댄다.
- kaku 외의 터미널에서는 탭 표식이 뜨지 않는다. BEL 을 어떻게 다루는지는 터미널마다
  다르다. WezTerm 계열이면 비슷하게 동작할 여지가 있으나 확인하지 않았다.
