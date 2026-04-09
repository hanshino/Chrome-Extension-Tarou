# 技能與攻擊提示音 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 玩家可點擊技能 icon 訂閱提示音，當該技能在戰鬥中觸發時播 `tip`；亦可開啟普通攻擊提示音，攻擊發出時播 `warning`。

**Architecture:** 新增 `useAbilitySound` composable 集中管理訂閱狀態與音效觸發邏輯。觸發點在 `useBattleLog.ts` 的 `handleAttackResultJson()` 裡，於 API response 進來、資料尚未寫入 storage 前呼叫，確保 reload 不重播。UI 在兩個 ActionList 元件的技能 img 上加點擊事件，combatPanel 版額外加普通攻擊開關。

**Tech Stack:** Vue 3 Composition API, TypeScript, UnoCSS, `useWebExtensionStorage`, `createNotification` (existing)

---

## File Map

| 操作 | 檔案 | 變更內容 |
|---|---|---|
| Modify | `src/logic/storage.ts` | 新增 2 個 storage refs |
| Create | `src/composables/useAbilitySound.ts` | 新增 composable（4 個函式） |
| Modify | `src/composables/useBattleLog.ts` | 新增 2 個觸發點 |
| Modify | `src/views/sidePanel/views/combat/components/ActionList.vue` | img click handler + ring highlight |
| Modify | `src/views/combatPanel/components/ActionList.vue` | img click handler + ring highlight + 普攻 toggle |

---

## Task 1: 新增 Storage Refs

**Files:**
- Modify: `src/logic/storage.ts:35-42`

- [ ] **Step 1: 在 BattleLog 區塊末尾新增兩個 ref**

在 `src/logic/storage.ts` 的 `// BattleLog` 區塊，`onlyShowSpecBuff` 那行之後加入：

```ts
// BattleLog
export const battleInfo = useWebExtensionStorage<Partial<BattleInfo>>('battleInfo', {})
export const battleRecord = useWebExtensionStorage<BattleRecord[]>('battleRecord', [])
export const specBossBuff = useWebExtensionStorage<string[]>('specBossBuff', [])
export const specPlayerBuff = useWebExtensionStorage<string[]>('specPlayerBuff', [])
export const questSetting = useWebExtensionStorage<QuestSetting[]>('questSetting', [])
export const onlyShowSpecBuff = useWebExtensionStorage<boolean>('onlyShowSpecBuff', false)
export const battleExportData = useWebExtensionStorage<Partial<BattleExport>>('battleExportData', {})
export const soundTriggerAbilities = useWebExtensionStorage<string[]>('soundTriggerAbilities', [])
export const soundNormalAttack = useWebExtensionStorage<boolean>('soundNormalAttack', false)
```

- [ ] **Step 2: Commit**

```bash
git add src/logic/storage.ts
git commit -m "feat(sound): 新增技能音效與普攻音效 storage refs"
```

---

## Task 2: 新增 `useAbilitySound` Composable

**Files:**
- Create: `src/composables/useAbilitySound.ts`

- [ ] **Step 1: 建立 composable 檔案**

建立 `src/composables/useAbilitySound.ts`，內容如下：

```ts
import { createNotification } from '~/composables/useNotification'
import { soundNormalAttack, soundTriggerAbilities } from '~/logic'

export function toggleAbilitySound(id: string) {
  const index = soundTriggerAbilities.value.indexOf(id)
  if (index >= 0)
    soundTriggerAbilities.value.splice(index, 1)
  else
    soundTriggerAbilities.value.push(id)
}

export function isAbilitySoundEnabled(id: string) {
  return soundTriggerAbilities.value.includes(id)
}

export function checkAndPlayAbilitySound(id: string | undefined) {
  if (id && soundTriggerAbilities.value.includes(id))
    createNotification({ message: '技能触发提示', sound: 'tip' })
}

export function checkAndPlayNormalAttackSound() {
  if (soundNormalAttack.value)
    createNotification({ message: '攻击已发出', sound: 'warning' })
}
```

- [ ] **Step 2: Commit**

```bash
git add src/composables/useAbilitySound.ts
git commit -m "feat(sound): 新增 useAbilitySound composable"
```

---

## Task 3: 在 `useBattleLog.ts` 加入觸發點

**Files:**
- Modify: `src/composables/useBattleLog.ts`

觸發點有兩處：
1. **技能觸發**：`type === 'ability'` 區塊，actionList.push 之後（原始行號約 767）
2. **普攻觸發**：`type === 'normal'` 區塊，normalAttackInfo 設定之後（原始行號約 835）

- [ ] **Step 1: 在檔案頂部加入 import**

在 `useBattleLog.ts` 既有的 import 區塊末尾加入：

```ts
import { checkAndPlayAbilitySound, checkAndPlayNormalAttackSound } from '~/composables/useAbilitySound'
```

- [ ] **Step 2: 修改 ability 區塊，抽出 id 並觸發音效**

找到 `type === 'ability'` 的 actionList.push（目前約 767 行），改成：

