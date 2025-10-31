# DroidAgent 系統架構技術文檔

> 基於多 LLM 協同的 Android GUI 自動化測試框架

## 目錄
1. [系統概述](#1-系統概述)
2. [APK 靜態分析與頁面解析](#2-apk-靜態分析與頁面解析)
3. [運行時 GUI 狀態獲取機制](#3-運行時-gui-狀態獲取機制)
4. [View 到 Widget 的轉換流程](#4-view-到-widget-的轉換流程)
5. [多 LLM 協同架構](#5-多-llm-協同架構)
6. [狀態機驅動的 Agent 調度](#6-狀態機驅動的-agent-調度)
7. [分層記憶系統設計](#7-分層記憶系統設計)
8. [完整執行流程](#8-完整執行流程)

---

## 1. 系統概述

### 1.1 研究動機

傳統的 Android GUI 測試工具（如 Monkey、Espresso）主要依賴隨機探索或預定義腳本，存在以下限制：

- **缺乏語義理解**：無法理解 UI 元件的功能和上下文
- **重複性探索**：缺乏記憶機制，重複執行無意義的操作
- **目標導向性不足**：無法根據測試目標規劃有意義的操作序列

### 1.2 核心創新

DroidAgent 透過以下機制解決上述問題：

| 機制 | 技術實現 | 效果 |
|-----|---------|------|
| **語義理解** | LLM 處理 UI 元件描述 | 理解「登入按鈕」vs「設定按鈕」的語義差異 |
| **知識累積** | ChromaDB 向量資料庫 | 記錄 Widget 行為，避免重複探索 |
| **任務導向** | Persona-driven task planning | 模擬真實用戶場景，而非隨機點擊 |
| **自我調整** | Critique 機制 | 定期評估策略並調整 |

### 1.3 系統架構概覽

```
┌─────────────────────────────────────────────────────────┐
│                     DroidAgent 主程式                     │
│                  (scripts/run_droidagent.py)             │
└───────────────┬─────────────────────────────────────────┘
                │
    ┌───────────┼───────────┐
    │           │           │
    ▼           ▼           ▼
┌────────┐ ┌────────┐ ┌──────────────┐
│APK 解析│ │設備通訊│ │Agent 狀態機   │
│Androguard DroidBot│ │4-LLM 協同    │
└────────┘ └────────┘ └──────┬───────┘
                             │
                    ┌────────▼────────┐
                    │  Memory System  │
                    │  (ChromaDB)     │
                    └─────────────────┘
```

---

## 2. APK 靜態分析與頁面解析

### 2.1 問題陳述

測試系統需要在啟動前獲取 APP 的結構資訊，包括：
- 應用程式包含哪些頁面（Activity）
- 主頁面（Main Activity）是哪一個
- 各頁面之間的導航關係

### 2.2 技術方案：Androguard

**檔案位置**：`droidbot/droidbot/app.py:28-39`

使用 Androguard 庫解析 APK 檔案的 `AndroidManifest.xml`：

```python
from androguard.core.apk import APK

class App:
    def __init__(self, app_path):
        self.apk = APK(app_path)

        # 提取應用程式基本資訊
        self.package_name = self.apk.get_package()
        self.main_activity = self.apk.get_main_activities()[0]
        self.activities = self.apk.get_activities()
        self.permissions = self.apk.get_permissions()
```

**Androguard 工作原理**：

1. **APK 解壓縮**：APK 本質上是 ZIP 壓縮檔
2. **二進位 XML 解析**：AndroidManifest.xml 是二進位格式，需要專門的解析器
3. **結構化資訊提取**：透過 XPath 或正規表達式提取 `<activity>` 標籤

**範例 AndroidManifest.xml**：
```xml
<manifest package="com.ichi2.anki">
    <application>
        <activity android:name=".IntentHandler">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <activity android:name=".DeckPicker"/>
        <activity android:name=".Reviewer"/>
        <activity android:name=".CardBrowser"/>
    </application>
</manifest>
```

### 2.3 頁面列表的後處理

**檔案位置**：`droidagent/config.py:124-144`

```python
def set_app(self, app):
    # 取得應用程式名稱和套件資訊
    self.app_name = app.apk.get_app_name()
    self.package_name = app.get_package_name()

    # 提取套件前綴（用於過濾）
    package_prefix = '.'.join(self.package_name.split('.')[:2])

    # 過濾並簡化頁面名稱
    app_activities = [
        ActivityNameManager.fix_activity_name(a.split('/')[-1])
        for a in app.activities
        if package_prefix in a
        and 'leakcanary' not in a  # 排除第三方除錯工具
        and 'CrashReportDialog' not in a  # 排除系統頁面
    ]

    self.app_activities = app_activities
```

**過濾邏輯**：
- 排除系統頁面（如 CrashReportDialog）
- 排除第三方庫頁面（如 LeakCanary）
- 只保留應用程式本身的頁面

**實際範例**（AnkiDroid）：
```python
app_activities = [
    "IntentHandler",
    "DeckPicker",
    "Reviewer",
    "CardBrowser",
    "NoteEditor",
    "Preferences"
]
```

### 2.4 應用場景

頁面列表在系統中的用途：

1. **測試覆蓋率計算**
   ```python
   coverage = len(visited_activities) / len(app_activities)
   ```

2. **任務規劃**
   ```python
   unvisited_pages = set(app_activities) - set(visited_activities.keys())
   # Planner 會優先規劃訪問未探索的頁面
   ```

3. **日誌記錄**
   ```python
   logger.info(f'Coverage: {len(visited_activities)}/{len(app_activities)}')
   ```

---

## 3. 運行時 GUI 狀態獲取機制

### 3.1 技術挑戰

Android 的安全機制限制了外部程式對應用程式 UI 的存取權限，主要挑戰包括：

- **跨程序 UI 讀取**：無法直接存取其他應用程式的 View Hierarchy
- **即時性要求**：需要在每次操作後立即獲取最新的 GUI 狀態
- **完整性保證**：必須獲取所有可見元件及其屬性

### 3.2 解決方案：Accessibility Service

Android 提供 Accessibility Service API，原本用於輔助技術（如螢幕閱讀器），可以合法地讀取 View Hierarchy。

**架構設計**：

```
┌──────────────┐                          ┌─────────────────┐
│ DroidAgent   │                          │ Android Device  │
│ (電腦端)      │                          │                 │
│              │    ① 請求 GUI 狀態        │ ┌─────────────┐ │
│              ├──────────────────────────►│ │DroidBotApp  │ │
│              │    Socket (port 7336)     │ │(輔助 APK)    │ │
│              │                          │ └──────┬──────┘ │
│              │                          │        │        │
│              │                          │        │ ② 呼叫  │
│              │                          │ ┌──────▼──────┐ │
│              │                          │ │Accessibility│ │
│              │                          │ │Service API  │ │
│              │                          │ └──────┬──────┘ │
│              │                          │        │        │
│              │                          │        │ ③ 讀取  │
│              │                          │ ┌──────▼──────┐ │
│              │    ④ 回傳 JSON           │ │View         │ │
│              │◄──────────────────────────┤ │Hierarchy    │ │
│              │                          │ └─────────────┘ │
└──────────────┘                          └─────────────────┘
```

### 3.3 DroidBotApp 實作細節

**檔案位置**：`droidbot/droidbot/adapter/droidbot_app.py`

#### 3.3.1 初始化流程

```python
class DroidBotAppConn(Adapter):
    def set_up(self):
        # 安裝輔助 APK
        droidbot_app_path = pkg_resources.resource_filename(
            "droidbot", "resources/droidbotApp.apk"
        )
        self.device.adb.run_cmd(["install", droidbot_app_path])

        # 啟用 Accessibility Service
        ACCESSIBILITY_SERVICE = (
            "io.github.ylimit.droidbotapp/"
            "io.github.privacystreams.accessibility.PSAccessibilityService"
        )
        self.device.adb.enable_accessibility_service(ACCESSIBILITY_SERVICE)
```

#### 3.3.2 Socket 通訊機制

```python
def connect(self):
    # 使用 ADB Port Forwarding
    forward_cmd = f"adb -s {self.device.serial} forward tcp:{self.port} tcp:7336"
    subprocess.check_call(forward_cmd.split())

    # 建立 Socket 連線
    self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    self.sock.connect(("localhost", self.port))

    # 啟動監聽執行緒
    listen_thread = threading.Thread(target=self.listen_messages)
    listen_thread.start()
```

#### 3.3.3 訊息接收與解析

```python
def listen_messages(self):
    while self.connected:
        # 讀取封包頭（6 bytes）
        _, _, message_len = self.read_head()

        # 讀取完整訊息
        message = self.sock_read(message_len)

        # 解析 Accessibility Event
        if "AccEvent >>>" in message:
            body = json.loads(message.split("AccEvent >>> ")[1])
            self.last_acc_event = body  # 快取最新狀態
```

### 3.4 View Hierarchy 資料格式

DroidBotApp 透過 `AccessibilityNodeInfo` API 獲取的資料結構：

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
    "checkable": false,
    "enabled": true,
    "visible": true,
    "focused": false,
    "checked": false,
    "selected": false,
    "children": [
      {
        "class": "android.widget.Button",
        "bounds": [100, 200, 300, 280],
        "resource_id": "com.ichi2.anki:id/deck_picker_add",
        "text": "Add",
        "clickable": true,
        "enabled": true,
        "children": []
      }
    ]
  }
}
```

**關鍵屬性說明**：

| 屬性 | 類型 | 說明 |
|-----|------|------|
| `class` | String | View 的完整類名（例如 `android.widget.Button`） |
| `bounds` | Array[4] | 元件在螢幕上的座標 `[left, top, right, bottom]` |
| `resource_id` | String | 開發者定義的資源 ID |
| `text` | String | 元件顯示的文字內容 |
| `content_description` | String | 無障礙描述文字 |
| `clickable` | Boolean | 是否可點擊 |
| `long_clickable` | Boolean | 是否可長按 |
| `scrollable` | Boolean | 是否可滑動 |
| `editable` | Boolean | 是否可編輯（輸入文字） |

---

## 4. View 到 Widget 的轉換流程

### 4.1 設計動機

從 DroidBotApp 獲取的原始 View 資料需要經過轉換，原因包括：

1. **抽象化**：隱藏 Android 底層細節，提供統一的 Widget 介面
2. **智能化**：自動推斷可執行的操作類型
3. **最佳化**：移除冗餘資訊，減少 LLM Token 消耗
4. **記憶化**：生成穩定的 Widget 識別符，支援知識累積

### 4.2 轉換架構

```
DroidBotApp JSON
      ↓
DroidBot DeviceState
  ├─ views (平面列表)
  └─ view_tree (樹狀結構)
      ↓
DroidAgent GUIState
  ├─ widgets (Widget 物件列表)
  └─ root_widgets (根 Widget)
      ↓
Widget 物件
  ├─ view_id
  ├─ widget_type
  ├─ possible_action_types ← 自動推斷
  ├─ signature ← 穩定識別符
  └─ children
```

### 4.3 DroidBot DeviceState 層

**檔案位置**：`droidbot/droidbot/device_state.py:9-83`

```python
class DeviceState:
    def __init__(self, device, views, foreground_activity, activity_stack, ...):
        self.device = device
        self.foreground_activity = foreground_activity  # 當前頁面
        self.activity_stack = activity_stack  # 頁面堆疊

        # 解析並展平 View 結構
        self.views = self.__parse_views(views)

        # 組裝樹狀結構
        self.view_tree = {}
        self.__assemble_view_tree(self.view_tree, self.views)
```

**資料結構轉換**：

原始巢狀結構：
```
LinearLayout (0)
  ├─ Button (1)
  └─ LinearLayout (2)
      ├─ TextView (3)
      └─ EditText (4)
```

轉換後的平面列表：
```python
views = [
    {temp_id: 0, class: 'LinearLayout', children: [1, 2]},
    {temp_id: 1, class: 'Button', parent: 0},
    {temp_id: 2, class: 'LinearLayout', parent: 0, children: [3, 4]},
    {temp_id: 3, class: 'TextView', parent: 2},
    {temp_id: 4, class: 'EditText', parent: 2}
]
```

**優點**：
- 快速索引：`views[id]` 直接存取
- 保留關係：透過 `parent` 和 `children` 維護層次結構

### 4.4 DroidAgent GUIState 層

**檔案位置**：`droidagent/types/gui_state.py:67-83`

```python
class GUIState:
    def from_droidbot_state(self, droidbot_state):
        # 儲存頁面資訊
        self.activity = ActivityNameManager.fix_activity_name(
            droidbot_state.foreground_activity
        )
        self.activity_stack = droidbot_state.activity_stack

        # 簡化 View Tree（移除不必要的屬性）
        view_tree = minimize_view_tree(droidbot_state.view_tree)

        # 遞迴遍歷，創建 Widget 物件
        self.widgets = []
        self.root_widgets = []
        for root_elem in view_tree:
            widget = traverse_widgets(root_elem, self.widgets, droidbot_state.views)
            self.root_widgets.append(widget)

        return self
```

### 4.5 核心演算法：traverse_widgets

**檔案位置**：`droidagent/types/gui_state.py:315-374`

這是系統的核心函數，負責：
1. 推斷可執行操作（`possible_action_types`）
2. 提取關鍵屬性
3. 創建 Widget 物件

```python
def traverse_widgets(elem, processed_widgets, original_views):
    new_elem = OrderedDict()
    possible_action_types = []
    state_properties = []

    # ==================== 操作類型推斷 ====================
    # 基於 Android Accessibility 屬性的規則映射

    # 規則 1：可點擊或可勾選 → touch
    if elem.get('clickable', False) or elem.get('checkable', False):
        possible_action_types.append('touch')

    # 規則 2：可長按 → long_touch
    if elem.get('long_clickable', False):
        possible_action_types.append('long_touch')

    # 規則 3：可編輯 → set_text
    if elem.get('editable', False):
        possible_action_types.append('set_text')

    # 規則 4：可滑動 → scroll
    if elem.get('scrollable', False):
        possible_action_types.append('scroll')

    # ==================== 狀態屬性提取 ====================

    if elem.get('focused', False):
        state_properties.append('focused')

    if elem.get('checked', False):
        state_properties.append('checked')

    if elem.get('selected', False):
        state_properties.append('selected')

    # ==================== 關鍵屬性提取 ====================

    # 只有可互動的元件才賦予 ID
    if 'temp_id' in elem and len(possible_action_types) > 0:
        new_elem['ID'] = elem['temp_id']
        new_elem['view_str'] = original_views[elem['temp_id']]['view_str']

    # Widget 類型（簡化類名）
    if 'class' in elem:
        # "android.widget.Button" → "Button"
        new_elem['widget_type'] = elem['class'].split('.')[-1]

    # 文字內容（限制長度）
    if 'text' in elem and elem['text']:
        new_elem['text'] = elem['text'][:100]

    # 無障礙描述
    if 'content_description' in elem:
        new_elem['content_description'] = elem['content_description']

    # Resource ID（簡化）
    if 'resource_id' in elem and elem['resource_id']:
        # "com.ichi2.anki:id/deck_picker_add" → "deck_picker_add"
        new_elem['resource_id'] = elem['resource_id'].split('/')[-1]

    # 密碼欄位標記
    if elem.get('is_password'):
        new_elem['is_password'] = True

    # 儲存推斷結果
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

### 4.6 轉換範例

**輸入（Android 原生 View）**：
```json
{
  "class": "android.widget.Button",
  "bounds": [100, 200, 300, 280],
  "resource_id": "com.ichi2.anki:id/deck_picker_add",
  "text": "Add",
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
  "resource_id": "deck_picker_add",
  "text": "Add",
  "possible_action_types": ["touch"],
  "bounds": [[100, 200], [300, 280]],
  "children": []
}
```

**轉換效果**：
- 類名簡化：`android.widget.Button` → `Button`
- Resource ID 簡化：`com.ichi2.anki:id/deck_picker_add` → `deck_picker_add`
- 操作推斷：`clickable: true` → `possible_action_types: ["touch"]`
- 移除冗餘：移除 `enabled`, `long_clickable` 等不必要屬性

### 4.7 Widget Signature 生成

**檔案位置**：`droidagent/types/widget.py:84-104`

為了在 Widget 重新載入後仍能識別，需要生成穩定的識別符：

```python
@cached_property
def signature(self):
    # 選擇不可變屬性
    immutable_props = ['content_description', 'resource_id']

    # 非輸入框的文字也是不可變的
    if 'set_text' not in self.possible_action_types:
        immutable_props.append('text')

    ingredients = []
    for prop in immutable_props:
        if prop in self.elem_dict and self.elem_dict[prop]:
            ingredients.append(self.elem_dict[prop])

    # 遞迴加上子元件的 signature
    ingredients.extend([child.signature for child in self.children])

    # 如果沒有任何識別屬性，使用座標
    if len(ingredients) == 0:
        ingredients = [str(self.elem_dict['bounds'])]

    # 加上 Widget 類型作為前綴
    ingredients.insert(0, self.widget_type)

    return '-'.join(ingredients)
```

**Signature 範例**：
- `Button-deck_picker_add-Add`
- `EditText-username_input`
- `ImageView-profile_picture`
- `TextView-[[100,200],[300,250]]`（無其他識別屬性時使用座標）

---

## 5. 多 LLM 協同架構

### 5.1 設計理念

DroidAgent 採用**分工協作**的多 LLM 架構，將測試任務分解為四個專門角色：

| 角色 | 職責 | 使用模型 | 輸出 |
|-----|------|---------|------|
| **Planner** | 任務規劃 | GPT-4o-mini | 新任務描述 + 第一步動作 |
| **Observer** | 狀態觀察 | GPT-4o-mini | GUI 變化摘要 |
| **Actor** | 動作選擇 | GPT-4o-mini | 下一步動作 |
| **Reflector** | 結果評估 | GPT-4o-mini | 成功/失敗 + 學習 |

### 5.2 協同工作流程

```
    ┌─────────────┐
    │  Planner    │  規劃任務："創建新筆記"
    │             │  第一步動作："點擊新增按鈕"
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Observer   │  觀察變化："開啟編輯頁面，有標題和內容輸入框"
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Actor      │  選擇動作："在標題欄位輸入 'My Note'"
    │             │  （每 4 步自我批判）
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Observer   │  觀察變化："標題欄位顯示文字"
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Actor      │  選擇動作："點擊儲存按鈕"
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Observer   │  觀察變化："回到主頁面，新筆記出現"
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Actor      │  選擇動作："結束任務"
    └──────┬──────┘
           │
           ▼
    ┌─────────────┐
    │  Reflector  │  評估："成功！學到創建筆記的流程"
    └──────┬──────┘
           │
           └──────► 回到 Planner（下一個任務）
```

### 5.3 各組件詳細設計

#### 5.3.1 Planner：任務規劃器

**檔案位置**：`droidagent/_planner.py:30-47`

**輸入資訊**：
```python
# 1. APP 結構資訊
app_activities = ["MainActivity", "EditorActivity", "SettingsActivity"]
visited_activities = {"MainActivity": 5, "EditorActivity": 2}
unvisited_pages = ["SettingsActivity"]

# 2. 歷史任務（從 ChromaDB 檢索）
task_history = memory.task_memory.retrieve_task_history(max_len=20)

# 3. 相關反思（語義檢索）
reflections = memory.task_memory.retrieve_task_reflections(current_state, N=5)

# 4. 當前 GUI 狀態（含 Widget 知識）
current_gui = current_state.describe_screen_w_memory(memory)

# 5. Persona 資訊
ultimate_goal = "盡可能訪問所有頁面並測試核心功能"
```

**Prompt 結構**（`droidagent/prompts/plan.py:18-73`）：
```python
system_message = f'''
你是任務規劃助手，為 {persona_name} 規劃使用 {app_name} 的任務。

APP 資訊：
- 所有頁面：{activities}
- 已訪問：{visited_activities}
- 未訪問：{unvisited_pages}
- 當前頁面：{current_activity}

任務需符合：
1. 真實性：對應實際使用場景
2. 重要性：優先核心功能
3. 多樣性：不重複過去任務
4. 難度：從當前狀態可達成
'''

user_message = f'''
過去任務歷史：{task_history}
過去反思：{reflections}
當前畫面：{current_gui}

請規劃下一個任務，包括：
- 任務描述
- 完成條件
- 粗略計劃
'''
```

**輸出範例**：
```
任務: 在筆記 APP 中創建新筆記
完成條件: 新筆記出現在筆記列表中
計劃: 點擊新增按鈕 → 輸入標題和內容 → 儲存
```

接著透過 Function Call 選擇第一步動作。

#### 5.3.2 Observer：狀態觀察器

**檔案位置**：`droidagent/_observer.py`

**工作流程**：
```python
def observe_action_result(self):
    old_state = AppState.previous_gui_state
    new_state = AppState.current_gui_state

    # 比較兩個狀態，找出差異
    changed, appeared, disappeared = old_state.diff_widgets(new_state)

    # 使用 LLM 總結變化
    observation = prompt_state_change_summary(
        old_state, new_state, changed, appeared, disappeared
    )

    # 記錄到 WorkingMemory
    self.memory.working_memory.add_step(observation, 'OBSERVATION')

    # 記錄到 SpatialMemory（Widget 知識庫）
    if observation and last_action.target_widget:
        self.memory.widget_knowledge.add_widget_wise_observation(
            page=old_state.activity,
            widget_signature=last_action.target_widget.signature,
            observation=observation,
            action=last_action
        )
```

**Prompt 範例**：
```python
system_message = f'''
總結 {persona_name} 執行動作後的 GUI 變化。

執行的動作：{last_action}
動作前頁面：{old_state.activity}
動作後頁面：{new_state.activity}

變化的元件：{changed_widgets}
新出現的元件：{appeared_widgets}
消失的元件：{disappeared_widgets}

用 1-2 句話總結主要變化。
'''
```

**輸出範例**：
```
"開啟了筆記編輯頁面，包含標題輸入框、內容輸入框和儲存按鈕"
```

#### 5.3.3 Actor：動作選擇器

**檔案位置**：`droidagent/_actor.py:31-58`

**批判機制**：
```python
CRITIQUE_COUNTDOWN = 4

def act(self):
    # 每 4 步動作後進行批判
    if self.critique_countdown == 0:
        self.critique_countdown = CRITIQUE_COUNTDOWN

        critique, workaround = prompt_critique(self.memory)

        if critique:
            full_critique = f'{critique} {workaround}'
            self.memory.working_memory.add_step(full_critique, 'CRITIQUE')

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
評估當前執行策略是否有效。

當前任務：{task.summary}
最近 4 步：{working_memory.get_recent_steps(4)}

問題：
1. 策略是否有效？
2. 是否陷入循環？
3. 是否需要調整？

輸出：批判 + 建議
'''
```

**Critique 範例**：
```
批判: 已經連續點擊三次新增按鈕但對話框都關閉了，可能遺漏了必要步驟
建議: 點擊確認前應先在輸入框填入筆記標題
```

#### 5.3.4 Reflector：結果評估器

**檔案位置**：`droidagent/_reflector.py`

**工作流程**：
```python
def reflect(self):
    task = self.memory.working_memory.task

    # LLM 評估任務結果
    assessment, result_summary, reflections = prompt_task_reflection(
        task,
        self.memory.working_memory,
        current_gui_state
    )

    # 更新任務物件
    task.assessment = assessment  # "成功" or "失敗"
    task.result_summary = result_summary
    task.reflections = reflections

    # 儲存到 TaskMemory
    self.memory.task_memory.record_task_result(task, reflections, ...)

    return assessment
```

**Prompt 範例**：
```python
system_message = f'''
評估任務執行結果。

任務：{task.summary}
完成條件：{task.end_condition}
執行歷史：{working_memory.to_string()}
當前畫面：{current_gui_state.describe_screen()}

問題：
1. 任務是否成功？
2. 可學到什麼知識？
'''
```

**輸出範例**：
```json
{
  "assessment": "成功",
  "result_summary": "成功創建了標題為 'My Note' 的新筆記",
  "reflections": [
    "創建筆記需要：點擊新增 → 輸入標題 → 輸入內容 → 儲存",
    "標題欄位必須填寫，否則儲存按鈕會保持停用狀態"
  ]
}
```

---

## 6. 狀態機驅動的 Agent 調度

### 6.1 設計模式

DroidAgent 使用**有限狀態機**（Finite State Machine）控制執行流程，確保各 Agent 的有序協作。

**狀態定義**：
```python
MODE_PLAN    = 'plan'      # 規劃模式
MODE_OBSERVE = 'observe'   # 觀察模式
MODE_ACT     = 'act'       # 行動模式
MODE_REFLECT = 'reflect'   # 反思模式
```

### 6.2 狀態轉換邏輯

**檔案位置**：`droidagent/agent.py:126-201`

```python
class TaskBasedAgent:
    def __init__(self, ...):
        self.mode = MODE_PLAN  # 初始狀態
        self.step_count = 0

        # 初始化四個 Agent
        self.planner = Planner(memory, prompt_recorder)
        self.observer = Observer(memory, prompt_recorder)
        self.actor = Actor(memory, prompt_recorder)
        self.reflector = Reflector(memory, prompt_recorder)

    def step(self, droidbot_state=None):
        """每次呼叫執行一步，根據當前模式調度不同 Agent"""

        self.step_count += 1

        if self.mode == MODE_PLAN:
            # ==================== 規劃新任務 ====================
            self.actor.reset()
            first_action = self.planner.plan_task()

            if first_action is not None:
                self.mode = MODE_OBSERVE  # 狀態轉換
                return first_action
            return None

        elif self.mode == MODE_OBSERVE:
            # ==================== 觀察變化 ====================
            observation = self.observer.observe_action_result()

            # 記錄到記憶系統
            if observation:
                self.memory.working_memory.add_step(observation, 'OBSERVATION')
                self.memory.widget_knowledge.add_observation(...)

            self.mode = MODE_ACT  # 狀態轉換
            return None

        elif self.mode == MODE_ACT:
            # ==================== 選擇動作 ====================

            # 檢查任務長度限制
            if self.actor.action_count >= MAX_ACTIONS:
                self.mode = MODE_REFLECT  # 強制反思
                self.inject_action_entry('任務過長，強制結束', 'TASK_ABORTED')
                return None

            next_action = self.actor.act()

            if next_action is None:
                # Actor 選擇結束任務
                self.mode = MODE_REFLECT
                return None
            else:
                self.mode = MODE_OBSERVE  # 執行後需觀察
                return next_action

        elif self.mode == MODE_REFLECT:
            # ==================== 評估結果 ====================
            task_result = self.reflector.reflect()

            self.mode = MODE_PLAN  # 準備下一個任務
            return None
```

### 6.3 主循環設計

**檔案位置**：`scripts/run_droidagent.py:59-107`

```python
def main(device, app, persona):
    agent = TaskBasedAgent(output_dir, app=app, persona=persona)
    device_manager = DeviceManager(device, app, output_dir)

    MAX_STEP = 8000

    while True:
        # 終止條件檢查
        if agent.step_count > MAX_STEP:
            break

        # Loading 狀態處理
        if is_loading_state(device_manager.current_state):
            time.sleep(1)
            device_manager.fetch_device_state()
            continue

        # 呼叫 Agent 執行一步
        action = agent.step()

        # 儲存記憶快照（用於除錯）
        agent.save_memory_snapshot()

        # 如果有動作，執行到設備
        if action is not None:
            # 轉換為 DroidBot 事件
            events = action.to_droidbot_event()

            # 透過 ADB 發送
            for event in events:
                device_manager.send_event_to_device(event, ...)

            # 更新 GUI 狀態
            agent.set_current_gui_state(device_manager.current_state)
```

### 6.4 狀態轉換圖

```
        開始
         │
         ▼
    ┌─────────┐
    │MODE_PLAN│ Planner 規劃任務
    └────┬────┘
         │ 回傳第一步動作
         ▼
  [主程式執行動作]
         │
         ▼
  ┌──────────────┐
  │MODE_OBSERVE  │ Observer 觀察變化
  └──────┬───────┘
         │
         ▼
  ┌─────────┐
  │MODE_ACT │ Actor 選擇動作
  └────┬────┘
       │
       ├──► 選擇動作 ───► [執行] ───► MODE_OBSERVE ┐
       │                                         │
       └──► 結束任務 ───────────────────────────►│
                                                 │
                                                 ▼
                                          ┌────────────┐
                                          │MODE_REFLECT│ Reflector 評估
                                          └─────┬──────┘
                                                │
                                                └──► MODE_PLAN
```

### 6.5 執行範例

```
Step 1, Mode: MODE_PLAN
→ Planner: 規劃任務「創建新筆記」
→ Planner: 第一步動作「touch_widget(ID=5)」
[主程式執行：點擊 ID 5]

Step 2, Mode: MODE_OBSERVE
→ Observer: 「開啟編輯頁面，有標題和內容輸入框」

Step 3, Mode: MODE_ACT
→ Actor: 「fill_text(ID=12, 'My Note')」
[主程式執行：輸入文字]

Step 4, Mode: MODE_OBSERVE
→ Observer: 「標題欄位顯示 'My Note'」

Step 5, Mode: MODE_ACT
→ Actor: 「fill_text(ID=13, '筆記內容')」
[主程式執行：輸入文字]

Step 6, Mode: MODE_OBSERVE
→ Observer: 「內容欄位顯示文字」

Step 7, Mode: MODE_ACT
→ Actor: 「touch_widget(ID=14)」（儲存按鈕）
[主程式執行：點擊儲存]

Step 8, Mode: MODE_OBSERVE
→ Observer: 「回到主頁面，新筆記出現在列表」

Step 9, Mode: MODE_ACT
→ Actor: 「end_task()」

Step 10, Mode: MODE_REFLECT
→ Reflector: 「成功！學到創建筆記流程」

Step 11, Mode: MODE_PLAN
→ Planner: 規劃新任務「開啟剛創建的筆記」
...
```

---

## 7. 分層記憶系統設計

### 7.1 架構概覽

DroidAgent 實作了四層記憶系統，模擬人類的記憶機制：

| 記憶類型 | 儲存位置 | 生命週期 | 用途 |
|---------|---------|---------|------|
| **WorkingMemory** | 內存 | 單一任務 | 當前任務的執行上下文 |
| **TaskMemory** | ChromaDB | 持久化 | 過去任務的歷史記錄 |
| **SpatialMemory** | ChromaDB | 持久化 | Widget 行為知識庫 |
| **PersistentStorage** | ChromaDB | 持久化 | 底層向量資料庫 |

**檔案位置**：`droidagent/memories/memory.py`

```python
class Memory:
    def __init__(self, name):
        # ChromaDB 向量資料庫（兩個 collection）
        self.history = PersistentStorage(f'{name}_primary')
        self.knowledge = PersistentStorage(f'{name}_knowledge')

        # 四層記憶
        self.working_memory = WorkingMemory()
        self.task_memory = TaskMemory(self.history, self.knowledge)
        self.widget_knowledge = SpatialMemory(self.knowledge)
```

### 7.2 WorkingMemory：短期記憶

**檔案位置**：`droidagent/memories/working_memory.py`

```python
class WorkingMemory:
    def __init__(self, task=None):
        self.task = task  # 當前任務
        self.steps = []   # (description, activity, type, timestamp)

    def add_step(self, description, activity, step_type):
        timestamp = time.strftime('%Y-%m-%d %H:%M:%S')
        self.steps.append((description, activity, step_type, timestamp))
```

**Step 類型**：
- `ACTION`：執行的動作（例如：點擊按鈕）
- `OBSERVATION`：觀察到的變化
- `CRITIQUE`：自我批判

**範例**：
```python
{
  "task": "創建新筆記",
  "steps": [
    ("touch_widget(ID=5)", "MainActivity", "ACTION", "2025-01-15 10:23:45"),
    ("開啟編輯頁面", "EditorActivity", "OBSERVATION", "2025-01-15 10:23:46"),
    ("fill_text(ID=12, 'My Note')", "EditorActivity", "ACTION", "2025-01-15 10:23:47"),
    ("標題欄位顯示文字", "EditorActivity", "OBSERVATION", "2025-01-15 10:23:48")
  ]
}
```

### 7.3 SpatialMemory：Widget 知識庫

**檔案位置**：`droidagent/memories/spatial_memory.py`

**設計理念**：記錄「在某頁面上，對某 Widget 執行某動作後，觀察到什麼變化」。

#### 7.3.1 知識記錄

```python
def add_widget_wise_observation(self, page, widget_signature, observation, action):
    # 更新互動計數
    if widget_signature not in self.widget_knowledge_map[page]:
        self.widget_knowledge_map[page][widget_signature] = {
            'action_count': defaultdict(int),
            'observation_count': 0
        }

    self.widget_knowledge_map[page][widget_signature]['action_count'][action.type] += 1

    # 儲存到 ChromaDB
    self.storage.add_entry(
        document=f"{page} page: {widget.stringify()}",
        metadata={
            'type': 'WIDGET',
            'page': page,
            'widget': widget_signature,
            'action': action.type,
            'observation': observation,
            'task': task.summary
        }
    )
```

**儲存範例**：
```json
{
  "document": "MainActivity page: button with resource_id 'add_note_button'",
  "metadata": {
    "type": "WIDGET",
    "page": "MainActivity",
    "widget": "Button-add_note_button-New Note",
    "action": "touch",
    "observation": "開啟筆記編輯頁面",
    "task": "創建新筆記"
  }
}
```

#### 7.3.2 知識檢索

```python
def retrieve_widget_knowledge(self, state, widget, N=5):
    # 語義檢索：用當前 GUI 狀態作為查詢
    results = self.storage.query(
        query_texts=[state.signature],
        n_results=N,
        where={
            'type': 'WIDGET',
            'page': state.activity,
            'widget': widget.signature
        }
    )

    # 提取觀察記錄
    observations = self.storage.stringify_entries(results, mode='widget_knowledge')

    # LLM 總結
    widget_role = prompt_summarized_widget_knowledge(
        widget.stringify(),
        observations
    )

    return widget_role
```

**檢索範例**：

查詢：「MainActivity 頁面上的 add_note_button」

檢索結果：
```
- touch → 開啟筆記編輯頁面
- touch → 跳轉到編輯器介面
- touch → 顯示新筆記編輯畫面
```

LLM 總結：
```
"此按鈕用於開啟新筆記的編輯介面"
```

#### 7.3.3 知識注入

**檔案位置**：`droidagent/types/gui_state.py:122-188`

```python
def describe_screen_w_memory(self, memory):
    for widget in self.widgets:
        widget_info = widget.to_dict()

        # 注入互動計數
        if len(widget.possible_action_types) > 0:
            action_counts = memory.widget_knowledge.get_performed_action_counts(
                self.activity, widget.signature
            )
            interaction_count = sum(action_counts.values())
            widget_info['num_prev_actions'] = interaction_count

        # 注入功能總結
        if memory.widget_knowledge.has_widget_knowledge(self.activity, widget.signature):
            widget_role = memory.widget_knowledge.retrieve_widget_knowledge(self, widget)
            if widget_role:
                widget_info['widget_role_inference'] = widget_role
```

**注入後的 Widget 描述**：
```json
{
  "ID": 5,
  "widget_type": "Button",
  "resource_id": "add_note_button",
  "text": "New Note",
  "possible_action_types": ["touch"],
  "num_prev_actions": 3,
  "widget_role_inference": "用於開啟新筆記的編輯介面"
}
```

### 7.4 ChromaDB：向量資料庫

**檔案位置**：`droidagent/memories/memory.py:13-72`

#### 7.4.1 為何使用向量資料庫

傳統關係型資料庫基於關鍵字匹配，無法處理語義查詢：

```python
# 傳統資料庫
SELECT * FROM memories WHERE widget = 'add_button'
# 只能找到完全一樣的關鍵字

# ChromaDB
query_texts=["如何創建新筆記"]
# 可以找到語義相似的文檔，即使沒有 '創建' 或 '筆記' 關鍵字
```

#### 7.4.2 向量嵌入原理

```python
# 文字 → 向量（Embedding）
"創建新筆記" → [0.12, 0.45, 0.78, -0.33, ..., 0.91]  # 1536 維
"新增筆記"   → [0.15, 0.43, 0.76, -0.35, ..., 0.89]  # 相似！
"刪除文件"   → [-0.82, 0.23, -0.11, 0.67, ..., -0.15] # 不相似

# 計算餘弦相似度
similarity("創建新筆記", "新增筆記") = 0.95
similarity("創建新筆記", "刪除文件") = 0.12
```

#### 7.4.3 實作細節

```python
class PersistentStorage:
    def __init__(self, name):
        self.name = name
        self.entry_id = 0
        self.db = PersistentStorageManager.create_storage(name)

    def add_entry(self, document, metadata):
        """新增記錄（自動生成 embedding）"""
        timestamp = time.strftime('%Y-%m-%d %H:%M:%S')

        self.db.add(
            documents=[document],  # ChromaDB 自動轉為向量
            metadatas=[{'timestamp': timestamp, **metadata}],
            ids=[str(self.entry_id)]
        )

        self.entry_id += 1

    def query(self, query_texts, n_results=5, where=None):
        """語義檢索"""
        return self.db.query(
            query_texts=query_texts,  # 查詢文字（轉為向量）
            n_results=n_results,      # 返回前 N 筆最相似結果
            where=where               # Metadata 過濾條件
        )
```

### 7.5 TaskMemory：任務記憶

**檔案位置**：`droidagent/memories/task_memory.py`

```python
class TaskMemory:
    def __init__(self, primary_storage, knowledge_storage):
        self.storage = primary_storage  # 儲存任務歷史
        self.knowledge_storage = knowledge_storage  # 儲存反思
        self.task_results = {}  # 實驗數據記錄

    def record_task_result(self, task, reflections, working_memory_steps):
        # 記錄任務結果到 primary storage
        self.storage.add_entry(
            document=task.result_summary,
            metadata={
                'type': 'TASK_RESULT',
                'task': task.summary,
                'task_result': task.assessment
            }
        )

        # 記錄反思到 knowledge storage
        for reflection in reflections:
            self.knowledge_storage.add_entry(
                document=reflection,
                metadata={
                    'type': 'TASK',
                    'reflection': reflection,
                    'task': task.summary
                }
            )
```

---

## 8. 完整執行流程

### 8.1 系統啟動流程

**入口檔案**：`scripts/run_droidagent.py`

```bash
$ python run_droidagent.py --app AnkiDroid --profile_id jade --is_emulator
```

```python
# 1. 解析命令列參數
args = parser.parse_args()

# 2. 建立輸出目錄
timestamp = time.strftime("%Y%m%d%H%M%S")
output_dir = f'evaluation/data_new/{args.app}/agent_run_{args.profile_id}_{timestamp}'

# 3. 連接 Android 設備
device = Device(device_serial='emulator-5554', is_emulator=True)
device.set_up()
device.connect()

# 4. 載入並安裝 APK
app = App(f'target_apps/{args.app}.apk')
device.install_app(app)  # 觸發 dumpsys package 解析
device.start_app(app)

# 5. 載入 Persona
persona = load_profile(args.profile_id)
persona['ultimate_goal'] = 'visit as many pages as possible'
persona['initial_knowledge'] = initial_knowledge_map(args.app, ...)

# 6. 初始化 Agent
agent = TaskBasedAgent(output_dir, app=app, persona=persona)

# 7. 初始化 DeviceManager
device_manager = DeviceManager(device, app, output_dir)

# 8. 進入主循環
main(device, app, persona)
```

### 8.2 主循環執行

```python
def main(device, app, persona):
    while True:
        # 終止條件檢查
        if agent.step_count > MAX_STEP:
            break

        # Loading 狀態等待
        if is_loading_state(device_manager.current_state):
            time.sleep(1)
            continue

        # Agent 執行一步
        action = agent.step()

        # 儲存記憶快照
        agent.save_memory_snapshot()

        # 執行動作到設備
        if action is not None:
            events = action.to_droidbot_event()
            for event in events:
                device_manager.send_event_to_device(event)

            # 更新 GUI 狀態
            agent.set_current_gui_state(device_manager.current_state)
```

### 8.3 完整執行範例

**場景**：測試 AnkiDroid（記憶卡 APP）

```
========== 啟動階段 ==========
✓ 解析 APK
  - 套件：com.ichi2.anki
  - 頁面：["IntentHandler", "DeckPicker", "Reviewer", "CardBrowser", "NoteEditor"]
✓ 連接設備：emulator-5554
✓ 安裝並啟動 APP
✓ 初始化 Agent（模式：MODE_PLAN）

========== Step 1 (MODE_PLAN) ==========
Planner 分析：
- 所有頁面：5 個
- 已訪問：{"DeckPicker": 1}
- 未訪問：["Reviewer", "CardBrowser", "NoteEditor"]
- 當前頁面：DeckPicker
- 當前畫面有：
  - Button (ID=10) "Add"
  - RecyclerView (ID=15) 顯示牌組列表

Planner 輸出：
→ 任務：「創建新牌組」
→ 第一步動作：touch_widget(ID=10)

[主程式執行：點擊 ID 10]

========== Step 2 (MODE_OBSERVE) ==========
Observer 比較：
- 前一頁面：DeckPicker
- 當前頁面：DeckPicker
- 新出現元件：
  - Dialog
  - EditText (ID=18) "Deck name"
  - Button (ID=20) "OK"

Observer 輸出：
→ 「出現對話框，包含牌組名稱輸入框和確認按鈕」

記錄到 SpatialMemory：
{
  page: "DeckPicker",
  widget: "Button-deck_picker_add-Add",
  action: "touch",
  observation: "出現對話框，包含牌組名稱輸入框"
}

========== Step 3 (MODE_ACT) ==========
Actor 看到：
- 任務：創建新牌組
- 執行歷史：
  1. 點擊 Add 按鈕
  2. 出現對話框
- 當前畫面：EditText (ID=18)

Actor 輸出：
→ fill_text(ID=18, "Vocabulary")

[主程式執行：輸入文字]

========== Step 4 (MODE_OBSERVE) ==========
Observer 輸出：
→ 「輸入框顯示 'Vocabulary'，確認按鈕變為可用」

========== Step 5 (MODE_ACT) ==========
Actor 輸出：
→ touch_widget(ID=20)

[主程式執行：點擊確認]

========== Step 6 (MODE_OBSERVE) ==========
Observer 輸出：
→ 「對話框關閉，牌組列表中出現新牌組 'Vocabulary'」

========== Step 7 (MODE_ACT) ==========
Actor 判斷任務完成，輸出：
→ end_task()

========== Step 8 (MODE_REFLECT) ==========
Reflector 評估：
- 任務：創建新牌組
- 完成條件：新牌組出現在列表中
- 當前狀態：有 "Vocabulary" 牌組

Reflector 輸出：
→ 評估：成功
→ 結果：「成功創建名為 'Vocabulary' 的新牌組」
→ 反思：
  - 「創建牌組需要：點擊 Add → 輸入名稱 → 確認」
  - 「牌組名稱必須至少包含一個字元」

記錄到 TaskMemory (ChromaDB)

========== Step 9 (MODE_PLAN) ==========
Planner 分析：
- 已訪問：{"DeckPicker": 2}
- 未訪問：["Reviewer", "CardBrowser", "NoteEditor"]
- 過去任務：✓ 創建新牌組（成功）

Planner 輸出：
→ 任務：「在 Vocabulary 牌組中新增卡片」
→ 第一步動作：touch_widget(ID=22)  # Vocabulary 牌組

[循環繼續...]
```

### 8.4 輸出資料結構

```
evaluation/data_new/AnkiDroid/agent_run_jade_20250115102345/
├── agent_config.json
├── exp_info.json
├── exp_data.json
├── prompts/
│   ├── plan_step_1.json
│   ├── act_step_3.json
│   └── ...
├── memory_snapshots/
│   ├── step_1/
│   │   ├── scratch.json
│   │   └── long_term_memory.txt
│   └── ...
└── events/
    ├── event_2025-01-15_102346.json
    └── ...
```

**exp_data.json**：
```json
{
  "app_activities": ["DeckPicker", "Reviewer", "CardBrowser", "NoteEditor"],
  "visited_activities": {
    "DeckPicker": 12,
    "Reviewer": 5,
    "CardBrowser": 3,
    "NoteEditor": 2
  },
  "task_results": {
    "創建新牌組": {
      "result": "成功",
      "summary": "成功創建名為 'Vocabulary' 的新牌組",
      "reflections": ["創建牌組需要：點擊 Add → 輸入名稱 → 確認"],
      "num_actions": 4,
      "num_critiques": 0
    }
  },
  "API_usage": {
    "gpt-4o-mini": {
      "prompt_tokens": 123456,
      "completion_tokens": 45678
    }
  }
}
```

---

## 9. 總結

### 9.1 技術貢獻

1. **多 LLM 協同架構**：透過分工合作提升測試效率
2. **語義化 Widget 表示**：結合 Accessibility Service 與 LLM 實現語義理解
3. **向量資料庫驅動的記憶系統**：支援知識累積與檢索
4. **狀態機控制流程**：確保 Agent 協作的確定性

### 9.2 與傳統工具的比較

| 面向 | 傳統工具 | DroidAgent |
|-----|---------|-----------|
| **探索策略** | 隨機或腳本 | Persona 驅動的任務規劃 |
| **語義理解** | 無 | LLM 理解 Widget 功能 |
| **知識累積** | 無 | ChromaDB 向量資料庫 |
| **自我調整** | 無 | Critique 機制 |
| **測試覆蓋** | 基於程式碼覆蓋率 | 基於頁面訪問和功能探索 |

---

**文件版本**：v1.0
**生成時間**：2025-01-15
**專案**：DroidAgent - Intent-Driven Android GUI Testing Framework
