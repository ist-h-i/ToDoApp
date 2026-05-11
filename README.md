# Angularデザイン刷新移行手順 — 工数効率最大化 + 安定性担保版

## 全体フロー

```text
1. 移行ルール固定
2. 対象画面棚卸し
3. Screen Contract作成 + semantic anchor追加
4. Baseline検査
5. Figma入力をDesign Packet化
6. Tailwind-firstでFigma移植
7. Contract検査 + Tailwind guard
8. 重要操作/重要フォームだけprobe test
9. AI差分レビュー
10. 最終検証
11. Migration Report作成
12. 次画面へ展開
```

---

## やること

```text
- Screen Contractを作る
- data-action-idを追加する
- data-form-idを追加する
- data-field-idを追加する
- required action / required field の欠落を検査する
- Tailwind-firstで新デザインを実装する
- 重要操作だけprobe testを追加する
- 重要フォームだけview→model / submit testを追加する
- Signal Formsへ移行しない
```

---

## 最小成果物構成

```text
.clinerules/
  design-migration.md

design-migration/
  contracts/
    <SCREEN_NAME>.screen-contract.yml
  figma/
    <SCREEN_NAME>/
      figma.html
      figma.css
      figma-tailwind.html
      notes.md
  reports/
    <SCREEN_NAME>.migration-report.md
    <SCREEN_NAME>.token-candidates.md

scripts/
  verify-screen-contract.mjs
  verify-tailwind-migration.mjs
```

---

# Step 1. 移行ルールを固定する

## Clineプロンプト

```markdown
# Task: Create design migration rules for Cline

Create or update `.clinerules/design-migration.md`.

Rules:

## Migration strategy
- Use Tailwind-first for the new design.
- Keep existing Angular component methods.
- Keep existing Reactive Forms / Template-driven Forms / ngModel.
- Do not migrate to Signal Forms.
- Do not perform Container/View refactoring unless explicitly requested.
- Do not change services, routing, permissions, validation logic, or API calls.

## Must preserve
- `data-action-id`
- `data-form-id`
- `data-field-id`
- existing `(click)`, `(submit)`, `(ngSubmit)`, `(change)`, `(input)` behavior
- existing method calls and payload semantics
- `[formGroup]`
- `formGroupName`
- `formArrayName`
- `formControlName`
- `[formControl]`
- `[(ngModel)]`
- custom two-way bindings such as `[(value)]`, `[(selected)]`, `[(checked)]`
- `routerLink`, `queryParams`
- `aria-*`, `i18n`, `data-testid`
- `[disabled]`, visibility, and permission conditions

## Tailwind rules
- Use Tailwind utility classes as the primary styling method.
- Do not construct Tailwind class names dynamically.
- Use complete static class names.
- Prefer design tokens.
- Avoid arbitrary values.
- If arbitrary values are necessary, record them as token candidates.
- Do not use `@apply` heavily.
- Do not create component CSS unless there is a documented exception.

## Forbidden
- Do not silently remove required actions.
- Do not silently remove required form fields.
- Do not rename form controls.
- Do not replace Reactive Forms with local component state.
- Do not use CSS classes as behavior/test selectors.
- Do not use `ViewEncapsulation.None`.
- Do not use `::ng-deep`.
- Do not use `!important` unless explicitly approved.

After creating the rule file, report only:
- changed file path
- key rules added
```

---

# Step 2. 対象画面を棚卸しする

## Clineプロンプト

```markdown
# Task: Inventory Angular screen before design migration

Read:
- `<COMPONENT_HTML_PATH>`
- `<COMPONENT_TS_PATH>`

Do not edit files.

Create an inventory for `<SCREEN_NAME>`.

Extract:

## Actions
- `(click)`
- `(submit)`
- `(ngSubmit)`
- `(change)`
- `(input)`
- child component output bindings
- method name
- argument expressions
- loop context such as `row`, `item`, `detail`
- disabled conditions
- visible conditions
- permission conditions

## Forms
- `[formGroup]`
- `formGroupName`
- `formArrayName`
- `formControlName`
- `[formControl]`
- `[(ngModel)]`
- custom two-way bindings
- submit handlers
- reset handlers
- validators visible in template
- validation error rendering

## Navigation
- `routerLink`
- `queryParams`
- navigation-related click handlers

## Attributes to preserve
- `data-testid`
- `aria-*`
- `role`
- `i18n`

## Risk classification
Classify each item as:
- critical
- normal
- low

Critical examples:
- save
- update
- delete
- submit
- approval
- detail navigation
- search
- CSV/download
- form fields used by `valueChanges`
- fields affecting rendering

Output:
- migration inventory
- ambiguous bindings
- high-risk areas
```

