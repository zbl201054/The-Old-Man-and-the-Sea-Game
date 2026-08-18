# 《老人与海》互动叙事游戏 —— 成就系统设计文档


## 一、设计原则

1. **激励探索**：鼓励玩家尝试不同的选择路径、数值组合与结局，提升多周目重玩价值。
2. **即时反馈**：在达成条件的瞬间弹出“成就解锁”提示（Toast 通知），增强爽感。
3. **轻量化 UI**：成就面板与现有“结局图鉴”并列，采用折叠面板或侧边栏，不干扰沉浸式叙事。
4. **数据持久化**：成就进度与解锁状态随 `localStorage` 存档一同保存，换浏览器或清缓存前永久保留。
5. **分级体系**：成就分为 **青铜（易）**、**白银（中）**、**黄金（难）**、**传说（隐藏/极难）** 四个稀有度，对应不同图标和奖励反馈。


## 二、成就列表总览（共 18 项）

| 编号 | 成就名称 | 稀有度 | 解锁条件 | 解锁提示语 |
|------|----------|--------|----------|------------|
| A01 | **初试锋芒** | 青铜 | 完成任意 1 个结局（首次通关） | “海的第一课，你已及格。” |
| A02 | **远海行者** | 青铜 | 游戏累计进行 3 次“再次出海”（≥3 周目） | “船底已沾满三个海港的沙。” |
| A03 | **时空旅人** | 青铜 | 使用“存档”或“读档”功能任意 1 次 | “你握住了时间的舵盘。” |
| A04 | **坚韧不拔** | 白银 | 单次通关中，**精神值** 达到上限 8（满格） | “他的眼里有火，骨里有铁。” |
| A05 | **海之哲人** | 白银 | 单次通关中，**感悟值** 达到上限 5（满格） | “你听见了大海深处的回响。” |
| A06 | **残血战神** | 白银 | **体力值** 降至 0 或 1 但最终未触发 E3（坚持通关） | “只剩最后一口气，也要把船划回家。” |
| A07 | **S/L 大师** | 白银 | 累计使用存档/读档功能 ≥ 10 次 | “你不相信命运，你相信 SL。” |
| A08 | **鲨鱼克星** | 白银 | 在 N18、N19、N20、N22 这 4 个鲨鱼节点**全部**选择正面战斗（绝不投降） | “鲨鱼们回家做了三天噩梦。” |
| A09 | **和平主义者** | 白银 | 在 N18、N19、N20、N22 这 4 个鲨鱼节点**全部**选择妥协/放弃/回避 | “你放下了屠刀，大海给了你拥抱。” |
| A10 | **狮王传承** | 黄金 | 解锁隐藏结局 E6（狮子的传人） | “古老的狮子在梦中低语。” |
| A11 | **慈悲之海** | 黄金 | 解锁特殊结局 E12（鱼之泪） | “放生一条鱼，养起整片海。” |
| A12 | **硬汉之魂** | 黄金 | 解锁金色结局 E1（硬汉的荣耀） | “一个人可以被毁灭，但不能被打败。” |
| A13 | **海的遗产** | 黄金 | 解锁金色结局 E9（渔夫的遗产） | “财富不在手中，而在心里。” |
| A14 | **孤高之王** | 白银 | 解锁结局 E2（孤独的雄狮）且全程未接受马诺林帮助（`hasBoy` 标记为假） | “王者的路，总是越走越安静。” |
| A15 | **全收集·海王** | 传说 | 解锁全部 12 个结局（图鉴集齐） | “你读完了海的每一种表情。” |
| A16 | **面面俱到** | 黄金 | 在一周目内，精神≥6、体力≥3、感悟≥4（三项均衡达标） | “勇猛、顽强与智慧，你全都要。” |
| A17 | **极限求生** | 传说 | 体力值连续低于等于 1 并存活超过 5 个节点（含鲨鱼战）后通关 | “在死神的渔网边跳了一支舞。” |
| A18 | **海明威的凝视** | 传说 | 单次通关中，在任意“放弃/投降”节点选择**拒绝投降**不少于 5 次，且最终达成金色结局 | “海明威在远方点了点头。” |


## 三、成就条件判定逻辑（伪代码）

开发时，只需要在 `renderNode()` 或 `makeChoice()` 中调用 `checkAchievements()` 函数，引擎会自动轮询所有成就条件。

