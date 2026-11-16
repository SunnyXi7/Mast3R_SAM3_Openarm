# 跨端介面契約

2026-11-12 第一輪盤點，涉及 OpenArm、Thor、Detection。執行 profile 為**唯讀實機**，使用既有相機串流與臨時 ROS 2 訂閱器。`camera1` 傳輸已實測，重建與識別的應用程式驗收待確認。第一版跨端資料流限於感知，手臂動作入口由 OpenArm 操作者掌控。

## 連接盤點

| 項目 | OpenArm | Thor | Detection |
|---|---|---|---|
| Codex 遠端專案 | `炫智的openarm` | `Thor` | `皓澤電腦` |
| SSH alias | `hrclab` | `thor` | `harold` |
| 工作目錄 | `/home/hrclab/jeff/openarm_dev` | `/home/hrc/mast3r_openvins` | `/home/harold/Desktop/try` |
| 主機 OS、實測 ROS 環境 | Ubuntu 24.04.4，`openarm-dev` 容器內 ROS 2 Humble，host network | Ubuntu 24.04.4，`reconstruction_pointcloud` 容器內 ROS 2 Jazzy，host network | Ubuntu 22.04.5，主機 ROS 2 Humble。識別程式所在子目錄 `VLMotion`，部署用 runtime 待確認 |

Codex 清單與既有 SSH 連線均已核對。OpenArm 工作區版本 `3eaa336`，Detection 的 `VLMotion` 版本 `5cfd64a`。Thor 的 `reconstruction_pointcloud` 源碼已確認以唯讀方式掛載至容器。Detection 主機上的既有 `vlmotion` 容器指向另一個工作區，本次探針在 `harold` 主機執行。

## OpenArm 影像來源

| 相機 | CAM-RGB 原始彩色影像 topic | CAM-INFO 內參 topic | `header.frame_id` | 設定與實測 |
|---|---|---|---|---|
| `camera1`（RealSense D415，第一輪共用來源） | `/camera/camera1/color/image_raw` | `/camera/camera1/color/camera_info` | `camera1_color_optical_frame` | 實測 640×360、`yuv422_yuy2`（YUY2）、`step=1280`，`rs_launch_top.py` 與當前程序皆設定 15 FPS。各端將 YUY2 轉為 RGB／BGR 處理 |
| `camera3`（RealSense D435） | `/camera/camera3/color/image_raw` | `/camera/camera3/color/camera_info` | `camera3_color_optical_frame` | 實測 640×360、`rgb8`、`step=1920`，`rs_launch_far.py` 與當前程序皆設定 15 FPS。本次在 OpenArm 取樣 |
| `camera_left` | 待確認 | 待確認 | 待確認 | `rs_launch_first.py` 有此配置。盤點時無對應 publisher 或影像，實際規格待確認 |

型號由各節點的唯讀 `device_info` 服務回報。`camera1` 韌體為 `5.17.0.10`，`camera3` 為 `5.13.0.55`，兩者回報 USB 3.2。實體安裝位置待確認。

`image_raw` 型別為 `sensor_msgs/msg/Image`，實測 offered QoS 為 **Best Effort / Volatile**。`camera_info` 型別為 `sensor_msgs/msg/CameraInfo`，實測 offered QoS 為 **Reliable / Volatile**。Publisher 的 history／depth 在 graph 回報 `UNKNOWN`，實際值待確認。

兩路運作中的來源皆有 `/color/image_raw/compressed`，型別為 `sensor_msgs/msg/CompressedImage`，實測 **Best Effort / Volatile**，payload 有 JPEG 標頭。`camera1` 的 `format` 回報 `yuv422_yuy2; jpeg compressed mono8`，`camera3` 回報 `rgb8; jpeg compressed bgr8`。第一輪選 `camera1` raw 保留彩色。壓縮路徑的色彩等價性與跨機收訊待確認。

### CameraInfo 與時間戳

