# 2026-05-28 — Future / async / await 이벤트 루프 디테일 (심화)

> 📌 **추가학습 출처**: `20260528_flutter_engine_구조.md` (다음에 공부할 개념 #3), `20260528_isolate_vs_이벤트루프.md`, `20260528_sound_null_safety_vs_typescript.md`
> 원문: *"Future / async / await 이벤트 루프 디테일 — platform channel 의 await 동작"*
> 선행 권장: `20260528_isolate_vs_이벤트루프.md` (이벤트 루프의 큰 그림)

---

## 핵심 정의

**Future = "미래에 도착할 값 1개" 를 담는 약속(promise) 객체. async/await 는 그 Future 를 동기 코드처럼 읽기 쉽게 쓰게 해주는 문법. 그리고 이 모든 건 단일 isolate 안의 이벤트 루프 위에서 돈다.**

isolate 편에서 "이벤트 루프" 의 큰 그림을 봤다면, 이 문서는 그 루프 안에서 **`await` 한 줄이 정확히 어떤 순서로 실행되는지** 를 현미경으로 들여다봐. 이걸 알면 "왜 print 순서가 내 예상과 다르지?" 같은 비동기의 미스터리가 전부 풀려.

---

## 왜 이 디테일이 중요한가

비동기 코드에서 가장 흔한 버그는 **"실행 순서를 잘못 예측" 하는 것**이야:

```dart
void main() {
  print('A');
  Future(() => print('B'));
  print('C');
}
// 예상: A B C ?
// 실제: A C B   ← 왜?!
```

이 "왜?!" 를 설명하려면 이벤트 루프의 **두 개의 큐** 와 **실행 규칙**을 알아야 해. 외워서 되는 게 아니라 원리를 알면 어떤 코드든 순서를 예측할 수 있어.

---

## Dart에서의 동작 방식

### 1) 이벤트 루프와 두 개의 큐

isolate 편 복습 + 심화. 각 isolate 에는 이벤트 루프와 **큐가 2개** 있어:

```
┌──────────────────── 하나의 Isolate ────────────────────┐
│                                                         │
│  [Microtask Queue]  ← scheduleMicrotask, Future.then 일부 │
│       (높은 우선순위 — 먼저, 전부 비움)                   │
│                                                         │
│  [Event Queue]      ← Future(...), Timer, IO, 사용자 입력  │
│       (낮은 우선순위 — 한 번에 하나씩)                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

**이벤트 루프의 절대 규칙:**
1. 지금 실행 중인 **동기 코드를 끝까지** 다 실행한다.
2. 그다음 **Microtask Queue 를 완전히 빌 때까지** 처리한다.
3. 그다음 Event Queue 에서 **딱 하나** 꺼내 실행한다.
4. 다시 2번으로 (microtask 비우기) → 3번 (event 하나) → 반복.

즉 **동기 코드 > 마이크로태스크 > 이벤트** 순서의 우선순위야. 마이크로태스크는 항상 이벤트보다 먼저, 그리고 한 번 처리할 때 **싹 다** 비워.

### 2) 아까 그 미스터리 풀이

```dart
void main() {
  print('A');                      // 동기 → 즉시
  Future(() => print('B'));        // Event Queue 에 등록 (나중에)
  print('C');                      // 동기 → 즉시
}
```

1. `print('A')` — 동기, 즉시 실행 → **A**
2. `Future(() => print('B'))` — 이 콜백은 **Event Queue 로** 들어감. 지금 실행 안 함.
3. `print('C')` — 동기, 즉시 실행 → **C**
4. main 의 동기 코드 끝. 이벤트 루프가 microtask(없음) 확인 후 Event Queue 에서 B 콜백 꺼내 실행 → **B**

→ 결과 **A C B**. "Future(...) 는 즉시 실행이 아니라 큐에 예약" 이라는 게 핵심.

### 3) 마이크로태스크 vs 이벤트 — 더 정밀하게

```dart
void main() {
  print('1');
  Future(() => print('2'));                    // Event Queue
  Future.microtask(() => print('3'));          // Microtask Queue
  scheduleMicrotask(() => print('4'));         // Microtask Queue
  print('5');
}
// 결과: 1 5 3 4 2
```

순서 분석:
1. 동기: `1`, `5` → **1 5**
2. 마이크로태스크 비우기: `3`, `4` (등록 순서대로) → **3 4**
3. 이벤트 하나: `2` → **2**

→ **1 5 3 4 2**. 마이크로태스크가 이벤트보다 먼저인 걸 눈으로 확인.

### 4) await 는 무슨 일을 하는가 (제일 중요)

`await` 는 마법이 아니라 **"이 Future 가 완료되면 그 다음 줄부터 이어서 실행해줘" 라고 콜백을 등록하고, 일단 함수에서 빠져나가는 것**이야.

```dart
Future<void> main() async {
  print('A');
  await Future.delayed(Duration(seconds: 1));  // ← 여기서 함수가 "일시 중단"
  print('B');   // delay 끝나면 이벤트 큐를 통해 이어서 실행
  print('C');
}
```

`await` 를 만나면:
1. 그 Future 가 끝날 때까지 **이 함수를 중단**하고 **호출자에게 제어를 돌려줘** (블로킹이 아님! 스레드는 안 멈춤).
2. 중단된 동안 이벤트 루프는 **다른 일을 자유롭게** 함.
3. Future 가 완료되면, `await` 다음 줄(`print('B')`)부터가 **콜백으로 큐에 올라가** 이어서 실행.

> 핵심 오해 정정: **`await` 는 "기다리는 동안 프로그램 전체를 멈추는 것" 이 아니야.** "나는 잠깐 빠질게, 끝나면 깨워줘" 하고 **이벤트 루프에 자리를 양보** 하는 거야. 그래서 `await` 중에도 UI 가 안 멈추고 다른 이벤트가 처리돼. 이게 단일 스레드로도 앱이 부드러운 비결.

### 5) async 함수는 항상 Future 를 반환한다

```dart
Future<int> getValue() async {
  return 42;   // int 를 return 해도
}
// 실제 반환 타입은 Future<int>. 호출 측은 await 로 받음.
```

`async` 를 붙이면 그 함수는 **자동으로 Future 로 감싸져.** `return 42` 는 "이 Future 를 42 로 완료시켜" 라는 뜻이 돼.

---

## 기본 예제 (순서 예측 훈련)

```dart
Future<void> main() async {
  print('1');

  Future(() => print('2'));                   // event queue

  await Future(() => print('3'));             // event queue, 그리고 여기서 await

  print('4');
}
```

분석:
1. `1` 동기 출력.
2. `2` 콜백 → event queue 등록.
3. `3` 콜백 → event queue 등록. 그리고 `await` 로 main 중단, 제어 양보.
4. 이벤트 루프: event queue 에서 순서대로 `2` → `3` 실행. (등록 순서)
5. `3` 의 Future 가 완료되니 `await` 다음 줄 `4` 가 이어서 실행.

→ **1 2 3 4**. (`2` 가 `3` 보다 먼저 등록돼서 먼저 나옴에 주의.)

---

## Flutter에서는 어떻게 쓰이는가

### 1) platform channel 의 await (출처 문서의 그 연결고리)

engine 편에서 본 platform channel 호출:

```dart
Future<int> getBatteryLevel() async {
  final level = await platform.invokeMethod('getBatteryLevel');
  return level;
}
```

`invokeMethod` 는 네이티브(Kotlin/Swift)에 메시지를 보내고 **답을 기다리는 Future** 를 돌려줘. `await` 가 하는 일을 이제 정확히 이해할 수 있어:
1. 네이티브에 "배터리 알려줘" 메시지 전송.
2. `await` 로 이 함수 중단, **이벤트 루프에 자리 양보** → 그동안 UI 는 계속 부드럽게 렌더링.
3. 네이티브가 답을 보내면 그 응답이 **event queue** 에 들어오고, 이벤트 루프가 처리하며 `await` 다음 줄(`return level`)을 이어 실행.

→ 그래서 platform channel 호출 중에도 화면이 안 멈추는 거야. 메시지 패싱(isolate 편)과 이벤트 루프(이 편)가 만나는 지점이야.

### 2) FutureBuilder — Future 를 UI 에 연결

```dart
FutureBuilder<int>(
  future: getBatteryLevel(),
  builder: (context, snapshot) {
    if (snapshot.connectionState == ConnectionState.waiting) {
      return const CircularProgressIndicator();   // 기다리는 중
    }
    if (snapshot.hasError) return Text('에러: ${snapshot.error}');
    return Text('배터리: ${snapshot.data}%');       // 완료
  },
)
```

Future 가 완료되면 builder 가 자동으로 다시 호출돼 UI 가 갱신돼. (Stream 편의 `StreamBuilder` 와 형제 — 값 1개면 FutureBuilder, 여러 개면 StreamBuilder.)

### 3) 무거운 작업과의 구분 (isolate 편 연결)

**중요**: `await` 는 **CPU 무거운 작업을 빠르게 만들지 않아.** `await` 는 "IO 대기처럼 시간이 걸리지만 CPU 는 안 쓰는 일" 에 자리를 양보할 뿐이야. 큰 JSON 파싱 같은 **CPU 작업을 그냥 await 하면 그 계산 동안 이벤트 루프가 막혀(UI 멈춤).** 그건 isolate(compute)로 보내야 해. **async/await = 동시성(자리 양보), isolate = 병렬성(다른 코어).** 이 구분이 isolate 편의 핵심이었고 여기서도 그대로 적용돼.

---

## 다른 언어와 비교

| 언어 | 단일 값 비동기 | 문법 | 큐 모델 |
|---|---|---|---|
| **Dart** | `Future<T>` | `async`/`await` | microtask + event queue |
| **JavaScript** | `Promise<T>` | `async`/`await` | microtask(job) + task queue |
| **Kotlin** | `suspend` 함수 / `Deferred` | 코루틴 | 디스패처 기반 |
| **Python** | `Coroutine` / `asyncio.Future` | `async`/`await` | event loop |

**JavaScript 와의 직접 비교**: Dart 의 `Future`+`async/await` 는 JS 의 `Promise`+`async/await` 와 **거의 똑같아.** JS 도 microtask queue(Promise 콜백)와 task queue(setTimeout 등)를 구분하고, microtask 가 우선이야. JS 의 `Promise.then` ≈ Dart 의 `Future.then`, JS 의 `Promise.resolve().then` ≈ Dart 의 microtask. JS 를 해봤다면 Dart 비동기는 거의 공짜로 이해돼.

**왜 Dart 도 같은 모델을 골랐나**: Dart 의 초기 목표가 JS 대체였고(컴파일 편), dart2js 로 JS 로 컴파일되니 JS 의 이벤트 루프 모델과 호환되는 게 자연스러웠어. 또한 단일 스레드 + 이벤트 루프는 UI 프레임워크에 검증된 안전한 모델이라 Flutter 에도 맞았어.

---

## 주의할 점

1. **`Future(...)` 는 즉시 실행이 아니다.** event queue 에 예약될 뿐. 동기 코드가 다 끝난 뒤에야 실행돼 (A C B 예제).
2. **`await` 는 스레드를 멈추지 않는다.** 자리를 양보할 뿐. "기다린다" 는 말에 속아 "프로그램이 멈춘다" 고 오해하면 안 돼.
3. **CPU 무거운 작업을 await 해도 안 빨라진다.** 오히려 그 계산 동안 UI 가 멈춰. 그건 isolate(compute)의 영역.
4. **마이크로태스크를 무한히 만들면 이벤트 큐가 굶는다(starvation).** microtask 가 또 microtask 를 끝없이 만들면 event queue(사용자 입력 등)가 영영 처리 안 됨. 무거운 반복은 event queue 쪽(`Future(...)`)으로.
5. **에러 처리는 try-catch.** `await` 한 Future 가 에러를 던지면 일반 try-catch 로 잡혀. `.then` 방식이면 `.catchError`. 안 잡으면 unhandled 로 터져.
6. **`async` 함수의 반환은 항상 Future.** `Future<int>` 함수에서 `return 42` 해도 호출 측은 `await` 해야 값을 얻어.

---

## 다음에 공부할 개념

1. **`Future.wait` / `Future.any`** — 여러 Future 를 병렬로 기다리기 (동시 네트워크 요청 등)
2. **에러 처리 심화** — try-catch-finally, `.catchError`, `whenComplete`, unhandled exception
3. **Stream 과의 통합** — `Stream` 의 각 이벤트도 이 큐 위에서 도는 것 (Stream 편과 교차)
4. **`Completer`** — Future 를 수동으로 만들고 완료시키는 도구 (콜백 API 를 Future 로 감쌀 때)