```javascript
// ============================================================
// 成就系统核心判定逻辑（ES5 兼容）
// ============================================================

// 1. 数据结构
var ACHIEVEMENT_DEFS = {
    'A01': { id:'A01', name:'初试锋芒', grade:'bronze', desc:'完成任意1个结局', unlocked:false },
    'A02': { id:'A02', name:'远海行者', grade:'bronze', desc:'累计出海≥3周目', unlocked:false },
    // ... 其余 16 个
};

// 2. 成就进度追踪器（存放在 gameState 中）
// gameState.achievementProgress = {
//     totalPlaythroughs: 0,          // 总通关次数
//     totalSaves: 0,                 // 存档次数
//     totalLoads: 0,                 // 读档次数
//     sharkFightCount: 0,            // 鲨鱼战中选择战斗的次数
//     sharkFleeCount: 0,             // 鲨鱼战中选择逃跑的次数
//     nodesSurvivedLowStamina: 0,    // 体力≤1时的节点存活数
//     rejectSurrenderCount: 0,       // 拒绝投降的次数
//     // 还可以添加更多计数器...
// };

// 3. 条件检测函数（示例）
function checkAchievements() {
    var p = gameState.achievementProgress;
    var s = gameState;
    var unlocked = gameState.achievements || [];

    // ---------- A01: 初试锋芒 ----------
    if (s.endingsUnlocked.length >= 1 && unlocked.indexOf('A01') === -1) {
        unlockAchievement('A01');
    }

    // ---------- A02: 远海行者 ----------
    if (p.totalPlaythroughs >= 3 && unlocked.indexOf('A02') === -1) {
        unlockAchievement('A02');
    }

    // ---------- A04: 坚韧不拔 ----------
    if (s.willpower >= 8 && unlocked.indexOf('A04') === -1) {
        unlockAchievement('A04');
    }

    // ---------- A05: 海之哲人 ----------
    if (s.insight >= 5 && unlocked.indexOf('A05') === -1) {
        unlockAchievement('A05');
    }

    // ---------- A06: 残血战神 ----------
    if (s.stamina <= 1 && s.endingsUnlocked.length > 0 && 
        s.endingsUnlocked.indexOf('E3') === -1 && unlocked.indexOf('A06') === -1) {
        // 注意：需要确保不触发E3结局，且已经通关
        unlockAchievement('A06');
    }

    // ---------- A08: 鲨鱼克星 ----------
    if (p.sharkFightCount >= 4 && unlocked.indexOf('A08') === -1) {
        unlockAchievement('A08');
    }

    // ---------- A09: 和平主义者 ----------
    if (p.sharkFleeCount >= 4 && unlocked.indexOf('A09') === -1) {
        unlockAchievement('A09');
    }

    // ---------- A10~A13: 特定结局解锁 ----------
    // (在 renderEnding 中触发更准确)
    // 但这里也可以做轮询

    // ---------- A15: 全收集 ----------
    if (s.endingsUnlocked.length >= 12 && unlocked.indexOf('A15') === -1) {
        unlockAchievement('A15');
    }

    // ---------- A16: 面面俱到 ----------
    if (s.willpower >= 6 && s.stamina >= 3 && s.insight >= 4 && unlocked.indexOf('A16') === -1) {
        unlockAchievement('A16');
    }

    // ---------- A17: 极限求生 ----------
    if (p.nodesSurvivedLowStamina >= 5 && s.endingsUnlocked.length > 0 && unlocked.indexOf('A17') === -1) {
        unlockAchievement('A17');
    }

    // ---------- A18: 海明威的凝视 ----------
    if (p.rejectSurrenderCount >= 5 && s.endingsUnlocked.indexOf('E1') !== -1 && unlocked.indexOf('A18') === -1) {
        unlockAchievement('A18');
    }
}

// 4. 解锁函数（触发弹窗 + 存档）
function unlockAchievement(id) {
    var def = ACHIEVEMENT_DEFS[id];
    if (!def) return;
    if (gameState.achievements.indexOf(id) !== -1) return;

    gameState.achievements.push(id);
    def.unlocked = true;

    // 触发 Toast 通知
    showToast('🏅 成就解锁：' + def.name, def.grade);

    // 保存到本地
    saveGameSilent();

    // 更新成就面板
    updateAchievementUI();
}
```


## 四、UI 交互设计

### 4.1 成就通知（Toast 弹窗）