| 光學座標系 | 寬×高 | `fx`, `fy`, `cx`, `cy`（像素，取六位小數） | 畸變 |
|---|---|---|---|
| `camera1_color_optical_frame` | 640×360 | `454.660309`, `453.967224`, `317.555908`, `176.475067` | `plumb_bob`，`D=[0,0,0,0,0]` |
| `camera3_color_optical_frame` | 640×360 | `452.868713`, `452.674957`, `325.398529`, `189.927048` | `plumb_bob`，`D=[0,0,0,0,0]` |

兩者 `R` 皆為單位矩陣，`P` 的左側 3×3 對應 `K`，最後一欄皆為零。消費者以收到的 `CameraInfo` 為準。去畸變處理、標定版本、光學 frame 到 OpenArm base 的 TF／外參待確認。2D 結果使用原始影像像素座標，原點為左上角，`u` 向右、`v` 向下。

RealSense driver 以影格時間填 `header.stamp`（`sec`、`nanosec`）。`camera1` metadata 取樣回報 `clock_domain=global_time`。`camera1` 的實測影像和對應內參具有相同 stamp、尺寸與 frame。兩端應以 `camera_id`、來源 stamp 和 frame 關聯資料，預處理後保留來源 stamp，裁切／縮放時保留回原圖的座標轉換。`camera3` 的時鐘域、跨主機時鐘同步與端到端延遲待確認。

## 兩端如何接同一路影像

使用 OpenArm 的單一 `camera1` 發布者。兩端透過 ROS 2 DDS 各自訂閱同一組 raw／CameraInfo topic。跨主機收訊已實測 `rmw_fastrtps_cpp`、`ROS_DOMAIN_ID=4`。部署用固定 IP、port、interface 綁定及額外 discovery 設定待確認。

| 端 | 已實測的接收環境 | 應用程式接法與待確認項 |
|---|---|---|
| OpenArm | 既有 Humble 相機程序使用 `ROS_DOMAIN_ID=4`、`ROS_LOCALHOST_ONLY=0`、`rmw_fastrtps_cpp`。 | 相機端持續提供原始影像與內參，由各端獨立取樣。 |
| Thor | Jazzy 容器的臨時訂閱器使用 domain 4、`ROS_AUTOMATIC_DISCOVERY_RANGE=SUBNET`、解除 `ROS_LOCALHOST_ONLY`，並載入容器內 `/opt/reconstruction_pointcloud/fastdds.xml` 的 UDPv4 profile。 | `reconstruction_pointcloud/src/node.py` 支援以 `--image-transport raw` 訂閱上述 raw topic，Best Effort / Volatile / Keep Last 1。`src/core.py` 支援 YUY2→RGB。現有推論入口訂閱影像，內參先由獨立訂閱核對。容器預設 domain 96，連接既有 OpenArm 時在接收程序指定 4。本輪未啟動推論。 |
| Detection | `harold` 主機的 Humble 臨時訂閱器使用 domain 4、`ROS_LOCALHOST_ONLY=0`、`rmw_fastrtps_cpp`，已接收並解碼 raw 及接收內參。 | `VLMotion/ros2_ws/src/white_point_pipeline` 的 `sam3_recognition_gui` 原始碼已指向同一組 topic，Best Effort / Volatile / Keep Last 5。既有 launch 的 `ros_domain_id` 預設 4，`start_camera` 與 `use_compressed_color` 預設 `false`，並設定 UDPv4 profile。該 launch 也包含位置 publisher。第一輪 2D 驗證只使用識別入口。主機的 `CvBridge` 合成 YUY2→BGR 測試通過，部署環境的解碼和模型推論待確認。 |

## 其他介面

