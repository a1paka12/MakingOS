# [Rust로 OS 만들기] 1일차 - Freestanding 실행파일

## 시작하며

컴퓨터 공학을 하는 사람이라면 누구나 접해보고 배워 봤을 OS를 직접 만들어 보고 싶었다.

### 왜 Rust인가

왜 Rust인가? 의문을 가지는 사람들이 있을 수 있다.
Rust는 C/C++ 수준의 성능을 내면서, 메모리 안전성과 데이터 레이스 방지를 컴파일 단계에서 보장하는 언어다.
Rust는 처음 코딩하기는 어렵지만 만들어 놓고 실제로 돌릴 때 안정성을 보장할 수 있기 때문에 최근 들어 많이 사용되고 있다.

여기까지는 Rust의 장점이다. 그래서 내가 왜 Rust를 선택하여 OS를 만들려고 도전하는가?
사실 큰 뜻은 없었다. OS는 동시성 제어에서 민감하게 작동하고 보장돼야 하는 시스템이라고 알고 있다.
동시성 문제의 일부(데이터 레이스)를 컴파일러가 막아 주고 성능도 빠른 언어로 제일 먼저 Rust가 생각났다.
또한, Rust를 언젠간 배워보고 싶다는 생각을 가지고 있었는데 이번 기회에 Rust도 배울 겸 도전하게 됐다.

### 참고 자료와 진행 방식

