# 技能與攻擊提示音功能設計

**日期**: 2026-04-09
**狀態**: 已核准

## 需求概述

玩家可以訂閱特定技能，當該技能在戰鬥中觸發時播放提示音（`tip`）。
玩家也可以開啟普通攻擊提示音，在攻擊發出後播放（`warning`），方便知道可以直接 refresh 跳過動畫。

## 架構

### 觸發時機

聲音必須在 `useBattleLog.ts` 的 `handleAttackResultJson()` 處理新 API response 時觸發，
具體時間點是組好本回合 `actionList` 之後、`push` 進 `battleRecord` 之前。

這確保了：
- 只有真正新進來的 response 才觸發聲音
- 頁面 reload 從 storage 載入舊資料時不會重播

### Storage（`src/logic/storage.ts`）

新增兩個 reactive ref：

```ts
export const soundTriggerAbilities = useWebExtensionStorage<string[]>('soundTriggerAbilities', [])
export const soundNormalAttack = useWebExtensionStorage<boolean>('soundNormalAttack', false)
```

沿用 `specBossBuff` 的相同模式。

### Composable（`src/composables/useAbilitySound.ts`）

新增 composable，提供：

| 函式 | 用途 |
|---|---|
| `toggleAbilitySound(id: string)` | push / remove id，給 ActionList UI 呼叫 |
| `isAbilitySoundEnabled(id: string)` | 判斷技能是否已訂閱，給 UI 顯示 highlight |
| `checkAndPlayAbilitySound(actions: Action[])` | 遍歷 actions，篩選 `type === 'ability' && id` 的行動，有符合 soundTriggerAbilities 的就播 `tip` |
| `checkAndPlayNormalAttackSound()` | 若 soundNormalAttack 為 true，播 `warning` |

`checkAndPlayAbilitySound` 和 `checkAndPlayNormalAttackSound` 只由 `useBattleLog.ts` 在資料處理管線中呼叫，不掛 watcher。

### `useBattleLog.ts` 修改

在 `handleAttackResultJson()` 中，組好回合資料後、push 進 battleRecord 前：

```ts
// 新增（偽代碼）
checkAndPlayAbilitySound(newTurn.actionList)
if (isNormalAttack) checkAndPlayNormalAttackSound()
battleRecord.value.push(newTurn) // 原有邏輯
```

### UI 修改

**ActionList（`sidePanel` 和 `combatPanel` 兩個元件）**：
- 技能 `<img>` 加 `@click.stop="toggleAbilitySound(action.id)"`
- 已訂閱的技能加 highlight（`ring-2 ring-yellow-4`）

**combatPanel ActionList**：
- 在 scrollbar 上方加一行：`el-switch` + label「攻擊後提示音」，綁定 `soundNormalAttack`

## 資料流

```
Game API response
  → useDataCenter.ts routing
  → useBattleLog.handleAttackResultJson()
    → 組建 actionList
    → checkAndPlayAbilitySound(actionList)   ← 技能匹配 → tip 音效
    → checkAndPlayNormalAttackSound()         ← 若開啟 → warning 音效
    → battleRecord.push(newTurn)              ← 寫入 storage
  → UI reactive 更新
```

## 音效對應

| 事件 | 音效 key | 檔名 |
|---|---|---|
| 訂閱技能觸發 | `tip` | `chat_se_1.mp3` |
| 普通攻擊發出 | `warning` | `help_se_2.mp3` |

## 影響範圍

| 檔案 | 變更類型 |
|---|---|
| `src/logic/storage.ts` | 新增 2 個 storage ref |
| `src/composables/useAbilitySound.ts` | 新增檔案 |
| `src/composables/useBattleLog.ts` | 新增呼叫點 |
| `src/views/sidePanel/views/combat/components/ActionList.vue` | 新增 click handler + highlight |
| `src/views/combatPanel/components/ActionList.vue` | 新增 click handler + highlight + toggle |
