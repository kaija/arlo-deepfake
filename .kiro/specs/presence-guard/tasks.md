# Implementation Plan

任務依「需要什麼環境」分成三段，讓沒有 Windows 機器的期間也能推進：

- **A 段（任務 1–9）**：純 .NET，可在 macOS 上開發與測試。
- **B 段（任務 10–16）**：需要 Windows 11 24H2 開發機。
- **C 段（任務 17）**：需要具備 NPU 的 Copilot+ PC。

每個任務都以測試先行：先寫失敗的測試，再寫最少的實作讓它通過。

## A 段：核心邏輯（macOS 可進行）

- [ ] 1. 建立方案結構與核心 CI
  - 建立 solution、`src/PresenceGuard.Core`（net10.0）與 `tests/PresenceGuard.Core.Tests`（xUnit）
  - 加入 `Directory.Build.props`（nullable、警告視為錯誤、分析器）與 `.editorconfig`
  - 建立 GitHub Actions `core` 工作流程：在 Linux 上建置、測試、產生覆蓋率報告，覆蓋率低於 80% 時失敗
  - 加入架構測試：斷言 `PresenceGuard.Core` 組件不引用任何 `Windows.*` 或 `Microsoft.UI.*` 組件
  - _Requirements: 19.3_

- [ ] 2. 實作領域模型
  - [ ] 2.1 建立觀測值與引擎的資料型別
    - 撰寫 `RectF`、`HeadPose`、`FaceObservation`、`FrameQuality`、`FrameObservation`
    - 撰寫 `EnvironmentState`、`ProtectionState`、`PauseReason`、`ManualOverride`、`EngineInput`、`EngineCommand` 階層
    - 撰寫 `IClock` 與測試用的 `FakeClock`
    - 加入測試：以反射斷言 `FrameObservation` 及其成員型別不含位元組陣列或 `Span` 欄位
    - _Requirements: 3.3, 3.4, 19.3_

  - [ ] 2.2 實作 `UserSettings` 與範圍驗證
    - 撰寫 `UserSettings`、`PeekReaction`、`TrustedNetwork`、`HotkeyBindings`，包含預設值
    - 撰寫 `SettingsValidator`：檢查離開等待（5–120 秒）、倒數（0–15 秒）、暫停時長的範圍，回傳具體的錯誤與允許範圍
    - 為每個欄位的上下界與預設值撰寫測試
    - _Requirements: 6.1, 9.9, 14.1, 14.6_

  - [ ] 2.3 實作 `SettingsStore`
    - 定義 `IFileStore` 抽象並撰寫記憶體內的測試替身
    - 實作 JSON 載入與儲存、變更事件、還原預設值
    - 實作損毀處理：改名備份、以預設值啟動、回報一次性通知事件
    - 實作 `SchemaVersion` 遷移的進入點
    - _Requirements: 2.7, 14.2, 14.3, 14.4, 14.5_

- [ ] 3. 實作日誌與遙測聚合
  - [ ] 3.1 實作受限型別的日誌 API
    - 撰寫 `IEventLog`，方法只接受列舉型別的事件代碼與數值參數，不接受任意物件
    - 實作滾動檔案寫入與 5 MB 總量上限
    - 測試：超過上限時最舊的檔案被移除；API 表面沒有可接收 `FrameBuffer` 或任意字串的多載
    - _Requirements: 3.6_

  - [ ] 3.2 實作 `TelemetryAggregator`
    - 撰寫 `TelemetryBatch`、`LatencyHistogram`、`ITelemetrySink` 與 `NullTelemetrySink`
    - 實作計數器累計與每日至多一批的送出節流
    - 測試：遙測關閉時 sink 從未被呼叫；批次型別不含任何識別碼欄位；sink 拋出例外時聚合器不向外傳播
    - _Requirements: 17.1, 17.2, 17.3, 17.4, 17.5, 17.7_