참고하는 자료는 Philipp Oppermann의 [Writing an OS in Rust](https://os.phil-opp.com/ko/) 한국어판이다.
요즘은 AI한테 시키면 금방 만들 수도 있겠지만, 이번에는 일부러 AI 없이 예전 방식대로 책을 보고, 검색하고, 직접 디버깅하면서 해 보려고 한다. 

일종의 AI 디톡스라고 할 수 있다.

취미로 시간 날 때만 하는 거라 "N일차"는 날짜가 아니라 작업한 횟수를 의미한다.

## 오늘의 목표

운영체제 없이 돌아가는 Rust 실행파일 만들기.

Rust 프로그램은 표준 라이브러리(`std`)를 쓰고, `std`는 스레드, 파일, 네트워크 같은 기능을 OS에 의존해서 돌아간다.
그런데 지금 만들려는 게 OS 자체라 OS 의존성을 없애야 한다. 그래서 `std` 없이도 컴파일되는 실행파일, 즉 freestanding 실행파일부터 만든다.

## 1. 프로젝트 만들기

```bash
cargo new blog_os --bin --edition 2024
```

책에는 `--edition 2018`로 나와 있는데, 나는 최신 에디션인 2024로 만들었다. 이게 뒤에서 작은 차이를 만든다.

## 2. `#![no_std]`: 표준 라이브러리 빼기

`main.rs` 맨 위에 `#![no_std]`를 붙이면 `println!`부터 못 쓴다. `println!`이 `std`에 들어 있는 매크로라서다.
`println!`을 지우고 빌드하면 이런 에러가 나온다.

```
error: `#[panic_handler]` function required, but not found
error: unwinding panics are not supported without std
```

## 3. 패닉 핸들러 직접 만들기

`std`가 없으니 패닉이 났을 때 뭘 할지도 직접 정해야 한다.

```rust
use core::panic::PanicInfo;

/// 패닉이 일어날 경우, 이 함수가 호출됩니다.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

반환 타입 `!`는 never 타입으로, 이 함수가 절대 반환하지 않는다는 뜻이다. (일종의 발산 함수라고 할 수 있다.) 지금은 할 수 있는 게 없으니 무한 루프만 돈다.

## 4. 패닉 시 스택 되감기(unwinding) 끄기

패닉 핸들러를 넣어도 두 번째 에러는 그대로 남는다.

```
error: unwinding panics are not supported without std
```

책과 다른 점: 책에서는 이 단계에서 `eh_personality` language item이 없다는 에러가 나온다고 한다. 지금 버전에서는 위처럼 메시지가 바뀌었는데, 원인은 같다.
패닉이 나면 Rust는 기본적으로 스택을 거꾸로 되감으면서(unwinding) 변수들을 정리하는데, 이 기능에는 OS 지원이 필요하다. 그래서 패닉이 나면 그냥 바로 멈추도록(abort) 바꾼다.

```toml
# Cargo.toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

## 5. `#![no_main]`과 `_start` 진입점

다음 에러는 이거다.

```
error: using `fn main` requires the standard library
  = help: use `#![no_main]` to bypass the Rust generated entrypoint and declare a platform specific entrypoint yourself, usually with `#[no_mangle]`
```

우리가 짜는 `main`은 사실 프로그램이 처음 실행하는 함수가 아니다. 그 전에 C 런타임(crt0)과 Rust 런타임이 먼저 돌고 나서 `main`을 부른다. OS가 없으면 이 런타임도 없으니, `main` 대신 진입점을 직접 만들어야 한다.

```rust
#![no_std]
#![no_main]

#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

- `#![no_main]`: Rust 컴파일러에게 일반적인 진입점(`main`)을 쓰지 않겠다고 알린다.
- `#[no_mangle]`: 컴파일러가 함수 이름을 다른 이름으로 바꾸지(mangling) 못하게 해서, 링커가 `_start`라는 이름 그대로 찾을 수 있게 한다.
- `extern "C"`: C 호출 규약을 따르게 한다.

**책과 다른 점:** 책대로 `#[no_mangle]`만 쓰면 2024 에디션에서는 에러가 난다.

```
error: unsafe attribute used without unsafe
help: wrap the attribute in `unsafe(...)`
```

2024 에디션부터 `no_mangle`은 unsafe 속성이 되어서 `#[unsafe(no_mangle)]`로 써야 한다. 이름이 겹치는 심볼을 만들면 위험할 수 있어서라고 한다. 

## 6. 링커 에러

코드는 다 고쳤는데 `cargo build`를 하면 이번엔 링커가 실패한다.

```
error: linking with `cc` failed: exit status: 1
  = note: Undefined symbols for architecture arm64:
            "_main", referenced from:
          ld: symbol(s) not found for architecture arm64
```

`cargo build`는 기본적으로 지금 쓰는 컴퓨터(나는 Apple Silicon 맥, `aarch64-apple-darwin`)용으로 빌드한다. 그래서 링커는 이 프로그램이 macOS 위에서 C 런타임과 함께 돌 거라고 생각하고 `main`을 찾다가 실패한다.

내 호스트 타겟은 `rustc --version --verbose`의 `host:` 줄에서 확인할 수 있다.

해결 방법은 OS가 없는 베어메탈 타겟으로 빌드하는 것이다. 책에 나온 대로 ARM 임베디드용 타겟을 썼다.

```bash
rustup target add thumbv7em-none-eabihf
cargo build --target thumbv7em-none-eabihf
```

타겟 이름의 `none`이 OS가 없다는 뜻이다. 이렇게 하니 빌드가 성공했다.

> 책에는 OS별 링커 인자를 넘기는 방법도 나온다. macOS라면 `cargo rustc -- -C link-args="-e __start -static -nostartfiles"`. 나는 타겟을 바꾸는 쪽을 택했다.

## 결과

최종 `src/main.rs`:

```rust
#![no_std] // Rust 표준 라이브러리를 링크하지 않도록 합니다
#![no_main] // Rust 언어에서 사용하는 실행 시작 지점 (main 함수)을 사용하지 않습니다

use core::panic::PanicInfo;

#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    loop {}
}

/// 패닉이 일어날 경우, 이 함수가 호출됩니다.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

아직 실행해서 볼 수 있는 건 없다. 빌드가 된다는 것 자체가 오늘의 결과다.

## 정리

- `std`는 OS에 기대고 있어서 커널에서는 쓸 수 없다. 대신 `core`만 쓴다.
- `std`가 해 주던 일(패닉 처리, 진입점)을 직접 정의해야 한다.
- 호스트용으로 빌드하면 링커가 OS를 가정하니까, OS 없는 타겟으로 빌드한다.
- 책과 지금 버전 사이에 에러 메시지나 문법(`unsafe(no_mangle)`)이 조금씩 다르다. 2일차에는 이게 훨씬 크게 터진다.

처음 언어를 배울 땐 맨땅에 헤딩식으로 언어 하나 정하고 프로젝트를 하면서 문법을 찾아보고 디버깅하면서 배우는 성격이다.
하지만 Rust는 악명 높게도 컴퓨터 구조와 시스템 같은 저수준 개념을 알아야 제대로 이해되는 언어라는 걸 체감하게 됐다. 비록 오늘은 그렇게까지 에러를 보지는 못했지만, 간단한 코드를 적는 것에서도 Rust의 정교함을 느낄 수 있었다.

## 다음 할 일

책 2장 "최소 기능을 갖춘 커널": 직접 만든 타겟으로 빌드해서 실제로 부팅하고 화면에 글자를 띄운다.