---

# Step 3. Screen Contract作成 + アンカー追加

## 目的

既存templateの関数・フォーム・ルーティング接続を、デザイン移植で消えないようにIDで固定する。

### 例

```html
<button
  data-action-id="OrderList.openDetail.v1"
  (click)="openDetail(row.id)"
>
  詳細
</button>

<form
  data-form-id="OrderSearch.searchForm.v1"
  [formGroup]="searchForm"
  (ngSubmit)="search()"
>
  <input
    data-field-id="OrderSearch.searchForm.keyword.v1"
    formControlName="keyword"
  />
</form>
```

## Clineプロンプト

```markdown
# Task: Create screen contract and add semantic anchors

Read:
- `<COMPONENT_HTML_PATH>`
- `<COMPONENT_TS_PATH>`

Create:
- `design-migration/contracts/<SCREEN_NAME>.screen-contract.yml`

Then update the existing template by adding semantic anchors only.

## Add anchors

For actions:
- add `data-action-id`

For forms:
- add `data-form-id`

For form fields:
- add `data-field-id`

## ID naming

Use this format:

Actions:
- `<ScreenName>.<actionName>.v1`

Examples:
- `OrderList.openDetail.v1`
- `OrderList.cancelOrder.v1`
- `OrderList.downloadCsv.v1`

Forms:
- `<ScreenName>.<formName>.v1`

Examples:
- `OrderSearch.searchForm.v1`

Fields:
- `<ScreenName>.<formName>.<controlName>.v1`

Examples:
- `OrderSearch.searchForm.keyword.v1`
- `OrderSearch.searchForm.status.v1`

## Contract content

Include:

```yaml
screen:
component:
source:
  template:
  class:

actions:
  - id:
    required:
    event:
    binding:
    handler:
      method:
      args:
        - expr:
          semantic:
    disabled_when:
    visible_when:
    preserve:
      - event
      - handler
      - payload

forms:
  - id:
    kind:
    group_expr:
    submit:
      binding:
      handler:
      required:
    controls:
      - id:
        name:
        binding:
        required:
        value_type:
    preserve:
      - formGroup
      - formControlName
      - ngSubmit
      - disabled
      - validation

navigation:
  - id:
    binding:
    required:

waivers: []
```

## Constraints

- Do not change business logic.
- Do not change method names.
- Do not change method arguments.
- Do not change form control names.
- Do not change visual structure.
- Do not change CSS classes.
- Do not migrate to Signal Forms.
- Do not refactor the component.

After editing, report:
- contract file created
- anchors added
- actions extracted
- forms extracted
- ambiguous items
```

---

# Step 4. Contract検査スクリプトを作る

## Clineプロンプト