- [ ] 4. 實作視覺前後處理
  - [ ] 4.1 實作 `FrameBuffer` 與緩衝池
    - 撰寫 `FrameBuffer`（BGR 位元組、寬、高）與 `FrameBufferPool`
    - 歸還時清零緩衝區
    - 測試：歸還後的緩衝區內容全為零；池不會無限成長
    - _Requirements: 3.1, 3.4_

  - [ ] 4.2 實作 `FrameQualityGate`
    - 計算平均亮度與變異數，判定 `Ok` 或 `TooDark`
    - 以合成的全黑、近黑、正常、均勻灰階影格撰寫測試
    - _Requirements: 9.1, 9.8_

  - [ ] 4.3 實作模型清單載入
    - 撰寫 `ModelSpec`、`ModelManifest` 與 JSON 載入器
    - 實作 SHA-256 驗證，雜湊不符時拒絕載入
    - 測試：缺欄位、雜湊不符、未知解碼器名稱的錯誤處理
    - _Requirements: 19.4, 20.1_

  - [ ] 4.4 實作 letterbox 前處理與座標還原
    - 依 `ModelSpec` 將 `FrameBuffer` 縮放並填入預先配置的張量，套用色彩順序與正規化
    - 實作模型座標到原始影格座標的還原
    - 測試：多種長寬比下，還原後的座標與已知位置一致；推論迴圈中不配置新陣列
    - _Requirements: 19.4, 19.5_

  - [ ] 4.5 實作人臉偵測解碼器與 NMS
    - 定義 `IModelRunner`、`IModelOutputSink`、`IFaceDetector`、`RawFace`
    - 實作 YuNet 輸出解碼（邊界框、5 個關鍵點、分數）與 NMS
    - 以固定的輸出張量夾具撰寫測試
    - _Requirements: 5.1, 19.2_

  - [ ] 4.6 實作人體偵測解碼器
    - 定義 `IBodyDetector`
    - 以 YOLOX-Nano 的輸出格式為第一候選實作解碼，只保留 person 類別並回傳是否存在
    - 以固定的輸出張量夾具撰寫測試
    - _Requirements: 7.1, 7.6, 19.2_

  - [ ] 4.7 實作 `FaceTracker` 與主要用戶選擇
    - 以 IoU 比對跨影格人臉並指派 `TrackId`
    - 實作主要用戶規則：無主要用戶時取最大者；維持同一軌跡直到消失超過容忍時間
    - 測試：其他人臉短暫變大時主要用戶不切換；主要用戶消失超過容忍時間後重新選擇
    - _Requirements: 4.1, 4.2, 4.3_

  - [ ] 4.8 實作 `LandmarkHeadPoseEstimator`
    - 定義 `IHeadPoseEstimator`
    - 由 5 個關鍵點推算 yaw 與 pitch；人臉邊長低於門檻時回傳 `null`
    - 以合成關鍵點（正面、左右轉、上下轉）對應已知角度範圍撰寫測試
    - _Requirements: 5.1, 5.7_

  - [ ] 4.9 組裝 `VisionPipeline`
    - 串接品質檢查、人臉偵測、追蹤、頭部姿態、人體偵測，輸出 `FrameObservation`
    - 實作級聯：有人臉時不執行人體偵測；只對非主要用戶計算頭部姿態；`TooDark` 時不執行任何模型
    - 以假的 `IModelRunner` 測試各分支的模型呼叫次數，並斷言處理後緩衝區已歸還
    - _Requirements: 3.4, 7.6, 9.1, 15.2_

- [ ] 5. 實作 `PresenceEngine`
  - [ ] 5.1 實作離開偵測狀態機
    - 實作 `Present`、`Absent`、`Countdown`、`Locked`、`Suspended` 與其轉移
    - 無人時間的累計與歸零：偵測到人、有輸入、例外解除
    - 倒數設為 0 時直接鎖定
    - `Quality != Ok` 與各抑制旗標進入 `Suspended`；`KeepUnlockedWhenDisplayRequired` 關閉時忽略顯示器保持開啟旗標
    - 以 `FakeClock` 測試每條轉移與邊界時間
    - _Requirements: 7.2, 7.3, 7.4, 7.5, 7.6, 8.2, 8.3, 8.4, 8.5, 9.4, 9.5, 9.6, 9.8, 9.9_

  - [ ] 5.2 實作防窺狀態機
    - 實作 `Idle`、`Candidate`、`Active`、`Clearing`、`Dismissed` 與其轉移
    - 以 `TrackId` 為單位累計朝向螢幕的持續時間
    - 沒有主要用戶或頭部姿態為 `null` 時不觸發
    - 測試：路人短暫出現不觸發；不同 `TrackId` 的時間不被加總；手動關閉後同一事件不再顯示
    - _Requirements: 5.2, 5.3, 5.4, 5.5, 5.6, 5.7, 6.7_

  - [ ] 5.3 實作 `FrameRatePolicy`
    - 依狀態機狀態、人臉數量變化與省電模式輸出 `Idle`、`Normal`、`Elevated`
    - 測試每個觸發條件與穩定 10 秒後降級
    - _Requirements: 15.2, 15.3, 15.8_

  - [ ] 5.4 組裝 `PresenceEngine.Step`
    - 合併兩個狀態機與幀率策略，依防窺反應等級產生 `ShowPeekHint`、`ShowPrivacyCover` 或 `LockSession`
    - 產生 `StartCountdown`、`CancelCountdown`、`HideOverlays`、`ReportStatus`
    - 離開偵測 `Suspended` 時防窺仍運作；`Quality != Ok` 時兩者都暫停
    - 防窺與離開偵測可分別停用
    - 測試：相同輸入序列產生相同命令序列（可決定性）
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 9.7, 12.6_

