# DroidAgent 系統架構完整說明

> 這份文檔用白話的方式解釋 DroidAgent 如何自動測試 Android APP

## 目錄
1. [系統是幹嘛的](#1-系統是幹嘛的)
2. [系統怎麼知道 APP 有哪些頁面](#2-系統怎麼知道-app-有哪些頁面)
3. [系統怎麼即時看到畫面上有什麼元件](#3-系統怎麼即時看到畫面上有什麼元件)
4. [元件怎麼變成 Widget（重要！）](#4-元件怎麼變成-widget重要)
5. [四個 LLM 怎麼協同工作](#5-四個-llm-怎麼協同工作)
6. [狀態機：怎麼決定下一個用哪個 Agent](#6-狀態機怎麼決定下一個用哪個-agent)
7. [記憶系統怎麼運作](#7-記憶系統怎麼運作)
8. [完整執行範例](#8-完整執行範例)

---

## 1. 系統是幹嘛的

### 簡單來說

DroidAgent 是一個**用 AI 自動測試 Android APP 的系統**。

傳統測試工具只會「亂點」，但 DroidAgent 會：
- **看懂畫面**：知道這是「登入按鈕」還是「設定按鈕」
- **記住經驗**：記得「點這個按鈕會跳到哪個頁面」
- **規劃任務**：像真人一樣「我要建立一個新資料夾」
- **自我檢討**：「我一直在點同一個按鈕，應該換個方式」

### 核心能力

| 能力 | 說明 |
|-----|------|
| **自動探索** | 根據「用戶角色」（Persona）智能地測試 APP |
| **任務導向** | 不是亂點，而是規劃有意義的任務（例如「創建新文件」） |
| **知識累積** | 記住每個按鈕的功能，不會重複做無意義的操作 |
| **自我批判** | 每隔幾步就檢討「我這樣做對嗎」 |

---

## 2. 系統怎麼知道 APP 有哪些頁面

### 問題

系統要規劃測試任務時，需要知道「這個 APP 總共有哪些頁面？」才能決定「我還有哪些頁面沒測試過」。

### 解決方法：解析 APK 檔案

每個 Android APP 的 APK 檔案裡都有一個 `AndroidManifest.xml`，裡面記錄了所有頁面（Activity）。

**實際做法**：

**檔案**：`droidbot/droidbot/app.py`

```python
from androguard.core.apk import APK

# 1. 用 Androguard 庫解析 APK
self.apk = APK(self.app_path)

# 2. 取得所有頁面列表
self.activities = self.apk.get_activities()
# 範例結果：
# ['com.example.app/.MainActivity',
#  'com.example.app/.SettingsActivity',
#  'com.example.app/.ProfileActivity', ...]

# 3. 找出主頁面（APP 開啟時第一個看到的頁面）
self.main_activity = self.apk.get_main_activities()[0]
# 範例結果：'com.example.app/.MainActivity'
```

**Androguard 怎麼做到的？**

APK 其實就是個 ZIP 檔，Androguard 會：
1. 解壓縮 APK
2. 讀取裡面的 `AndroidManifest.xml`（是二進位格式）
3. 把二進位 XML 轉回可讀格式
4. 用正規表達式找出所有 `<activity>` 標籤

**範例 AndroidManifest.xml**：
```xml
<manifest package="com.example.app">
    <application>
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <activity android:name=".SettingsActivity"/>
        <activity android:name=".ProfileActivity"/>
    </application>
</manifest>
```

### 儲存到配置

**檔案**：`droidagent/config.py:124-144`

```python
def set_app(self, app):
    # 取得 APP 名稱
    self.app_name = app.apk.get_app_name()  # 例如："Gmail"

    # 取得套件名稱
    self.package_name = app.get_package_name()  # 例如："com.google.android.gm"

    # 取得所有頁面，並過濾掉系統頁面
    package_prefix = '.'.join(self.package_name.split('.')[:2])  # "com.google"

    app_activities = []
    for activity in app.activities:
        # 排除系統和第三方庫的頁面
        if package_prefix in activity and 'leakcanary' not in activity:
            # 簡化頁面名稱（去掉套件前綴）
            activity_name = activity.split('/')[-1]  # "MainActivity"
            app_activities.append(activity_name)

    self.app_activities = app_activities
```

**最後得到的頁面列表**：
```python
app_activities = [
    "MainActivity",
    "SettingsActivity",
    "ProfileActivity",
    "ComposeActivity",
    ...
]
```

### 用途

這個頁面列表會用在：

1. **規劃任務時**：LLM 會看到「還有哪些頁面沒訪問過」
2. **計算覆蓋率**：`已訪問頁面數 / 總頁面數`
3. **決定終止條件**：「所有頁面都訪問過了」

---

## 3. 系統怎麼即時看到畫面上有什麼元件

### 問題

系統需要即時知道「現在畫面上有哪些按鈕、輸入框、文字」，才能決定要點什麼。

但 Android 不允許外部程式直接讀取其他 APP 的 UI！

### 解決方法：DroidBotApp + Accessibility Service

**核心概念**：

Android 有個合法的「無障礙服務」（Accessibility Service），本來是給視障人士用的，可以讀取整個畫面的元件資訊。

DroidAgent 就是利用這個服務！

### 架構圖

```
┌──────────────────┐                    ┌─────────────────┐
│  電腦             │                    │  Android 手機    │
│  DroidAgent      │                    │                 │
│                  │                    │  ┌───────────┐  │
│                  │  ①要求取得畫面資訊  │  │ DroidBotApp│  │
│                  ├────────────────────►│  │(輔助 APP)  │  │
│                  │                    │  └─────┬─────┘  │
│                  │                    │        │        │
│                  │                    │        ②讀取    │
│                  │                    │        │        │
│                  │                    │  ┌─────▼─────┐  │
│                  │                    │  │Accessibility│  │
│                  │                    │  │  Service   │  │
│                  │                    │  │(系統服務)   │  │
│                  │                    │  └─────┬─────┘  │
│                  │                    │        │        │
│                  │                    │        ③取得    │
│                  │                    │        │        │
│                  │                    │  ┌─────▼─────┐  │
│                  │   ④回傳 JSON 資料  │  │View        │  │
│                  │◄────────────────────┤  │Hierarchy  │  │
│                  │                    │  │(UI 樹狀結構)│  │
└──────────────────┘                    └──┴───────────┴──┘
```

### 詳細流程

#### 步驟 1：安裝 DroidBotApp

**檔案**：`droidbot/droidbot/adapter/droidbot_app.py:67-96`

```python
# 系統啟動時，會先把 DroidBotApp 這個輔助 APP 裝到手機
droidbot_app_path = "droidbot/resources/droidbotApp.apk"
device.adb.run_cmd(["install", droidbot_app_path])

# 啟用 Accessibility Service
ACCESSIBILITY_SERVICE = "io.github.ylimit.droidbotapp/.../PSAccessibilityService"
device.adb.enable_accessibility_service(ACCESSIBILITY_SERVICE)
```

#### 步驟 2：建立 Socket 連線

```python
# 透過 ADB 把電腦的某個 port 轉發到手機的 7336 port
forward_cmd = f"adb -s {device_serial} forward tcp:{port} tcp:7336"

# 電腦連線到手機
self.sock = socket.socket()
self.sock.connect(('localhost', port))
```

#### 步驟 3：持續監聽 UI 變化

**檔案**：`droidbot/droidbot/adapter/droidbot_app.py:135-172`

```python
def listen_messages(self):
    while self.connected:
        # 讀取訊息頭（6 bytes）
        _, _, message_len = self.read_head()

        # 讀取完整訊息
        message = self.sock_read(message_len)

        # 解析訊息
        if "AccEvent >>>" in message:
            # DroidBotApp 透過 Accessibility 讀取到的 UI 資訊
            body = json.loads(message.split("AccEvent >>> ")[1])
            self.last_acc_event = body  # 儲存最新的 UI 狀態
```

#### 步驟 4：DroidBotApp 回傳的資料格式

```json
{
  "root_node": {
    "class": "android.widget.LinearLayout",
    "bounds": [0, 0, 1080, 1920],
    "resource_id": "com.example.app:id/root_layout",
    "text": null,
    "clickable": false,
    "scrollable": false,
    "editable": false,
    "long_clickable": false,
    "enabled": true,
    "visible": true,
    "checked": false,
    "focused": false,
    "children": [
      {
        "class": "android.widget.Button",
        "bounds": [100, 200, 300, 280],
        "resource_id": "com.example.app:id/submit_button",
        "text": "Submit",
        "clickable": true,
        "enabled": true,
        "children": []
      },
      {
        "class": "android.widget.EditText",
        "bounds": [100, 300, 500, 380],
        "resource_id": "com.example.app:id/username_input",
        "text": "",
        "editable": true,
        "children": []
      }
    ]
  }
}
```

### 關鍵欄位說明

每個元件（View）都有這些屬性：

| 屬性 | 說明 | 範例 |
|-----|------|------|
| `class` | 元件類型 | `android.widget.Button` |
| `bounds` | 螢幕上的位置 | `[100, 200, 300, 280]` (左上 x,y 到右下 x,y) |
| `resource_id` | 開發者定義的 ID | `com.example.app:id/submit_button` |
| `text` | 顯示的文字 | `"Submit"` |
| `clickable` | 能不能點 | `true` / `false` |
| `editable` | 能不能輸入文字 | `true` / `false` |
| `scrollable` | 能不能滑動 | `true` / `false` |
| `long_clickable` | 能不能長按 | `true` / `false` |
| `children` | 子元件列表 | `[...]` |

---

## 4. 元件怎麼變成 Widget（重要！）

### 為什麼要轉換

從 DroidBotApp 拿到的是「Android 原生的 View 資料」，但 DroidAgent 需要的是「更高階的 Widget 物件」，原因：

1. **簡化資訊**：原生資料有很多不重要的欄位
2. **加上智慧**：自動判斷「這個元件可以做什麼操作」
3. **統一格式**：讓 LLM 更容易理解

### 轉換流程圖

```
DroidBotApp 回傳的 JSON
        │
        ▼
DroidBot DeviceState (平面列表 + 樹狀結構)
        │
        ▼
DroidAgent GUIState (Widget 物件列表)
        │
        ├─► Widget 1: 按鈕
        ├─► Widget 2: 輸入框
        └─► Widget 3: 文字標籤
```

### 步驟 1：DroidBot 整理資料

**檔案**：`droidbot/droidbot/device_state.py:9-83`

```python
class DeviceState:
    def __init__(self, device, views, foreground_activity, ...):
        # 儲存當前頁面名稱
        self.foreground_activity = foreground_activity  # "MainActivity"

        # 把巢狀的樹狀結構「攤平」成列表
        self.views = []  # 平面列表，方便索引
        self.view_tree = {}  # 保留樹狀結構，方便理解層次

        # 轉換並賦予每個 view 一個 temp_id
        self.__parse_views(views)
```

**為什麼要轉成列表？**

原本的資料是這樣（巢狀）：
```
LinearLayout
  ├─ Button
  └─ LinearLayout
      ├─ TextView
      └─ EditText
```

轉成列表後：
```python
views = [
    {id: 0, type: 'LinearLayout', children: [1, 2]},
    {id: 1, type: 'Button', parent: 0},
    {id: 2, type: 'LinearLayout', parent: 0, children: [3, 4]},
    {id: 3, type: 'TextView', parent: 2},
    {id: 4, type: 'EditText', parent: 2}
]
```

這樣就可以用 `views[2]` 快速找到某個元件！

### 步驟 2：DroidAgent 創建 GUIState

**檔案**：`droidagent/types/gui_state.py:67-83`

```python
def from_droidbot_state(self, droidbot_state):
    self.activity = droidbot_state.foreground_activity  # "MainActivity"

    # 簡化 view tree（移除不必要的屬性）
    view_tree = minimize_view_tree(droidbot_state.view_tree)

    # 遞迴遍歷每個元件，轉換成 Widget 物件
    self.widgets = []
    for root_elem in view_tree:
        widget = traverse_widgets(root_elem, self.widgets, droidbot_state.views)
        self.root_widgets.append(widget)
```

### 步驟 3：最關鍵！traverse_widgets 函數

這個函數會：
1. **判斷元件可以做什麼操作**（這就是你問的重點！）
2. **提取重要屬性**
3. **創建 Widget 物件**

**檔案**：`droidagent/types/gui_state.py:315-374`

```python
def traverse_widgets(elem, processed_widgets, original_views):
    new_elem = OrderedDict()
    possible_action_types = []  # 這個元件可以做什麼
    state_properties = []       # 這個元件的狀態

    # ==================== 判斷可執行的操作 ====================
    # 這裡就是系統「知道可以做什麼操作」的關鍵！

    # 規則 1：能點或能勾選 → 可以 touch
    if elem.get('clickable', False) or elem.get('checkable', False):
        possible_action_types.append('touch')

    # 規則 2：能長按 → 可以 long_touch
    if elem.get('long_clickable', False):
        possible_action_types.append('long_touch')

    # 規則 3：能編輯 → 可以輸入文字
    if elem.get('editable', False):
        possible_action_types.append('set_text')

    # 規則 4：能滑動 → 可以 scroll
    if elem.get('scrollable', False):
        possible_action_types.append('scroll')

    # ==================== 判斷元件的狀態 ====================

    if elem.get('focused', False):
        state_properties.append('focused')  # 正在被聚焦

    if elem.get('checked', False):
        state_properties.append('checked')  # 已勾選

    if elem.get('selected', False):
        state_properties.append('selected')  # 已選取

    # ==================== 提取重要屬性 ====================

    # 只有可互動的元件才給 ID（省記憶體）
    if len(possible_action_types) > 0 and 'temp_id' in elem:
        new_elem['ID'] = elem['temp_id']

    # 元件類型（簡化，只取類名最後一段）
    if 'class' in elem:
        # "android.widget.Button" → "Button"
        new_elem['widget_type'] = elem['class'].split('.')[-1]

    # 顯示文字
    if 'text' in elem and elem['text']:
        new_elem['text'] = elem['text'][:100]  # 最多 100 字

    # 內容描述（給視障人士聽的）
    if 'content_description' in elem:
        new_elem['content_description'] = elem['content_description']

    # Resource ID（簡化）
    if 'resource_id' in elem and elem['resource_id']:
        # "com.example.app:id/submit_button" → "submit_button"
        new_elem['resource_id'] = elem['resource_id'].split('/')[-1]

    # 是否為密碼欄位
    if elem.get('is_password'):
        new_elem['is_password'] = True

    # 儲存判斷結果
    if len(state_properties) > 0:
        new_elem['state'] = state_properties

    if len(possible_action_types) > 0:
        new_elem['possible_action_types'] = possible_action_types

    # 元件位置
    new_elem['bounds'] = elem['bounds']

    # ==================== 遞迴處理子元件 ====================

    children_widgets = []
    for child in elem.get('children', []):
        child_widget = traverse_widgets(child, processed_widgets, original_views)
        children_widgets.append(child_widget)

    new_elem['children'] = children_widgets

    # ==================== 創建 Widget 物件 ====================

    widget = Widget().from_dict(new_elem)
    processed_widgets.append(widget)

    return widget
```

### 轉換範例

**輸入（Android 原生格式）**：
```json
{
  "class": "android.widget.Button",
  "bounds": [100, 200, 300, 280],
  "resource_id": "com.example.app:id/submit_button",
  "text": "Submit",
  "clickable": true,
  "long_clickable": false,
  "editable": false,
  "scrollable": false,
  "enabled": true,
  "children": []
}
```

**輸出（Widget 物件）**：
```json
{
  "ID": 15,
  "widget_type": "Button",
  "resource_id": "submit_button",
  "text": "Submit",
  "possible_action_types": ["touch"],
  "bounds": [[100, 200], [300, 280]],
  "children": []
}
```

### 重點整理

**系統怎麼知道 Widget 可以做什麼操作？**

答案：**看 Accessibility Service 回傳的屬性！**

```python
if clickable == true:
    → 可以 touch

if long_clickable == true:
    → 可以 long_touch

if editable == true:
    → 可以 set_text (輸入文字)

if scrollable == true:
    → 可以 scroll (滑動)
```

這些屬性是 **Android 系統自己標記的**，開發者在寫 APP 時就會設定！

---

## 5. 四個 LLM 怎麼協同工作

### 概念

DroidAgent 不是用一個 LLM 做所有事，而是把工作拆成四個角色：

| 角色 | 負責的事 | 用的 LLM |
|-----|---------|---------|
| **Planner** | 規劃新任務 | GPT-4o-mini |
| **Observer** | 觀察畫面變化 | GPT-4o-mini |
| **Actor** | 選擇下一步動作 | GPT-4o-mini |
| **Reflector** | 評估任務結果 | GPT-4o-mini |

### 為什麼要這樣設計？

1. **分工明確**：每個 LLM 只專注做一件事，效果更好
2. **可以調整**：例如可以讓 Planner 用更聰明的模型
3. **方便研究**：可以做消融實驗（Ablation Study）看哪個組件最重要

### 循環示意圖

```
    ┌─────────────┐
    │  Planner    │  規劃新任務：「創建一個新資料夾」
    │             │  選擇第一步動作：「點擊新增按鈕」
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Observer   │  觀察變化：「出現了一個對話框，有輸入框和確認按鈕」
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Actor      │  選擇動作：「在輸入框輸入 'My Folder'」
    │             │  （每 4 步會自我批判一次）
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Observer   │  觀察變化：「輸入框顯示了文字」
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Actor      │  選擇動作：「點擊確認按鈕」
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Observer   │  觀察變化：「對話框消失，出現了新資料夾」
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Actor      │  選擇動作：「結束任務」
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Reflector  │  評估結果：「成功！學到：要創建資料夾，
    │             │  需要點新增→輸入名稱→確認」
    └──────┬──────┘
           │
           └───────► 回到 Planner（規劃下一個任務）
```

---

## 6. 狀態機：怎麼決定下一個用哪個 Agent

### 核心概念

系統用**狀態機**（State Machine）來控制流程，有 4 個狀態（Mode）：

```
MODE_PLAN    → 規劃模式（用 Planner）
MODE_OBSERVE → 觀察模式（用 Observer）
MODE_ACT     → 行動模式（用 Actor）
MODE_REFLECT → 反思模式（用 Reflector）
```

### 狀態轉換規則

**檔案**：`droidagent/agent.py:126-201`

讓我用更白話的方式解釋：

```python
class TaskBasedAgent:
    def __init__(self, ...):
        # 初始化時，從「規劃模式」開始
        self.mode = MODE_PLAN
        self.step_count = 0

    def step(self):
        """每次主程式呼叫 step()，Agent 會根據當前模式做不同的事"""

        self.step_count += 1
        print(f"第 {self.step_count} 步，當前模式：{self.mode}")

        # ==================== 模式 1：規劃新任務 ====================
        if self.mode == MODE_PLAN:
            print("👉 用 Planner 規劃新任務...")

            # 重置 Actor 的計數器
            self.actor.reset()

            # Planner 規劃任務並選擇第一步動作
            first_action = self.planner.plan_task()

            if first_action is not None:
                # 有任務 → 切換到「觀察模式」
                self.mode = MODE_OBSERVE
                return first_action  # 回傳動作給主程式執行
            else:
                # 沒任務（可能 LLM 失敗了）→ 不做事
                return None

        # ==================== 模式 2：觀察畫面變化 ====================
        elif self.mode == MODE_OBSERVE:
            print("👉 用 Observer 觀察變化...")

            # Observer 比較前後畫面，總結變化
            observation = self.observer.observe_action_result()

            if observation:
                print(f"觀察到：{observation}")
                # 記錄到記憶系統
                self.memory.working_memory.add_step(observation, 'OBSERVATION')
                self.memory.widget_knowledge.add_observation(...)  # 累積知識

            # 切換到「行動模式」
            self.mode = MODE_ACT
            return None  # 不執行動作，只是觀察

        # ==================== 模式 3：選擇下一步動作 ====================
        elif self.mode == MODE_ACT:
            print("👉 用 Actor 選擇動作...")

            # 檢查：如果已經執行太多步（13 步），強制結束任務
            if self.actor.action_count >= 13:
                print("⚠️ 任務太長了，強制結束")
                self.mode = MODE_REFLECT
                return None

            # Actor 選擇下一步動作
            next_action = self.actor.act()

            if next_action is None:
                # Actor 選擇「結束任務」→ 切換到「反思模式」
                self.mode = MODE_REFLECT
                return None
            else:
                # 有動作 → 切換到「觀察模式」（執行完後要看結果）
                self.mode = MODE_OBSERVE
                return next_action

        # ==================== 模式 4：評估任務結果 ====================
        elif self.mode == MODE_REFLECT:
            print("👉 用 Reflector 評估任務...")

            # Reflector 評估任務成功/失敗，並提取學習
            task_result = self.reflector.reflect()
            print(f"任務結果：{task_result}")

            # 切換回「規劃模式」（準備規劃下一個任務）
            self.mode = MODE_PLAN
            return None
```

### 主程式的無限迴圈

**檔案**：`scripts/run_droidagent.py:59-107`

```python
def main(device, app, persona):
    # 初始化 Agent
    agent = TaskBasedAgent(output_dir, app=app, persona=persona)
    device_manager = DeviceManager(device, app, output_dir)

    # 無限迴圈（直到達到最大步數或 2 小時）
    while True:
        if agent.step_count > 8000:
            print("達到最大步數，結束測試")
            break

        # 檢查畫面是否在載入中
        if is_loading_state(device_manager.current_state):
            print("畫面載入中，等待...")
            time.sleep(1)
            continue

        # ========== 關鍵：呼叫 Agent 的 step() ==========
        action = agent.step()

        # 儲存記憶快照（除錯用）
        agent.save_memory_snapshot()

        # 如果 Agent 回傳了動作，就執行到手機
        if action is not None:
            print(f"執行動作：{action}")

            # 把動作轉成 DroidBot 事件
            events = action.to_droidbot_event()

            # 透過 ADB 發送到手機
            for event in events:
                device_manager.send_event_to_device(event)

            # 更新 Agent 的 GUI 狀態
            agent.set_current_gui_state(device_manager.current_state)
```

### 狀態轉換流程圖

```
開始
  │
  ▼
┌─────────────┐
│ MODE_PLAN   │ Planner 規劃任務
└──────┬──────┘
       │ 回傳第一步動作
       ▼
┌─────────────┐
│主程式執行動作 │ 例如：點擊按鈕
└──────┬──────┘
       │
       ▼
┌─────────────┐
│MODE_OBSERVE │ Observer 觀察變化
└──────┬──────┘
       │ 記錄觀察結果
       ▼
┌─────────────┐
│ MODE_ACT    │ Actor 選擇下一步
└──────┬──────┘
       │
       ├─ 選擇動作 ──────► 主程式執行 ──► 回到 MODE_OBSERVE
       │
       └─ 選擇結束任務 ──► MODE_REFLECT
                           │
                           ▼
                    ┌─────────────┐
                    │MODE_REFLECT │ Reflector 評估結果
                    └──────┬──────┘
                           │
                           └──────────► 回到 MODE_PLAN（規劃下一個任務）
```

### 實際執行範例

```
Step 1, Mode: MODE_PLAN
👉 Planner: 規劃任務「創建新資料夾」
👉 Planner: 第一步動作「點擊新增按鈕 (ID: 10)」
[主程式執行動作]

Step 2, Mode: MODE_OBSERVE
👉 Observer: 觀察到「出現了對話框，有輸入框和確認按鈕」

Step 3, Mode: MODE_ACT
👉 Actor: 選擇動作「在輸入框輸入 'My Folder' (ID: 15)」
[主程式執行動作]

Step 4, Mode: MODE_OBSERVE
👉 Observer: 觀察到「輸入框顯示了文字」

Step 5, Mode: MODE_ACT
👉 Actor: 選擇動作「點擊確認按鈕 (ID: 16)」
[主程式執行動作]

Step 6, Mode: MODE_OBSERVE
👉 Observer: 觀察到「對話框消失，出現新資料夾」

Step 7, Mode: MODE_ACT
👉 Actor: 選擇「結束任務」

Step 8, Mode: MODE_REFLECT
👉 Reflector: 任務成功！學到「創建資料夾需要：點新增→輸入名稱→確認」

Step 9, Mode: MODE_PLAN
👉 Planner: 規劃新任務「開啟剛創建的資料夾」
...
```

### 為什麼要用狀態機？

1. **清楚的控制流程**：不會亂掉，永遠知道下一步該做什麼
2. **容易除錯**：看 log 就知道卡在哪個模式
3. **可以暫停/恢復**：儲存 `self.mode` 就能恢復狀態

---

## 7. 記憶系統怎麼運作

### 四種記憶

| 記憶類型 | 存在哪裡 | 存什麼 | 用途 |
|---------|---------|-------|------|
| **WorkingMemory** | 內存（RAM） | 當前任務的執行步驟 | 給 LLM 看「我做了什麼」 |
| **TaskMemory** | ChromaDB 向量資料庫 | 過去任務的歷史記錄 | 避免重複做相同任務 |
| **SpatialMemory** | ChromaDB 向量資料庫 | Widget 的行為知識 | 記住「點這個按鈕會怎樣」 |
| **PersistentStorage** | ChromaDB 向量資料庫 | 底層儲存 | 提供語義搜尋功能 |

### WorkingMemory：短期記憶

**就像人的「工作記憶」**，只記住當前任務的事。

**檔案**：`droidagent/memories/working_memory.py`

```python
class WorkingMemory:
    def __init__(self, task):
        self.task = task  # 當前任務
        self.steps = []   # 執行步驟列表

    def add_step(self, description, activity, step_type):
        """記錄一個步驟"""
        timestamp = time.strftime('%Y-%m-%d %H:%M:%S')
        self.steps.append((description, activity, step_type, timestamp))
```

**內容範例**：
```python
{
  "task": "創建新資料夾",
  "steps": [
    ("點擊新增按鈕 (ID: 10)", "MainActivity", "ACTION", "2025-01-15 10:23:45"),
    ("出現對話框，有輸入框", "MainActivity", "OBSERVATION", "2025-01-15 10:23:46"),
    ("輸入文字 'My Folder' 到輸入框 (ID: 15)", "MainActivity", "ACTION", "2025-01-15 10:23:47"),
    ("輸入框顯示文字", "MainActivity", "OBSERVATION", "2025-01-15 10:23:48"),
    ("點擊確認按鈕 (ID: 16)", "MainActivity", "ACTION", "2025-01-15 10:23:49"),
    ("對話框消失，新資料夾出現", "MainActivity", "OBSERVATION", "2025-01-15 10:23:50")
  ]
}
```

**Step 類型**：
- `ACTION`：執行的動作
- `OBSERVATION`：觀察到的變化
- `CRITIQUE`：自我批判

### SpatialMemory：Widget 知識庫

**記住每個按鈕、輸入框的功能**，不用每次都重新探索。

**檔案**：`droidagent/memories/spatial_memory.py`

**記錄時機**：每次 Observer 觀察到變化後

```python
def add_widget_wise_observation(self, page, widget_signature, observation, action):
    """
    記錄：在某個頁面上，對某個 Widget 執行某個動作後，觀察到什麼
    """

    # 儲存到 ChromaDB
    self.storage.add_entry(
        document=f"{page} page: {widget.stringify()}",  # 用於語義搜尋
        metadata={
            'type': 'WIDGET',
            'page': page,                    # 例如："MainActivity"
            'widget': widget_signature,      # 例如："Button-add_button-Add"
            'action': action.type,           # 例如："touch"
            'observation': observation,      # 例如："出現對話框"
            'task': task.summary
        }
    )
```

**儲存範例**：
```json
{
  "document": "MainActivity page: a button with resource_id 'add_button'",
  "metadata": {
    "type": "WIDGET",
    "page": "MainActivity",
    "widget": "Button-add_button-Add",
    "action": "touch",
    "observation": "A dialog appeared for creating a new folder",
    "task": "創建新資料夾"
  }
}
```

**檢索時機**：Actor 規劃動作時

```python
def retrieve_widget_knowledge(self, state, widget, N=5):
    """
    檢索：這個 Widget 過去被操作過幾次？每次的結果是什麼？
    """

    # 用當前畫面狀態作為查詢（語義搜尋）
    results = self.storage.query(
        query_texts=[state.signature],  # 例如："MainActivity page: button..."
        n_results=N,
        where={
            'type': 'WIDGET',
            'page': state.activity,
            'widget': widget.signature
        }
    )

    # 取得過去的觀察記錄
    observations = []
    for metadata in results['metadatas']:
        observations.append(f"- {metadata['action']}: {metadata['observation']}")

    # 用 LLM 總結
    summary = llm_summarize(widget, observations)

    return summary
```

**檢索範例**：

當 Actor 看到「新增按鈕」時：

```
查詢: "MainActivity page: button with resource_id 'add_button'"

檢索結果（前 5 筆）:
1. touch → A dialog appeared for creating a new folder
2. touch → A dialog with input field appeared
3. touch → Opens the folder creation interface

LLM 總結:
"This button opens a dialog for creating a new folder"
```

這個總結會被加到 Widget 的描述中：

```json
{
  "ID": 10,
  "widget_type": "Button",
  "resource_id": "add_button",
  "text": "Add",
  "possible_action_types": ["touch"],
  "num_prev_actions": 3,  ← 記憶！
  "widget_role_inference": "Opens a dialog for creating a new folder"  ← 記憶！
}
```

### ChromaDB：向量資料庫

**為什麼用向量資料庫而不是普通資料庫？**

因為需要**語義搜尋**！

**傳統資料庫**：
```sql
SELECT * FROM memories WHERE widget = 'add_button'
```
只能找到完全一樣的關鍵字。

**ChromaDB**：
```python
query_texts=["如何創建新資料夾"]
```
會找到語義相似的文檔，即使沒有「資料夾」這個詞！

**原理**：

1. **文字 → 向量（Embedding）**
   ```
   "創建新資料夾" → [0.12, 0.45, 0.78, -0.33, ...]  (1536 維向量)
   "新增檔案夾"   → [0.15, 0.43, 0.76, -0.35, ...]  (很接近！)
   "刪除文件"     → [-0.82, 0.23, -0.11, 0.67, ...] (很遠)
   ```

2. **計算相似度（餘弦相似度）**
   ```
   similarity("創建新資料夾", "新增檔案夾") = 0.95  (很像！)
   similarity("創建新資料夾", "刪除文件")   = 0.12  (不像)
   ```

3. **返回最相似的文檔**

---

## 8. 完整執行範例

讓我用一個完整的例子，串起所有概念。

### 場景：測試一個筆記 APP

**APP 資訊**：
- 套件名稱：`com.example.notes`
- 頁面列表：`["MainActivity", "EditorActivity", "SettingsActivity"]`
- Persona：名叫 Jade 的使用者，目標是「盡可能測試所有功能」

### 執行流程

#### 啟動階段

```bash
$ python run_droidagent.py --app Notes --profile_id jade
```

```
1. 解析 APK
   ✓ 套件名稱：com.example.notes
   ✓ 主頁面：MainActivity
   ✓ 所有頁面：["MainActivity", "EditorActivity", "SettingsActivity"]

2. 連接設備
   ✓ 設備：emulator-5554
   ✓ 安裝 DroidBotApp
   ✓ 啟用 Accessibility Service

3. 安裝並啟動 APP
   ✓ 安裝 Notes.apk
   ✓ 啟動 APP (MainActivity)

4. 初始化 Agent
   ✓ 創建 Planner, Observer, Actor, Reflector
   ✓ 初始化 ChromaDB 記憶系統
   ✓ 模式：MODE_PLAN
```

#### Step 1-3：第一個任務

```
========== Step 1 ==========
Mode: MODE_PLAN

Planner 看到的資訊：
- 所有頁面：["MainActivity", "EditorActivity", "SettingsActivity"]
- 已訪問：{"MainActivity": 1}
- 未訪問：["EditorActivity", "SettingsActivity"]
- 當前頁面：MainActivity
- 當前畫面：
  {
    "page_name": "MainActivity",
    "children": [
      {
        "ID": 5,
        "widget_type": "Button",
        "resource_id": "add_note_button",
        "text": "New Note",
        "possible_action_types": ["touch"]
      },
      {
        "ID": 8,
        "widget_type": "ListView",
        "possible_action_types": ["scroll"]
      }
    ]
  }

Planner 的輸出：
任務: Create a new note
第一步動作: touch_widget(target_widget_ID=5)

→ 切換到 MODE_OBSERVE
→ 回傳動作給主程式

[主程式執行動作：點擊 ID 5 的按鈕]
```

```
========== Step 2 ==========
Mode: MODE_OBSERVE

Observer 看到的變化：
前一個頁面：MainActivity
當前頁面：EditorActivity

新出現的元件：
- EditText (ID: 12) - 標題輸入框
- EditText (ID: 13) - 內容輸入框
- Button (ID: 14) - 儲存按鈕

Observer 的輸出：
"Opened the note editor page with title and content input fields"

→ 記錄到 WorkingMemory
→ 記錄到 SpatialMemory:
  {
    "page": "MainActivity",
    "widget": "Button-add_note_button-New Note",
    "action": "touch",
    "observation": "Opened the note editor page"
  }
→ 切換到 MODE_ACT
```

```
========== Step 3 ==========
Mode: MODE_ACT

Actor 看到的資訊：
- 當前任務：Create a new note
- 執行歷史：
  1. [ACTION] 點擊 New Note 按鈕
  2. [OBSERVATION] 開啟筆記編輯頁面
- 當前頁面：EditorActivity
- 當前畫面：
  {
    "ID": 12,
    "widget_type": "EditText",
    "resource_id": "title_input",
    "possible_action_types": ["set_text"]
  }

Actor 的輸出：
fill_text(target_widget_ID=12, text="My First Note")

→ 切換到 MODE_OBSERVE
→ 回傳動作給主程式
```

#### Step 4-8：繼續執行任務

```
Step 4 (OBSERVE): 觀察到標題欄位顯示文字
Step 5 (ACT): 在內容欄位輸入文字
Step 6 (OBSERVE): 觀察到內容欄位顯示文字
Step 7 (ACT): 點擊儲存按鈕
Step 8 (OBSERVE): 觀察到回到 MainActivity，新筆記出現在列表中
```

#### Step 9：反思任務

```
========== Step 9 ==========
Mode: MODE_ACT

Actor 判斷任務已完成，選擇「結束任務」

→ 切換到 MODE_REFLECT
```

```
========== Step 10 ==========
Mode: MODE_REFLECT

Reflector 看到的資訊：
- 任務：Create a new note
- 任務完成條件：The task is completed when a new note appears in the note list
- 執行歷史：
  1. 點擊 New Note 按鈕
  2. 觀察到開啟編輯頁面
  3. 在標題欄位輸入 "My First Note"
  4. 觀察到標題顯示
  5. 在內容欄位輸入內容
  6. 觀察到內容顯示
  7. 點擊儲存按鈕
  8. 觀察到回到主頁面，新筆記出現
- 當前頁面：MainActivity（有新筆記）

Reflector 的輸出：
評估: 成功
結果摘要: Successfully created a new note titled "My First Note"
反思:
  - To create a note in this app, click "New Note", fill in title and content, then click save
  - The save button must be clicked to persist the note

→ 記錄到 TaskMemory (ChromaDB)
→ 切換到 MODE_PLAN
```

#### Step 11：規劃第二個任務

```
========== Step 11 ==========
Mode: MODE_PLAN

Planner 看到的資訊：
- 已訪問頁面：{"MainActivity": 2, "EditorActivity": 1}
- 未訪問：["SettingsActivity"]
- 過去任務：
  1. ✓ Create a new note（成功）
- 過去反思：
  - To create a note, click "New Note", fill in fields, then save

Planner 的輸出：
任務: Open the settings page to explore configuration options
第一步動作: touch_widget(target_widget_ID=3)  ← 設定按鈕

→ 繼續循環...
```

---

## 總結

### 系統的核心機制

1. **靜態分析 APK** → 知道有哪些頁面
2. **Accessibility Service** → 即時看到畫面元件
3. **智能轉換** → 元件轉 Widget，自動判斷可執行操作
4. **狀態機控制** → PLAN → OBSERVE → ACT → REFLECT 循環
5. **ChromaDB 記憶** → 記住 Widget 功能，累積測試知識

### 與傳統測試工具的差異

| 傳統工具 | DroidAgent |
|---------|-----------|
| 隨機點擊 | 有目的的任務規劃 |
| 不記得操作過什麼 | 記住每個按鈕的功能 |
| 重複點同樣的東西 | 知識累積，避免重複 |
| 看不懂 UI | 理解「這是登入按鈕」 |
| 無法學習 | 反思機制，累積經驗 |

---

**文件生成時間**：2025-01-15
**作者**：Claude (Anthropic)
**專案**：DroidAgent - Intent-Driven Android GUI Testing Framework
