# Storybook 워크스페이스 패키지명 충돌 사전 감지

> **기여 범위**
> Node.js 모듈 해석 원인 분석 · CLI 충돌 감지 로직 구현 · init/upgrade 경로 대응 · 회귀 테스트 추가

[storybookjs/storybook PR #34290](https://github.com/storybookjs/storybook/pull/34290)

## 문제 상황

npm, pnpm, yarn 워크스페이스에서 프로젝트 패키지명이 `"storybook"`이면, `node_modules/storybook`이 실제 Storybook 패키지가 아니라 로컬 워크스페이스 패키지를 가리키는 심볼릭 링크가 될 수 있습니다.

그 결과 Storybook이 `storybook/internal/...`을 불러올 때 실제 패키지를 찾지 못하고 `Cannot find module storybook/internal/...` 오류가 발생할 수 있습니다.

![Storybook 패키지명 충돌 구조 비교](./assets/storybook-package-name-conflict-flow.png)

즉, 패키지가 누락된 문제가 아니라 **동일한 패키지명으로 인해 로컬 워크스페이스 패키지가 실제 Storybook 패키지를 가리는 이름 충돌 문제**였습니다.

---

## 원인 분석

문제는 Storybook 내부 모듈이 누락된 것이 아니라, **워크스페이스 심볼릭 링크가 실제 `storybook` 패키지를 가리는 이름 충돌**이었습니다.

```text
package name = "storybook"
        ↓
node_modules/storybook → workspace symlink
        ↓
실제 Storybook 패키지를 가림
        ↓
storybook/internal/... 해석 실패
```

따라서 오류가 발생한 뒤 원인을 추적하게 하기보다, CLI에서 `package.json`의 이름을 미리 검사해 충돌을 알려주는 방식으로 해결했습니다.

---

## 해결

`storybook upgrade`에서 사용되는 automigrate에 패키지명 충돌 검사를 추가했습니다.

```ts
const packageName =
  packageManager.primaryPackageJson.packageJson.name;

if (packageName === 'storybook') {
  return { packageName };
}
```

충돌이 발견되면 자동으로 이름을 변경하지 않고 원인과 해결 방법을 경고하도록 했습니다.

패키지명은 프로젝트 식별자이기 때문에 CLI가 임의로 수정하는 것보다 사용자가 직접 이름을 선택하도록 하는 것이 적절하다고 판단했습니다.

초기 구현은 `storybook upgrade`만 대상으로 했지만, 리뷰 과정에서 신규 사용자의 `storybook init`에서도 동일한 문제가 발생할 수 있다는 피드백을 받았습니다.

이를 반영해 `PreflightCheckCommand`에도 같은 검사를 추가했습니다.

```text
storybook upgrade → automigrate
storybook init    → preflight check
```

두 경로 모두 충돌을 발견하면 실행을 막지 않고 경고만 출력하도록 기존 CLI 동작 방식에 맞췄습니다.

---

## 검증

다음 경우를 테스트했습니다.

* 패키지명이 `"storybook"`이면 충돌 감지
* 다른 이름이면 감지하지 않음
* `name`이 없으면 감지하지 않음
* 경고 메시지에 충돌 원인과 이름 변경 안내가 포함되는지 확인
* `storybook init`에서도 동일한 경고가 발생하는지 확인

---

## 결과

**실행 중 발생하던 모듈 해석 오류를 CLI 단계에서 사전에 발견할 수 있도록 변경했습니다.**

워크스페이스 패키지명이 `"storybook"`일 때 발생하는 심볼릭 링크 충돌을 감지하고, 사용자가 실제 오류 원인과 해결 방법을 바로 확인할 수 있도록 했습니다.

또한 메인테이너 피드백을 반영해 처음 구현한 `upgrade` 경로뿐 아니라 `init` 경로까지 확장했으며, 수정 사항은 Storybook에 최종 반영됐습니다.