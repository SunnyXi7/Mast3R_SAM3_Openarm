# OpenArm + MASt3R + Detection

本目錄是三端總控工作區。Claude 接手請先讀 [CLAUDE.md](CLAUDE.md) 和 [交接手冊](HANDOFF.md)。三個執行端分別負責 **Thor 重建**、**影像識別**、**OpenArm 與相機傳輸**。總控在本機維護分工、介面、驗證紀錄；各執行端在自己的工作區開發。

| 角色 | Codex 專案／主機 | 工作目錄 | 狀態 |
|---|---|---|---|
| Coordinator | `Openarm+Mast3r+detection`／本機 | `本倉庫` | 已登記 |
| Thor／MASt3R | `Thor`／`thor` | `/home/hrc/mast3r_openvins` | 已核對，臨時訂閱收到 `camera1` |
| Detection／影像識別 | `皓澤電腦`／`harold` | `/home/harold/Desktop/try` | 已核對，臨時訂閱收到 `camera1` |
| OpenArm／相機傳輸 | `炫智的openarm`／`hrclab` | `/home/hrclab/jeff/openarm_dev` | 已核對，既有 `camera1`、`camera3` 串流有資料 |

- [AGENTS.md](AGENTS.md)：Codex 總控的工作規則與各端責任。
- [ARCHITECTURE.md](ARCHITECTURE.md)：資料流、所有權、故障隔離與第一輪驗收。
- [INTERFACE.md](INTERFACE.md)：跨端契約草案與待確認欄位。

### 第一輪驗證

唯讀盤點已確認三端可接收同一份 `camera1` 原始影像。後續先在隔離環境以同一份錄影驗證 Thor 重建與 Detection 2D 輸出，再用靜止相機驗證兩端同時處理。介面、測試證據及待確認事項見 [INTERFACE.md](INTERFACE.md)。

### MASt3R-SLAM 即時 RGB-D

三端各自在專案根目錄提供 `啓動指令_master_openarm.md`，對應 `master-openarm-openarm`、`master-openarm-thor`、`master-openarm-detection`。OpenArm 轉送既有 `camera1` 影像，Thor 產生同影格相對深度和即時點雲，Detection 將目標預覽點回傳 OpenArm。啟動參數與目標點的使用邊界見 [跨端介面](INTERFACE.md#mast3r-slam-容器介面)。
