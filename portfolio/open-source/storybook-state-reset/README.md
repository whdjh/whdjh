# Storybook Vue 3 args/globals 초기화 버그 수정

> **기여 범위**
> 원인 분석 · Vue 3 렌더러 상태 동기화 로직 수정 · 회귀 테스트 추가

[storybookjs/storybook PR #34409](https://github.com/storybookjs/storybook/pull/34409)

## 문제 상황

Storybook은 컴포넌트에 전달하는 값인 `args`와 여러 스토리에서 공통으로 사용하는 설정값인 `globals`를 관리합니다.

Vue 렌더러에서는 값의 변경을 감지할 수 있도록 반응형 객체를 유지하면서 내부 속성을 추가, 수정, 삭제하는 방식으로 상태를 동기화하고 있었습니다.

따라서 기존 상태를 초기화하기 위해 빈 객체가 전달되면, 반응형 객체는 유지하되 내부 값은 모두 삭제되어야 합니다.

```ts
// 현재 상태
{ name: 'Kim', age: 20 }

// nextArgs = {} 적용 후 기대 상태
{}
```

하지만 실제로는 빈 객체를 적용해도 기존 `name`과 `age`가 남을 수 있었습니다. 이로 인해 상태를 초기화하거나 다른 스토리로 이동했을 때 화면 상태와 Storybook 내부 상태가 달라지는 문제가 발생했습니다.

![스토리 이동 후 args와 globals 초기화 동작 비교](./assets/state-reset-before-after.png)

---

## 원인 분석

상태를 갱신하는 `updateArgs()`에는 `nextArgs`가 빈 객체일 때 함수를 바로 종료하는 조건이 있었습니다.

```ts
if (Object.keys(nextArgs).length === 0) {
  return;
}
```

여기서 `{}`는 **변경할 값이 없다**는 의미가 아니라 **기존 값을 모두 제거한 상태로 변경한다**는 의미로도 사용됩니다.

기존 동기화 로직은 현재 객체의 키를 순회하면서 다음 상태에 존재하지 않는 값을 삭제하고 있었습니다.

```ts
Object.keys(currentArgs).forEach((key) => {
  if (!(key in nextArgs)) {
    delete currentArgs[key];
  }
});
```

`nextArgs`가 빈 객체라면 모든 기존 키가 삭제되어야 하지만, 조기 종료 때문에 이 로직까지 도달하지 못하고 이전 값이 그대로 남았습니다.

---

## 해결

빈 객체도 정상적인 다음 상태로 처리할 수 있도록 조기 종료 조건을 제거했습니다.

```diff
function updateArgs(reactiveArgs, nextArgs) {
-  if (Object.keys(nextArgs).length === 0) {
-    return;
-  }

  const currentArgs = isReactive(reactiveArgs)
    ? reactiveArgs
    : reactive(reactiveArgs);

  // 기존 상태에만 존재하는 값을 삭제
  Object.keys(currentArgs).forEach((key) => {
    if (!(key in nextArgs)) {
      delete currentArgs[key];
    }
  });

  Object.assign(currentArgs, nextArgs);
}
```

수정 후에는 `nextArgs = {}`가 들어와도 기존 상태와 비교하는 과정이 실행됩니다. 다음 상태에 포함되지 않은 모든 키가 삭제되므로 반응형 객체를 유지하면서 내부 상태를 빈 객체로 초기화할 수 있습니다.

---

## 검증

빈 객체가 전달됐을 때 기존 값이 모두 삭제되는 경우를 검증하는 회귀 테스트를 추가했습니다.

```ts
it('clears all args when nextArgs is empty -> updateArgs()', () => {
  const reactiveArgs = reactive({
    argFoo: 'foo',
    argBar: 'bar',
  });

  updateArgs(reactiveArgs, {} as any);

  expect(reactiveArgs).toEqual({});
});
```

PR의 Before/After 영상에서는 스토리를 이동해 상태를 초기화했을 때, 수정 전에는 이전 값이 남고 수정 후에는 빈 상태가 유지되는 것도 비교했습니다.

---

## 결과

**기존 객체를 유지하면서 내부 값을 동기화하는 구조에서는 빈 객체 역시 하나의 정상적인 상태**라는 점을 반영했습니다.

별도의 초기화 로직을 추가하지 않고 기존 상태 삭제 로직을 그대로 활용해 `args`와 `globals` 초기화 동작을 정상화했으며, 수정 사항은 Storybook에 최종 반영됐습니다.