| ID | 資料流 | 來源與待確認項 |
|---|---|---|
| CAM-DEPTH | OpenArm → 有需要的消費者 | 深度 topic、實際配準、時間差、encoding、無效值與單位待確認。第一輪共用輸入為 RGB 與 CameraInfo。 |
| MAP | Thor → 展示／記錄 | 程式預設 `/mast3r/live_cloud` 為 `sensor_msgs/msg/PointCloud2`，frame `mast3r_live_optical`，保留來源 stamp。`/mast3r/live_cloud/status` 為 `std_msgs/msg/String`，含 `session_id`。兩者原始碼 QoS 均為 Reliable / Volatile / Keep Last 1。輸出是逐張替換的點雲，尺度、公尺校正、姿態與重啟行為待驗收。 |
| DET-2D | Detection → 展示／記錄 | 程式宣告 `/sam3_recognition/result` 為 JSON 字串（`std_msgs/msg/String`，Reliable / Transient Local / Keep Last 1）。`/sam3_recognition/overlay` 為 `sensor_msgs/msg/Image`。現有 RGB callback 只傳遞像素，跨端輸出須補齊來源 `camera_id`、`header.stamp`、`header.frame_id`，並核對類別、置信度、原圖 bbox 格式、空結果及過期語義。部署模型版本與實際輸出待確認。 |
| ARM-STATE | OpenArm → 監看／記錄 | Joint names、單位、source stamp、base frame、driver、topic／型別／QoS 與頻率待確認。 |

## 驗證紀錄與下一輪測試

| 執行端 | 指令或 fixture | 預期與實際 | 結果 |
|---|---|---|---|
| 三端 | Codex 專案清單，既有 SSH `pwd`／OS 檢查 | 名稱、主機與實際路徑一致 | PASS |
| OpenArm | `openarm-dev` 內約 8 秒的 `rclpy` 訂閱與 graph 檢查 | `camera1`、`camera3` 收到可解碼 raw、對應 CameraInfo 與帶 JPEG 標頭的 payload。`camera_left` 尚無發布者 | PASS（兩路運作中的串流） |
| OpenArm、Thor、Detection | 同時以 Best Effort / Volatile / depth 5 訂閱，核對 `camera1` 來源 stamp、frame、尺寸與原始 payload 的 BLAKE2b-64 | 三端有 **63 張** raw 同時符合相同 stamp 與雜湊。例如 `1794478798.482570068`，雜湊 `3190ca5300a3e964`，frame `camera1_color_optical_frame`。三端均收到對應內參。OpenArm 探針曾出現一次 `sequence size exceeds remaining buffer`，原因及長時間丟幀率待確認 | PASS（收訊與解碼） |
| Detection | 主機 Humble `CvBridge().imgmsg_to_cv2(..., desired_encoding="bgr8")`，合成 2 像素 YUY2 | 成功回傳 `(1, 2, 3)` BGR 陣列 | PASS（主機轉色） |
| Thor、Detection | 同一份錄影與靜止相機的應用程式驗收 | 重建、識別模型、帶來源 header 的輸出及斷流恢復待驗證 | 待執行 |

1. **錄影重播。** 唯讀查詢確認 OpenArm `rosbag/0029` 的 `camera1` 有 781 筆 JPEG 與 790 筆 CameraInfo，可用於讀包和 stamp 核對。本次檢查的 OpenArm 錄影中未找到 `camera1` raw／CameraInfo 成對錄影。完整彩色驗收可在相機靜止時補錄這兩條 topic。將同一份錄影放在隔離的測試 ROS domain，重播時只選取影像和內參 topic，保留原始 header。通過條件為 Thor 與 Detection 能解出彩色影像、配對來源時間和內參，且點雲與 2D 結果能按來源 stamp 對回原圖。
2. **靜止相機。** 錄影驗收後，沿用 OpenArm 已有的相機發布者，兩端同時接 raw／CameraInfo。保持相機與手臂靜止，記錄各端收訊速率、最後來源 stamp、解碼錯誤及影格取樣差異。在重播環境以暫停／恢復重播驗證過期與恢復行為。通過條件為兩端可持續取得有效影格與對應輸出，且在輸入斷流時可識別過期資料。

本輪僅更新總控文件，臨時訂閱器已退出。無 runtime 變更需要回退。

## MASt3R-SLAM 容器介面

此輪採用三個獨立容器 `master-openarm-openarm`、`master-openarm-thor`、`master-openarm-detection`。在已核對的 domain 4 使用 Fast DDS 和 host network。新資料流使用 `/master_openarm` namespace，沿用既有 `camera1` 作為輸入。

