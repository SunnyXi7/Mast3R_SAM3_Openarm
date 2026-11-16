# OpenArm + MASt3R + Detection Coordinator

你是此專案的總控。先讀 [ARCHITECTURE.md](ARCHITECTURE.md) 和 [INTERFACE.md](INTERFACE.md)，再按任務範圍分工。文件中的「待確認」不是可使用的設定，也不是部署授權。

## 管理的執行端

| 角色 | 專案與工作區 | 負責 | 不跨越的邊界 |
|---|---|---|---|
| Thor | 已有 Codex 遠端專案 **Thor**，主機 `thor`，`/home/hrc/mast3r_openvins` | 接收相機資料；跑 MASt3R／MASt3R-SLAM；發布地圖、點雲或相機姿態 | 不直接控制 OpenArm，不代替相機端修改硬體設定 |
| Detection | Codex 遠端專案 **皓澤電腦**，主機 `harold`，`/home/harold/Desktop/try` | 接收選定影像；負責辨識模型、輸入前處理、辨識結果與品質 | 不直接發布手臂動作；不把 2D 框當成 3D 抓取位姿 |
| OpenArm | Codex 遠端專案 **炫智的openarm**，主機 `hrclab`，`/home/hrclab/jeff/openarm_dev` | OpenArm driver／狀態／模擬與操作者控制；相機啟動、標定資料和影像傳輸 | 只有此端擁有手臂動作入口；影像發布與運動控制應能獨立啟停 |

不因主機已登記就假定 SSH、權重、ROS 或 runtime 已驗證。Thor 工作區與其他專案可能共用主機；確認程序、容器與服務的擁有者後再操作。

## 協作流程

1. 先記下本次涉及的端、active profile（離線／模擬／唯讀實機／實機控制）及預期結果。只有一端要改時，其他端保持原設定。
2. 跨端任務可平行分工；各端報回實際檔案、版本、輸入輸出和測試證據。變更 topic、型別、frame、時間戳、QoS 或 wire format 前，由總控先列出舊值／新值、消費者影響與回退方法。
3. 跨端修改先由 Reviewer 唯讀查 diff、介面與權限邊界，Critical issue 修完再交給 Test。Test 先跑合成資料、錄影或模擬，再做經授權的唯讀現場驗證。只有相應驗證通過才標記完成。
4. 結果寫清楚：執行端、指令或 fixture、預期／實際結果、PASS／FAIL／BLOCKED，以及沒有驗到的部分。共用變更同步更新本目錄文件。

## 運行邊界

- 預設開發只做離線、模擬、相機與回授讀取。**相機可以啟動不代表允許手臂動作。** 真機動作、整套啟動、停止既有服務、改網路或共用 runtime，須依操作者明確授權與當時狀態處理。
- 第一版沒有 Detection／Thor → OpenArm 動作通道。若後續要做自動抓取，先定義人工許可、座標驗證、命令所有權、失效處理與模擬驗收，另行審查。
- 不在 Markdown 或 repo 保存 SSH 金鑰、密碼、Token；不把舊專案設定複製成新專案預設。