在游戏主界面右上方（或屏幕中央顶部）弹出优雅的成就提示：

```html
<!-- 成就通知容器（置于页面最顶层） -->
<div id="achievement-toast" style="display:none;position:fixed;top:30px;right:30px;background:rgba(10,30,38,0.95);border:2px solid #c9a86c;border-radius:12px;padding:18px 28px;z-index:9999;box-shadow:0 8px 40px rgba(0,0,0,0.9);max-width:380px;backdrop-filter:blur(4px);">
    <div style="display:flex;align-items:center;gap:16px;">
        <span id="toast-icon" style="font-size:36px;">🏅</span>
        <div>
            <div id="toast-title" style="font-size:18px;font-weight:bold;color:#f0d6a8;">成就解锁</div>
            <div id="toast-name" style="font-size:16px;color:#e8dcc8;">硬汉之魂</div>
            <div id="toast-grade" style="font-size:12px;color:#8aa9b3;margin-top:4px;">黄金</div>
        </div>
    </div>
</div>
```

**显示逻辑**：
- 弹窗从右边缘滑入（`translateX(120%)` → `0`），停留 4 秒后淡出消失。
- 稀有度颜色边框：青铜（#b08d6b）、白银（#c0c0c0）、黄金（#ffd700）、传说（#ff6b6b 闪动）。

### 4.2 成就面板（与结局图鉴并列）

在底部“结局图鉴”旁边，增加“🏆 成就”按钮，点击展开成就列表。

```html
<div style="display:flex;gap:12px;margin-top:8px;border-top:1px solid #1f4350;padding-top:12px;">
    <span id="gallery-title" onclick="toggleGallery()" style="cursor:pointer;color:#6a8f9a;font-size:14px;">📖 结局图鉴</span>
    <span id="achievement-title" onclick="toggleAchievement()" style="cursor:pointer;color:#6a8f9a;font-size:14px;">🏅 成就（<span id="ach-count">0</span>/18）</span>
</div>
<div id="achievement-grid" style="display:none;margin-top:10px;display:flex;flex-wrap:wrap;gap:6px;">
    <!-- 动态生成成就徽章 -->
</div>
```

每个成就徽章样式：
```css
.ach-badge {
    display: inline-flex; align-items: center; gap: 6px;
    padding: 4px 12px 4px 8px;
    border-radius: 30px; font-size: 12px;
    background: #0a1e26; border: 1px solid #2d5a68; color: #6a8f9a;
}
.ach-badge.unlocked {
    border-color: var(--ach-color);
    color: #f0e6d0;
    background: #1d3f4b;
}
.ach-badge .ach-icon { font-size: 16px; }
.ach-badge.locked .ach-icon { filter: grayscale(1); opacity: 0.4; }
/* 稀有度颜色变量 */
.ach-badge.gold .ach-border { border-color: #ffd700; }
.ach-badge.silver .ach-border { border-color: #c0c0c0; }
.ach-badge.bronze .ach-border { border-color: #b08d6b; }
.ach-badge.legend .ach-border { border-color: #ff6b6b; animation: legend-glow 1.5s infinite; }
@keyframes legend-glow { 0%{box-shadow:0 0 4px #ff6b6b;} 50%{box-shadow:0 0 16px #ff6b6b;} 100%{box-shadow:0 0 4px #ff6b6b;} }
```


## 五、深度集成方案（与主代码无缝衔接）

### 5.1 对 `gameState` 的改造

```javascript
// 在游戏状态中增加成就相关字段
var gameState = {
    // ...原有字段
    achievements: [],          // 已解锁的成就ID列表
    achievementProgress: {
        totalPlaythroughs: 0,
        totalSaves: 0,
        totalLoads: 0,
        sharkFightCount: 0,
        sharkFleeCount: 0,
        nodesSurvivedLowStamina: 0,
        rejectSurrenderCount: 0,
        maxWillpowerReached: 0,
        maxInsightReached: 0,
        // 用于判断残血存活节点数的临时计数
        _lowStaminaStreak: 0
    }
};
```

### 5.2 在关键函数中“埋点”

