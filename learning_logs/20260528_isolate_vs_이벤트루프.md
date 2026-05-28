# 2026-05-28 — Dart isolate 모델 vs JS 단일 이벤트 루프

> 관련 선행 학습: `20260522_dart_의_javascript_컴파일.md`, `20260528_dart_to_js_vs_순수js.md`

## Q
> Dart isolate 모델 — JS 의 단일 이벤트 루프와 어떻게 다른가 (Flutter 의 비동기 처리 이해에 필수) 도 알려줘

## A

## 핵심 정의

**Isolate = 각자 독립된 메모리를 가지고, 메모리를 공유하지 않으며, 오직 메시지로만 통신하는 Dart 의 동시성 단위.**

가장 먼저 깨야 할 오해:

> ⚠️ **async/await ≠ isolate**

- **async/await, Future** → 한 isolate **안에서** 일어나는 **동시성(concurrency)**. 이벤트 루프가 처리. JS 와 동일.
- **Isolate** → 진짜 **병렬성(parallelism)**. 다른 CPU 코어에서 동시 실행. 멀티스레드에 가깝지만 **메모리를 안 나눠 씀**.

비교: **Dart 단일 isolate ≈ JS 메인 스레드 + 이벤트 루프**. **Dart 추가 isolate ≈ JS Web Worker**.

---

## 왜 등장했는가

동시성을 다루는 3가지 철학 중 Dart 는 의도적으로 ③을 골랐다.

| 모델 | 대표 | 장점 | 단점 |
|---|---|---|---|
| ① 단일 스레드 | JavaScript | 안전 (race 없음) | CPU 무거운 작업이 UI 멈춤 |
| ② 공유 메모리 멀티스레드 | Java, C++ | 강력, 빠름 | race condition·deadlock·lock 지옥 |
| ③ **격리된 메모리 + 메시지** | **Dart**, Erlang, Elixir | 병렬 + 안전 | 메시지 복사 비용 |

②가 위험한 이유: 여러 스레드가 같은 메모리를 동시에 건드리면 race condition, lock 을 걸면 deadlock. 잡기 힘든 버그의 원천.

Dart 의 통찰: **"메모리를 안 나눠 쓰면, 나눠 쓸 때 생기는 모든 버그가 원천적으로 사라진다."** 이것이 **isolate(고립)** 라는 이름의 뜻. Erlang/Elixir 의 process 모델에서 영감.

---

## Dart에서의 동작 방식 (2층 구조)

### 1층 — 단일 isolate 안: 이벤트 루프

각 isolate 는 JS 처럼 **이벤트 루프 + 큐 2개**를 가진다:

```
┌─────────────── 하나의 Isolate ───────────────┐
│  [Microtask Queue]  ← scheduleMicrotask, 우선순위 높음 │
│  [Event Queue]      ← Future, Timer, IO, 사용자 입력   │
│  이벤트 루프: microtask 다 비우고 → event 1개 → 반복   │
└───────────────────────────────────────────────┘
```

`async/await` 가 이 안에서 돈다. 스레드가 하나라 진짜 동시 실행이 아니라 "기다리는 동안 다른 일 하기":

```dart
Future<void> main() async {
  print('1');
  await Future.delayed(Duration(seconds: 1));  // 대기 중 이벤트 루프는 다른 일 가능
  print('2');
}
// 출력: 1 → (1초) → 2
```

> 핵심: 네트워크·파일 읽기 같은 IO 는 isolate 불필요. OS 가 비동기 처리하고 끝나면 이벤트 큐에 알림. 이벤트 루프 하나로 충분.

### 2층 — 여러 isolate: 진짜 병렬성

CPU 를 오래 잡아먹는 작업(이미지 처리, 큰 JSON 파싱, 암호화)은 새 isolate 를 띄운다:

```dart
import 'dart:isolate';

void heavyWork(SendPort sendPort) {
  int sum = 0;
  for (int i = 0; i < 1000000000; i++) {
    sum += i;
  }
  sendPort.send(sum);   // 결과를 메시지로 전송 (메모리 공유 아님)
}

void main() async {
  final receivePort = ReceivePort();
  await Isolate.spawn(heavyWork, receivePort.sendPort);  // 다른 코어에서 실행
  final result = await receivePort.first;                // 메시지 받기
  print('결과: $result');
}
```