| Topic | 發布端 → 消費端 | 型別／內容 | QoS |
|---|---|---|---|
| `/master_openarm/camera1/image_raw`、`/master_openarm/camera1/camera_info` | OpenArm → Thor | `sensor_msgs/msg/Image`、`sensor_msgs/msg/CameraInfo`。影像和內參原始 header 不改寫 | Best Effort / Volatile |
| `/master_openarm/rgbd/image_raw`、`/master_openarm/rgbd/camera_info` | Thor → Detection | `sensor_msgs/msg/Image` 的 `rgb8`，以及原始像素格的 `sensor_msgs/msg/CameraInfo` | Best Effort / Volatile |
| `/master_openarm/rgbd/depth_relative` | Thor → Detection | `sensor_msgs/msg/Image` 的 `32FC1`。Z 為相機光軸深度，單位為 `model_units`，無效像素為 NaN。影像尺寸及光學 frame 與 RGB 相同 | Best Effort / Volatile |
| `/master_openarm/rgbd/status` | Thor → Detection | `std_msgs/msg/String`，JSON 含 `stamp`（`sec`、`nanosec`）、`frame_id`、`session_id`、`tracking_valid`、`depth_units`、`metric_verified` | Best Effort / Volatile |
| `/master_openarm/slam/points` | Thor → Detection 端監看 | `sensor_msgs/msg/PointCloud2`。以同一影格的光學座標及模型單位發布 XYZ | Best Effort / Volatile |
| `/master_openarm/detection/result` | Detection → 展示／記錄 | `std_msgs/msg/String`，帶來源 stamp、frame、session 與時效狀態的 VLMotion 辨識結果 | Reliable / Volatile |
| `/master_openarm/detection/target_preview` | Detection → OpenArm | `std_msgs/msg/String`，JSON 含 `stamp`、`frame_id`、`session_id`、`point`（`x`、`y`、`z`）、`depth_units: "model_units"`、`metric_verified: false`、`grasp_eligible: false` | Reliable / Volatile |

Detection 以來源 stamp 和 frame 精確配對 RGB、深度、內參與狀態。候選點由配對內參和有效 Z 反投影，OpenArm 以時效、順序、單位和有效值核對後預覽。公尺尺度、經驗證的基座座標及夾取姿態待確認。單一候選點不提供夾爪姿態或安全路徑。

Detection 沿用 VLMotion 的 `controller.launch.py` 與 `worker.launch.py`。新容器透過 `vlmotion_gui.py` 重用原 SAM3 GUI 的畫面、按鈕與 worker 請求，將辨識當下鎖定的影格和 Thor 相對深度配對。原 `sam3_recognition_gui.launch.py` 的 RealSense 裝置處理與位置 publisher 不屬於此資料流。現有 `11000`／`22000` 已由同一 VLMotion 工作區的 `vlpoint-sam3` 容器使用。新 GUI 可由操作者指定連接現有 controller，獨立服務預設使用 loopback 的 `11100`／`22100`。主機和新非特權 GPU 容器均實測 CUDA 初始化失敗，獨立 worker 待驗證。

原有 `/camera/camera1/color/*` 及 `/mast3r/live_cloud*` 保留原介面。新訂閱端使用上表的 namespaced topics。回退時停止本輪三個 `master-openarm-*` 容器，既有相機與其他消費者仍可使用原 topic。停止或重啟已存在的相機、手臂或其他專案服務需依各端操作權限處理。

2026-11-12 核對 Detection 的 UFW 入站限制，確認 Thor 的 domain 4 UDP 封包未抵達。操作者授權後按來源主機及介面放行該 DDS UDP 範圍，三端直連測試通過。現場規則與回退參數在 Detection 主機核對。

### 容器測試紀錄