```ts
const pushedId = hit.subAbility
  ? hit.subAbility.find(a => a.index === String(payload.ability_sub_param[0]))?.id
  : hit.id
currentRaid.actionQueue.at(-1)?.actionList.push({
  ...hit,
  icon: hit.subAbility ? hit.subAbility.find(a => a.index === String(payload.ability_sub_param[0]))?.icon : hit.icon,
  id: pushedId,
  aim,
})
checkAndPlayAbilitySound(pushedId)
```

- [ ] **Step 3: 修改 normal 區塊，觸發普攻音效**

找到 `type === 'normal'` 的區塊（目前約 830 行），改成：

```ts
if (type === 'normal') {
  handleNormalAttackJson(data)
  const index = dieIndex !== -1 ? -1 : -2
  if (currentRaid.actionQueue.at(index)) {
    currentRaid.actionQueue.at(index)!.actionList.push({ icon: 'attack', id: 'attack', type: 'attack' })
    currentRaid.actionQueue.at(index)!.normalAttackInfo = { ...battleInfo.value.normalAttackInfo! }
    checkAndPlayNormalAttackSound()
  }
}
```

- [ ] **Step 4: Commit**

```bash
git add src/composables/useBattleLog.ts
git commit -m "feat(sound): 在 handleAttackResultJson 加入技能與普攻音效觸發"
```

---

## Task 4: sidePanel ActionList UI

**Files:**
- Modify: `src/views/sidePanel/views/combat/components/ActionList.vue`

目前約第 37 行的 `<img h-47px :src="getActionIcon(action)">` 只顯示 icon，無點擊互動。

- [ ] **Step 1: 修改技能 img，加入 click handler 與 ring highlight**

`toggleAbilitySound` 和 `isAbilitySoundEnabled` 來自 `src/composables/useAbilitySound.ts`，已被 unplugin-auto-import 自動引入，無需手動 import。

將原本的：

```html
<img h-47px :src="getActionIcon(action)">
```

改成：

```html
<img
  h-47px
  :src="getActionIcon(action)"
  :class="action.type === 'ability' && action.id ? [isAbilitySoundEnabled(action.id) ? 'ring-2 ring-yellow-4' : '', 'cursor-pointer rounded-sm'] : ''"
  @click.stop="action.type === 'ability' && action.id && toggleAbilitySound(action.id)"
>
```

- [ ] **Step 2: Commit**

```bash
git add src/views/sidePanel/views/combat/components/ActionList.vue
git commit -m "feat(sound): sidePanel ActionList 加入技能音效訂閱點擊互動"
```

---

## Task 5: combatPanel ActionList UI

**Files:**
- Modify: `src/views/combatPanel/components/ActionList.vue`

此元件比 sidePanel 版多一項：普通攻擊音效 toggle，放在 scrollbar 上方。
`soundNormalAttack` 是 storage ref，需明確 import（非 composable，不會自動引入）。

- [ ] **Step 1: 在 `<script setup>` 加入 import**

在 `src/views/combatPanel/components/ActionList.vue` 的 script setup 區塊，`battleInfo, battleRecord, combatPanelSetting` import 行後加入：

```ts
import { soundNormalAttack } from '~/logic'
```

- [ ] **Step 2: 修改技能 img，加入 click handler 與 ring highlight**

找到 `<img h-47px :src="getActionIcon(action)">` 那行（約第 69 行），改成：

```html
<img
  h-47px
  :src="getActionIcon(action)"
  :class="action.type === 'ability' && action.id ? [isAbilitySoundEnabled(action.id) ? 'ring-2 ring-yellow-4' : '', 'cursor-pointer rounded-sm'] : ''"
  @click.stop="action.type === 'ability' && action.id && toggleAbilitySound(action.id)"
>
```

- [ ] **Step 3: 在 scrollbar 上方加入普攻音效 toggle**

找到 `<el-scrollbar` 開頭那行（約第 33 行），在它之前插入：

```html
<div flex items-center gap-2 px-3 py-6px bg-neutral-8 border-b="1 solid #414243">
  <el-switch v-model="soundNormalAttack" size="small" />
  <span text-sm text-neutral-3>攻擊後提示音</span>
</div>
```

- [ ] **Step 4: Commit**

```bash
git add src/views/combatPanel/components/ActionList.vue
git commit -m "feat(sound): combatPanel ActionList 加入技能音效訂閱與普攻音效開關"
```

---

## Task 6: Build 驗證

- [ ] **Step 1: 執行 dev build**

```bash
pnpm ba
```

預期：build 成功，無 TypeScript 錯誤，無 lint 錯誤。

- [ ] **Step 2: 若有錯誤，根據錯誤訊息修正**

常見問題：
- `checkAndPlayAbilitySound` / `checkAndPlayNormalAttackSound` import 路徑錯誤 → 確認 `~/composables/useAbilitySound`
- `soundNormalAttack` 未 import → 確認 `from '~/logic'`
- UnoCSS ring 色彩 class 語法 → 參考現有 `ring-neutral-7` 用法

- [ ] **Step 3: Commit 修正（若有）**

```bash
git add -p
git commit -m "fix(sound): 修正 build 錯誤"
```
