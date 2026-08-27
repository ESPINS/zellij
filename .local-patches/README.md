# 로컬 패치 — file descriptor 고갈로 세션이 통째로 죽는 문제

기준: `v0.45.0` / 업스트림 이슈 https://github.com/zellij-org/zellij/issues/5314

## 무엇이 문제였나

zellij 서버는 탭이 늘 때마다 fd를 ~15개씩 더 쓴다. status-bar / tab-bar 같은 WASM
플러그인이 **탭마다 인스턴스로 뜨고**, 인스턴스마다 WASI preopen 디렉토리를 4개씩
잡기 때문이다. 실측(탭 8개 세션): 서버가 연 fd 156개 중 **125개가 DIR**.

macOS launchd는 데몬에게 soft `RLIMIT_NOFILE` 을 **256** 으로 준다(hard 는 unlimited).
256 / 15 ≈ 17 이라 탭 10~16개쯤에서 `Command::spawn()` 이 EMFILE 로 실패하고,
그 지점이 `.expect("failed to spawn")` 이라 **서버가 패닉하며 세션 전체가 날아간다.**

## 패치 3개

| # | 파일 | 내용 |
|---|---|---|
| 1 | `zellij-server/src/lib.rs` | `start_server_impl()` 진입 시 `raise_nofile_limit()` — soft 를 min(65536, hard) 까지 올린다. 실패하면 절반씩 낮춰 재시도(macOS 는 per-process cap 초과 시 EINVAL). |
| 2 | `zellij-server/src/os_input_output_unix.rs` | `.expect("failed to spawn")` 제거 → `Err` 전파. 호출부(`pty.rs`)가 `non_fatal()` 로 받으므로 **해당 페인만 실패하고 세션은 산다.** |
| 3 | `zellij-server/src/os_input_output_unix.rs` | `CommandNotFound` 조기 반환 경로의 **pty fd 2개 누수** 수정. `into_raw_fd()` 로 소유권을 가져온 뒤 닫지 않고 return 하던 것. |

패치 1이 본 해결이고, 2·3은 안전망이다.

## 원격 구성

| remote | 대상 | 용도 |
|---|---|---|
| `origin` | `zellij-org/zellij` | 업스트림. 태그·업데이트를 여기서 가져온다 |
| `fork` | `ESPINS/zellij` | 개인 백업. 패치 브랜치를 여기 push |

패치는 `patch/fd-limits` 브랜치에 커밋되어 있다.

## 새 zellij 버전으로 옮길 때

rebase 가 정석이다 (패치 파일 수동 적용보다 충돌 처리가 낫다):

```sh
cd ~/IdeaProjects/zellij
git fetch origin --tags
git checkout patch/fd-limits
git rebase <새태그>          # 충돌 나면 해결 후 --continue
cargo xtask install ~/bin/zellij
git push -f fork patch/fd-limits
```

⚠ **빌드하면 `zellij-utils/assets/plugins/*.wasm` 13개가 재생성되어 변경으로 잡힌다.**
바이트만 다른 산출물이니 커밋하지 말고 `git restore -- 'zellij-utils/assets/plugins/*.wasm'`.

`.local-patches/fd-limits.patch` 는 rebase 가 불가능할 때를 위한 보조 사본이다
(`git apply --3way`). 소스를 고치면 `git diff <기준태그> -- zellij-server/src > .local-patches/fd-limits.patch` 로 갱신할 것.

## 빌드 준비물

- `brew install protobuf` (protoc)
- `rustup target add wasm32-wasip1`
- rust 툴체인은 `rust-toolchain.toml` 이 자동으로 고정본을 받아온다

## 검증법

서버가 스스로 한도를 올리는지 확인 — 일부러 256으로 낮춘 셸에서 세션을 띄우고,
그 안에서 실행되는 프로세스의 한도를 본다.

```sh
zsh -c 'ulimit -n 256; zellij -s fdcheck'
# 세션 안에서:
ulimit -n     # 65536 이면 패치가 동작한 것
```

서버가 실제로 몇 개를 쓰는지: `lsof -p $(pgrep -f "zellij --server") | awk 'NR>1{print $5}' | sort | uniq -c`
