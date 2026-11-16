# 三端總控交接

## 主機與入口

| SSH alias | 工作目錄 | 專用容器 | 職責 |
|---|---|---|---|
| `hrclab` | `/home/hrclab/jeff/openarm_dev` | `master-openarm-openarm` | 胸口 camera1、影像轉送、目標預覽與本地 MoveIt 規劃 |
| `thor` | `/home/hrc/mast3r_openvins` | `master-openarm-thor` | MASt3R-SLAM 即時點雲、影像對齊深度、AprilTag 定尺 |
| `harold` | `/home/harold/Desktop/try` | `master-openarm-detection` | VLMotion controller、CUDA worker、SAM3 GUI、目標候選 |

每端工作目錄的 `啓動指令_master_openarm.md` 是操作入口。先核對檔案與當前程序，再決定是否需要啟動。三端皆使用 ROS 2 domain 4、Fast DDS 和 host network。Thor 使用 Jazzy，其餘兩端使用 Humble。現有 DDS 防火牆放行與 Thor 本機共享記憶體配置應從各主機唯讀核對。

## 資料流與判讀

1. OpenArm 轉送 camera1 的 RGB 與內參，保留來源時間戳和 `camera1_color_optical_frame`。
2. Thor 發布 `/master_openarm/slam/points`（`PointCloud2`，模型單位）及與 RGB 配準的相對深度。RViz2 可在獨立視覺化環境訂閱此 topic，Fixed Frame 用 `camera1_color_optical_frame`，Reliability 用 Best Effort。
3. AprilTag 使用 25h9、ID 0、黑框邊長 77.8 mm。`depth_tag_scaled` 是標籤定尺的光軸 Z，單位為公尺，供預覽。`depth_metric` 依同影格 D415 對齊深度逐像素核對，差距大於 10 mm 的像素為 NaN。開啟此路徑時，Thor 校正程式須帶 `--reference-depth-topic /camera/camera1/aligned_depth_to_color/image_raw`。
4. Detection 用同影格 RGB、深度與內參反投影。`MASTER_OPENARM_METRIC=1` 使用 `depth_metric` 及同影格 `metric_status`，通過時才送 `/master_openarm/detection/target_metric`。GUI、controller 和 worker 分別啟動。最近觀測的獨立 controller 和 worker 埠是 11100／22100，GUI 從皓澤的圖形桌面啟動。部署也有舊服務埠 11000／22000，連線前以實際程序為準。
5. OpenArm 規劃器使用已接入的胸口相機外參，將合格目標轉到手臂基座。模擬已通過左臂接近規劃。此交接未驗到現場端到端的公尺目標、基座座標及真機夾取。

## 最近一次實測與接手檢查

- Thor 已收到實際即時點雲。標籤定尺回報 `tag_scale_verified: true`。最近操作採純標籤定尺，狀態為 `metric_verified: false`，原因為 `independent depth is unavailable`。較早啟用對齊深度時有間歇取得同時間戳的公尺影格。
- 最近 ROS graph 中 `depth_tag_scaled` 有一個 publisher、零訂閱者，未發現 `target_metric`。判斷是否接通須觀察新鮮的實際訊息，並核對 stamp、frame 和 session，僅看到 topic 名稱不足以判定。
- 先查三端指定容器的 `docker ps`、`docker top`。在 Thor 容器 source ROS 後，唯讀取樣下列兩條 topic。對目標 topic 的取樣安排在操作員於 GUI 觸發識別時。

```bash
ssh thor
docker exec -it master-openarm-thor bash -lc '
  source /opt/ros/jazzy/setup.bash
  timeout 8 ros2 topic echo --once --full-length --qos-reliability best_effort /master_openarm/rgbd/metric_status std_msgs/msg/String
  timeout 20 ros2 topic echo --once --full-length /master_openarm/detection/target_metric std_msgs/msg/String
'
```
