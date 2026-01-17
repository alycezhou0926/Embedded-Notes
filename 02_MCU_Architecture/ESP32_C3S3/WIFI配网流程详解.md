# WIFI配网流程详解

### 第一幕：准备阶段（基础设施搭建）

**场景**：单片机刚上电，还是一张白纸，它需要先把自己的基础能力（记忆、网络、听觉）准备好。



| **代码片段 (Dri_WIFI.c)**             | **对应的操作与含义**                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| **`nvs_flash_init()`**                | **准备笔记本（NVS）**                                     <br/>配网的最终目的是把 Wi-Fi 账号密码存起来。NVS 就是它的“非易失性存储区”（断电不丢）。如果初始化失败（比如满了），代码里还写了自动擦除重来。 |
| **`esp_netif_init()`**                | **安装网卡驱动**<br/>初始化底层的 TCP/IP 协议栈。没有它，就算有了密码也连不上网。 |
| **`esp_event_loop_create_default()`** | **竖起耳朵（事件循环）** <br/>ESP32 的核心机制是“事件驱动”。这行代码创建了一个系统级的“监听中心”，用来接收各种信号（比如“蓝牙连上了”、“Wi-Fi 连上了”）。 |
| **`esp_event_handler_register(...)`** | **分配接线员** <br/>你注册了很多 `event_handler`。意思就是告诉系统：“如果有 `WIFI_PROV_EVENT`（配网事件）或者 `IP_EVENT`（IP事件）发生，请把消息转给 `event_handler` 这个函数处理。” |

### 第二幕：开店迎客（启动配网管理器）

**场景**：基础设施好了，现在要决定用什么方式来接待用户（手机 App）。



| **代码片段**                                                 | **对应的操作与含义**                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| **`wifi_prov_mgr_config_t config = { .scheme = wifi_prov_scheme_ble ... }`** | **制定接待策略：<br/>蓝牙** 这里你明确指定了 `.scheme = wifi_prov_scheme_ble`。意思是：“我要用**蓝牙**通道来和手机说话，而不是 SoftAP。” |
| **`wifi_prov_mgr_init(config)`**                             | **聘请配网经理** <br/>初始化配网管理器（Provisioning Manager）。它是一个专门负责处理配网逻辑的幕后大佬。 |
| **`wifi_prov_mgr_is_provisioned(&provisioned)`**             | **查账本** <br/>这是**最关键的分岔路口**！ 它会去 NVS（笔记本）里看一眼：**“我以前存过 Wi-Fi 密码吗？”** 👉 **如果有**：跳过配网，直接去连网。 👉 **如果没有**：进入下面的“第三幕”，开始配网。 |



### 第三幕：如果没配过网，开始营业（广播与连接）

**场景**：单片机发现自己没有密码，于是它开启蓝牙，向周围广播“我在这里，快来连我”



#### 1. 准备店名和暗号

```
// 这里的 pop = "abcd1234" 就是“暗号”（Proof of Possession）。
// 手机连上蓝牙后，必须提供这个码，才能证明它是设备的主人。
const char *pop = "abcd1234"; 

// 生成设备名，例如 "PROV_123456"
get_device_service_name(service_name, sizeof(service_name)); 
```

#### 2. 开启自定义窗口（你之前的疑惑）

```
// 开设一个叫 "custom-data" 的窗口
wifi_prov_mgr_endpoint_create("custom-data"); 
// 指定在这个窗口办事的人是 custom_prov_data_handler
wifi_prov_mgr_endpoint_register("custom-data", custom_prov_data_handler, NULL);
```

> **解释**：这相当于在标准办事大厅旁边，专门开了一个“VIP 自定义服务窗口”。如果 App 想传用户 ID 或 Token，就往这里扔。

#### 3. 正式开张

```
// 启动配网服务！此时蓝牙开始广播，手机可以搜到设备了。
wifi_prov_mgr_start_provisioning(security, (const void *) sec_params, service_name, service_key);
```

#### 4. 打印传单（二维码

