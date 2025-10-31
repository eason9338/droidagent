# DroidAgent 系統架構分析（務實版）

## 目錄
1. [系統概述](#1-系統概述)
2. [APK 靜態分析與初始化](#2-apk-靜態分析與初始化)
3. [運行時 GUI 狀態獲取](#3-運行時-gui-狀態獲取)
4. [多 LLM 協同架構](#4-多-llm-協同架構)
5. [記憶系統設計](#5-記憶系統設計)
6. [任務規劃機制](#6-任務規劃機制)
7. [動作執行與批判](#7-動作執行與批判)
8. [知識累積與檢索](#8-知識累積與檢索)
9. [完整執行流程](#9-完整執行流程)

---

## 1. 系統概述

### 1.1 核心能力

DroidAgent 是一個**基於多 LLM 協同的 Android GUI 自動化測試框架**，具備以下核心能力：

- **自動探索**：基於 Persona（用戶角色）驅動的智能探索
- **任務導向**：將測試拆分為有意義的任務單元（Task-based）
- **知識累積**：記錄並重用 UI 元件行為知識
- **自我批判**：定期評估當前執行策略並調整
- **測試腳本生成**：自動生成可重現的 UIAutomator2 測試腳本

### 1.2 系統架構圖

```
┌─────────────────────────────────────────────────────────────────┐
│                      DroidAgent 主程式                           │
│                   (scripts/run_droidagent.py)                    │
└─────────────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┼────────────────────┐
         │                    │                    │
         ▼                    ▼                    ▼
┌────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  APK 靜態分析   │  │  Android 設備     │  │  Agent 主循環     │
│  (Androguard)  │  │  (ADB + DroidBot)│  │  (4 LLM 組件)    │
└────────────────┘  └──────────────────┘  └──────────────────┘
         │                    │                    │
         └────────────────────┼────────────────────┘
                              ▼
                   ┌─────────────────────┐
                   │   記憶系統 (Memory)  │
                   │   - ChromaDB 向量庫  │
                   │   - 4 種記憶類型     │
                   └─────────────────────┘
```

---

## 2. APK 靜態分析與初始化

### 2.1 APK 解析流程

**關鍵檔案**：`droidbot/droidbot/app.py`

當系統啟動時，首先使用 **Androguard** 庫解析 APK 檔案：

```python
from androguard.core.apk import APK

self.apk = APK(self.app_path)
self.package_name = self.apk.get_package()           # 套件名稱
self.main_activity = self.apk.get_main_activities()  # 主 Activity
self.permissions = self.apk.get_permissions()        # 權限列表
self.activities = self.apk.get_activities()          # 所有 Activity
```

#### 實際獲取的資訊

| 資訊項目 | 獲取方式 | 實際來源 |
|---------|---------|---------|
| **Package Name** | `apk.get_package()` | AndroidManifest.xml 的 `<manifest package="...">` |
| **Main Activity** | `apk.get_main_activities()` | 尋找帶有 `MAIN` action 和 `LAUNCHER` category 的 Activity |
| **所有 Activities** | `apk.get_activities()` | AndroidManifest.xml 的 `<activity>` 標籤列表 |
| **Permissions** | `apk.get_permissions()` | AndroidManifest.xml 的 `<uses-permission>` 標籤 |

**Androguard 原理**：
- 解析 APK 內的 `AndroidManifest.xml`（已編譯成二進位格式）
- 將二進位 XML 轉換回可讀格式
- 提供 Python API 存取各項 metadata

### 2.2 安裝時的動態驗證

**關鍵檔案**：`droidbot/droidbot/device.py:616-658`

APK 安裝後，系統會執行 `dumpsys package` 來驗證並補充資訊：

```bash
adb shell dumpsys package <package_name>
```

**為什麼需要 dumpsys？**

1. **驗證安裝**：確認 APK 已正確安裝
2. **獲取實際 Main Activity**：有些 APK 的 Manifest 可能不完整，dumpsys 能取得系統實際使用的 Main Activity
3. **獲取 Intent Filters**：更詳細的 Intent 資訊

**解析範例**：
```python
# 尋找帶有 MAIN action 和 LAUNCHER category 的 Activity
for activity in activities:
    if "android.intent.action.MAIN" in activities[activity]["actions"] \
            and "android.intent.category.LAUNCHER" in activities[activity]["categories"]:
        main_activity = activity
```

### 2.3 AgentConfig 初始化

**關鍵檔案**：`droidagent/config.py:124-144`

系統將解析結果儲存到全域配置物件：

```python
def set_app(self, app):
    self.app_name = app.apk.get_app_name()
    self.package_name = app.get_package_name()
    self.main_activity = ActivityNameManager.fix_activity_name(
        app.get_main_activity().split('/')[-1]
    )

    # 過濾出屬於此 APP 的 Activities（排除系統和第三方庫）
    package_name_prefix = '.'.join(self.package_name.split('.')[:2])

    app_activities = [
        ActivityNameManager.fix_activity_name(a.split('/')[-1])
        for a in app.activities
        if 'leakcanary' not in a
        and 'CrashReportDialog' not in a
        and package_name_prefix in a
    ]

    self.app_activities = app_activities
```

**Activities 列表用途**：
1. **覆蓋率計算**：追蹤已訪問 vs. 未訪問的頁面
2. **任務規劃**：LLM 會看到「還有哪些頁面未訪問」來規劃新任務
3. **終止條件**：可設定「訪問所有頁面後結束」

**實際範例**（AnkiDroid APP）：
```json
{
  "app_name": "AnkiDroid",
  "package_name": "com.ichi2.anki",
  "main_activity": "IntentHandler",
  "app_activities": [
    "DeckPicker",
    "Reviewer",
    "CardBrowser",
    "NoteEditor",
    "Preferences",
    "Statistics",
    ...
  ]
}
```

---

## 3. 運行時 GUI 狀態獲取

### 3.1 DroidBotApp：中間代理機制

**為什麼需要 DroidBotApp？**

Android 不允許外部程式直接讀取其他 APP 的 UI 結構，必須透過 **Accessibility Service**（無障礙服務）這個合法途徑。

**DroidBotApp 架構**：

```
┌──────────────────┐          Socket (port 7336)          ┌─────────────────┐
│  電腦端           │ ◄───────────────────────────────────►│  Android 設備    │
│  DroidAgent      │          ADB Port Forwarding         │  DroidBotApp     │
│                  │                                       │  (AccessService) │
└──────────────────┘                                       └─────────────────┘
         │                                                          │
         │ 請求 GUI 狀態                                              │
         ├──────────────────────────────────────────────────────────►
         │                                                          │
         │                                        ┌─────────────────┴────────┐
         │                                        │ Accessibility API        │
         │                                        │ AccessibilityNodeInfo    │
         │                                        │ 取得整個 View Hierarchy  │
         │                                        └─────────────────┬────────┘
         │                                                          │
         │ 回傳 JSON (AccEvent)                                      │
         ◄──────────────────────────────────────────────────────────┤
         │                                                          │
```

**關鍵檔案**：`droidbot/droidbot/adapter/droidbot_app.py`

### 3.2 View Hierarchy 資料結構

DroidBotApp 透過 Accessibility Service 獲取的 View Tree 範例：

```json
{
  "root_node": {
    "class": "android.widget.LinearLayout",
    "bounds": [0, 0, 1080, 1920],
    "resource_id": "com.ichi2.anki:id/root_layout",
    "text": null,
    "content_description": null,
    "clickable": false,
    "long_clickable": false,
    "scrollable": false,
    "editable": false,
    "enabled": true,
    "visible": true,
    "checked": false,
    "focused": false,
    "selected": false,
    "children": [
      {
        "class": "android.widget.Button",
        "bounds": [100, 200, 300, 280],
        "resource_id": "com.ichi2.anki:id/study_button",
        "text": "Study Now",
        "clickable": true,
        "children": []
      }
    ]
  }
}
```

### 3.3 資料轉換流程

**階段 1：DroidBot DeviceState**

`droidbot/droidbot/device_state.py`

將樹狀結構轉為平面列表（方便索引）：

```python
self.views = []  # 平面列表
self.view_tree = {}  # 保留樹狀結構

# 將巢狀結構展開成列表，每個 view 分配一個 temp_id
for view in view_tree:
    view['temp_id'] = len(self.views)
    self.views.append(view)
```

**階段 2：DroidAgent GUIState**

`droidagent/types/gui_state.py:67-83`

轉換為 DroidAgent 內部的 Widget 物件：

```python
def from_droidbot_state(self, droidbot_state):
    self.activity = droidbot_state.foreground_activity  # 當前頁面名稱
    self.activity_stack = droidbot_state.activity_stack  # 頁面堆疊

    # 遞迴遍歷 view tree，為每個元件創建 Widget 物件
    self.widgets = []
    for root_elem in view_tree:
        traverse_widgets(root_elem, self.widgets, droidbot_state.views)
```

**階段 3：Widget 物件**

`droidagent/types/widget.py`

每個 Widget 包含：

- **識別屬性**：`view_id`, `signature`, `resource_id`, `text`
- **類型資訊**：`widget_type` (Button, EditText, etc.)
- **可執行操作**：`possible_action_types` (touch, scroll, set_text, etc.)
- **狀態屬性**：`checked`, `focused`, `selected`
- **層次結構**：`children` (子元件列表)

**判斷可執行操作的邏輯**：

```python
possible_action_types = []

if elem.get('clickable') or elem.get('checkable'):
    possible_action_types.append('touch')

if elem.get('long_clickable'):
    possible_action_types.append('long_touch')

if elem.get('editable'):
    possible_action_types.append('set_text')

if elem.get('scrollable'):
    possible_action_types.append('scroll')
```

### 3.4 Widget Signature：穩定識別符

**關鍵檔案**：`droidagent/types/widget.py:84-104`

**為什麼需要 Signature？**

- View ID (`temp_id`) 每次重新載入頁面會改變
- 需要一個穩定的識別符來追蹤「同一個元件在不同時間點」

**Signature 生成邏輯**：

```python
def signature(self):
    # 選擇不可變的屬性
    immutable_props = ['content_description', 'resource_id']

    # 非輸入框的 text 也是不可變的
    if 'set_text' not in self.possible_action_types:
        immutable_props.append('text')

    ingredients = []
    for prop in immutable_props:
        if self.elem_dict.get(prop):
            ingredients.append(self.elem_dict[prop])

    # 遞迴加上子元件的 signature
    ingredients.extend([child.signature for child in self.children])

    # 如果沒有任何識別屬性，使用座標
    if len(ingredients) == 0:
        ingredients = [str(self.elem_dict['bounds'])]

    # 加上元件類型作為前綴
    ingredients.insert(0, self.widget_type)

    return '-'.join(ingredients)
```

**範例**：
- `Button-study_button-Study Now`
- `EditText-deck_name_input`
- `ImageView-[[100,200],[300,250]]` (沒有其他屬性時用座標)

---

## 4. 多 LLM 協同架構

### 4.1 四組件循環

**關鍵檔案**：`droidagent/agent.py:86-201`

DroidAgent 使用**狀態機模式**，在 4 個模式間循環：

```
    ┌─────────────┐
    │ MODE_PLAN   │  規劃新任務
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │ MODE_OBSERVE│  觀察動作結果
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  MODE_ACT   │  執行下一步動作
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │ MODE_REFLECT│  反思任務結果
    └──────┬──────┘
           │
           └──────► 回到 MODE_PLAN
```

### 4.2 Planner（任務規劃器）

**關鍵檔案**：`droidagent/_planner.py:30-47`

**輸入資訊**：
1. **APP 頁面列表**：從 APK 解析得到的 `app_activities`
2. **已訪問頁面統計**：`visited_activities`（記錄每個頁面的訪問次數）
3. **當前頁面狀態**：`current_gui_state`（包含所有 Widget）
4. **歷史任務記錄**：從 ChromaDB 檢索相關的過去任務
5. **Persona 目標**：`ultimate_goal`（例如："盡可能訪問所有頁面並嘗試核心功能"）

**關鍵檔案**：`droidagent/prompts/plan.py:13-73`

**Planner 的 Prompt 結構**：

```python
system_message = f'''
你是一個任務規劃助手，為名為 "{persona_name}" 的使用者規劃使用 {app_name} APP 的任務。

用戶資料：
{persona_profile}

用戶終極目標：{ultimate_goal}

APP 資訊：
- 所有頁面：{activities}  # 從 APK 解析得到
- 已訪問頁面：{visited_activities}  # 即時追蹤
- 當前頁面：{current_activity}
- 未訪問頁面：{unvisited_pages}  # 即時計算

任務需符合以下特性：
1. (真實性) 對應真實使用場景，不要規劃 "導航到 X" 這種模糊任務
2. (重要性) 優先測試核心功能，不要在同一頁面停留過久
3. (多樣性) 不要重複過去失敗的任務
4. (難度) 從當前狀態幾步就能完成的任務
'''

user_message = f'''
規劃下一個任務，參考以下資訊：

過去任務歷史：
{task_memory.retrieve_task_history()}  # 從 ChromaDB 檢索

過去任務反思：
{task_memory.retrieve_task_reflections(current_state)}  # 語義檢索

當前頁面結構：
{current_gui_state.describe_screen_w_memory(memory)}

輸出格式：
- 新任務推理：<1-2 句話>
- 下一個任務：<1 句話，動詞開頭>
- 任務完成條件：<1 句話>
- 第一步動作推理：<推理過程>
- 粗略計劃：<1 句話>
'''
```

**實際輸出範例**：

```
新任務推理: The user has visited the main deck picker page but hasn't created any new decks yet, which is a core function. This task is realistic and achievable.

下一個任務: Create a new deck for studying vocabulary

任務完成條件: The task is known to be completed when a new deck appears in the deck list with the name "Vocabulary"

第一步動作推理: I should click on the "Add" button to start creating a new deck

粗略計劃: I plan to click the add button, enter a deck name, and confirm the creation
```

**接著 Planner 會呼叫 Function Call 來選擇第一個動作**：

```python
# 可用的 function 列表
functions = [
    {
        "name": "touch_widget",
        "parameters": {
            "target_widget_ID": {"type": "integer"}
        }
    },
    {
        "name": "scroll_widget",
        "parameters": {
            "target_widget_ID": {"type": "integer"},
            "direction": {"enum": ["up", "down", "left", "right"]}
        }
    },
    {
        "name": "fill_text",
        "parameters": {
            "target_widget_ID": {"type": "integer"}
        }
    },
    {
        "name": "go_back"
    }
]

# LLM 選擇: touch_widget(target_widget_ID=15)
```

### 4.3 Observer（觀察者）

**關鍵檔案**：`droidagent/_observer.py`

**任務**：比較動作前後的 GUI 狀態，總結變化

**方法**：

```python
def observe_action_result(self):
    old_state = AppState.previous_gui_state
    new_state = AppState.current_gui_state

    # 比較兩個狀態
    changed_widgets, appeared_widgets, disappeared_widgets = \
        old_state.diff_widgets(new_state)

    # 使用 LLM 總結變化
    observation = prompt_state_change_summary(
        old_state, new_state,
        changed_widgets, appeared_widgets, disappeared_widgets
    )

    # 記錄到 Working Memory
    self.memory.working_memory.add_step(observation, 'OBSERVATION')

    # 記錄到 Spatial Memory (Widget 知識庫)
    if observation:
        self.memory.widget_knowledge.add_widget_wise_observation(
            page=old_state.activity,
            widget_signature=last_action.target_widget.signature,
            observation=observation,
            action=last_action
        )
```

**Observer 的 Prompt**：

```python
system_message = f'''
總結 {persona_name} 執行動作後的 GUI 變化。

動作前頁面：{old_state.activity}
動作後頁面：{new_state.activity}

執行的動作：{last_action}

變化的元件：
{format_widget_changes(changed_widgets)}

新出現的元件：
{format_widgets(appeared_widgets)}

消失的元件：
{format_widgets(disappeared_widgets)}

用 1-2 句話總結主要變化。
'''
```

**實際範例**：

```
動作: 點擊 "Add" 按鈕 (ID: 15)
觀察: 一個新的對話框出現，包含一個文字輸入框 (resource_id: deck_name_input) 和兩個按鈕 ("Cancel" 和 "OK")
```

### 4.4 Actor（執行者）

**關鍵檔案**：`droidagent/_actor.py:31-58`

**任務**：選擇下一步動作

**批判機制**：每 4 步動作後會自我批判一次

```python
def act(self):
    # 每 4 步動作後進行批判
    if self.critique_countdown == 0:
        self.critique_countdown = 4

        critique, workaround = prompt_critique(self.memory)

        if critique:
            self.memory.working_memory.add_step(
                f'{critique} {workaround}',
                'CRITIQUE'
            )

    self.critique_countdown -= 1

    # 選擇下一步動作
    action = prompt_action(self.memory)

    if action:
        self.action_count += 1
        self.memory.working_memory.add_step(action, 'ACTION')

    return action
```

**Critique Prompt**：

```python
system_message = f'''
評估 {persona_name} 當前的任務執行策略。

當前任務：{task.summary}
任務計劃：{task.plan}
任務完成條件：{task.end_condition}

過去 4 步的執行歷史：
{working_memory.get_recent_steps(4)}

問題：
1. 當前策略是否有效？
2. 是否陷入重複循環？
3. 是否需要調整策略？

輸出格式：
批判：<具體問題，或 "無">
建議：<替代方案，或 "無">
'''
```

**實際範例**：

```
批判: The user has clicked the "Add" button three times but the dialog keeps closing without creating a deck. This suggests the user might be missing a required step.

建議: Before clicking "OK", the user should first fill in the deck name input field with a meaningful name.
```

### 4.5 Reflector（反思者）

**關鍵檔案**：`droidagent/_reflector.py`

**任務**：任務完成後評估成功/失敗，並提取可重用的知識

**Reflector Prompt**：

```python
system_message = f'''
評估 {persona_name} 的任務執行結果。

任務：{task.summary}
任務完成條件：{task.end_condition}

完整執行歷史：
{working_memory.to_string()}

當前頁面狀態：
{current_gui_state.describe_screen()}

問題：
1. 任務是否成功完成？
2. 從這次任務學到什麼可重用的知識？
'''
```

**輸出範例**：

```json
{
  "assessment": "成功",
  "result_summary": "Successfully created a new deck named 'Vocabulary'. The deck now appears in the main deck list.",
  "reflections": [
    "To create a new deck in AnkiDroid, the user needs to: 1) click the Add button, 2) enter a deck name, 3) click OK",
    "The deck name input field requires at least one character, otherwise the OK button remains disabled"
  ]
}
```

這些反思會被儲存到 **Knowledge Storage**（ChromaDB），供未來任務檢索使用。

---

## 5. 記憶系統設計

### 5.1 四種記憶類型

**關鍵檔案**：`droidagent/memories/memory.py`

```python
class Memory:
    def __init__(self, name):
        # ChromaDB 向量資料庫（兩個 collection）
        self.history = PersistentStorage(f'{name}_primary')      # 任務歷史
        self.knowledge = PersistentStorage(f'{name}_knowledge')  # 知識庫

        # 四種記憶
        self.working_memory = WorkingMemory()              # 短期記憶
        self.task_memory = TaskMemory(self.history, ...)  # 任務記憶
        self.widget_knowledge = SpatialMemory(...)         # 空間記憶（Widget）
```

| 記憶類型 | 儲存位置 | 內容 | 用途 |
|---------|---------|------|------|
| **WorkingMemory** | 內存 | 當前任務的執行步驟 | 提供給 LLM 作為上下文 |
| **TaskMemory** | ChromaDB `primary` | 過去任務的歷史記錄 | 避免重複任務 |
| **SpatialMemory** | ChromaDB `knowledge` | Widget 行為知識 | 告訴 LLM "這個按鈕做什麼" |
| **PersistentStorage** | ChromaDB | 底層向量資料庫 | 語義檢索 |

### 5.2 WorkingMemory：短期記憶

**關鍵檔案**：`droidagent/memories/working_memory.py`

**內容**：當前任務的執行步驟序列

```python
class WorkingMemory:
    def __init__(self, task=None):
        self.task = task
        self.steps = []  # (description, activity, type, timestamp)

    def add_step(self, description, activity, step_type):
        self.steps.append((description, activity, step_type, timestamp))
```

**Step 類型**：
- `ACTION`：執行的動作（例如：點擊按鈕）
- `OBSERVATION`：觀察到的變化（例如：對話框出現）
- `CRITIQUE`：自我批判（例如：策略需調整）

**實際範例**：

```json
{
  "task": "Create a new deck for studying vocabulary",
  "steps": [
    {
      "type": "ACTION",
      "description": "Touch on button with resource_id 'add_button' (ID: 15)",
      "activity": "DeckPicker",
      "timestamp": "2025-01-15 10:23:45"
    },
    {
      "type": "OBSERVATION",
      "description": "A dialog appeared with a text input field and OK/Cancel buttons",
      "activity": "DeckPicker",
      "timestamp": "2025-01-15 10:23:46"
    },
    {
      "type": "ACTION",
      "description": "Fill in text 'Vocabulary' to textfield (ID: 18)",
      "activity": "DeckPicker",
      "timestamp": "2025-01-15 10:23:47"
    },
    {
      "type": "OBSERVATION",
      "description": "The OK button became enabled",
      "activity": "DeckPicker",
      "timestamp": "2025-01-15 10:23:48"
    },
    {
      "type": "ACTION",
      "description": "Touch on button 'OK' (ID: 20)",
      "activity": "DeckPicker",
      "timestamp": "2025-01-15 10:23:49"
    },
    {
      "type": "OBSERVATION",
      "description": "The dialog closed and a new deck named 'Vocabulary' appeared in the deck list",
      "activity": "DeckPicker",
      "timestamp": "2025-01-15 10:23:50"
    }
  ]
}
```

**用途**：提供給 Actor 和 Reflector 作為上下文

### 5.3 TaskMemory：任務記憶

**關鍵檔案**：`droidagent/memories/task_memory.py`

**儲存內容**：
1. 任務描述（Task summary）
2. 任務結果（成功/失敗）
3. 任務反思（Reflections）

**ChromaDB 儲存格式**：

```python
{
  "document": "Successfully created a new deck named 'Vocabulary'",  # 任務結果摘要
  "metadata": {
    "type": "TASK_RESULT",
    "task": "Create a new deck for studying vocabulary",
    "task_result": "成功",
    "timestamp": "2025-01-15 10:23:50"
  }
}
```

**檢索方法**：

```python
def retrieve_task_history(self, max_len=20):
    # 取得最近 20 筆任務記錄
    entries = self.storage.get(where={
        '$or': [
            {'type': 'TASK_RESULT'},
            {'type': 'INITIAL_KNOWLEDGE'}
        ]
    })

    return self.storage.stringify_entries(entries, max_len=max_len)
```

**用途**：
- Planner 用來避免重複規劃相同任務
- 提供給 LLM 作為歷史上下文

### 5.4 SpatialMemory：Widget 知識庫

**關鍵檔案**：`droidagent/memories/spatial_memory.py`

**核心概念**：記錄「對某個 Widget 執行某個動作後，觀察到什麼變化」

**儲存格式**：

```python
{
  "document": "DeckPicker page: a button that has resource_id 'add_button'",  # 狀態簽名
  "metadata": {
    "type": "WIDGET",
    "page": "DeckPicker",
    "widget": "Button-add_button-Add",  # Widget signature
    "action": "touch",
    "observation": "A dialog appeared for creating a new deck",
    "task": "Create a new deck"
  }
}
```

**記錄流程**：

```python
def add_widget_wise_observation(self, page, widget_signature, observation, action):
    # 記錄 Widget 被操作的次數
    if widget_signature not in self.widget_knowledge_map[page]:
        self.widget_knowledge_map[page][widget_signature] = {
            'action_count': defaultdict(int),
            'observation_count': 0
        }

    self.widget_knowledge_map[page][widget_signature]['action_count'][action.type] += 1

    # 儲存到 ChromaDB
    self.storage.add_entry(
        document=state_signature,
        metadata={
            'type': 'WIDGET',
            'observation': observation,
            'page': page,
            'widget': widget_signature,
            'action': action.type
        }
    )
```

**檢索方法**：語義相似度檢索

```python
def retrieve_widget_knowledge(self, state, widget, N=5):
    # 使用當前 GUI 狀態作為查詢
    relevant_entries = self.storage.query(
        query_texts=[state.signature],  # 語義查詢
        n_results=N,
        where={
            'type': 'WIDGET',
            'page': state.activity,
            'widget': widget.signature
        }
    )

    # 用 LLM 總結檢索到的觀察
    widget_role_summary = prompt_summarized_widget_knowledge(
        widget.stringify(),
        relevant_widget_observations
    )

    return widget_role_summary
```

**實際檢索範例**：

當 Actor 看到 `add_button` 時，系統會檢索過去的觀察記錄：

```
查詢: "DeckPicker page: button with resource_id 'add_button'"

檢索結果:
- result of touch: A dialog appeared for creating a new deck
- result of touch: A dialog with deck name input field appeared
- result of touch: Opens the new deck creation interface

LLM 總結:
widget_role_inference: "This button opens a dialog for creating a new deck"
```

這個總結會被注入到 GUI 狀態描述中：

```json
{
  "ID": 15,
  "widget_type": "Button",
  "resource_id": "add_button",
  "text": "Add",
  "possible_action_types": ["touch"],
  "num_prev_actions": 3,
  "widget_role_inference": "This button opens a dialog for creating a new deck"
}
```

**用途**：告訴 LLM 每個 Widget 的功能，避免盲目探索

### 5.5 ChromaDB：向量資料庫

**關鍵檔案**：`droidagent/memories/memory.py:13-72`

**為什麼使用向量資料庫？**

傳統資料庫用關鍵字查詢，無法理解語義。ChromaDB 使用**向量嵌入**（Embedding）來實現語義檢索。

**範例**：

查詢："如何創建新卡片組？"

傳統資料庫：找不到（因為沒有 "卡片組" 這個關鍵字）

ChromaDB：
```
查詢向量: [0.12, 0.45, 0.78, ...]  # "如何創建新卡片組" 的 embedding
    ↓ 計算餘弦相似度
最相似文檔: "Create a new deck for studying vocabulary"
相似度: 0.89
```

**初始化**：

```python
class PersistentStorageManager:
    chroma_client = chromadb.Client()

    @classmethod
    def create_storage(cls, storage_id):
        # 刪除舊的 collection（如果存在）
        try:
            cls.chroma_client.delete_collection(name=storage_id)
        except ValueError:
            pass

        # 創建新 collection
        cls.active_storages[storage_id] = \
            cls.chroma_client.create_collection(name=storage_id)

        return cls.active_storages[storage_id]
```

**兩個 Collection**：

1. **`primary`**：儲存任務歷史
2. **`knowledge`**：儲存 Widget 和 Task 知識

**新增資料**：

```python
def add_entry(self, document, metadata):
    timestamp = time.strftime('%Y-%m-%d %H:%M:%S')

    self.db.add(
        documents=[document],  # 會自動生成 embedding
        metadatas=[{
            'timestamp': timestamp,
            **metadata
        }],
        ids=[str(self.entry_id)]
    )

    self.entry_id += 1
```

**查詢資料**：

```python
def query(self, query_texts, n_results=5, where=None):
    return self.db.query(
        query_texts=query_texts,  # 查詢文字（會轉為 embedding）
        n_results=n_results,      # 返回前 N 筆最相似結果
        where=where               # 額外的 metadata 過濾條件
    )
```

---

## 6. 任務規劃機制

### 6.1 任務規劃的輸入

**關鍵檔案**：`droidagent/prompts/plan.py:13-73`

Planner 會看到以下資訊來規劃任務：

#### 1. APP 頁面資訊（來自 APK 靜態分析）

```python
# 所有頁面列表
activities = [
    "DeckPicker", "Reviewer", "CardBrowser",
    "NoteEditor", "Preferences", ...
]

# 已訪問頁面統計
visited_activities = {
    "DeckPicker": 5,
    "Reviewer": 2,
    "CardBrowser": 1
}

# 未訪問頁面
unvisited_pages = ["NoteEditor", "Preferences", ...]
```

這些資訊在 Prompt 中呈現為：

```
- AnkiDroid app 有以下頁面: [DeckPicker, Reviewer, CardBrowser, NoteEditor, Preferences, ...]
- 目前已訪問的頁面: {DeckPicker: 5, Reviewer: 2, CardBrowser: 1}
- 目前所在頁面: DeckPicker
- 尚未訪問的頁面: [NoteEditor, Preferences, ...]
```

#### 2. 歷史任務（來自 ChromaDB TaskMemory）

```python
task_history = memory.task_memory.retrieve_task_history(max_len=20)
```

**實際內容**：

```
2025-01-15 10:15:30: [INITIAL_KNOWLEDGE] Jade started the AnkiDroid app
2025-01-15 10:16:45: [TASK_RESULT] Successfully browsed the main deck picker page. Discovered that the app has 3 pre-installed sample decks.
2025-01-15 10:18:20: [TASK_RESULT] Failed to create a new deck. The task was abandoned after multiple attempts without entering a deck name.
2025-01-15 10:20:10: [TASK_RESULT] Successfully created a new deck named 'Vocabulary'. The deck now appears in the deck list.
```

#### 3. 相關反思（語義檢索自 ChromaDB）

```python
reflections = memory.task_memory.retrieve_task_reflections(current_gui_state, N=5)
```

**檢索邏輯**：
- 將當前 GUI 狀態轉為自然語言描述
- 用此描述作為查詢向量
- 檢索最相關的 5 筆過去任務反思

**實際內容**：

```
- To create a new deck in AnkiDroid, click the Add button, enter a deck name, then click OK
- The deck name input field requires at least one character
- Clicking the study button on a deck opens the Reviewer page
```

#### 4. 當前 GUI 狀態（包含 Widget 知識）

```python
current_page = current_gui_state.describe_screen_w_memory(memory)
```

**實際內容**（JSON 格式）：

```json
{
  "page_name": "DeckPicker",
  "children": [
    {
      "ID": 15,
      "widget_type": "Button",
      "resource_id": "add_button",
      "text": "Add",
      "possible_action_types": ["touch"],
      "num_prev_actions": 3,
      "widget_role_inference": "Opens a dialog for creating a new deck"
    },
    {
      "ID": 18,
      "widget_type": "LinearLayout",
      "children": [
        {
          "widget_type": "TextView",
          "text": "Vocabulary"
        },
        {
          "widget_type": "TextView",
          "text": "0 cards"
        }
      ],
      "possible_action_types": ["touch"],
      "num_prev_actions": 1,
      "widget_role_inference": "Opens the study interface for this deck"
    }
  ]
}
```

注意 `num_prev_actions` 和 `widget_role_inference` 都來自記憶系統！

### 6.2 任務規劃的四個準則

Planner 的 Prompt 明確要求任務符合：

1. **真實性（Realism）**
   - 對應真實使用場景
   - ❌ 錯誤範例："Navigate to the settings page"
   - ✅ 正確範例："Change the app theme to dark mode"

2. **重要性（Importance）**
   - 優先核心功能
   - 不要在同一頁面停留過久
   - 參考 `visited_activities` 來平衡探索

3. **多樣性（Diversity）**
   - 不重複過去的任務
   - 參考 `task_history` 避免重複
   - 注意 `num_prev_actions`，優先點擊未操作過的 Widget

4. **難度（Difficulty）**
   - 從當前狀態幾步就能完成
   - 參考 `widget_role_inference` 來判斷可行性

### 6.3 任務輸出格式

Planner 需要輸出：

```
1. 任務推理 (Reasoning)
2. 任務描述 (Task description)
3. 任務完成條件 (End condition)
4. 第一步動作推理 (First action reasoning)
5. 粗略計劃 (Rough plan)
6. 第一步動作 (First action) - Function call
```

**實際範例**：

```
Reasoning about Jade's new task: Jade has successfully created a deck and should now try studying it, which is a core function of the app. This is realistic and achievable from the current state.

Jade's next task: Study flashcards in the Vocabulary deck

End condition of Jade's next task: The task is known to be completed when Jade has reviewed at least one flashcard and returned to the deck picker page

Reasoning of the first action: I should touch on the "Vocabulary" deck item to open the study interface

Rough plan for the task in Jade's perspective: I plan to select the Vocabulary deck, review the flashcards, and return to the main screen

[Function Call] touch_widget(target_widget_ID=18)
```

### 6.4 任務物件

**關鍵檔案**：`droidagent/types/task.py`

```python
class Task:
    def __init__(self, summary, description, plan=None, end_condition=None):
        self.summary = summary              # 簡短描述
        self.description = description      # 詳細描述（可選）
        self.plan = plan                    # 粗略計劃
        self.end_condition = end_condition  # 完成條件

        self.explored_states = []           # 任務中訪問的 GUI 狀態
        self.explored_activities = set()    # 任務中訪問的頁面

        self.assessment = None              # 任務結果（成功/失敗）
        self.result_summary = None          # 結果摘要
        self.reflections = []               # 反思列表
        self.entry_id = None                # ChromaDB entry ID
```

---

## 7. 動作執行與批判

### 7.1 Action 物件

**關鍵檔案**：`droidagent/types/action.py`

```python
class Action:
    def __init__(self, event_type, target_widget=None, **kwargs):
        self.event_type = event_type  # touch, scroll, set_text, key_press
        self.target_widget = target_widget

        # 特定類型的參數
        self.text = kwargs.get('text')           # for set_text
        self.direction = kwargs.get('direction') # for scroll
        self.key_name = kwargs.get('name')       # for key_press

        self.event_records = []  # 執行後的事件記錄
```

**支援的動作類型**：

| 類型 | 參數 | DroidBot 事件 | 範例 |
|-----|------|--------------|------|
| `touch` | `target_widget` | `TouchEvent` | 點擊按鈕 |
| `long_touch` | `target_widget` | `LongTouchEvent` | 長按元件 |
| `set_text` | `target_widget`, `text` | `SetTextEvent` | 輸入文字 |
| `scroll` | `target_widget`, `direction` | `ScrollEvent` | 滾動列表 |
| `key_press` | `name` | `KeyEvent` | 按返回鍵 |

### 7.2 Actor 的動作選擇

**關鍵檔案**：`droidagent/prompts/act.py`

**Actor Prompt 結構**：

```python
system_message = f'''
你是 {persona_name}，正在使用 {app_name}。

當前任務：{task.summary}
任務計劃：{task.plan}
任務完成條件：{task.end_condition}

可用動作類型：
- Scroll on a scrollable widget
- Touch on a clickable widget
- Long touch on a long-clickable widget
- Fill in an editable widget
- Navigate back
'''

user_message = f'''
任務執行歷史：
{working_memory.to_string()}

當前頁面：
{current_gui_state.describe_screen_w_memory(memory)}

選擇下一步動作（呼叫 function）
'''
```

**Function Call 範例**：

```python
# LLM 選擇
{
  "function": {
    "name": "touch_widget",
    "arguments": {
      "target_widget_ID": 18
    }
  }
}

# 轉換為 Action 物件
action = Action(
    event_type='touch',
    target_widget=gui_state.get_widget_by_id(18)
)

# 轉換為 DroidBot 事件
droidbot_event = TouchEvent(view=widget.to_droidbot_view())

# 透過 ADB 發送到設備
device.send_event(droidbot_event)
```

### 7.3 批判機制

**關鍵檔案**：`droidagent/_actor.py:34-48`

**觸發條件**：每執行 4 步動作後

```python
CRITIQUE_COUNTDOWN = 4

def act(self):
    if self.critique_countdown == 0:
        # 重置計數器
        self.critique_countdown = CRITIQUE_COUNTDOWN

        # 獲取批判
        critique, workaround = prompt_critique(self.memory)

        if critique:
            full_critique = f'{critique} {workaround}'
            self.memory.working_memory.add_step(
                full_critique,
                AppState.current_activity,
                'CRITIQUE'
            )

    self.critique_countdown -= 1
```

**Critique Prompt**：

```python
system_message = f'''
評估 {persona_name} 的任務執行策略。

當前任務：{task.summary}
任務計劃：{task.plan}
任務完成條件：{task.end_condition}

最近 4 步執行歷史：
{working_memory.get_recent_steps(4)}

問題：
1. 策略是否有效？
2. 是否陷入重複循環？
3. 是否偏離任務目標？

輸出格式：
批判：<具體問題，或輸出 "無">
建議：<替代方案，或輸出 "無">
'''
```

**實際範例**：

```
最近 4 步：
1. [ACTION] Touch on button "Study" (ID: 25)
2. [OBSERVATION] Opened Reviewer page showing "No cards to review"
3. [ACTION] Press back button
4. [OBSERVATION] Returned to DeckPicker page

批判: The user keeps trying to study the Vocabulary deck but it has no cards. This is causing a repetitive loop.

建議: Before studying, the user should first add some flashcards to the deck by creating new notes.
```

這個批判會被加入到 Working Memory，影響下一步的動作選擇。

### 7.4 任務中止機制

**關鍵檔案**：`droidagent/agent.py:173-178`

**規則**：單一任務最多執行 13 步動作

```python
MAX_ACTIONS = 13

if self.actor.action_count >= MAX_ACTIONS:
    self.mode = MODE_REFLECT
    self.inject_action_entry(
        'The task gets too long, so I am going to put off the task and start a new task',
        'TASK_ABORTED'
    )
    return None
```

**為什麼限制 13 步？**

1. 防止無限循環
2. 確保任務粒度適當（太複雜的任務應拆分）
3. 控制 LLM 上下文長度

---

## 8. 知識累積與檢索

### 8.1 Widget 知識的累積

**流程**：

```
ACTION → OBSERVATION → 儲存到 SpatialMemory
```

**實際程式碼**：`droidagent/_observer.py`

```python
def observe_action_result(self):
    old_state = AppState.previous_gui_state
    new_state = AppState.current_gui_state
    last_action = self.memory.working_memory.get_last_action()

    # 比較狀態變化
    changed, appeared, disappeared = old_state.diff_widgets(new_state)

    # LLM 總結變化
    observation = prompt_state_change_summary(
        old_state, new_state, changed, appeared, disappeared
    )

    # 記錄到 Working Memory
    self.memory.working_memory.add_step(observation, 'OBSERVATION')

    # 記錄到 Spatial Memory
    if observation and last_action.target_widget:
        self.memory.widget_knowledge.add_widget_wise_observation(
            page=old_state.activity,
            state_signature=old_state.signature,
            widget_signature=last_action.target_widget.signature,
            observation=observation,
            action=last_action,
            task=self.memory.working_memory.task
        )
```

**ChromaDB 儲存**：

```python
{
  "document": "DeckPicker page: button with resource_id 'add_button'",
  "metadata": {
    "type": "WIDGET",
    "page": "DeckPicker",
    "widget": "Button-add_button-Add",
    "action": "touch",
    "observation": "A dialog appeared for creating a new deck",
    "task": "Create a new deck for studying vocabulary",
    "timestamp": "2025-01-15 10:23:46"
  }
}
```

### 8.2 Widget 知識的檢索與總結

**關鍵檔案**：`droidagent/memories/spatial_memory.py:17-39`

**觸發時機**：Actor 規劃動作時，GUI 狀態描述會包含 Widget 知識

**流程**：

```python
def retrieve_widget_knowledge(self, state, widget, N=5):
    # 1. 語義檢索：用當前狀態作為查詢
    relevant_entries = self.storage.query(
        query_texts=[state.signature],  # 例如："DeckPicker page: button with..."
        n_results=N,
        where={
            'type': 'WIDGET',
            'page': state.activity,
            'widget': widget.signature
        }
    )

    # 2. 提取所有觀察記錄
    observations = []
    for metadata in relevant_entries['metadatas'][0]:
        observations.append(f"- result of {metadata['action']}: {metadata['observation']}")

    # 3. 用 LLM 總結
    widget_role_summary = prompt_summarized_widget_knowledge(
        widget.stringify(),
        '\n'.join(observations)
    )

    return widget_role_summary
```

**總結 Prompt**：

```python
system_message = f'''
總結這個 Widget 的功能，基於過去的觀察記錄。

Widget: {widget_description}

過去觀察記錄：
{observations}

用 1 句話總結這個 Widget 的功能。
'''
```

**實際範例**：

```
Widget: a button that has resource_id 'add_button' and text 'Add'

過去觀察記錄：
- result of touch: A dialog appeared for creating a new deck
- result of touch: A dialog with deck name input field appeared
- result of touch: Opens the new deck creation interface

LLM 總結:
"Opens a dialog for creating a new deck"
```

### 8.3 知識注入到 GUI 描述

**關鍵檔案**：`droidagent/types/gui_state.py:122-188`

```python
def describe_screen_w_memory(self, memory):
    for widget in self.widgets:
        widget_info = widget.to_dict()

        # 注入操作次數
        if len(widget.possible_action_types) > 0:
            action_counts = memory.widget_knowledge.get_performed_action_counts(
                self.activity, widget.signature
            )
            interaction_count = sum(action_counts.values())
            widget_info['num_prev_actions'] = interaction_count

        # 注入功能總結
        if memory.widget_knowledge.has_widget_knowledge(self.activity, widget.signature):
            widget_role = memory.widget_knowledge.retrieve_widget_knowledge(
                self, widget
            )
            if widget_role:
                widget_info['widget_role_inference'] = widget_role

    return json.dumps(view_hierarchy, indent=2)
```

**最終效果**：Actor 看到的 Widget 描述

```json
{
  "ID": 15,
  "widget_type": "Button",
  "resource_id": "add_button",
  "text": "Add",
  "possible_action_types": ["touch"],
  "num_prev_actions": 3,  ← 知識！
  "widget_role_inference": "Opens a dialog for creating a new deck"  ← 知識！
}
```

這讓 LLM 能夠：
1. 知道哪些 Widget 還沒操作過（`num_prev_actions == 0`）
2. 知道每個 Widget 的功能（`widget_role_inference`）
3. 做出更明智的動作選擇

### 8.4 Task 知識的累積

**關鍵檔案**：`droidagent/_reflector.py`

**流程**：

```python
def reflect(self):
    task = self.memory.working_memory.task

    # 1. LLM 評估任務結果
    assessment, result_summary, reflections = prompt_task_reflection(
        task,
        self.memory.working_memory,
        current_gui_state
    )

    # 2. 更新任務物件
    task.assessment = assessment  # "成功" or "失敗"
    task.result_summary = result_summary
    task.reflections = reflections

    # 3. 儲存到 TaskMemory
    self.memory.task_memory.record_task_result(
        task,
        reflections,
        self.memory.working_memory.steps
    )

    return assessment
```

**Reflection Prompt**：

```python
system_message = f'''
評估 {persona_name} 的任務執行結果。

任務：{task.summary}
任務計劃：{task.plan}
任務完成條件：{task.end_condition}

完整執行歷史：
{working_memory.to_string()}

當前頁面狀態：
{current_gui_state.describe_screen()}

問題：
1. 任務是否成功完成？（根據任務完成條件判斷）
2. 可以學到什麼可重用的知識？（2-3 句獨立的陳述）

輸出格式：
評估：<"成功" 或 "失敗">
結果摘要：<1-2 句話>
反思：<可重用的知識，每行一條>
'''
```

**實際範例**：

```
評估: 成功

結果摘要: Successfully added a new flashcard to the Vocabulary deck. The card now appears in the card browser.

反思:
- To add a new flashcard in AnkiDroid, navigate to the deck, click "Add", select a note type, fill in the fields, and click "Save"
- The "Front" field must be filled in before the card can be saved
- After saving, the app returns to the card browser showing the newly created card
```

**儲存格式**：

```python
# 儲存到 knowledge storage (ChromaDB)
{
  "document": "To add a new flashcard in AnkiDroid, navigate to the deck, click Add...",
  "metadata": {
    "type": "TASK",
    "reflection": "...",
    "task": "Add a flashcard to the Vocabulary deck",
    "timestamp": "2025-01-15 10:25:30"
  }
}
```

---

## 9. 完整執行流程

### 9.1 系統啟動流程

**入口檔案**：`scripts/run_droidagent.py`

```python
def main():
    # 1. 解析命令列參數
    parser = argparse.ArgumentParser()
    parser.add_argument('--app', default='AnkiDroid')
    parser.add_argument('--output_dir', default=None)
    parser.add_argument('--profile_id', default='jade')
    parser.add_argument('--is_emulator', action='store_true')
    args = parser.parse_args()

    # 2. 設定輸出目錄
    timestamp = time.strftime("%Y%m%d%H%M%S")
    output_dir = f'evaluation/data_new/{args.app}/agent_run_{args.profile_id}_{timestamp}'

    # 3. 連接 Android 設備
    device = Device(device_serial='emulator-5554', is_emulator=args.is_emulator)
    device.set_up()
    device.connect()

    # 4. 載入並安裝 APK
    app_path = f'target_apps/{args.app}.apk'
    app = App(app_path)

    device.install_app(app)  # ← 這裡會執行 dumpsys package
    device.start_app(app)

    # 5. 載入 Persona
    persona = load_profile(args.profile_id)  # resources/personas/jade.txt
    persona.update({
        'ultimate_goal': 'visit as many pages as possible while trying core functionalities',
        'initial_knowledge': initial_knowledge_map(args.app, persona['name'], app.apk.get_app_name())
    })

    # 6. 初始化 Agent
    agent = TaskBasedAgent(output_dir, app=app, persona=persona)

    # 7. 初始化 DeviceManager
    device_manager = DeviceManager(device, app, output_dir)
    agent.set_current_gui_state(device_manager.current_state)

    # 8. 進入主循環
    run_agent_loop(agent, device_manager)
```

### 9.2 主循環流程

```python
def run_agent_loop(agent, device_manager):
    MAX_STEP = 8000

    while agent.step_count < MAX_STEP:
        # 1. 檢查 loading 狀態
        if is_loading_state(device_manager.current_state):
            time.sleep(1)
            device_manager.fetch_device_state()
            continue

        # 2. Agent 執行一步（狀態機）
        action = agent.step()

        # 3. 儲存記憶快照
        agent.save_memory_snapshot()  # memory_snapshots/step_{N}/

        # 4. 如果有動作，執行到設備
        if action is not None:
            events = action.to_droidbot_event()
            for event in events:
                event_dict = device_manager.send_event_to_device(
                    event,
                    capture_intermediate_state=True,
                    agent=agent
                )
                action.add_event_records([event_dict])

            # 5. 恢復 Activity Stack（處理意外跳出）
            recover_activity_stack(device_manager, agent)

            # 6. 更新 Agent 的 GUI 狀態
            agent.set_current_gui_state(device_manager.current_state)

        # 7. 儲存實驗數據
        with open(f'{output_dir}/exp_data.json', 'w') as f:
            json.dump(agent.exp_data, f, indent=2)
```

### 9.3 狀態機詳細流程

```python
def agent.step():
    self.step_count += 1

    if self.mode == MODE_PLAN:
        # ==================== PLAN ====================
        # 1. 重置 Actor
        self.actor.reset()

        # 2. Planner 規劃任務
        task_desc, end_cond, plan, first_action = planner.plan_task()

        # 3. 創建 Task 物件
        task = Task(task_desc, plan=plan, end_condition=end_cond)

        # 4. 記錄到 TaskMemory
        task.entry_id = self.memory.task_memory.record_task(task, ...)

        # 5. 初始化 WorkingMemory
        self.memory.working_memory = WorkingMemory(task)
        self.memory.working_memory.add_step(first_action, 'ACTION')

        # 6. 切換狀態
        self.mode = MODE_OBSERVE
        self.actor.action_count += 1

        return first_action

    elif self.mode == MODE_OBSERVE:
        # ==================== OBSERVE ====================
        # 1. Observer 觀察變化
        observation = observer.observe_action_result()

        # 2. 記錄到 WorkingMemory
        self.memory.working_memory.add_step(observation, 'OBSERVATION')

        # 3. 記錄到 SpatialMemory
        last_action = self.memory.working_memory.get_last_action()
        if observation and last_action.target_widget:
            self.memory.widget_knowledge.add_widget_wise_observation(...)

        # 4. 切換狀態
        self.mode = MODE_ACT

        return None

    elif self.mode == MODE_ACT:
        # ==================== ACT ====================
        # 1. 檢查是否超過最大動作數
        if self.actor.action_count >= MAX_ACTIONS:
            self.mode = MODE_REFLECT
            self.inject_action_entry('Task gets too long, aborting...', 'TASK_ABORTED')
            return None

        # 2. 每 4 步動作後批判
        if self.actor.critique_countdown == 0:
            critique, workaround = prompt_critique(self.memory)
            if critique:
                self.memory.working_memory.add_step(f'{critique} {workaround}', 'CRITIQUE')
            self.actor.critique_countdown = CRITIQUE_COUNTDOWN

        self.actor.critique_countdown -= 1

        # 3. Actor 選擇動作
        next_action = actor.act()

        # 4. 記錄到 WorkingMemory
        if next_action:
            self.memory.working_memory.add_step(next_action, 'ACTION')
            self.actor.action_count += 1
            self.mode = MODE_OBSERVE
            return next_action
        else:
            # Actor 選擇結束任務
            self.mode = MODE_REFLECT
            return None

    elif self.mode == MODE_REFLECT:
        # ==================== REFLECT ====================
        # 1. Reflector 評估任務
        assessment, result_summary, reflections = reflector.reflect()

        # 2. 更新 Task 物件
        task = self.memory.working_memory.task
        task.assessment = assessment
        task.result_summary = result_summary
        task.reflections = reflections

        # 3. 記錄到 TaskMemory
        self.memory.task_memory.record_task_result(
            task,
            reflections,
            self.memory.working_memory.steps
        )

        # 4. 切換狀態
        self.mode = MODE_PLAN

        return None
```

### 9.4 資料輸出結構

**輸出目錄**：`evaluation/data_new/{APP_NAME}/agent_run_{PROFILE}_{TIMESTAMP}/`

```
agent_run_jade_20250115102345/
├── agent_config.json           # Agent 配置
├── exp_info.json              # 實驗資訊（開始時間、設備等）
├── exp_data.json              # 實驗數據（即時更新）
│
├── prompts/                   # 所有 LLM Prompt 記錄
│   ├── plan_step_1.json
│   ├── act_step_3.json
│   ├── critique_step_7.json
│   └── ...
│
├── memory_snapshots/          # 每步的記憶快照
│   ├── step_1/
│   │   ├── scratch.json           # WorkingMemory
│   │   └── long_term_memory.txt   # TaskMemory
│   ├── step_2/
│   └── ...
│
├── events/                    # 每個動作的詳細記錄
│   ├── event_2025-01-15_102346.json
│   └── ...
│
└── views/                     # Widget 截圖
    ├── view_abc123.png
    └── ...
```

**exp_data.json 內容**：

```json
{
  "app_activities": [
    "DeckPicker", "Reviewer", "CardBrowser", ...
  ],
  "visited_activities": {
    "DeckPicker": 12,
    "Reviewer": 5,
    "CardBrowser": 3,
    "NoteEditor": 2
  },
  "task_results": {
    "Create a new deck for studying vocabulary": {
      "result": "成功",
      "summary": "Successfully created a new deck named 'Vocabulary'",
      "reflections": [
        "To create a new deck, click Add button, enter name, click OK"
      ],
      "num_actions": 4,
      "num_critiques": 0,
      "visited_pages_during_task": ["DeckPicker"],
      "task_execution_history": [...]
    },
    "Add a flashcard to the Vocabulary deck": {
      "result": "成功",
      ...
    }
  },
  "API_usage": {
    "gpt-4o-mini": {
      "prompt_tokens": 123456,
      "completion_tokens": 45678,
      "total_tokens": 169134
    }
  },
  "API_response_time": {
    "gpt-4o-mini": [1.2, 0.8, 1.5, ...]
  }
}
```

---

## 10. 總結

### 10.1 系統關鍵特性

| 特性 | 實現方式 | 關鍵檔案 |
|-----|---------|---------|
| **APK 頁面列表獲取** | Androguard 解析 AndroidManifest.xml | `droidbot/app.py` |
| **即時 GUI 狀態獲取** | DroidBotApp + Accessibility Service | `droidbot/adapter/droidbot_app.py` |
| **Widget 穩定識別** | Signature 機制（resource_id + text + children） | `droidagent/types/widget.py` |
| **任務規劃** | LLM + 頁面訪問統計 + 歷史任務檢索 | `droidagent/prompts/plan.py` |
| **知識累積** | ChromaDB 向量資料庫 | `droidagent/memories/` |
| **自我批判** | 每 4 步動作後 LLM 評估 | `droidagent/_actor.py` |
| **語義檢索** | ChromaDB embedding + 餘弦相似度 | `droidagent/memories/memory.py` |

### 10.2 數據流總覽

```
APK 檔案
    │
    ├─► Androguard 解析
    │       │
    │       ├─► Package Name
    │       ├─► Main Activity
    │       └─► Activities 列表 ──────┐
    │                                 │
    └─► ADB 安裝                      │
            │                         │
            └─► dumpsys package       │
                    │                 │
                    └─► 驗證 Activities
                            │
                            ▼
                    AgentConfig.app_activities ─────┐
                                                    │
Android 設備                                        │
    │                                               │
    ├─► DroidBotApp (Accessibility Service)        │
    │       │                                       │
    │       └─► View Hierarchy JSON                │
    │               │                               │
    │               ├─► DroidBot DeviceState        │
    │               │       │                       │
    │               │       └─► DroidAgent GUIState │
    │               │               │               │
    │               │               └─► Widget 物件  │
    │               │                   │           │
    │               │                   ├─► view_id │
    │               │                   ├─► signature
    │               │                   └─► possible_actions
    │               │                       │
    │               └─► Screenshot         │
    │                                       │
    └─► ADB 執行動作 ◄─────────────────────┘
            │
            └─► 狀態變化
                    │
                    ▼
            Observer 總結 ──────► ChromaDB (SpatialMemory)
                    │                   │
                    │                   └─► Widget 知識累積
                    ▼
            Planner 規劃任務 ◄──────────┘
                    │               (檢索相關知識)
                    ├─► 未訪問頁面列表 (from AgentConfig)
                    ├─► 歷史任務 (from ChromaDB TaskMemory)
                    └─► 當前 GUI (with Widget 知識)
                            │
                            ▼
                    Actor 選擇動作
                            │
                            ├─► 每 4 步批判一次
                            └─► 最多 13 步後反思
                                    │
                                    ▼
                            Reflector 評估 ──► ChromaDB (TaskMemory)
                                                    │
                                                    └─► Task 反思累積
```

### 10.3 與傳統測試框架的差異

| 傳統框架 | DroidAgent |
|---------|-----------|
| 隨機探索或腳本錄製 | LLM 驅動的智能探索 |
| 無記憶，重複操作 | 記錄 Widget 行為知識 |
| 無目標，盲目點擊 | Persona 驅動的任務導向 |
| 無法理解 GUI 語義 | 理解「這個按鈕做什麼」 |
| 無法從失敗中學習 | 反思機制累積經驗 |
| 需手動指定測試場景 | 自動規劃測試任務 |

---

**文件生成時間**：2025-01-15
**作者**：Claude (Anthropic)
**專案**：DroidAgent - Intent-Driven Android GUI Testing Framework