| 執行端 | 指令或 fixture | 預期與實際 | 結果 |
|---|---|---|---|
| OpenArm → Thor | ROS domain 61，60 組 640×360 合成 RGB／CameraInfo，使用 VLMotion 測試影像與小幅像素位移 | 收到 42 組 `tracking_valid=true` 狀態及 18 組完整配對。配對的深度有效率約 76–81%，每組 10,881–11,655 點。點雲與同像素 Z 和內參的反投影相符。完整配對約 0.74 Hz，長時間速率及封包完整率待確認 | PASS |
| OpenArm → Thor | domain 4 既有 `camera1`，短暫啟動轉送器和 Thor 臨時訂閱器 | 86 組具有同一來源 stamp、尺寸、frame、RGB 資料雜湊及 K。Humble 轉送器另出現 `sequence size exceeds remaining buffer`，原因待確認 | PASS（影像轉送） |
| OpenArm | domain 61 跨主機合成 `target_preview` | 接受新鮮、非公尺候選，拒絕逾期、重複及標為可夾取的候選 | PASS（預覽接收） |
| Detection | Qt GUI、合成 RGB-D 與隔離模擬 worker | 延遲回應仍對應按下辨識時的來源影格。追蹤失效時結果標示 `source_current=false`，且沒有發出預覽點 | PASS（GUI 接線，實際模型未驗證） |
| Thor → Detection（變更前） | domain 4 獨立 ROS 2 端點與單向 UDP 封包 | Thor 封包未抵達 `harold`，同一測試埠可收到 OpenArm 封包 | BLOCKED（直連入站） |
| 三端（UFW 放行後） | 實際容器映像、domain 4、16 秒合成 DDS 測試 | 三端各收到另兩端的可靠訊息。Detection 直接收到 Thor 的 15 組 640×360 RGB、`32FC1`、CameraInfo、狀態與 10,000 點點雲，來源 stamp 對齊，資料核對無誤。Humble 仍出現 `sequence size exceeds remaining buffer` 訊息，本次合成輸入沒有配對缺漏 | PASS（跨端傳輸） |

## AprilTag 尺度與 OpenArm 規劃介面

本輪工作 profile 為離線／模擬。`camera1` 採指定上游版本的 AprilTag 25h9、ID 0，黑框邊長 77.8 mm。同影格的標籤定尺、物體深度的獨立核對，以及 `camera1_color_optical_frame` 到 OpenArm 基座的外參各自需要驗證。OpenArm 是唯一接受本地操作者確認及發出規劃請求的一端。

| 新增 topic | 發布端 → 消費端 | 型別與條件 | QoS |
|---|---|---|---|
| `/master_openarm/rgbd/depth_tag_scaled` | Thor → 預覽 | `sensor_msgs/msg/Image`，`32FC1`，以 AprilTag 平面定尺的光學 Z。保留來源 header | Best Effort / Volatile |
| `/master_openarm/rgbd/depth_metric` | Thor → Detection | `sensor_msgs/msg/Image`，`32FC1`，光學 Z 的單位為公尺。僅保留與獨立同影格深度相差不超過 10 mm 的像素，其餘為 NaN | Best Effort / Volatile |
| `/master_openarm/rgbd/metric_status` | Thor → Detection、OpenArm | `std_msgs/msg/String`，JSON 含原始 `stamp`、`frame_id`、`session_id`、`calibration_id`、`depth_units: "m"`、`tag_scale_verified`、`metric_verified`。只有標籤尺度通過時可提供 `tag_pose_camera`。獨立深度失效時 `metric_verified=false` | Best Effort / Volatile |
| `/master_openarm/detection/target_metric` | Detection → OpenArm | `std_msgs/msg/String`，JSON 含同一筆 `stamp`、`frame_id`、`session_id`、`calibration_id`、`depth_units: "m"`、`metric_verified: true` 及 `point`（`x`、`y`、`z`）。資料只用於本地候選驗證與規劃 | Reliable / Volatile |

既有 `depth_relative`、`rgbd/status` 和 `target_preview` 的 topic、型別和單位沿用既有契約。Detection 只在公尺深度、RGB、內參及同一筆驗證狀態完整配對時發公尺候選。OpenArm 再核對對應的 Thor 狀態、時效、外參、工作空間與規劃結果。回退時停用新增校正／規劃入口，原有相對深度與預覽可獨立運行。

