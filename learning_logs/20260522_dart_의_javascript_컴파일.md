# 2026-05-22 — Dart 의 JavaScript 컴파일 (dart2js / source-to-source)

## Q (첫 질문)
> dart 는 웹에서 javascript 를 대체하기 위해 나왔다고 했는데 결국에는 javascript 로 컴파일이 가능한 구조로 바뀌었다고 들었어, javascript 자체가 컴파일이 되지 않는 언어 아닌가? 어떻게 컴파일 한다는거지,,, 그 정적언어? 가 아니잖아 javascript 는,,, dart 로 코딩한게 javascript 화 된다는건가?

## A

## 핵심 정의

**"컴파일" = 한 언어로 쓴 코드를 다른 표현으로 자동 변환하는 과정.** 꼭 "기계어"여야 한다는 법은 없다. Dart 가 JavaScript 로 변환되는 것도 엄연한 컴파일이고, 정확히는 **source-to-source compilation** (또는 transpilation, 트랜스파일) 이라 부른다. 즉 "Dart 로 코딩한 게 JavaScript 화 된다" 는 직관은 본질적으로 맞다.

---

## 왜 등장했는가

Dart 의 역사는 "야망 → 좌절 → 피벗" 의 교과서다.

1. **2011년 — 야망**: Google 이 Dart 발표. 목표는 **"JavaScript 를 대체"**. JS 의 느슨한 타입, 예측 불가능한 동작, 대규모 앱에서의 유지보수성을 해결하겠다고 선언.
2. **Dartium 프로젝트**: Chrome 안에 **Dart VM 을 직접 박아넣는** 프로토타입 브라우저까지 만들었다. `<script type="application/dart">` 가 표준화되길 노렸다.
3. **2015년 — 좌절**: Mozilla(Firefox), Apple(Safari), Microsoft 모두 거부. "JS 라는 단일 표준을 깨는 건 안 된다." Google 도 결국 Dartium 포기.
4. **피벗**: "JS 를 죽일 수 없다면, JS 와 친해지자." → **dart2js** 라는 트랜스파일러 등장. Dart 로 짜고, 배포는 JS 로.
5. **2017~ Flutter 의 등장**: 진짜 승부는 웹이 아니라 **모바일 UI** 에서 났다. 이때부터 Dart 는 "웹 언어"가 아니라 "**Flutter 의 언어**" 로 정체성이 바뀜.

> 즉 "Dart 가 JS 로 컴파일된다" 는 건 원래 계획이 아니라 **타협의 결과물**이다. 이걸 알면 Dart 가 왜 그렇게 여러 백엔드(컴파일 타깃)를 갖게 됐는지 이해된다.

---

## Dart에서의 동작 방식

**같은 Dart 소스코드가 환경에 따라 4가지 결과물로 변환된다.**

| 컴파일러 | 입력 | 출력 | 언제 |
|---|---|---|---|
| `dart2js` | Dart | **JavaScript** | Flutter Web (기본), 일반 웹 배포 |
| `dart2wasm` | Dart | **WebAssembly (.wasm)** | Flutter Web 신형 — 더 빠름 |
| **AOT** (Ahead-of-Time) | Dart | **ARM/x64 머신코드** | Flutter mobile/desktop **릴리즈** 빌드 |
| **JIT** (Just-in-Time) | Dart | **실행 중 즉석 머신코드** | Flutter 개발 모드 (hot reload 가능한 이유) |

핵심 오해 풀기:

> "JavaScript 는 정적언어가 아니라서 컴파일 안 되는 거 아닌가?"

**아니다. JavaScript 도 컴파일된다.** 단지 "**언제**" 가 다를 뿐.

- **정적 컴파일** (C, Rust): 빌드 시점에 컴파일 → 실행 파일 생성.
- **JIT 컴파일** (JavaScript, Java, Dart 개발 모드): 실행 **순간** 엔진이 컴파일. JS 도 V8 안에서 핫한 함수는 머신코드로 컴파일돼서 돌아간다.
- **트랜스파일** (TypeScript → JS, Dart → JS): 언어 A 의 문법을 언어 B 의 문법으로 변환. 변환 결과물은 다시 그 환경의 엔진(V8 등)이 JIT 컴파일.

그래서 dart2js 의 흐름은:

```
Dart 소스 → [dart2js 컴파일] → JavaScript → [브라우저 V8] → JIT 컴파일 → 머신코드 → 실행
```

**컴파일이 두 번 일어난다.**

---

## 기본 예제

```dart
// hello.dart
void main() {
  int sum = 0;
  for (int i = 1; i <= 10; i++) {
    sum += i;
  }
  print('합계: $sum');
}
```

`dart compile js hello.dart -o hello.js` 로 변환하면 (실제 출력은 훨씬 길고 난독화됨) 대략 이런 JS 가 나온다:

```javascript
// hello.js (개념적으로 단순화)
(function() {
  function main() {
    var sum = 0;
    for (var i = 1; i <= 10; i++) {
      sum = sum + i;
    }
    console.log("합계: " + sum);
  }
  main();
})();
```

