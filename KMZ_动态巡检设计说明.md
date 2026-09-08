# Dock 3 KMZ 与动态目标巡检设计

## 已实现的离线航线职责

生成器把 Cloud API 返回的 Dock 3 起飞参考点写入 WPML 的
`wpml:takeOffRefPoint`，把接口中的返航高度写入 `wpml:globalRTHHeight`。
返航高度字段缺失、不是数值或不在 2–1500 m 时，固定使用 45 m。任务的
`finishAction` 是 `goHome`，因此巡检点完成后由飞行器返回该 Dock 3，而不是
把第一个巡检点误当作返航点。

建议 Cloud API 为生成器提供一个稳定、已鉴权的只读接口（所有坐标均为 WGS84）：

```json
{
  "data": {
    "takeoff_ref_point": {
      "longitude": 119.971325,
      "latitude": 30.365020,
      "ellipsoid_height": 15.14
    },
    "rth_height": 100
  }
}
```

页面也兼容 `lng` / `lat`，并递归寻找 `rth_height`、`rthHeight`、
`globalRTHHeight`、`return_home_height` 等常用字段。实际接入时应让 Cloud API
设置 CORS 白名单；令牌仅用于本次浏览器请求，不会写入 KMZ。

## 固定巡检任务

1. 在页面中先加载 Cloud API 的 Dock 3 参数。
2. 地图上按顺序标记巡检点 1 到 15（页面限制最多 15 个）。
3. 不勾选“闭合巡检点”；生成的任务从 Dock 3 起飞、经过巡检点，最后以
   `goHome` 返回 Dock 3。
4. 上传前必须在 DJI Pilot / FlightHub 的任务校验和真实受控环境中复核：
   WGS84 坐标、禁飞/限高、RTK/海拔基准、返航高度与障碍物净空。

## 发现新目标后的正确职责划分

KMZ/WPML 是上传前就确定的**静态任务文件**，不能在飞行中自行接收识别结果、
插入航点并在完成后恢复到原任务。因此不要试图仅通过修改 KMZ 达成“飞行中动态
改线”。应由运行时服务调用 DJI Cloud API/设备能力来编排，建议流程如下：

```text
机载视频/照片 -> 第三方识别服务 -> 目标事件
                                  -> 安全与坐标服务
                                  -> 任务编排服务（Cloud API）
                                  -> 无人机执行临时复拍任务
                                  -> 恢复原任务或从最近未完成巡检点继续
```

### 目标事件的最小数据契约

识别方只需把结果提交给任务编排服务；不应直接生成 KMZ：

```json
{
  "event_id": "uuid",
  "mission_id": "cloud-mission-id",
  "coordinate": {"longitude": 119.97, "latitude": 30.36, "altitude": 70, "crs": "WGS84"},
  "confidence": 0.92,
  "capture": {"heading_deg": 120, "gimbal_pitch_deg": -45, "hover_seconds": 5}
}
```

编排服务应执行：去重/置信度阈值、地理围栏与障碍物净空检查、剩余电量和返航裕量
检查、目标坐标精度检查、人工确认策略。通过后，暂停或取消原任务，创建一个只包含
“当前安全位置 → 目标上方复拍点 → 安全退出点”的临时任务；在目标点悬停、云台对准并
拍照。临时任务结束后，按 Cloud API 支持的恢复能力继续原任务；若不支持断点恢复，则
生成“Dock 3 → 下一个未完成巡检点 → 剩余点 → Dock 3”的新任务，且不要重复执行已完成点。

只有在 DJI 当前 Cloud API、机型固件与监管策略明确支持对应的任务暂停、替换和恢复
能力时才能自动执行；否则将目标事件展示给操作员并要求人工确认。动态改线是飞行安全
关键功能，必须保留完整审计日志和一键返航优先级。