| 函数名 | 埋点内容 |
|--------|----------|
| `makeChoice(index)` | 检测鲨鱼节点（N18~N22）的选项，累加 `sharkFightCount` 或 `sharkFleeCount`；检测“拒绝投降”选项（即非放弃类选项），累加 `rejectSurrenderCount` |
| `renderNode(nodeId)` | 检测当前 `stamina ≤ 1` 时，累加 `nodesSurvivedLowStamina`；若 `stamina > 1`，重置 `_lowStaminaStreak = 0` |
| `renderEnding(endingId)` | 每次到达结局时，`totalPlaythroughs += 1`；检测特定结局ID（E1,E6,E9,E12,E2），直接解锁对应的成就 |
| `saveGame()` / `loadGame()` | `totalSaves += 1` / `totalLoads += 1` |
| `restartGame()` | 在重开后调用 `checkAchievements()`（因为某些成就需要在特定周目边界触发） |

### 5.3 成就进度可视化（Hover 提示）

鼠标悬停到未解锁的成就徽章上时，显示达成条件（略带神秘感）：

```html
<span class="ach-badge locked" title="❓ 条件：解锁全部12个结局">
    ❓ 全收集·海王
</span>
<span class="ach-badge unlocked gold" title="✅ 已解锁">
    🏆 硬汉之魂
</span>
```


## 六、18 项成就详细数据表（可直接复制用于 JSON）

| ID | 名称 | 稀有度 | 解锁条件（代码逻辑） | 提示文本 |
|----|------|--------|----------------------|----------|
| A01 | 初试锋芒 | 青铜 | `endingsUnlocked.length >= 1` | “海的第一课，你已及格。” |
| A02 | 远海行者 | 青铜 | `achievementProgress.totalPlaythroughs >= 3` | “船底已沾满三个海港的沙。” |
| A03 | 时空旅人 | 青铜 | `achievementProgress.totalSaves + totalLoads >= 1` | “你握住了时间的舵盘。” |
| A04 | 坚韧不拔 | 白银 | `willpower >= 8` | “他的眼里有火，骨里有铁。” |
| A05 | 海之哲人 | 白银 | `insight >= 5` | “你听见了大海深处的回响。” |
| A06 | 残血战神 | 白银 | `stamina <= 1` 且通关且未触发 E3 | “只剩最后一口气，也要把船划回家。” |
| A07 | S/L 大师 | 白银 | `achievementProgress.totalSaves + totalLoads >= 10` | “你不相信命运，你相信 SL。” |
| A08 | 鲨鱼克星 | 白银 | 在 N18,N19,N20,N22 **全部**选战斗 | “鲨鱼们回家做了三天噩梦。” |
| A09 | 和平主义者 | 白银 | 在 N18,N19,N20,N22 **全部**选放弃/回避 | “你放下了屠刀，大海给了你拥抱。” |
| A10 | 狮王传承 | 黄金 | 解锁结局 E6（狮子的传人） | “古老的狮子在梦中低语。” |
| A11 | 慈悲之海 | 黄金 | 解锁结局 E12（鱼之泪） | “放生一条鱼，养起整片海。” |
| A12 | 硬汉之魂 | 黄金 | 解锁结局 E1（硬汉的荣耀） | “一个人可以被毁灭，但不能被打败。” |
| A13 | 海的遗产 | 黄金 | 解锁结局 E9（渔夫的遗产） | “财富不在手中，而在心里。” |
| A14 | 孤高之王 | 白银 | 解锁结局 E2 且 `hasBoy` 标记不存在 | “王者的路，总是越走越安静。” |
| A15 | 全收集·海王 | 传说 | `endingsUnlocked.length >= 12` | “你读完了海的每一种表情。” |
| A16 | 面面俱到 | 黄金 | `willpower >= 6 && stamina >= 3 && insight >= 4` | “勇猛、顽强与智慧，你全都要。” |
| A17 | 极限求生 | 传说 | `nodesSurvivedLowStamina >= 5`（连续或累计）且通关 | “在死神的渔网边跳了一支舞。” |
| A18 | 海明威的凝视 | 传说 | `rejectSurrenderCount >= 5` 且通关结局 E1 | “海明威在远方点了点头。” |

> **提示**：`totalPlaythroughs` 在每次进入结局界面时加 1（`renderEnding` 触发）；`nodesSurvivedLowStamina` 在每进入一个新节点且 `stamina <= 1` 时加 1，若 `stamina > 1` 则不清零（累计制），以降低难度。


## 七、完整前置代码（可直接粘贴至现有 HTML）

将以下代码片段插入到 `<script>` 中 `window.onload` 之前，即可启用成就系统（需配合上文 UI 改动）。