```markdown
# Task: Create simple screen contract verifier

Create:
- `scripts/verify-screen-contract.mjs`

The verifier should:

1. Read all files under:
   - `design-migration/contracts/*.screen-contract.yml`

2. For each contract, read the target Angular template.

3. Verify required actions:
   - every required `actions[].id` exists as `data-action-id`
   - the same element or a nearby actionable element has an event binding
   - if possible, verify the handler method name appears in the binding

4. Verify required forms:
   - every required `forms[].id` exists as `data-form-id`
   - the declared form group expression appears in `[formGroup]`
   - every required control exists as `data-field-id`
   - every required control has the expected `formControlName`, `[formControl]`, or `[(ngModel)]`

5. Verify navigation:
   - required router bindings are present

6. Verify waivers:
   - if a required action or field is missing, allow it only when a matching waiver exists

7. Print clear errors and exit with non-zero status on failure.

Also update `package.json`:

```json
{
  "scripts": {
    "verify:screen-contract": "node scripts/verify-screen-contract.mjs"
  }
}
```

Constraints:
- Keep the verifier simple.
- Do not add external npm packages unless already available.
- Do not change application code.

After editing, report:
- files changed
- command to run
```

---

# Step 5. 現行画面のままBaseline検査を通す

## Clineプロンプト

```markdown
# Task: Run baseline screen contract verification

Run:

```bash
npm run verify:screen-contract
npm run build
```

If either command fails:

1. Identify whether the failure is caused by:
   - wrong contract
   - missing anchor
   - incorrect template path
   - existing build issue

2. Fix only:
   - contract content
   - semantic anchors
   - verifier bug

Do not:
- change business logic
- change visual structure
- change CSS classes
- change form control names
- migrate forms
- refactor the component

After completion, report:
- commands run
- pass/fail result
- fixes made
- unresolved baseline issues
```

---

# Step 6. Figma入力をDesign Packet化する

## Clineプロンプト

```markdown
# Task: Prepare Figma design packet

Create directory:

```text
design-migration/figma/<SCREEN_NAME>/
```

Create files if they do not exist:

```text
design-migration/figma/<SCREEN_NAME>/figma.html
design-migration/figma/<SCREEN_NAME>/figma.css
design-migration/figma/<SCREEN_NAME>/figma-tailwind.html
design-migration/figma/<SCREEN_NAME>/notes.md
```

Use the copied Figma outputs provided by the user.

In `notes.md`, summarize:

```markdown
# <SCREEN_NAME> Design Notes

## Target states
- default
- hover
- disabled
- loading
- error
- empty

## Important UI areas
- search area
- list/table/card area
- action buttons
- form fields
- navigation
- modal/menu if any

## Existing Angular bindings to preserve
Reference:
- `design-migration/contracts/<SCREEN_NAME>.screen-contract.yml`

## Token candidates
- colors
- spacing
- radius
- typography
- shadow
```

Do not edit Angular source files in this task.

After completion, report:
- files created
- missing Figma inputs
- token candidates found
```

---

# Step 7. Figma/Tailwind移植を行う

## Clineプロンプト

```markdown
# Task: Apply Figma design using Tailwind while preserving screen contract

Read:
- `<COMPONENT_HTML_PATH>`
- `<COMPONENT_TS_PATH>`
- `design-migration/contracts/<SCREEN_NAME>.screen-contract.yml`
- `design-migration/figma/<SCREEN_NAME>/figma-tailwind.html`
- `design-migration/figma/<SCREEN_NAME>/figma.html`
- `design-migration/figma/<SCREEN_NAME>/figma.css`
- `design-migration/figma/<SCREEN_NAME>/notes.md`

Goal:
- Replace the old visual HTML/CSS with the new Figma-based design.
- Use Tailwind-first styling.
- Preserve Angular behavior exactly according to the screen contract.

## Must preserve

Actions:
- preserve every required `data-action-id`
- preserve event trigger
- preserve method name
- preserve method argument semantics
- preserve disabled conditions
- preserve visible/permission conditions

Forms:
- preserve `data-form-id`
- preserve `data-field-id`
- preserve `[formGroup]`
- preserve `formGroupName`
- preserve `formArrayName`
- preserve `formControlName`
- preserve `[formControl]`
- preserve `[(ngModel)]`
- preserve custom two-way bindings
- preserve `(ngSubmit)`
- preserve submit button `type="submit"`
- preserve validation error rendering unless explicitly waived

Navigation:
- preserve `routerLink`
- preserve `queryParams`
- preserve navigation handlers

Attributes:
- preserve `data-testid`
- preserve `aria-*`
- preserve `role`
- preserve `i18n`

## Allowed

- Replace DOM structure
- Replace CSS classes
- Use Tailwind utility classes
- Change visual grouping
- Change labels to match Figma
- Move an action to another element if the same `data-action-id` and behavior are preserved

## Forbidden

- Do not remove required actions.
- Do not remove required form fields.
- Do not change method names.
- Do not change payload expressions unless contract is updated.
- Do not change form control names.
- Do not replace Reactive Forms with local state.
- Do not migrate to Signal Forms.
- Do not change service calls.
- Do not change routing logic.
- Do not change permission logic.
- Do not construct Tailwind classes dynamically.
- Do not use CSS classes as behavior selectors.
- Do not use `ViewEncapsulation.None`.
- Do not use `::ng-deep`.
- Do not use `!important`.

## Tailwind requirements

- Use complete static Tailwind class names.
- Prefer existing design tokens.
- If an arbitrary value is necessary, keep it minimal and record it in:
  - `design-migration/reports/<SCREEN_NAME>.token-candidates.md`

After editing, run:

```bash
npm run verify:screen-contract
npm run build
```

After completion, report:
- changed files
- preserved actions
- preserved forms
- changed action locations
- token candidates
- commands run
- unresolved risks
```

---

# Step 8. Tailwind migration guardを作る

## Clineプロンプト

```markdown
# Task: Create Tailwind migration guard

Create:
- `scripts/verify-tailwind-migration.mjs`

The script should scan changed Angular files under `src/` and detect:

Forbidden:
- `ViewEncapsulation.None`
- `::ng-deep`
- `!important`
- dynamic Tailwind class construction patterns such as:
  - `` `bg-${...}` ``
  - `"bg-" +`
  - `'text-' +`
  - `class="{{`
  - `[class]` with string concatenation
- excessive arbitrary values:
  - count classes containing `[`
  - warn when count is high

Also update `package.json`:

```json
{
  "scripts": {
    "verify:tailwind-migration": "node scripts/verify-tailwind-migration.mjs"
  }
}
```

Do not add external npm packages unless already available.

After editing, run:

```bash
npm run verify:tailwind-migration
```

Report:
- files changed
- warnings
- failures
```

---

# Step 9. 重要操作だけProbe Testを追加する

## 対象

```text
必須:
- save
- update
- delete
- approval
- submit
- detail navigation
- search
- CSV/download
- permission-sensitive actions

基本スキップ:
- tooltip
- hover
- 単なる開閉
- 低重要度の表示切替
```

## Clineプロンプト

```markdown
# Task: Generate high-risk action probe tests

Read:
- `<COMPONENT_HTML_PATH>`
- `<COMPONENT_TS_PATH>`
- `design-migration/contracts/<SCREEN_NAME>.screen-contract.yml`

Generate or update the component spec file for `<SCREEN_NAME>`.

Create probe tests only for high-risk actions:

Include actions related to:
- save
- update
- delete
- approval
- submit
- detail navigation
- search
- CSV/download
- permission-sensitive actions

Test rules:

1. Query elements only by:
   - `data-action-id`

2. Do not query by:
   - CSS class
   - DOM hierarchy
   - text label
   - nth-child

3. Use probe values to detect wrong payload mapping.

Example:

```ts
component.rows = [
  {
    id: '__EXPECTED_ID__',
    customerId: '__WRONG_CUSTOMER_ID__',
    detail: {
      id: '__WRONG_DETAIL_ID__',
    },
  } as any,
];

const spy = spyOn(component, 'openDetail');

fixture.detectChanges();

const el = fixture.nativeElement.querySelector(
  '[data-action-id="OrderList.openDetail.v1"]'
) as HTMLElement;

el.click();

expect(spy).toHaveBeenCalledWith('__EXPECTED_ID__');
```

4. Keep tests minimal.
5. Mock only what is necessary.
6. Do not refactor component logic.
7. Do not migrate forms.
8. Do not change production code unless a test exposes a real migration issue.

After editing, run:

```bash
npm test -- --watch=false
```

Report:
- tests added
- commands run
- failures
- production issues found
```

---

# Step 10. 重要フォームだけProbe Testを追加する

## 対象

```text
必須:
- 検索条件
- 保存/更新payloadに使われるfield
- submit対象field
- valueChangesで画面更新/API呼び出しに関わるfield
- validationやdisabledに影響するfield

基本スキップ:
- 表示補助だけのfield
- 低重要度filter
- デザイン変更で動作に影響しない装飾field
```

## Clineプロンプト

```markdown
# Task: Generate high-risk form probe tests

Read:
- `<COMPONENT_HTML_PATH>`
- `<COMPONENT_TS_PATH>`
- `design-migration/contracts/<SCREEN_NAME>.screen-contract.yml`

Generate or update the component spec file for `<SCREEN_NAME>`.

Create probe tests only for important forms and fields.

Prioritize:
- search fields
- submit fields
- fields used by `valueChanges`
- fields controlling rendering
- fields controlling validation
- fields affecting save/update/delete payloads

## Required test types

### 1. view to model

Input/select change updates the correct FormControl or ngModel.

Query by:
- `data-field-id`

Example:

```ts
const input = fixture.nativeElement.querySelector(
  '[data-field-id="OrderSearch.searchForm.keyword.v1"]'
) as HTMLInputElement;

input.value = '__KEYWORD_PROBE__';
input.dispatchEvent(new Event('input'));

expect(component.searchForm.controls.keyword.value)
  .toBe('__KEYWORD_PROBE__');
```

### 2. submit

Submitting the form reaches the existing handler.

Query by:
- `data-form-id`

Example:

```ts
const spy = spyOn(component, 'search');

const form = fixture.nativeElement.querySelector(
  '[data-form-id="OrderSearch.searchForm.v1"]'
) as HTMLFormElement;

form.dispatchEvent(new Event('submit'));

expect(spy).toHaveBeenCalled();
```

### 3. valueChanges only when relevant

If a field is documented as triggering `valueChanges`, test that the effect is reached.

Use `fakeAsync` and `tick()` only when debounce exists.

## Forbidden

- Do not query by CSS class.
- Do not query by DOM hierarchy.
- Do not migrate to Signal Forms.
- Do not change form control names.
- Do not replace Reactive Forms.
- Do not rewrite component logic.

After editing, run:

```bash
npm test -- --watch=false
```

Report:
- form tests added
- fields covered
- commands run
- unresolved mocking issues
```

---

# Step 11. AI差分レビューを行う

## Clineプロンプト

```markdown
# Task: Review migration diff against screen contract

Read:
- `design-migration/contracts/<SCREEN_NAME>.screen-contract.yml`
- `<COMPONENT_HTML_PATH>`
- `<COMPONENT_TS_PATH>`
- changed spec files
- changed style files if any

Review the migration.

Report only these sections:

## Contract compliance

### Actions
For each required action:
- id
- present: yes/no
- handler preserved: yes/no
- payload preserved: yes/no
- disabled/visible condition preserved: yes/no
- issue if any

### Forms
For each required form:
- id
- formGroup preserved: yes/no
- submit preserved: yes/no

For each required field:
- id
- binding preserved: yes/no
- control name preserved: yes/no
- issue if any

### Navigation
- routerLink preserved
- queryParams preserved
- navigation handlers preserved

### Attributes
- data-testid preserved
- aria preserved
- i18n preserved

## Tailwind review
- dynamic class construction found
- arbitrary values found
- token candidates
- unnecessary component CSS

## Forbidden changes
Check whether any of these changed:
- services
- API calls
- routing logic
- permission logic
- validation logic
- form architecture
- Signal Forms migration

## Required waivers
List missing or intentionally moved items that need waiver.

## Final status
- pass
- pass with warnings
- fail
```

---

# Step 12. Waiverが必要な場合だけ追加する

## Clineプロンプト

```markdown
# Task: Add waiver for intentional migration change

Read:
- `design-migration/contracts/<SCREEN_NAME>.screen-contract.yml`
- current migrated template

Add waiver only for the following intentional change:

```text
<DESCRIBE_INTENTIONAL_CHANGE>
```

Waiver format:

```yaml
waivers:
  - id: <CONTRACT_ITEM_ID>
    decision: moved | removed | replaced
    replacement: <NEW_CONTRACT_ITEM_ID_OR_NULL>
    reason: <REASON>
    approved_by: <APPROVER_ROLE_OR_NAME>
```

Rules:
- Do not add waivers for accidental missing bindings.
- Do not add waivers without a clear reason.
- Do not change production code in this task.
- Do not remove required items unless the waiver documents it.

After editing, run:

```bash
npm run verify:screen-contract
```

Report:
- waiver added
- reason
- verification result
```

---

# Step 13. 最終検証を実行する

## Clineプロンプト

```markdown
# Task: Run final migration verification

Run:

```bash
npm run verify:screen-contract
npm run verify:tailwind-migration
npm run build
npm test -- --watch=false
```

If any command fails:

1. Classify the failure:
   - contract violation
   - Tailwind guard violation
   - build error
   - test failure
   - existing unrelated issue

2. Fix only migration-related failures.

Do not:
- change business logic
- change services
- change routing
- change form architecture
- migrate to Signal Forms
- refactor unrelated code

After completion, report only:
- commands run
- pass/fail
- fixes made
- remaining issues
```

---

# Step 14. Migration Reportを生成する

## Clineプロンプト

```markdown
# Task: Generate migration report

Create:
- `design-migration/reports/<SCREEN_NAME>.migration-report.md`

Include only:

```markdown
# <SCREEN_NAME> Migration Report

## Changed files
- ...

## Preserved actions
| id | handler | payload | status |
|---|---|---|---|

## Preserved forms
| id | form group | submit | status |
|---|---|---|---|

## Preserved fields
| id | binding | status |
|---|---|---|

## Navigation
| id | status |
|---|---|

## Waivers
| id | decision | reason |
|---|---|---|

## Tailwind notes
- token candidates
- arbitrary values
- component CSS exceptions

## Tests and verification
- npm run verify:screen-contract
- npm run verify:tailwind-migration
- npm run build
- npm test -- --watch=false

## Remaining risks
- ...
```

Do not modify application code.
```

---

# Step 15. 次画面へ展開する

## Clineプロンプト

```markdown
# Task: Prepare next screen migration from previous successful pattern

Use the previous migration as a template.

Reference:
- `design-migration/contracts/<PREVIOUS_SCREEN>.screen-contract.yml`
- `design-migration/reports/<PREVIOUS_SCREEN>.migration-report.md`
- `.clinerules/design-migration.md`

Prepare migration for:
- `<NEXT_SCREEN_NAME>`
- `<NEXT_COMPONENT_HTML_PATH>`
- `<NEXT_COMPONENT_TS_PATH>`

Do:
1. Inventory the next screen.
2. Create `<NEXT_SCREEN_NAME>.screen-contract.yml`.
3. Add semantic anchors.
4. Run baseline verification.

Do not apply Figma design yet.

After completion, report:
- contract created
- anchors added
- high-risk actions
- high-risk forms
- blockers before Figma migration
```

---

# 最小CI設定

初期はこれで十分。

```json
{
  "scripts": {
    "verify:migration": "npm run verify:screen-contract && npm run verify:tailwind-migration && npm run build"
  }
}
```

テスト基盤が安定している場合だけ追加する。

```json
{
  "scripts": {
    "verify:migration": "npm run verify:screen-contract && npm run verify:tailwind-migration && npm run build && npm test -- --watch=false"
  }
}
```

---

# 最終チェックリスト

```markdown
## Scope
- [ ] business logicを変更していない
- [ ] service / APIを変更していない
- [ ] routing logicを変更していない
- [ ] permission logicを変更していない
- [ ] validation logicを変更していない
- [ ] Signal Formsへ移行していない

## Contract
- [ ] Screen Contractがある
- [ ] required actionが残っている
- [ ] required formが残っている
- [ ] required fieldが残っている
- [ ] waiverなしの削除がない

## Template
- [ ] data-action-idが残っている
- [ ] data-form-idが残っている
- [ ] data-field-idが残っている
- [ ] formControlNameが維持されている
- [ ] ngSubmitが維持されている
- [ ] routerLink/queryParamsが維持されている

## Tailwind
- [ ] Tailwind classを動的生成していない
- [ ] arbitrary valueを乱用していない
- [ ] token候補を記録している
- [ ] ViewEncapsulation.Noneを追加していない
- [ ] ::ng-deepを追加していない
- [ ] !importantを追加していない

## Test
- [ ] verify:screen-contract成功
- [ ] verify:tailwind-migration成功
- [ ] build成功
- [ ] 重要action probe test成功
- [ ] 重要form probe test成功
```

---
