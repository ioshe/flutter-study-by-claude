# 2026-05-28 — Sound Null Safety vs TypeScript strict null check

> 관련 선행 학습: `20260528_dart_to_js_vs_순수js.md` (트랜스파일 언어, TS 개념)

## Q
> sound null safety — Dart 2.12+ 의 정적 안전성 보장. TypeScript 의 strict null check 와 비교하면 재밌다 → typescript 를 잘 모르는데 비교해볼래?

## A

## 핵심 정의

**Null safety = 타입 시스템이 "이 변수가 null 이 될 수 있는가"를 추적해서, null 일 수 있는 값을 안전하게 다루지 않으면 컴파일을 거부하는 기능.**

Dart 의 것은 **sound(건전한) null safety**. "sound" 는 **컴파일러가 OK 하면 런타임에 예상치 못한 null 참조가 절대 안 난다**는 보장을 뜻한다. 이것이 TS 와의 결정적 차이.

---

## 왜 등장했는가

`null` 은 Tony Hoare 가 1965년 만들고 훗날 **"10억 달러짜리 실수"** 라 사과한 개념.

```javascript
// JavaScript
const user = getUser();        // null 일 수도 있는데 타입에 안 적혀 있음
console.log(user.name);        // 💥 user 가 null 이면 런타임 폭발
```

JS 에선 어떤 값이든 null/undefined 가 끼어들 수 있고, 터지는 건 항상 런타임. Dart 도 2.12(2021) 이전엔 동일했다. 그래서 sound null safety 도입.

---

## Dart에서의 동작 방식

핵심 원칙: **모든 타입은 기본적으로 non-nullable.** null 을 원하면 `?` 를 붙인다.

```dart
String name = 'Lee';     // non-nullable
String? nickname = null; // nullable
```

nullable 값은 그냥은 못 쓴다:

```dart
String? name;
print(name.length);   // ❌ 컴파일 에러
```

null 연산자 4개:

| 연산자 | 이름 | 의미 |
|---|---|---|
| `?` | nullable 선언 | `String?` |
| `?.` | null-aware 접근 | null 이면 null 반환 (`user?.name`) |
| `??` | null 기본값 | null 이면 대체값 (`name ?? '익명'`) |
| `!` | null 단언 | "null 아님 보증" (틀리면 런타임 에러) |

```dart
String? name;
print(name?.length);       // null (안 터짐)
print(name ?? '이름없음');  // '이름없음'
print(name!.length);       // 💥 null 이면 여기서 터짐
```

`late`: "지금은 초기화 안 하지만 쓰기 전엔 반드시 채울게, null 아니야".

```dart
late String token;   // 초기화 전 쓰면 LateInitializationError
```

---

## 기본 예제

```dart
int getLength(String s) => s.length;
getLength('hi');   // 2
getLength(null);   // ❌ 컴파일 거부

String greet(String? name) => '안녕, ${name ?? '손님'}';
```

---

## Flutter에서는 어떻게 쓰이는가

```dart
// 1. 비동기 데이터 — 오기 전엔 nullable
String? userName;
Widget build(BuildContext context) => Text(userName ?? '로딩 중...');

// 2. 옵셔널 위젯 파라미터
Text('hello', style: maybeStyle);   // TextStyle? 일 수 있음

// 3. State 의 late 초기화
class _MyState extends State<MyWidget> {
  late final AnimationController _controller;
  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this);
  }
}
```

`?.`, `??`, `late` 는 Flutter 에서 하루에도 수십 번 쓴다.

---

## 다른 언어와 비교 (TypeScript 중심)

**TS 가 뭔지**: JavaScript 에 타입을 붙인 트랜스파일 언어. `tsc` 가 타입 검사 후 순수 JS 로 변환. `strictNullChecks` 옵션을 켜면 Dart 처럼 null 추적:

```typescript
let name: string | null = null;
console.log(name.length);    // ❌ 컴파일 에러
console.log(name?.length);   // ✅ OK
```

문법은 비슷하지만 결정적 차이 2개:

### 차이 1: opt-in vs 기본값
- TS: `strictNullChecks` 를 켜야 작동. 레거시는 대부분 꺼져 있음.
- Dart: 기본이자 강제. 끌 수 없음.

### 차이 2: unsound vs sound (핵심)
TS 는 빠져나갈 구멍이 많다:

```typescript
const data = JSON.parse(response) as User;   // "as" 로 타입 강제 단언
console.log(data.name);   // 컴파일 OK. 실제 응답에 name 없으면 런타임 폭발 💥
```

이유:
1. `any` 로 타입 검사를 통째로 끌 수 있음.
2. `as` 로 컴파일러를 속일 수 있음.
3. **TS 타입은 런타임에 사라짐(타입 소거).** 실행 중엔 타입 검사가 없음.

→ TS 는 **unsound**: "컴파일 통과"가 "런타임 안전"을 보장 못 함.

Dart 는 **sound**:
- 빠져나갈 구멍이 훨씬 적고,
- 타입 정보가 런타임까지 살아있어 컴파일러의 약속이 진짜로 지켜짐.

> 비유: TS 의 null check 는 **성실하지만 뇌물(`any`, `as`)에 약한 경비원**. Dart 의 sound null safety 는 **우회로 자체가 없는 시스템**.

Kotlin, Swift 도 Dart 와 유사한 null safety 를 가진다. 모던 언어들이 같은 방향으로 수렴.

---

## 주의할 점

1. **`!` (단언) 남발 금지.** null safety 보호를 스스로 끄는 것. 확실할 때만.
2. **`late` 는 초기화 전 접근하면 `LateInitializationError`.**
3. **nullable 을 습관적으로 붙이지 마라.** 기본은 non-nullable, null 이 의미 있을 때만 `?`.
4. **`?.` 체인 결과는 nullable.** `a?.b?.c` 결과도 nullable.

---

## 다음에 공부할 개념

1. **Future / async / await 의 이벤트 루프 디테일** — nullable 한 비동기 결과(`Future<String?>`) 처리와 연결
2. **Stream** — 비동기 값의 연속, Flutter 상태관리 기반
3. **generic + null safety** — `List<String>` vs `List<String?>` vs `List<String>?` 차이

---
