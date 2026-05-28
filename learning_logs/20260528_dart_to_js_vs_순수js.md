# 2026-05-28 — Dart→JS 컴파일 vs 순수 JS 작성, 뭐가 다른가

> 관련 선행 학습: `20260522_dart_의_javascript_컴파일.md` (Dart 가 JS 로 컴파일된다는 의미)

## Q
> 근데 dart 를 js 로 바꾼거랑 js 를 그냥 짜는거랑 뭐가 다른건데??

## A

## 핵심 정의

**결과물(브라우저에서 도는 JS)은 비슷하지만, "코드를 짜는 동안의 경험·안전성·생산성"이 완전히 다르다.** 한 줄 요약: **컴파일러가 나 대신 버그를 잡아주고, 최적화도 해주느냐**의 차이.

이것은 사실상 **"어차피 JS 되는데 왜 TypeScript 써?"** 와 똑같은 질문이다.

---

## 왜 이 질문이 중요한가

"어차피 JS 로 바뀔 거면 그냥 JS 짜지" 는 마라톤 결승선만 보면 맞는 말이다. 그러나 개발은 결승선이 아니라 **달리는 과정**이 90%다. 차이는 그 과정에서 나온다. 구체적으로 5가지.

---

## Dart에서의 동작 방식 (핵심 차이 5가지)

### 1. 타입 안전성 — 버그를 "언제" 잡느냐

```javascript
// 순수 JavaScript
function getLength(str) {
  return str.length;
}
getLength(null);   // 💥 런타임에 터짐 — 사용자가 앱 쓰다가 발견
// TypeError: Cannot read properties of null (reading 'length')
```

```dart
// Dart
int getLength(String str) {
  return str.length;
}
getLength(null);   // ❌ 컴파일 거부 — 빌드 자체가 안 됨. 배포 전에 막힘.
```

> 비유: JS 는 **맞춤법 검사 없는 메모장**, Dart 는 **빨간 줄 그어주는 워드**. 둘 다 글은 써지지만, 하나는 오타를 독자가 발견하고 하나는 내가 쓰는 중에 발견한다.

이것이 **sound null safety**. 실무 버그의 상당수가 "null 인 줄 모르고 접근" 이라 이 하나만으로도 가치가 크다.

### 2. IDE 지원 — 자동완성과 리팩토링

타입 정보가 있으니 IDE 가 정확히 안다:
- `.` 찍으면 그 객체가 가진 메서드만 정확히 자동완성
- 변수명 바꾸면 관련된 곳 전부 안전하게 일괄 변경 (rename refactoring)
- "정의로 이동" 이 100% 정확

순수 JS 는 타입이 없어 IDE 가 추측만 한다. 큰 프로젝트일수록 차이가 커진다.

### 3. 최적화 — tree shaking

dart2js 는 **whole-program optimization** 을 한다. 안 쓰는 코드를 전부 잘라내고 JS 를 만든다:

```
import 한 라이브러리에서 함수 100개 중 3개만 쓰면
→ dart2js 는 3개만 남기고 나머지 97개를 결과 JS 에서 제거
```

손으로 JS 짜면 죽은 코드 제거를 webpack 등으로 직접 설정해야 한다.

### 4. 언어 표현력

Dart 는 처음부터 일관되게 설계된 클래스, mixin, 제네릭, 연산자 오버로딩, sealed class 등을 가진다. JS 도 많이 따라왔지만 `this` 동작, 호이스팅, `==` vs `===` 같은 역사적 함정이 남아있다. Dart 는 그런 함정을 처음부터 안 만들었다.

### 5. 단일 코드베이스 (진짜 핵심)

- 순수 JS → **웹만** 나온다.
- Dart → 같은 코드로 **iOS + Android + 웹 + 데스크톱** (Flutter).

---

## Flutter에서는 어떻게 쓰이는가

위젯 코드를 Dart 로 한 번 짜면:

```bash
flutter build web     # dart2js → 웹용 JS
flutter build apk      # AOT → 안드로이드 네이티브
flutter build ios      # AOT → iOS 네이티브
```

JS 로 짰다면 웹 버전만 있고 모바일은 따로 짜야 한다. "한 번 짜서 다 배포" 가 Dart 를 쓰는 가장 큰 실무 이유.

---

## 다른 언어와 비교

**TypeScript vs JavaScript 논쟁과 정확히 같은 구조**:

| | 순수 JS | TypeScript→JS | Dart→JS |
|---|---|---|---|
| 타입 검사 | ❌ | ✅ | ✅ (더 엄격, sound) |
| JS 생태계(npm) | ✅ 직접 | ✅ 직접 | △ interop 필요 |
| 독립 언어인가 | — | JS superset | **완전 독립 언어** |
| 멀티플랫폼 | 웹만 | 웹만 | **모바일+웹+데스크톱** |

TypeScript 는 "JS 위에 타입 안전망만 얹는" 보수적 선택, Dart 는 "아예 새 언어 만들고 JS 는 배포 타깃 중 하나로 취급" 하는 급진적 선택이다.

---

## 주의할 점

**"그럼 Dart 가 항상 낫냐?" → 아니다.** 정직하게:

- **순수 웹 프론트엔드 작은 프로젝트**라면 JS/TS 가 유리. npm 생태계 직접 쓰고, 번들 크기 작고, 채용 시장도 크다.
- **dart2js 결과물은 번들이 크다.** Dart 런타임 포함. 작은 위젯 하나도 수백 KB.
- **npm 패키지를 바로 못 쓴다.** `package:js` 로 interop 필요.
- 그래서 **Dart 를 "웹 때문에" 쓰는 사람은 거의 없다.** Dart 의 존재 이유는 **Flutter 멀티플랫폼**이고, 웹은 보너스.

> 정리: "Dart→JS vs 순수 JS" 만 놓고 보면 웹 전용 프로젝트엔 JS/TS 가 합리적일 때가 많다. 하지만 **"모바일 앱도 만들 거다" 가 들어오는 순간 Dart 의 단일 코드베이스가 압도적**이 된다.

---

## 다음에 공부할 개념

방금 **sound null safety** 를 맛봤으니:

1. **sound null safety 깊게** — `?`, `!`, `late`, `?.` 가 컴파일 타임에 어떻게 보장되는지
2. **JS interop** — Dart 에서 npm/JS 라이브러리를 어떻게 끌어쓰는지
3. **AOT vs JIT** — 모바일은 왜 JS 를 안 거치고 네이티브가 되는지

추천: **1번 (null safety)**. 방금 가장 강력한 차이로 등장했으니 바로 이어가면 흐름이 좋다.

---