- [ ] 6. 實作 `ActivationPolicy`
  - [ ] 6.1 實作網路信任判定
    - 撰寫 `NetworkContext`（連線類型、SSID）與判定函式
    - 以表格式測試涵蓋：無連線、只有有線、Wi-Fi 在清單內外、Wi-Fi 與有線並存
    - _Requirements: 11.1, 11.2, 11.3, 11.5, 11.6_

  - [ ] 6.2 實作保護狀態解析
    - 依判定順序合併暫停原因、用戶暫停計時、手動覆寫與網路信任
    - `ForcedOff` 在網路情境改變時重設為 `Auto`；`ForcedOn` 不受網路影響
    - 暫停計時到期後自動恢復，並提供剩餘時間
    - 輸出「因未知網路而自動啟用」的一次性事件
    - 以完整組合表測試
    - _Requirements: 8.7, 8.8, 9.2, 9.3, 11.4, 11.8, 12.2, 12.3, 12.4, 12.5_

- [ ] 7. 實作 `GuardRuntime`
  - 在 Core 定義平台服務介面：`IFrameSource`、`IInputIdleSource`、`IEnvironmentSource`、`INetworkSource`、`ISessionLocker`、`IOverlayPresenter`、`IStatusPresenter`
  - 實作推論迴圈：依幀率等級取最新影格、執行 `VisionPipeline`、呼叫 `PresenceEngine.Step`、分派命令
  - 影格通道容量為 1，舊影格被覆蓋
  - 倒數期間以 100 ms 週期輪詢輸入閒置時間
  - 保護 `Disabled` 或 `Paused` 時停止影格來源且不執行推論；恢復時重新啟動
  - 鎖定失敗時改顯示遮蓋覆蓋層
  - 以假的平台服務撰寫端對端測試：無人 → 倒數 → 鎖定命令；倒數中有輸入 → 取消；工作階段鎖定 → 影格來源停止
  - _Requirements: 3.7, 8.2, 8.7, 8.8, 15.6, 15.7_

- [ ] 8. 建立重播測試基礎設施
  - [ ] 8.1 實作觀測值追蹤檔與重播器
    - 定義 `EngineInput` 的 JSON Lines 序列化格式
    - 撰寫 `ReplayRunner`：逐筆餵給 `PresenceEngine` 並收集命令
    - 撰寫指標計算：誤鎖次數、鎖定延遲、防窺檢出率與誤觸發次數
    - _Requirements: 16.1, 16.2, 16.3, 16.4, 16.6_

  - [ ] 8.2 撰寫合成情境與重播測試
    - 以程式產生追蹤檔：低頭閱讀、轉頭交談、戴口罩（只有人體）、背景路人經過、旁觀者正對螢幕、真正離開
    - 建立 `tests/PresenceGuard.Replay.Tests`，對每個情境斷言指標符合驗收門檻
    - 將重播測試加入 `core` 工作流程
    - _Requirements: 16.1, 16.2, 16.3, 16.4, 16.5_

