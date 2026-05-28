# 2026-05-28 — Flutter Engine 구조 (Dart ↔ Skia/Impeller ↔ Platform Channel)

> 관련 선행 학습: `20260522_dart_의_javascript_컴파일.md` (AOT), `20260528_isolate_vs_이벤트루프.md` (메시지 패싱)

## Q
> Flutter engine 구조 — Dart 코드 ↔ Skia/Impeller 렌더러 ↔ 플랫폼 채널 흐름 이것도 알려줘

## A

## 핵심 정의

**Flutter engine = 네가 Dart 로 짠 위젯을 실제 화면 픽셀로 직접 그려주는 C++ 기반 런타임.**

Flutter 의 가장 급진적인 결정: **OS 의 네이티브 위젯을 안 쓴다.** OS 한테서 **빈 캔버스 한 장**만 받아 그 위의 **모든 픽셀을 직접 그린다.** 게임 엔진과 같은 접근.

---

## 왜 등장했는가

| 방식 | 대표 | 방법 | 문제 |
|---|---|---|---|
| 네이티브 위젯 빌리기 | React Native | JS 가 OS 위젯 조종 | JS 브릿지 병목, 플랫폼마다 모양·버그 다름 |
| 웹뷰 | Cordova | HTML 을 웹뷰에 | 느리고 네이티브 느낌 없음 |
| **직접 그리기** | **Flutter** | 빈 캔버스에 픽셀 직접 렌더 | (모든 걸 자체 구현해야) |

Flutter 의 통찰: **"OS 위젯을 안 빌리고 내가 다 그리면, 어디서나 똑같고 브릿지 병목도 없다."** 대가로 버튼·텍스트·스크롤·애니메이션을 전부 직접 구현했지만, 모든 플랫폼에서 픽셀 단위로 동일한 UI 와 60/120fps 를 얻었다.

---

## Flutter의 3층 구조

```
┌────────────────────────────────────────────┐
│  1. Framework (Dart)  ← 네가 코드 짜는 곳     │
│     Widget / Element / RenderObject 트리      │
│     Material, Cupertino, 애니메이션, 제스처    │
├────────────────────────────────────────────┤
│  2. Engine (C++)  ← 진짜 엔진                │
│     Skia / Impeller (렌더링)                  │
│     Dart Runtime (네 코드 실행)               │
│     텍스트 레이아웃, Platform Channel          │
├────────────────────────────────────────────┤
│  3. Embedder (플랫폼별)  ← OS 에 끼워넣는 층   │
│     Android / iOS / Windows / macOS / Linux   │
│     캔버스(surface), 입력 이벤트, 스레드 제공   │
└────────────────────────────────────────────┘
```

- **Framework**: 네가 만지는 99%. 전부 Dart.
- **Engine**: C++ 심장. Dart 코드 실행 + 결과를 픽셀로 변환.
- **Embedder**: 각 OS 에 engine 을 심는 얇은 어댑터.

---

## 렌더링 파이프라인 — build() 가 픽셀이 되기까지

```
Widget tree      (설계도 — 가볍고 자주 새로 만듦)
   ↓
Element tree     (설계도의 실체 — 상태 유지, 재사용)
   ↓
RenderObject tree (실제 크기·위치 계산, 레이아웃 & 페인트)
   ↓
Layer tree       (GPU 에 넘길 그리기 명령 묶음)
   ↓
Skia / Impeller  (그리기 명령을 GPU 명령으로 변환)
   ↓
GPU → 화면 픽셀
```

> `setState()` 는 전체가 아니라 바뀐 부분의 Widget→Element→RenderObject 만 다시 계산. 그래서 빠르다.

### Skia vs Impeller

| | Skia | Impeller |
|---|---|---|
| 정체 | 구글 범용 2D 그래픽 라이브러리 (Chrome 사용) | Flutter 전용 신형 렌더러 |
| 문제/해결 | 셰이더를 첫 사용 시 컴파일 → shader jank | 셰이더를 빌드 때 미리 컴파일 → jank 제거 |
| 2026 현황 | 레거시 | iOS 기본, Android 도 기본 전환 단계 |

Impeller 의 등장 이유 자체가 "첫 애니메이션이 버벅이는 shader jank 제거". OS 위젯을 안 쓰기에 직접 풀어야 했던 숙제.

---

## Platform Channel — Dart 가 네이티브 기능 쓰기

카메라·GPS·블루투스·센서 같은 OS 고유 기능은 Dart 가 직접 못 건드린다. **Platform Channel** 로 네이티브 코드(Android=Kotlin, iOS=Swift)에 메시지를 보내 위임:

```dart
// Dart 쪽
import 'package:flutter/services.dart';
const platform = MethodChannel('com.example/battery');

Future<int> getBatteryLevel() async {
  final int level = await platform.invokeMethod('getBatteryLevel');
  return level;
}
```

```kotlin
// Android 네이티브(Kotlin) 쪽
methodChannel.setMethodCallHandler { call, result ->
  if (call.method == "getBatteryLevel") {
    result.success(batteryLevel)
  }
}
```

> **이것도 메시지 패싱이다.** isolate 끼리 메모리 안 나누고 메시지로 통신했듯, Dart ↔ 네이티브도 메모리 공유 없이 비동기 메시지로 통신. `await` 가 붙는 이유 — 네이티브에 보내고 답을 기다리는 비동기 작업. 이벤트 루프 위에서 돈다.

---

## 다른 프레임워크와 비교

| | 위젯 | 통신 | 결과 |
|---|---|---|---|
| **React Native** | OS 네이티브 위젯 | JS ↔ 네이티브 브릿지 | 네이티브 느낌, but 브릿지 병목·플랫폼 차이 |
| **Flutter** | 자체 렌더링 (OS 위젯 X) | 픽셀 직접 + 필요시 platform channel | 완전 동일 UI, 고성능, but 다 재구현 |
| **Unity (게임엔진)** | 자체 렌더링 | — | Flutter 렌더링 철학과 가장 유사 |

**왜 이렇게 선택했는가**: "한 번 짜서 모든 플랫폼에서 똑같이" 가 목표. OS 위젯을 빌리면 플랫폼마다 미묘하게 달라지는 걸 못 막는다. 그래서 렌더링을 통째로 가져왔다. Dart AOT(네이티브 머신코드)와 합쳐져 게임 같은 부드러움.

---

## 주의할 점

1. **OS 위젯을 안 쓰는 대가.** 접근성, 텍스트 입력기(IME, 한글 조합), 복사붙여넣기를 Flutter 가 전부 재구현해야 했다. 초기엔 약했으나 지금은 성숙.
2. **웹 Flutter 는 다르다.** `CanvasKit`(Skia→wasm, 정확하지만 무거움) 과 HTML 렌더러 두 경로. 모바일과 렌더링 경로가 다름.
3. **Platform channel 은 비동기.** 항상 `await`. 자주 호출하면 직렬화 비용.
4. **무거운 네이티브 작업**은 네이티브 쪽 스레드 관리도 필요 (Dart isolate 와 별개).

---

## 다음에 공부할 개념

1. **Widget vs Element vs RenderObject 3트리** — `setState` 가 왜 빠른지, `const` 위젯이 왜 성능에 좋은지
2. **BuildContext 의 정체** — 위젯 트리에서 "내 위치". Element 와 직결
3. **Future / async / await 이벤트 루프 디테일** — platform channel 의 `await` 동작

추천: **1번 (3트리)**. 렌더링 파이프라인 앞부분을 봤으니 이어서 보면 작동 원리가 완성된다.

---