두 isolate 는 메모리를 1바이트도 공유하지 않는다. `SendPort`/`ReceivePort` 우편함으로 메시지를 주고받고, 메시지는 **복사(deep copy)** 되어 전달된다. 그래서 lock 이 필요 없다.

---

## Flutter에서는 어떻게 쓰이는가

Flutter UI 는 **메인 isolate** 에서 돈다. 60fps 면 프레임당 **16ms**, 120fps 면 **8ms** 안에 한 프레임을 그려야 한다. 메인 isolate 에서 무거운 작업을 하면 화면이 얼어붙는다(jank).

```dart
// ❌ 나쁜 예: 큰 JSON 파싱을 메인 isolate 에서
onPressed: () {
  final data = jsonDecode(hugeJsonString);  // 200ms 걸리면 화면이 200ms 얼어붙음
}
```

`compute()` 헬퍼로 무거운 작업을 임시 isolate 로 보낸다:

```dart
// ✅ 좋은 예
onPressed: () async {
  final data = await compute(jsonDecode, hugeJsonString);
  // 파싱이 다른 코어에서 도는 동안 UI 는 부드럽게 유지
}
```

> 실무 규칙: **"이 작업이 16ms 넘게 걸리나?" → isolate(compute) 로.** 네트워크 대기는 이벤트 루프로 충분, CPU 연산은 isolate 필요.

---

## 다른 언어와 비교

| 언어 | 모델 | 메모리 공유? | Dart 와의 관계 |
|---|---|---|---|
| **JavaScript** | 이벤트 루프 + Web Worker | ❌ (Worker 도 postMessage) | 가장 비슷. Worker ≈ isolate |
| **Java/Kotlin** | 스레드 + lock | ✅ 공유 | Dart 가 일부러 피한 모델 |
| **Erlang/Elixir** | actor/process | ❌ | Dart isolate 의 사상적 조상 |
| **Go** | goroutine + channel | ✅ (channel 권장) | 통신은 비슷, 메모리는 공유 |

**왜 Dart 는 이렇게 선택했는가**: Dart 의 주 무대는 모바일 UI. UI 는 절대 멈추면 안 되는 메인 스레드를 가지는데, 거기에 Java 식 공유 메모리 스레드를 붙이면 race condition 으로 UI 상태가 깨질 위험이 크다. "격리 + 메시지" 는 그 위험을 원천 차단하면서 멀티코어를 쓰게 해주는 절충안. JS 의 Web Worker 와 거의 같은 결론.

---

## 주의할 점

1. **async/await 는 isolate 가 아니다 (제일 중요).** async 는 동시성(한 스레드가 번갈아), isolate 는 병렬성(여러 코어). 99%의 비동기 코드는 isolate 없이 async/await 만으로 충분.
2. **IO 작업엔 isolate 쓰지 마라.** 네트워크·파일·DB 대기는 OS 가 비동기 처리. 이벤트 루프로 끝. isolate 는 CPU 바운드 전용.
3. **isolate 는 공짜가 아니다.** 생성 비용이 있고 메시지는 복사된다. 작은 작업을 isolate 로 보내면 복사 비용이 더 클 수 있다.
4. **메시지로 보낼 수 있는 것에 제약이 있다.** 기본 타입·List·Map 은 OK, 일부 객체(열린 소켓 핸들 등)는 불가.
5. **`compute()` 는 1회성.** 반복적 양방향 통신이 필요하면 직접 `Isolate.spawn` + 포트를 설계해야 한다.

---

## 다음에 공부할 개념

1. **Future / async / await 의 큐 동작** — microtask vs event queue 우선순위, `Future.then` 실행 시점
2. **Stream** — Future 가 "값 1개" 라면 Stream 은 "값 여러 개의 비동기 흐름". Flutter 상태관리(BLoC)의 기반
3. **sound null safety** — 계속 추천 중

추천: **1번 (Future 와 이벤트 루프 디테일)**. 방금 이벤트 루프를 큰 그림으로 봤으니 그 안에서 `await` 가 어떻게 도는지 들어가면 비동기가 완전히 손에 잡힌다.

---