```javascript
// ============================================================
//  成就系统模块 (ES5 兼容)
// ============================================================

// ----- 7.1 成就定义 -----
var ACHIEVEMENT_LIST = [
    { id:'A01', name:'初试锋芒', grade:'bronze', desc:'完成任意1个结局', hint:'海的第一课，你已及格。' },
    { id:'A02', name:'远海行者', grade:'bronze', desc:'累计出海≥3周目', hint:'船底已沾满三个海港的沙。' },
    { id:'A03', name:'时空旅人', grade:'bronze', desc:'使用存档/读档1次', hint:'你握住了时间的舵盘。' },
    { id:'A04', name:'坚韧不拔', grade:'silver', desc:'精神值达到上限8', hint:'他的眼里有火，骨里有铁。' },
    { id:'A05', name:'海之哲人', grade:'silver', desc:'感悟值达到上限5', hint:'你听见了大海深处的回响。' },
    { id:'A06', name:'残血战神', grade:'silver', desc:'体力≤1且通关(非E3)', hint:'只剩最后一口气，也要把船划回家。' },
    { id:'A07', name:'S/L大师', grade:'silver', desc:'累计存档/读档≥10次', hint:'你不相信命运，你相信SL。' },
    { id:'A08', name:'鲨鱼克星', grade:'silver', desc:'鲨鱼节点全选战斗', hint:'鲨鱼们回家做了三天噩梦。' },
    { id:'A09', name:'和平主义者', grade:'silver', desc:'鲨鱼节点全选放弃', hint:'你放下了屠刀，大海给了你拥抱。' },
    { id:'A10', name:'狮王传承', grade:'gold', desc:'解锁结局E6', hint:'古老的狮子在梦中低语。' },
    { id:'A11', name:'慈悲之海', grade:'gold', desc:'解锁结局E12', hint:'放生一条鱼，养起整片海。' },
    { id:'A12', name:'硬汉之魂', grade:'gold', desc:'解锁结局E1', hint:'一个人可以被毁灭，但不能被打败。' },
    { id:'A13', name:'海的遗产', grade:'gold', desc:'解锁结局E9', hint:'财富不在手中，而在心里。' },
    { id:'A14', name:'孤高之王', grade:'silver', desc:'解锁E2且未接受帮助', hint:'王者的路，总是越走越安静。' },
    { id:'A15', name:'全收集·海王', grade:'legend', desc:'解锁全部12结局', hint:'你读完了海的每一种表情。' },
    { id:'A16', name:'面面俱到', grade:'gold', desc:'精神≥6，体力≥3，感悟≥4', hint:'勇猛、顽强与智慧，你全都要。' },
    { id:'A17', name:'极限求生', grade:'legend', desc:'体力≤1存活≥5节点', hint:'在死神的渔网边跳了一支舞。' },
    { id:'A18', name:'海明威的凝视', grade:'legend', desc:'拒绝投降≥5次且E1结局', hint:'海明威在远方点了点头。' }
];

// ----- 7.2 Toast 通知函数 -----
function showAchievementToast(name, grade) {
    var toast = document.getElementById('achievement-toast');
    if (!toast) {
        // 若未创建DOM，则动态创建
        createToastElements();
        toast = document.getElementById('achievement-toast');
    }
    var icon = document.getElementById('toast-icon');
    var nameEl = document.getElementById('toast-name');
    var gradeEl = document.getElementById('toast-grade');
    var titleEl = document.getElementById('toast-title');
    
    var gradeMap = { bronze:'青铜', silver:'白银', gold:'黄金', legend:'传说' };
    var iconMap = { bronze:'🥉', silver:'🥈', gold:'🥇', legend:'👑' };
    
    icon.innerHTML = iconMap[grade] || '🏅';
    nameEl.innerHTML = name;
    gradeEl.innerHTML = gradeMap[grade] || '';
    titleEl.innerHTML = '🏆 成就解锁！';
    toast.style.borderColor = grade === 'legend' ? '#ff6b6b' : 
                              grade === 'gold' ? '#ffd700' : 
                              grade === 'silver' ? '#c0c0c0' : '#b08d6b';
    
    toast.style.display = 'block';
    toast.style.transform = 'translateX(0)';
    toast.style.opacity = '1';
    toast.style.transition = 'all 0.4s ease';
    
    // 自动隐藏
    clearTimeout(toast._timer);
    toast._timer = setTimeout(function() {
        toast.style.opacity = '0';
        toast.style.transform = 'translateX(120%)';
        setTimeout(function() {
            toast.style.display = 'none';
        }, 500);
    }, 4500);
}

function createToastElements() {
    var div = document.createElement('div');
    div.id = 'achievement-toast';
    div.style.cssText = 'display:none;position:fixed;top:24px;right:24px;background:rgba(10,30,38,0.96);border:2px solid #c9a86c;border-radius:16px;padding:20px 28px;z-index:9999;box-shadow:0 12px 48px rgba(0,0,0,0.9);max-width:360px;transform:translateX(120%);opacity:0;transition:all 0.4s cubic-bezier(0.34,1.56,0.64,1);backdrop-filter:blur(6px);';
    div.innerHTML = '<div style="display:flex;align-items:center;gap:16px;">' +
        '<span id="toast-icon" style="font-size:40px;">🏅</span>' +
        '<div><div id="toast-title" style="font-size:16px;font-weight:bold;color:#f0d6a8;">成就解锁</div>' +
        '<div id="toast-name" style="font-size:20px;font-weight:bold;color:#e8dcc8;margin:4px 0;">硬汉之魂</div>' +
        '<div id="toast-grade" style="font-size:13px;color:#8aa9b3;">黄金</div></div></div>';
    document.body.appendChild(div);
}

// ----- 7.3 成就解锁核心逻辑 -----
function unlockAchievement(id) {
    if (gameState.achievements.indexOf(id) !== -1) return;
    var def = null;
    for (var i=0; i<ACHIEVEMENT_LIST.length; i++) {
        if (ACHIEVEMENT_LIST[i].id === id) { def = ACHIEVEMENT_LIST[i]; break; }
    }
    if (!def) return;
    
    gameState.achievements.push(id);
    showAchievementToast(def.name, def.grade);
    saveGameSilent();
    updateAchievementUI();
    updateUI(); // 刷新计数
}

// ----- 7.4 成就检测（每次节点渲染或选择后调用）-----
function checkAchievements() {
    if (!gameState) return;
    var p = gameState.achievementProgress || {};
    var s = gameState;
    var unlocked = s.achievements || [];

    // A01
    if (s.endingsUnlocked.length >= 1 && unlocked.indexOf('A01') === -1) unlockAchievement('A01');
    // A02
    if ((p.totalPlaythroughs || 0) >= 3 && unlocked.indexOf('A02') === -1) unlockAchievement('A02');
    // A03
    if ((p.totalSaves || 0) + (p.totalLoads || 0) >= 1 && unlocked.indexOf('A03') === -1) unlockAchievement('A03');
    // A04
    if (s.willpower >= 8 && unlocked.indexOf('A04') === -1) unlockAchievement('A04');
    // A05
    if (s.insight >= 5 && unlocked.indexOf('A05') === -1) unlockAchievement('A05');
    // A06
    if (s.stamina <= 1 && s.endingsUnlocked.length > 0 && s.endingsUnlocked.indexOf('E3') === -1 && unlocked.indexOf('A06') === -1) {
        unlockAchievement('A06');
    }
    // A07
    if ((p.totalSaves || 0) + (p.totalLoads || 0) >= 10 && unlocked.indexOf('A07') === -1) unlockAchievement('A07');
    // A08
    if ((p.sharkFightCount || 0) >= 4 && unlocked.indexOf('A08') === -1) unlockAchievement('A08');
    // A09
    if ((p.sharkFleeCount || 0) >= 4 && unlocked.indexOf('A09') === -1) unlockAchievement('A09');
    // A10
    if (s.endingsUnlocked.indexOf('E6') !== -1 && unlocked.indexOf('A10') === -1) unlockAchievement('A10');
    // A11
    if (s.endingsUnlocked.indexOf('E12') !== -1 && unlocked.indexOf('A11') === -1) unlockAchievement('A11');
    // A12
    if (s.endingsUnlocked.indexOf('E1') !== -1 && unlocked.indexOf('A12') === -1) unlockAchievement('A12');
    // A13
    if (s.endingsUnlocked.indexOf('E9') !== -1 && unlocked.indexOf('A13') === -1) unlockAchievement('A13');
    // A14
    if (s.endingsUnlocked.indexOf('E2') !== -1) {
        var hasBoy = false;
        for (var f=0; f<s.flags.length; f++) { if (s.flags[f] === 'hasBoy') { hasBoy = true; break; } }
        if (!hasBoy && unlocked.indexOf('A14') === -1) unlockAchievement('A14');
    }
    // A15
    if (s.endingsUnlocked.length >= 12 && unlocked.indexOf('A15') === -1) unlockAchievement('A15');
    // A16
    if (s.willpower >= 6 && s.stamina >= 3 && s.insight >= 4 && unlocked.indexOf('A16') === -1) unlockAchievement('A16');
    // A17
    if ((p.nodesSurvivedLowStamina || 0) >= 5 && s.endingsUnlocked.length > 0 && unlocked.indexOf('A17') === -1) {
        unlockAchievement('A17');
    }
    // A18
    if ((p.rejectSurrenderCount || 0) >= 5 && s.endingsUnlocked.indexOf('E1') !== -1 && unlocked.indexOf('A18') === -1) {
        unlockAchievement('A18');
    }
    
    updateAchievementUI();
}

// ----- 7.5 UI 更新 -----
function updateAchievementUI() {
    var grid = document.getElementById('achievement-grid');
    if (!grid) return;
    var count = gameState.achievements ? gameState.achievements.length : 0;
    var total = ACHIEVEMENT_LIST.length;
    document.getElementById('ach-count').innerHTML = count;
    
    grid.innerHTML = '';
    for (var i=0; i<ACHIEVEMENT_LIST.length; i++) {
        var def = ACHIEVEMENT_LIST[i];
        var unlocked = gameState.achievements.indexOf(def.id) !== -1;
        var span = document.createElement('span');
        span.className = 'ach-badge ' + (unlocked ? 'unlocked ' + def.grade : 'locked');
        span.innerHTML = (unlocked ? '✅ ' : '❓ ') + def.name;
        span.title = unlocked ? '已解锁：' + def.desc : '🔒 未解锁：' + def.desc;
        grid.appendChild(span);
    }
}

function toggleAchievement() {
    var grid = document.getElementById('achievement-grid');
    if (!grid) return;
    if (grid.style.display === 'none' || grid.style.display === '') {
        grid.style.display = 'flex';
        updateAchievementUI();
    } else {
        grid.style.display = 'none';
    }
}

// 在游戏启动时，确保成就进度对象存在
function initAchievementSystem() {
    if (!gameState.achievements) gameState.achievements = [];
    if (!gameState.achievementProgress) {
        gameState.achievementProgress = {
            totalPlaythroughs: 0,
            totalSaves: 0,
            totalLoads: 0,
            sharkFightCount: 0,
            sharkFleeCount: 0,
            nodesSurvivedLowStamina: 0,
            rejectSurrenderCount: 0
        };
    }
    // 创建Toast DOM
    if (!document.getElementById('achievement-toast')) {
        createToastElements();
    }
    // 创建成就面板按钮（如果不存在则动态追加）
    var galleryDiv = document.getElementById('gallery');
    if (galleryDiv && !document.getElementById('achievement-title')) {
        var titleDiv = document.createElement('div');
        titleDiv.style.cssText = 'display:flex;gap:16px;margin-top:12px;border-top:1px solid #1f4350;padding-top:12px;';
        titleDiv.innerHTML = '<span id="gallery-title" onclick="toggleGallery()" style="cursor:pointer;color:#6a8f9a;font-size:14px;">📖 结局图鉴</span>' +
                             '<span id="achievement-title" onclick="toggleAchievement()" style="cursor:pointer;color:#6a8f9a;font-size:14px;">🏅 成就（<span id="ach-count">0</span>/18）</span>';
        galleryDiv.insertBefore(titleDiv, galleryDiv.firstChild);
        var grid = document.createElement('div');
        grid.id = 'achievement-grid';
        grid.style.cssText = 'display:none;margin-top:10px;flex-wrap:wrap;gap:6px;';
        galleryDiv.appendChild(grid);
    }
    updateAchievementUI();
}
```

