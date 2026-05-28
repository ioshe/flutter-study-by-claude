# 📚 심화 학습 체크리스트

각 "다음에 공부할 개념" 항목을 **어느 문서에서 파생됐는지(출처)** 표기하고, 별도 심화 문서로 정리한 목록.

---

## 2026-05-28 심화 (5개)

- [x] **Stream** — 값 여러 개의 비동기 흐름, BLoC 의 기반
  - 출처: [`20260528_isolate_vs_이벤트루프.md`](20260528_isolate_vs_이벤트루프.md), [`20260528_sound_null_safety_vs_typescript.md`](20260528_sound_null_safety_vs_typescript.md)
  - 심화 문서: [`20260528_stream.md`](20260528_stream.md)

- [x] **Flutter 자체 렌더링(게임 엔진 접근)의 장단점**
  - 출처: [`20260528_flutter_engine_구조.md`](20260528_flutter_engine_구조.md)
  - 심화 문서: [`20260528_flutter_자체렌더링_장단점.md`](20260528_flutter_자체렌더링_장단점.md)

- [x] **Widget vs Element vs RenderObject 3트리** — setState/const 가 왜 빠른지
  - 출처: [`20260528_flutter_engine_구조.md`](20260528_flutter_engine_구조.md)
  - 심화 문서: [`20260528_3트리_widget_element_renderobject.md`](20260528_3트리_widget_element_renderobject.md)

- [x] **BuildContext 의 정체** — 위젯 트리에서 "내 위치"(= Element)
  - 출처: [`20260528_flutter_engine_구조.md`](20260528_flutter_engine_구조.md)
  - 심화 문서: [`20260528_buildcontext.md`](20260528_buildcontext.md)

- [x] **Future / async / await 이벤트 루프 디테일** — platform channel 의 await 동작
  - 출처: [`20260528_flutter_engine_구조.md`](20260528_flutter_engine_구조.md), [`20260528_isolate_vs_이벤트루프.md`](20260528_isolate_vs_이벤트루프.md)
  - 심화 문서: [`20260528_future_async_await_이벤트루프.md`](20260528_future_async_await_이벤트루프.md)

---

## 📥 다음 심화 후보 (위 5개 문서에서 새로 파생됨)

- [ ] StreamBuilder vs FutureBuilder 나란히 비교 — 출처: stream.md, future_async_await.md
- [ ] rxdart (BehaviorSubject, debounce 등) — 출처: stream.md
- [ ] 상태관리 진화: setState → BLoC → Provider → Riverpod — 출처: stream.md, buildcontext.md
- [ ] InheritedWidget 직접 구현 — 출처: 3트리.md, buildcontext.md
- [ ] Key 심화 (ValueKey/ObjectKey/GlobalKey) — 출처: 3트리.md
- [ ] Impeller 렌더러 / shader jank — 출처: 자체렌더링_장단점.md
- [ ] Flutter web 렌더러 (CanvasKit vs HTML) — 출처: 자체렌더링_장단점.md
- [ ] Semantics 와 접근성 — 출처: 자체렌더링_장단점.md
- [ ] Future.wait / Future.any (병렬 대기) — 출처: future_async_await.md
- [ ] Completer — 출처: future_async_await.md

---

> 사용법: 새 심화 항목을 공부하면 위 후보에서 골라 문서를 만들고, 완료되면 `[x]` 로 체크 + 심화 문서 링크를 추가한다.