브라우저는 이 `.js` 를 평소처럼 읽어 실행한다. **브라우저 입장에선 Dart 의 존재를 모른다.**

---

## Flutter에서는 어떻게 쓰이는가

Flutter 빌드 명령들은 위의 컴파일러들을 호출하는 wrapper 다:

```bash
flutter run                 # 개발: JIT — hot reload 가능
flutter build apk           # Android 릴리즈: AOT → ARM 머신코드
flutter build ios           # iOS 릴리즈: AOT → ARM 머신코드
flutter build web           # 웹: dart2js → JavaScript
flutter build web --wasm    # 웹 (신형): dart2wasm → WebAssembly
```

**hot reload 가 빠른 이유**가 바로 JIT. 개발 중에 코드 변경하면 변경된 함수만 즉석에서 다시 컴파일해 메모리에 갈아끼운다. 앱 재시작 없이 1초 안에 반영.

릴리즈 빌드에서는 hot reload 가 필요 없고 성능이 중요하니 AOT 로 미리 머신코드를 만든다. Flutter 앱이 React Native 보다 일반적으로 빠른 이유 중 하나다 (RN 은 JS 브릿지 경유, Flutter 는 네이티브 머신코드 + 자체 렌더 엔진).

**한 코드베이스가 안드로이드/iOS/웹/Windows/macOS/Linux 로 다 빌드되는 것**이 이 다중 백엔드 덕분이다.

---

## 다른 언어와 비교

| 언어 | JS 로 컴파일? | 핵심 차이 |
|---|---|---|
| **TypeScript** | ✅ 거의 1:1. 타입 정보만 떼어내면 JS. | **타입 검사기 + 트랜스파일러**가 본질. 런타임은 JS. |
| **Kotlin** | ✅ (Kotlin/JS) | Dart 와 전략이 가장 유사. JVM/Native/JS 다중 타깃. |
| **Elm** | ✅ | 함수형 + 강한 정적 타입. 런타임 에러 거의 없는 게 자랑. |
| **Scala.js** | ✅ | JVM 언어인 Scala 를 JS 로. 큰 번들 크기가 단점. |
| **Dart** | ✅ + Wasm + AOT + JIT | **웹뿐 아니라 네이티브까지 같이 노리는 게 특이.** TypeScript 가 JS 보강책이라면, Dart 는 독립적인 언어 + 다중 백엔드. |

**"왜 Dart 는 이렇게 선택했는가"**: TypeScript 처럼 "JS 위에 타입만 얹는" 전략은 JS 의 한계를 그대로 물려받는다. Dart 는 처음부터 독자 언어로 설계됐기 때문에 sound null safety, isolate 기반 동시성, AOT 컴파일 같은 걸 자유롭게 채택할 수 있었다. 대신 JS 생태계와의 호환성은 어느 정도 포기했다.

---

## 주의할 점

1. **"정적언어 = 컴파일언어 / 동적언어 = 인터프리터언어" 는 틀린 도식.** JavaScript 도 V8 안에서 JIT 으로 머신코드로 변환된다. Python 도 CPython 에서 바이트코드로 컴파일된다. **모든 모던 언어는 어떤 형태로든 컴파일된다.**
2. **dart2js 결과물은 사람이 거의 못 읽는다.** 변수명 난독화 + 트리 셰이킹 + 최적화로 한 줄짜리 압축 형태. 디버깅은 **source map** 으로 한다.
3. **dart2js 빌드 결과 크기**: 작은 Hello World 도 수십~수백 KB. Dart 런타임 라이브러리가 포함돼서다. 큰 앱일수록 비율은 줄어든다.
4. **dart2wasm 은 dart2js 보다 빠르지만 아직 모든 패키지가 호환되진 않는다.** 2026년 현재 기본은 여전히 dart2js, 옵션으로 wasm.
5. **Flutter 앱이 JS 위에서 돈다는 오해**: 아니다. **모바일/데스크톱에서는 AOT 로 네이티브 머신코드가 되고, JS 는 거치지 않는다.** dart2js 는 **웹 타깃일 때만**.

---

## 다음에 공부할 개념

지금 질문은 **컴파일 모델** 이라는 깊은 영역을 건드렸다. 다음으로 자연스러운 흐름:

1. **AOT vs JIT 의 차이를 더 깊게** — hot reload 가 어떻게 가능한가, 릴리즈 빌드는 왜 더 빠른가
2. **Dart isolate 모델** — JS 의 단일 이벤트 루프와 어떻게 다른가 (Flutter 의 비동기 처리 이해에 필수)
3. **sound null safety** — Dart 2.12+ 의 정적 안전성 보장. TypeScript 의 strict null check 와 비교하면 재밌다
4. **Flutter engine 구조** — Dart 코드 ↔ Skia/Impeller 렌더러 ↔ 플랫폼 채널 흐름

추천 순서: **2 → 3 → 1 → 4**. isolate 부터 보면 Dart 의 진짜 색깔이 보인다.

---
