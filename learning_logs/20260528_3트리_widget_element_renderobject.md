# 2026-05-28 — Widget vs Element vs RenderObject 3트리 (심화)

> 📌 **추가학습 출처**: `20260528_flutter_engine_구조.md` (다음에 공부할 개념 #1)
> 원문: *"Widget vs Element vs RenderObject 3트리 — setState 가 왜 빠른지, const 위젯이 왜 성능에 좋은지"*

---

## 핵심 정의

**Flutter 는 화면을 그리기 위해 3개의 평행한 트리를 운영한다:**

| 트리 | 정체 | 비유 | 수명 |
|---|---|---|---|
| **Widget** | UI 의 **설계도(불변, immutable)** | 건물 도면 | 아주 짧음 (수시로 새로 만듦) |
| **Element** | 설계도와 실물을 잇는 **관리자(상태 보유)** | 현장 감독관 | 길다 (가능하면 재사용) |
| **RenderObject** | 실제 **레이아웃·페인트 담당** | 실제 지어진 건물 | 길다 (재사용) |

> 한 문장 비유: **Widget 은 "이렇게 생겨야 해" 라고 적힌 도면, Element 는 "그 도면과 실제 건물을 대조하며 관리하는 감독관", RenderObject 는 "실제로 공간을 차지하고 칠해지는 건물"** 이야.

이 3트리 구조를 모르면 `setState` 가 왜 안 느린지, `const` 가 왜 성능에 좋은지, `key` 가 왜 필요한지를 영원히 "그냥 그렇대" 로 외우게 돼. 알면 전부 한 번에 풀려.

---

## 왜 3개씩이나 필요한가

순진하게 생각하면 "위젯 하나가 알아서 그리면 되지 왜 3개로 나눠?" 싶어. 이유는 **성능** 이야.

문제 상황: Flutter 에선 화면이 바뀔 때마다 `build()` 가 호출되고 **위젯 트리를 통째로 새로 만들어.** 1초에 60번 그린다면 위젯 트리를 60번 새로 만드는 거야. 그런데 위젯에 대응하는 "실제 그려지는 무거운 객체(RenderObject)" 까지 매번 새로 만들고 부수면? → 끔찍하게 느려.

해결: **가벼운 것(Widget)은 막 새로 만들고 버리되, 무거운 것(RenderObject)은 재사용하자.** 그 사이에서 "새 도면과 기존 건물을 대조해서, 바뀐 부분만 건물에 반영" 하는 중개자가 필요해 — 그게 **Element** 야.

```
build() 호출  →  새 Widget 트리 (싸다, 막 만듦)
                      ↓ Element 가 "기존 것과 비교(diff)"
                 Element 트리 (재사용, 상태 보존)
                      ↓ 바뀐 부분만 명령
                 RenderObject 트리 (재사용, 무거움)
```

이 "새 도면 vs 기존 관리자 비교" 과정을 **reconciliation(재조정)** 이라고 해. React 의 Virtual DOM diffing 과 같은 아이디어야.

---

## Dart에서의 동작 방식 (각 트리의 역할)

### 1) Widget — 불변 설계도

```dart
class MyButton extends StatelessWidget {
  final String label;
  const MyButton(this.label);   // 모든 필드 final → 불변

  @override
  Widget build(BuildContext context) => Text(label);
}
```

Widget 은 **immutable(불변)** 이야. 한 번 만들어지면 못 바꿔. 색을 바꾸고 싶으면? "색이 다른 새 Widget" 을 만들어. 그래서 막 만들고 막 버려도 되는, 가볍고 일회용인 **설정값 묶음**이야. `build()` 가 리턴하는 게 바로 이 Widget 트리.

### 2) Element — 살아있는 관리자

Element 는 우리가 직접 안 만들어. Flutter 가 Widget 을 화면에 처음 올릴 때 각 Widget 에 대응하는 Element 를 **자동 생성**해. Element 가 하는 일:

- 자신이 어떤 Widget 에 대응하는지 기억
- 부모-자식 관계(트리 구조) 유지
- **State 객체를 보관** (StatefulWidget 의 상태가 여기 살아있음!)
- 새 Widget 이 오면 기존 것과 비교해서 RenderObject 를 갱신할지/재사용할지/버릴지 결정

**핵심**: build 로 Widget 은 매번 새로 만들어져 버려지지만, **Element 는 살아남아.** 그래서 StatefulWidget 의 `State` (카운터 값 등)가 위젯이 재생성돼도 유지되는 거야 — State 는 Widget 이 아니라 Element 에 붙어 있으니까.

### 3) RenderObject — 실제 그리는 일꾼

RenderObject 가 진짜 무거운 일을 해:

- **layout**: 내가 얼마나 큰지(크기), 어디 있는지(위치) 계산
- **paint**: 실제로 캔버스에 픽셀을 칠함 (engine 편의 Skia/Impeller 로 넘어감)
- **hit test**: 이 좌표를 탭하면 나를 누른 건지 판정

이게 비싸니까 재사용하는 거야. `RenderObject` 트리가 engine 의 Skia/Impeller 로 그리기 명령을 넘기는 마지막 Dart 단계야 (engine 편 렌더링 파이프라인의 그 RenderObject 트리).

### 세 트리의 대응 관계

대부분의 Widget 은 1:1:1 로 대응하지만, 항상은 아니야:
- `StatelessWidget`/`StatefulWidget` 같은 **조립용 위젯**은 Element 는 있지만 RenderObject 가 없어 (자식에게 위임).
- `Text`, `Padding`, `ColoredBox` 같은 **실제 그리는 위젯**만 RenderObject 를 가져.

---

## 기본 예제로 보는 "재조정"

```dart
// 변경 전
Container(color: Colors.red, child: Text('Hello'))

// setState 후, build 가 다시 호출되어
Container(color: Colors.blue, child: Text('Hello'))
```

이때 일어나는 일:
1. 새 Widget 트리 생성 (color 만 다른 Container, 동일한 Text).
2. Element 가 **같은 타입(Container)인지 확인** → 같으면 Element·RenderObject **재사용**.
3. RenderObject 에게 "색만 red→blue 로 바꿔" 라고 명령. **객체를 새로 안 만들고 속성만 갱신.**
4. Text 는 내용도 같으니 아무것도 안 함.

→ 무거운 RenderObject 를 새로 안 만들고 **색 속성 하나만 갱신** = 빠름. 이게 3트리의 위력이야.

만약 타입이 바뀌면? (`Container` → `Padding`) Element 는 "다른 타입이네" 하고 기존 걸 버리고 새로 만들어. 그래서 위젯 타입을 자주 바꾸면 재사용 이점이 줄어.

---

## Flutter에서는 어떻게 쓰이는가 (실전 3대 연결고리)

### 1) setState 가 왜 빠른가

```dart
setState(() { _count++; });
```

`setState` 는 "이 Element 를 dirty(다시 빌드 필요)로 표시" 할 뿐이야. 그러면 **그 Element 의 서브트리만** build 가 다시 돌고, 위에서 본 재조정으로 **바뀐 RenderObject 속성만 갱신**돼. 화면 전체를 새로 그리는 게 아니라서 빠른 거야. (단, build 메서드를 무겁게 짜면 그 build 자체가 느려지니 주의.)

### 2) const 위젯이 왜 성능에 좋은가

```dart
const Text('고정 텍스트')   // const!
```

`const` 위젯은 **컴파일 타임에 단 하나의 인스턴스로 고정**돼. build 가 100번 호출돼도 매번 같은 인스턴스를 가리켜. Element 가 재조정할 때 "어? 이거 이전과 **완전히 동일한 const 인스턴스**네(identical)" 라고 즉시 판단하고 **그 서브트리 비교를 통째로 건너뛰어(skip).** 즉 const 는 "이 부분은 절대 안 바뀌니 검사도 하지 마" 라는 최적화 힌트야. 안 바뀌는 위젯엔 const 를 붙이는 게 좋은 습관인 이유.

### 3) key 가 왜 필요한가

Element 가 새 Widget 과 기존 Element 를 짝지을 때, 기본은 **"같은 위치 + 같은 타입"** 으로 판단해. 그런데 리스트에서 항목 **순서가 바뀌거나 삭제**되면 이 판단이 어긋나서 엉뚱한 Element 와 짝지어져 (상태가 섞이는 버그). `Key`(특히 `ValueKey`)를 주면 Element 가 "위치가 아니라 이 key 로 짝을 찾아" 라고 알아들어서 상태가 올바르게 따라가. 리스트 아이템에 key 를 권장하는 게 이 3트리 매칭 때문이야.

---

## 다른 프레임워크와 비교

| 프레임워크 | 가벼운 설계도 | 재조정 중개 | 실제 화면 |
|---|---|---|---|
| **Flutter** | Widget | **Element** | RenderObject |
| **React** | JSX/Element | **Virtual DOM + Fiber** | 실제 DOM |
| **SwiftUI** | View(struct) | (내부 그래프) | 렌더 레이어 |

**핵심 공통점**: 셋 다 "선언적 UI" 야 — 개발자는 "지금 상태면 화면이 이렇게 생겨야 한다(Widget/JSX/View)" 만 선언하고, 프레임워크가 "이전 상태와 비교해서 실제로 뭘 바꿀지" 를 알아서 처리해. **Widget = React 의 가상 표현, Element ≈ React Fiber, RenderObject ≈ 실제 DOM** 으로 매핑하면 React 경험자는 바로 이해해.

**왜 Flutter 는 Element 를 따로 뒀나**: React 는 실제 DOM 이 브라우저 소유라 Virtual DOM 으로 diff 해서 DOM API 를 호출해. Flutter 는 DOM 이 없고 RenderObject 까지 자기가 소유하니, "가벼운 Widget ↔ 무거운 RenderObject" 의 수명 차이를 직접 관리할 중개자(Element)가 필요했어. 즉 직접 그리기(engine 편) 철학의 자연스러운 귀결이야.

---

## 주의할 점

1. **State 는 Widget 이 아니라 Element 에 산다.** StatefulWidget 의 위젯 객체는 매번 새로 만들어져도 State 가 유지되는 이유. 이걸 모르면 "왜 변수가 초기화 안 되지?" 또는 반대로 "왜 초기화되지?" 에서 혼란이 옴.
2. **build() 안에서 무거운 연산·객체 생성 금지.** build 는 자주 호출돼. 무거운 건 `initState` 나 별도 isolate(compute)로.
3. **const 를 붙일 수 있으면 붙여라.** 공짜 최적화. 린트(`prefer_const_constructors`)가 권장해주기도 해.
4. **리스트·동적 자식엔 Key 를 고려.** 특히 순서가 바뀌거나 추가/삭제되는 리스트.
5. **위젯 타입을 불필요하게 바꾸지 마라.** 조건부로 `Container` ↔ `Padding` 처럼 타입을 갈아끼우면 Element 재사용이 깨져. 가능하면 같은 타입 유지하고 속성만 바꿔.

---

## 다음에 공부할 개념

1. **BuildContext 의 정체** — 사실 BuildContext 는 바로 이 **Element** 야. 3트리를 알았으니 BuildContext 가 명확해짐 (바로 다음 학습으로 딱)
2. **Key 심화** — ValueKey, ObjectKey, GlobalKey 의 차이와 용도
3. **InheritedWidget** — Element 트리를 타고 데이터를 효율적으로 내려보내는 메커니즘 (Provider/Theme 의 기반)
4. **RenderObject 직접 만들기** — CustomPaint, 커스텀 레이아웃 (고급)