- [ ] 9. 建立模型工具鏈與 CPU 推論
  - [ ] 9.1 撰寫模型轉檔工具
    - 建立 `tools/convert`（Python、`pyproject.toml`）
    - 撰寫腳本：取得候選人臉與人體偵測模型、固定輸入尺寸、匯出 ONNX、產生 INT8 QDQ 量化版本
    - 撰寫量化前後輸出的比對腳本並輸出報告
    - 產生 `models/manifest.json` 項目，如實填寫授權欄位與 `commercialDistribution`
    - _Requirements: 19.4, 19.5, 20.1_

  - [ ] 9.2 實作 `OrtModelRunner`
    - 以 `Microsoft.ML.OnnxRuntime` 實作 `IModelRunner`，`SessionOptions` 由外部工廠注入
    - 啟動時建立 session 並預先配置輸入輸出張量；記錄每次推論耗時
    - 整合測試（CPU EP）：以授權允許的測試圖片跑完整 `VisionPipeline`，斷言偵測到預期數量的人臉與人體
    - _Requirements: 15.7, 19.2_

## B 段：Windows 平台與 UI（需要 Windows 開發機）

- [ ] 10. 建立 Windows 專案與封裝
  - 建立 `PresenceGuard.Inference`、`PresenceGuard.Platform`、`PresenceGuard.App`（WinUI 3）專案
  - 撰寫 MSIX 資訊清單：`runFullTrust`、`webcam`、`internetClient` capability 與 `windows.startupTask` 擴充；不宣告定位與 Wi-Fi 控制
  - 建立 `build` 工作流程：建置 x64 與 ARM64、執行 Windows 專案測試、產生 MSIX
  - 加入測試：解析資訊清單並斷言 capability 清單與預期完全一致
  - _Requirements: 1.6, 18.6, 18.8_

- [ ] 11. 實作 `ExecutionProviderManager`
  - 以薄包裝隔離 `ExecutionProviderCatalog`，使探測邏輯可用替身測試
  - 實作 `ProbeAsync`：依 EP 名稱判定 `Ready`、`NeedsDownload`、`NotSupported`
  - 實作 `EnsureReadyAsync`：轉送下載進度，處理失敗狀態，成功後註冊
  - 實作 `CreateNpuSessionOptions`：篩選 NPU 裝置並附加，停用 CPU fallback；每次重新列舉裝置
  - 測試：無相容 EP、下載失敗、裝置清單為空的行為
  - _Requirements: 1.1, 1.2, 1.3, 1.4, 15.1_

- [ ] 12. 實作平台服務
  - [ ] 12.1 實作 `CameraFrameSource`
    - 以 `SharedReadOnly` 初始化 `MediaCapture`，失敗時回報 `Unavailable` 且不以獨占模式重試
    - 以 `MediaFrameReader` 取得影格並轉為 `FrameBuffer`；格式改變時重新配置
    - 不呼叫任何設定格式或控制項的 API
    - `StopAsync` 釋放 `MediaCapture`；開啟失敗後每 30 秒重試
    - 實作鏡頭列舉、依設定選擇鏡頭、辨識內建鏡頭
    - 權限被拒時回報專屬狀態
    - _Requirements: 1.5, 2.3, 3.5, 3.7, 9.2, 10.1, 10.2, 10.3, 10.4, 10.5, 14.8_

  - [ ] 12.2 實作輸入閒置、鎖定與工作階段監看
    - `InputIdleMonitor`：以 `GetLastInputInfo` 回報閒置時間，不安裝任何 hook
    - `SessionLocker`：呼叫 `LockWorkStation` 並回報成功與否
    - `SessionMonitor`：訂閱工作階段鎖定與解鎖通知
    - _Requirements: 7.4, 7.7, 8.4, 8.6, 8.7, 8.8, 8.9_

  - [ ] 12.3 實作環境偵測器與 `EnvironmentMonitor`
    - `LidMonitor`、`PresentationDetector`、`DisplayRequestDetector`、`PowerModeMonitor`
    - 每個偵測器實作共同介面，失敗時回報「未偵測到」並寫入日誌
    - `EnvironmentMonitor` 彙整為不可變的 `EnvironmentState`
    - 以替身偵測器測試彙整邏輯與失敗隔離
    - _Requirements: 9.3, 9.4, 9.9, 15.8_

  - [ ] 12.4 實作 `CallDetector`
    - 麥克風策略：列舉預設擷取端點的音訊工作階段，排除本行程
    - 鏡頭策略：讀取 `ConsentStore\webcam` 的使用時間戳，排除本 App；可由設定旗標關閉
    - 任一策略成立即回報通話中
    - 以替身策略測試合併邏輯；以登錄檔資料夾具測試時間戳解析
    - _Requirements: 9.5_

  - [ ] 12.5 實作 `NetworkMonitor`
    - 訂閱網路狀態變更，輸出 `NetworkContext`
    - 以 `GetConnectedSsid` 取得 SSID；不呼叫任何需要定位權限的 API
    - _Requirements: 11.1, 11.2, 11.3, 11.5, 11.6_

  - [ ] 12.6 實作 `HotkeyService`
    - 以 `RegisterHotKey` 註冊開關保護、暫停、關閉覆蓋層的快捷鍵
    - 註冊失敗時回報衝突；設定變更時重新註冊
    - _Requirements: 6.6, 12.1, 12.7_