### 在主流程中调用 `checkAchievements()` 的位置：

```javascript
// 在 makeChoice 末尾添加
function makeChoice(index) {
    // ... 原有逻辑 ...
    checkAchievements(); // 新增
}

// 在 renderEnding 中添加埋点
function renderEnding(endingId) {
    // ... 原有逻辑 ...
    gameState.achievementProgress.totalPlaythroughs = (gameState.achievementProgress.totalPlaythroughs || 0) + 1;
    checkAchievements(); // 新增
}

// 在 saveGame / loadGame 中添加计数
function saveGame() {
    gameState.achievementProgress.totalSaves = (gameState.achievementProgress.totalSaves || 0) + 1;
    saveGameSilent();
    checkAchievements();
}
function loadGame() {
    // ... 读取逻辑 ...
    gameState.achievementProgress.totalLoads = (gameState.achievementProgress.totalLoads || 0) + 1;
    checkAchievements();
}
```

### 在鲨鱼节点选择时记录：

在 `N18, N19, N20, N22` 的 `makeChoice` 判定中（可通过检测 `currentNodeObj.id` 实现）：

```javascript
// 在 makeChoice 中增加埋点（放在应用deltas之后）
var nodeId = currentNodeObj ? currentNodeObj.id : '';
var sharkNodes = ['N18','N19','N20','N22'];
if (sharkNodes.indexOf(nodeId) !== -1) {
    var isFight = (choice.text.indexOf('战斗') !== -1 || choice.text.indexOf('刺') !== -1 || 
                   choice.text.indexOf('砸') !== -1 || choice.text.indexOf('拼') !== -1);
    var isFlee = (choice.text.indexOf('放弃') !== -1 || choice.text.indexOf('保命') !== -1 || 
                  choice.text.indexOf('躺') !== -1 || choice.text.indexOf('算了') !== -1);
    if (isFight) gameState.achievementProgress.sharkFightCount = (gameState.achievementProgress.sharkFightCount || 0) + 1;
    if (isFlee) gameState.achievementProgress.sharkFleeCount = (gameState.achievementProgress.sharkFleeCount || 0) + 1;
}
// 检测“拒绝投降” (所有非放弃选项，且不在鲨鱼节点内单独计数)
if (choice.text.indexOf('放弃') === -1 && choice.text.indexOf('保命') === -1 && choice.text.indexOf('算了') === -1 && choice.text.indexOf('躺') === -1) {
    gameState.achievementProgress.rejectSurrenderCount = (gameState.achievementProgress.rejectSurrenderCount || 0) + 1;
}
```

