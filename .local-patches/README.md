# 로컬 패치 — macOS 기본 fd 한도(256)에서 zellij 가 탭 ~20개에 죽는 문제

기준: `v0.45.0` / 업스트림 이슈 https://github.com/zellij-org/zellij/issues/5314

## 문제

zellij 서버는 **탭당 fd 를 약 10개** 쓴다(실측). status-bar / tab-bar 같은 WASM
플러그인이 탭마다 인스턴스로 뜨고 인스턴스마다 WASI preopen 디렉토리를 4개씩 잡기
때문이다. 붙은 클라이언트가 늘면 그만큼 더 든다.

macOS launchd 는 데몬에 soft `RLIMIT_NOFILE` **256** 을 준다(hard 는 unlimited).
`(256 - 26) / 10 ≈ 22` 탭에서 EMFILE 이 나고 서버가 죽는다.

실측(패치본, 탭당 소비율):

| 탭 | fd | 구간 소비율 |
|---:|---:|---:|
| 1 | 36 | — |
| 10 | 136 | 11.11 |
| 25 | 287 | 10.07 |
| 50 | 537 | 10.00 |
| 80 | 836 | 9.97 |

완전히 선형 = 누수 없음. 80탭 시점 fd 836개 중 649개가 DIR(플러그인 preopen).

## 패치 2개

| 파일 | 내용 |
|---|---|
| `zellij-server/src/lib.rs` | `raise_nofile_limit()` — 서버 기동 시 soft 를 `min(16384, hard)` 까지 올린다. 실패하면 절반씩 낮춰 재시도. `start_server_impl()` 첫 줄에서 호출(**daemonize 이후**라야 데몬 프로세스에 적용된다). |
| `zellij-server/src/os_input_output_unix.rs` | `handle_openpty` 에서 `into_raw_fd()` 를 `command_exists` 검사 **뒤로** 옮김. 앞에 있으면 존재하지 않는 명령으로 페인을 열 때마다 pty fd 2개가 영구 누수된다(`OwnedFd` 의 Drop 이 못 돈다). |

16384 는 이 맥 `kern.maxfilesperproc`(122880)의 13% 이고 탭 약 1600개 분량이다.
65536 은 `close_fds` 폴백 비용(측정: 6.7ms/spawn)과 머신 fd 테이블 점유가 커서 낮췄다.

## ⚠ 의도적으로 하지 않은 것

fd 가 **진짜** 고갈되면 zellij 는 그냥 죽는다. 그게 맞다.
한때 EMFILE 에서 패닉하는 모든 경로(`spawn`/`openpty`/IPC accept/`Signals::new` 등)를
비치명적으로 바꾸는 283줄짜리 버전을 만들었다가 **전부 폐기**했다 — 그 코드가 오히려
① 빈 탭 → 마지막 탭 닫힘 → **세션 조용히 종료**
② 리스너 영구 정지 → **재접속 불가 좀비 세션**
③ 보이지도 닫히지도 않는 유령 페인
을 만들었다. 목표는 **한도를 제대로 잡는 것**이지 고갈 상태에서 연명시키는 게 아니다.

## 빌드 · 배포

```sh
brew install protobuf              # protoc
rustup target add wasm32-wasip1
cargo xtask install ~/bin/zellij
codesign -f -s - ~/bin/zellij      # ⚠ 필수
```

⚠ **`codesign` 을 빼먹으면 `rc=137`(SIGKILL)로 아무 출력 없이 죽는다.** macOS 는
ad-hoc 서명 바이너리를 제자리에서 덮어쓰면 커널의 서명 캐시를 무효화한다
(`codesign -v` 는 통과하는데도).

⚠ 빌드하면 `zellij-utils/assets/plugins/*.wasm` 13개가 재생성되어 변경으로 잡힌다.
바이트만 다른 산출물이니 **커밋하지 말 것**: `git checkout -- zellij-utils/assets/plugins/`

⚠ `cargo check --workspace --all-targets` 는 debug 프로파일 wasm 플러그인을 요구한다
(`zellij-utils/src/consts.rs` 의 `include_bytes!`). 먼저 `cargo xtask build` 를 돌려야 한다.

## 원격 · 새 버전으로 옮기기

| remote | 대상 |
|---|---|
| `origin` | `zellij-org/zellij` (업스트림, 태그 출처) |
| `fork` | `ESPINS/zellij` (개인 백업) |

```sh
git fetch origin --tags
git checkout patch/fd-limits
git rebase <새태그>
cargo xtask install ~/bin/zellij && codesign -f -s - ~/bin/zellij
git checkout -- zellij-utils/assets/plugins/
git push -f fork patch/fd-limits
```

`fd-limits.patch` 는 rebase 가 불가능할 때의 보조 사본이다.
소스를 고치면 `git diff <기준태그> -- zellij-server/src > .local-patches/fd-limits.patch` 로 갱신할 것.

## 검증법

```sh
# 1) 한도가 실제로 올라가는가 — soft 만 낮추고 hard 는 그대로 두어야 한다
zsh -c 'ulimit -Sn 256; zellij attach --create-background fdcheck'
grep RLIMIT_NOFILE "$TMPDIR/zellij-$(id -u)/zellij-log/zellij.log"
#   → "raised RLIMIT_NOFILE soft limit from 256 to 16384"

# 2) 소비율 — 탭을 늘리며 재고, 선형이면 누수 없음
P=$(pgrep -f 'zellij --server.*fdcheck'); lsof -p $P | wc -l
```

⚠ soft·hard 를 **둘 다** 낮추면 `raise_nofile_limit` 이 조기 반환해서 아무것도
검증되지 않는다(한 번 당했다).