- [ ] 13. 實作啟動流程與系統匣
  - [ ] 13.1 實作 composition root 與啟動流程
    - 組裝 Core、Inference、Platform 的實作並啟動 `GuardRuntime`
    - 啟動時執行硬體探測：不支援畫面、EP 下載進度畫面、找不到鏡頭畫面
    - 引導流程未完成時不建立 `CameraFrameSource`
    - 未處理例外時寫入日誌、釋放鏡頭後結束
    - _Requirements: 1.2, 1.3, 1.4, 1.5, 2.6_

  - [ ] 13.2 實作系統匣
    - 撰寫 `Shell_NotifyIcon` 包裝
    - 四種狀態各一個形狀不同的圖示；工具提示顯示狀態與原因
    - 選單：啟用或停用、暫停與立即恢復（含剩餘時間）、防窺反應等級、開啟設定、結束
    - 「結束」會停止 `GuardRuntime`、釋放鏡頭並結束行程
    - 自動啟動時不建立主視窗
    - _Requirements: 12.1, 12.4, 12.5, 13.1, 13.2, 13.3, 13.4, 13.5, 13.6, 18.5_

- [ ] 14. 實作覆蓋層
  - [ ] 14.1 實作覆蓋視窗基底與 `PeekHintWindow`
    - 無邊框、最上層、不搶焦點的視窗基底；顯示器配置改變時重建
    - `PeekHintWindow`：每個顯示器的邊緣細框與角落圖示，滑鼠穿透
    - 實作 `IOverlayPresenter` 的顯示與隱藏
    - _Requirements: 6.2, 6.5, 6.8_

  - [ ] 14.2 實作 `PrivacyCoverWindow`
    - 每個顯示器一個全螢幕遮蓋，以不透明底色為基準，系統允許時疊加壓克力材質
    - 點擊或快捷鍵關閉，並呼叫 `PresenceEngine.NotifyCoverDismissed`
    - _Requirements: 6.3, 6.5, 6.6, 6.7_

  - [ ] 14.3 實作 `CountdownWindow`
    - 顯示剩餘秒數與取消方式，每秒更新
    - 覆蓋層顯示時發出 UI Automation 通知事件（倒數與防窺提示皆同）
    - _Requirements: 8.1, 21.5_

- [ ] 15. 實作引導與設定畫面
  - [ ] 15.1 實作首次引導流程
    - 步驟：隱私說明、鏡頭授權、遙測選擇（預設不同意）、信任目前網路、完成
    - 鏡頭權限被拒時顯示說明與 Windows 隱私設定捷徑
    - 完成後寫入 `OnboardingCompleted`
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5, 2.7_

  - [ ] 15.2 實作設定畫面
    - 功能開關、離開等待與倒數秒數、防窺反應等級、暫停時長、顯示器保持開啟時不鎖定、自動啟用、登入時啟動、快捷鍵
    - 信任網路的檢視、新增目前網路與移除
    - 鏡頭選擇；顯示目前鏡頭、NPU 類型與 App 版本
    - 遙測開關與實際回傳欄位的說明
    - 超出範圍的輸入顯示允許範圍；還原預設值
    - _Requirements: 11.7, 11.8, 12.6, 12.7, 14.1, 14.2, 14.5, 14.6, 14.7, 14.8, 17.5, 17.6, 18.4_

  - [ ] 15.3 實作未知網路提示與關於畫面
    - 保護因未知網路自動啟用時顯示一次提示，提供標記為信任的動作
    - 由 `models/manifest.json` 與第三方元件清單產生授權聲明頁
    - _Requirements: 11.4, 20.3_

  - [ ] 15.4 實作在地化與無障礙
    - 所有字串移入繁體中文與英文資源檔；依系統語言選擇並可在設定中切換
    - 為所有互動元件設定自動化名稱；確認完整的鍵盤操作順序
    - 支援淺色、深色與高對比主題
    - 加入測試：兩種語言的資源鍵集合完全一致
    - _Requirements: 21.1, 21.2, 21.3, 21.4, 21.6_