在 `renderNode` 中记录低体力存活：

```javascript
if (gameState.stamina <= 1) {
    gameState.achievementProgress.nodesSurvivedLowStamina = (gameState.achievementProgress.nodesSurvivedLowStamina || 0) + 1;
}
```


## 八、开发排期（成就系统独立模块）

| 阶段 | 内容 | 工时 |
|------|------|------|
| 设计 | 定义 18 项成就与稀有度，编写条件逻辑伪代码 | 0.5 天 |
| 前端 | 创建成就面板 UI + Toast 通知样式 | 0.5 天 |
| 逻辑 | 编写 `checkAchievements` 及埋点代码 | 1 天 |
| 测试 | 逐项验收 18 个成就的触发条件（结合已有的 30 节点） | 1 天 |
| 调优 | 调整部分成就的判定严格度（如 A17 从连续改为累计） | 0.5 天 |
| **总计** | | **3.5 天** |


## 九、附录：稀有度图标映射建议

| 稀有度 | CSS 类 | 图标 | 边框颜色 | 解锁弹窗音效（可选项） |
|--------|--------|------|----------|------------------------|
| 青铜 | `bronze` | 🥉 | #b08d6b | 短促“叮” |
| 白银 | `silver` | 🥈 | #c0c0c0 | 中音“啵” |
| 黄金 | `gold` | 🥇 | #ffd700 | 金色光辉“轰” |
| 传说 | `legend` | 👑 | #ff6b6b（带脉冲动画） | 号角声🎺 |


## 十、总结

本成就系统为《老人与海》互动叙事游戏提供了 **18 个明确的探索目标**，覆盖了：
- 🎯 **结局探索**（黄金/隐藏/传说结局）
- 📊 **数值极限**（精神/体力/感悟）
- 🧭 **行为模式**（战斗狂 vs 和平主义者）
- 🔄 **多周目循环**（通关次数、S/L次数）

所有代码均采用 **ES5** 写法，兼容 IE11，数据随 `localStorage` 持久化。玩家在沉浸于海明威的文学世界时，能不断收到“成就解锁”的惊喜反馈，极大提升游戏的耐玩性与社区分享欲望。

**“海是无限的，成就是有限的——但你永远不知道下一个浪头会带来什么。”** 🌊🏅