標籤設定來源見 [上游固定版本](https://github.com/hrc-pme/Mast3R_delta_workspace/blob/3d33d6f8f93e2b0836b6fa0dad69e22a989ab8ac/mast3r_openvins/workspace_local_map_unity_v2/config.json#L16-L20)。OpenArm 既有 36h11／80 mm 設定供另一套流程使用。本輪校正工具採固定相機安裝。外參需在 `openarm_body_link0` 獨立量測至少三個驗證標籤位置，通過 10 mm／2° 門檻後才輸出 `verified_static` 檔案。目標接近路徑使用 MoveIt 規劃，實際控制驗收狀態另記錄。

Thor 專用容器的 Fast DDS 由 UDPv4-only 改為 UDPv4 加 16 MiB 本機共享記憶體。共享記憶體承載同一容器內的 MASt3R-SLAM → AprilTag 大影格。其他主機仍使用 UDPv4、domain 4 及既有 topic／QoS。回退時，在 Thor 專用容器停機後把原 `fastdds.udp-only.xml` 還原成 `fastdds.xml`。 [Fast DDS 共享記憶體](https://fast-dds.docs.eprosima.com/en/2.x/fastdds/transport/shared_memory/shared_memory.html)

| 執行端 | 指令或 fixture | 實際 | 結果 |
|---|---|---|---|
| 本機 | `apriltag_deployment/test_offline.py`，OpenCV 4.11.0、NumPy 2.3.5 | 8 項通過，涵蓋 25h9 影像偵測與錯誤 ID、尺度留出檢查、獨立深度遮罩、外參及規劃輸入拒收 | PASS（合成資料） |
| Detection | `MASTER_OPENARM_METRIC=1`，隔離 ROS domain 73，實際 VLMotion `CameraNode` | 合成的同影格輸入保留相對 Z=2.0 並發公尺 Z=0.75。狀態失效、stamp／session 不符及未啟用時保留相對預覽 | PASS（隔離應用測試） |
| Thor | 已部署的原映像，隔離 ROS domain 63 | 完整 640×360 合成 RGB-D 通過。無獨立深度只出標籤預覽，缺件及錯 ID 拒收，外參工具取到 6 筆 pose | PASS（隔離應用測試） |
| OpenArm | 已部署的 `grasp_plan.py`，斷網無設備映射、fake-hardware MoveIt domain 73 | 實際左臂鏈 `left_arm_fingertip` → `openarm_left_fingertip_center` 通過 FK 與規劃。TTY 確認後兩段模擬軌跡成功，部分接近拒收。右臂、夾爪未動 | PASS（模擬手臂接近） |
| 現場標籤 | camera1 既有 RGB 與 Thor 實際 MASt3R-SLAM | 81 張 RGB 皆辨識到 25h9 ID 0，PnP 中位 Z 0.56038 m。Thor 存下 12 組同影格定尺，標籤平面留出 RMS 中位 0.91 mm。檔案為 `observations/camera1_tag_observed_20261112141708Z.json` | PASS（標籤平面） |
| 現場相機 | 操作者明確授權只重啟 camera1，啟用深度、同步及對齊 | OpenArm 端驗到 6 組相同 stamp、optical frame 與尺寸的 RGB、CameraInfo、16UC1 對齊深度 | PASS（相機配對） |
| Thor 現場公尺影格 | camera1 的同影格 16UC1 參考深度與實際 MASt3R-SLAM | 10 組 `depth_metric` 和狀態精確配對，每張通過獨立核對的像素中位 15,632。另有 157 組追蹤影格缺同 stamp 的參考深度，只保留標籤定尺預覽。實測檔為 `observations/camera1_tag_metric_observed_20261112142402Z.json` | PASS（公尺資料有間歇） |
| 皓澤現場公尺影格 | 臨時 ROS 訂閱器 | 觀測容器的暫存 log 目錄初始化失敗，這次未量到現場接收。隔離 `CameraNode` 測試已通過 | 待驗證 |
| 胸口外參與夾取 | 操作者確認胸口安裝，核對舊 `sam3_chest_moveit.launch.py` | 已將舊流程的 `chest_camera_calibration_v1.json` 掛入新規劃器。數值與舊流程轉換矩陣一致，錯誤 frame 會拒收。舊流程的基座方向目標補償和實機夾取待驗收。Thor 的標籤負責尺度校正 | PASS（外參接入、數值驗證） |