- [ ] 16. 實作啟動、更新與發布
  - [ ] 16.1 實作 `StartupService`
    - 查詢與切換 `StartupTask` 狀態，與 `LaunchAtLogin` 設定同步
    - _Requirements: 18.4, 18.5_

  - [ ] 16.2 實作 `UpdateCoordinator`
    - 每日檢查更新；有新版本時等待安全時機（工作階段鎖定或保護 `Disabled`）
    - 在安全時機登記重新啟動並套用更新
    - 以替身測試安全時機的判定：保護啟用中不觸發
    - _Requirements: 18.2, 18.3_

  - [ ] 16.3 建立發布工作流程
    - 撰寫模型授權檢查腳本：`manifest.json` 中有 `commercialDistribution: false` 時失敗
    - 撰寫 `.appinstaller` 範本（2021 schema、背景更新、不顯示提示、不阻擋啟動）
    - 建立 `release` 工作流程：授權檢查、建置兩種架構、簽章步驟（憑證來源以參數注入）、產生 `.appinstaller`、上傳成品
    - 為授權檢查腳本撰寫測試
    - _Requirements: 18.1, 18.2, 18.8, 20.2_

  - [ ] 16.4 實作遙測傳送
    - 實作對接選定服務的 `ITelemetrySink`，並接上當機報告
    - 遙測同意狀態改變時即時切換為 `NullTelemetrySink` 或實際 sink
    - 測試：序列化後的內容只含 `TelemetryBatch` 定義的欄位
    - _Requirements: 17.2, 17.3, 17.4, 17.5, 17.7_

## C 段：實機驗證工具（需要 Copilot+ PC）

- [ ] 17. 建立實機測試與量測工具
  - [ ] 17.1 撰寫 NPU 實機測試
    - 建立 `tests/PresenceGuard.Device.Tests`
    - 測試：每個模型在停用 CPU fallback 的設定下成功建立 session
    - 測試：各模型的單次推論延遲低於各幀率等級的可用時間
    - 建立 `device` 工作流程，在 self-hosted runner 上執行
    - _Requirements: 1.1, 15.1, 15.7_

  - [ ] 17.2 撰寫觀測值錄製工具
    - 建立 `tools/recorder`：在開發建置中把 `GuardRuntime` 每個週期的 `EngineInput` 寫成追蹤檔
    - 以編譯條件確保發布建置不含錄製功能，並加入測試驗證
    - _Requirements: 3.1, 16.6_

  - [ ] 17.3 撰寫資源量測腳本
    - 撰寫腳本：定時記錄行程的 private working set 與電池電量，輸出 CSV
    - 提供三種模式：未啟動 App、只開鏡頭不推論、保護完整啟用
    - _Requirements: 15.4, 15.5_

## 不在任務清單內的前置事項

以下不是程式碼工作，但會卡住部分任務，需要另外安排：

| 事項 | 影響的任務 |
|---|---|
| 取得 Intel Core Ultra Copilot+ PC | B 段與 C 段全部 |
| 決定申請憑證的法律主體並購買 OV 程式碼簽章憑證 | 16.3 的簽章步驟 |
| 選定遙測服務 | 16.4 |
| 選定人體偵測模型（需在 NPU 上評估算子覆蓋率） | 4.6、9.1 可能需調整解碼器 |
| 錄製驗收測試影片並取得參與者同意 | 以真實資料驗證 Requirement 16 |
| 量測鏡頭耗電與記憶體，決定是否調整架構 | 可能新增任務（見設計文件風險 1、5） |