```
// 在串口日志里打印二维码连接。
// 这一步是为了让 App 扫码就能拿到 Bluetooth Name 和 pop，不用手输。
wifi_prov_print_qr(service_name, username, pop, PROV_TRANSPORT_BLE);
```

------

### 第四幕：客户接待（Event Handler 的全过程）

**场景**：现在 App 搜到了设备，开始交互。**所有的动作都在 `event_handler` 函数里触发**。这是整个流程最动态的部分。

请看着你的 `static void event_handler(...)` 函数：

**第一步：握手 (BLE Event)**

1. 手机 App 连上蓝牙。
2. 触发 `PROTOCOMM_TRANSPORT_BLE_CONNECTED`。
3. **日志**：`ESP_LOGI(TAG, "BLE transport: Connected!");`
4. 此时物理连接建立。

**第二步：安全认证 (Security Session Event)**

1. App 发送加密握手包（包含对 `pop` "abcd1234" 的验证）。
2. 如果密码对，触发 `PROTOCOMM_SECURITY_SESSION_SETUP_OK`。
3. **日志**：`ESP_LOGI(TAG, "Secured session established!");`
4. 此时，一条加密通道建立完成。

**第三步：交换 Wi-Fi 密码 (Provisioning Event)**

1. **App 发送**：App 将家里的 Wi-Fi SSID 和 Password 发送给设备。
2. **设备接收**：触发 `WIFI_PROV_CRED_RECV`。
   - 代码：`wifi_sta_cfg->ssid`, `wifi_sta_cfg->password` 被提取出来。
   - **日志**：`Received Wi-Fi credentials...`
3. **设备尝试连接**：配网管理器拿着收到的账号密码，偷偷在后台试着连一下路由器。

**第四步：验证结果 (Provisioning Event)**

- **情况 A：连上了 (Happy Path)**
  - 触发 `WIFI_PROV_CRED_SUCCESS`。
  - **日志**：`Provisioning successful`。
  - 触发 `WIFI_PROV_END` -> 调用 `wifi_prov_mgr_deinit()`（蓝牙任务完成，关闭蓝牙，释放内存）。
- **情况 B：密码错误/连不上 (Sad Path)**
  - 触发 `WIFI_PROV_CRED_FAIL`。
  - **日志**：`Provisioning failed!`。
  - 设备会告诉 App：“你给的密码不对，请重试。”



### 第五幕：正式上岗（连接 Wi-Fi）

**场景**：配网成功结束，或者一开始查账本时发现已经配过网。

```
static void wifi_init_sta(void)
{
    // 设置为 Station 模式（连接路由器的模式）
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));
    // 启动 Wi-Fi
    ESP_ERROR_CHECK(esp_wifi_start());
}
```

当 `esp_wifi_start()` 运行后，因为 NVS 里已经存好了密码，ESP32 就会自动去连接路由器。

**最终回响：**

1. 路由器分配 IP 给 ESP32。

2. 触发 `IP_EVENT_STA_GOT_IP` 事件。

3. 进入 `event_handler` 的 IP 分支：

   ```
   // 打印 IP 地址
   ESP_LOGI(TAG, "Connected with IP Address:" ...);
   // 调用你注册的回调函数，通知主程序：网连好了，开始干活吧！
   if(wifiCallback) { wifiCallback(); }
   ```

------

### 总结流程图

1. **初始化** (NVS, Netif, Loop)
2. **检查** (配过网了吗？)
   - **Yes** -> 这里的代码直接跳转步骤 5。
   - **No** -> 继续步骤 3。
3. **启动蓝牙配网** (Start Provisioning) -> **手机 App 介入**
4. **交互** (Handler): 蓝牙连上 -> 校验 PoP -> 接收 SSID/密码 -> 试连接。
5. **关闭蓝牙** (Deinit Manager)。
6. **正式连接 Wi-Fi** (Start Station)。
7. **获取 IP** -> 触发 `wifiCallback` -> **成功！**





![ESP32wifi配网系统流程图](assets/ESP32wifi配网系统流程图.png)