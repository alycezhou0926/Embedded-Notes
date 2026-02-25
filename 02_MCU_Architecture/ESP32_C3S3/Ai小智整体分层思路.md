### 整体分层思路（先给结论）

整体架构是典型“三层”结构：

- 最上层 App/（业务应用层）：负责流程编排、状态机和业务逻辑（配网 → 激活 → 语音交互 → IoT 控制 & 显示）。

- 中间层 Dri/（协议/驱动封装层）：对 ESP-IDF 的 WiFi 配网、WebSocket、HTTP Client 做“功能级封装”，屏蔽具体协议细节。

- 底层 Inf/（设备 & 能力抽象层）：对具体硬件和算法能力做“接口级抽象”，包括 LCD、按键、LED 灯条、音频编解码、语音前端、IoT 描述等。

调用方向是单向自上而下：App → Dri / Inf，Dri 再去调 ESP-IDF 和第三方库，Inf 再去直接操作硬件或算法库。

上层之间通过事件标志组 + 环形缓冲区 + 回调松耦合协作。

------

### 各目录职责与主要模块

#### 1. main.c（应用入口）

void app_main(void)

{

  App_Application_Start();

}

- 职责：作为 ESP-IDF 的入口，只做一件事：启动顶层应用调度 App_Application_Start。

- 意义：所有系统初始化和业务流程都集中到 App_Application，入口很干净。

------

#### 2. App/ 目录 —— 应用/业务层

核心文件：

- App_Application.c

- App_Display.c

- App_Audio.c

- App_Communication.c

- App_OTA.c

##### 2.1 App_Application：全局应用编排中心

职责：系统主流程和状态机调度。

主要流程（App_Application_Start）：

1. 全局事件组创建：global_event = xEventGroupCreate();

1. 显示初始化与提示

- 调 App_Display_Init() 做 LVGL + LCD 初始化。

- App_Display_SetTitleText("配网中..."); 显示当前状态。

1. WiFi 配网与二维码

- 在 Dri_WIFI 中注册两个回调：

- Dri_WIFI_RegisterConnectedCallback(App_Application_WifiCallback);

- Dri_WIFI_RegisterShowQrCodeCallback(App_Application_ShowQRCode);

- 调 Dri_WIFI_Init() 开始 BLE 配网，底层生成二维码，回调到 App_Display_ShowQRCode 在屏幕上画二维码。

1. 按键输入

- 调 Inf_Key_Init() 初始化 ADC 按键。

- 注册按键回调：

- KEY1 单击：App_Application_KeyCallback，触发唤醒、服务器连接和 Hello。

- KEY2 长按松开：擦除配网信息 + esp_restart()。

1. 等待 WiFi 联网成功

- 等待事件位 WIFI_CONNECTED。

1. OTA 激活流程

- 标题改为“启动中…”，删除二维码。

- 调 App_OTA_Init() 初始化 HTTP 客户端。

- 调 App_OTA_Activity() 周期性向云端汇报固件信息，等待是否已激活：

- 成功：设置 WebSocket url/token，置 ACTIVATION_SUCCESS 后退出循环。

- 未激活：从云返回 activation_code，在屏上提示“请拿着激活码[...]到官网先激活”。

1. 音频子系统 + 通信子系统 + LED

- 创建语音数据的多级环形缓冲区：es8311_to_sr_buffer、sr_to_encoder_buff、encoder_to_ws_buff、ws_to_decoder_buff。

- 调 App_Audio_Init(App_Application_Wakup, App_Application_VadChange)：

- 把 “唤醒回调” 和 “VAD 状态变化回调” 交给音频层。

- 调 App_Communication_Init()：初始化 WebSocket 和上传音频任务。

- 调 Inf_Led_Init()：准备好 IoT 灯控制。

与其他模块关系：

- UI 显示：调用 App_Display_*，只关心“要显示什么”，不关心 LVGL / LCD 细节。

- 网络 & 云端交互：通过 Dri_WIFI / Dri_HttpClient / Dri_Websocket 间接访问。

- 音频与唤醒：App 只实现唤醒和 VAD 事件上的业务逻辑，音频底层由 App_Audio + Inf_* 处理。

- IoT 控制：自己不直接控制 IO，只通过 Inf_Led/Inf_ES8311 等在 App_Communication 中被调用。

##### 2.2 App_Display：UI 层（基于 LVGL）

职责：封装 LVGL UI 结构，暴露简单的显示接口：

- App_Display_Init()

- 先调 Inf_Lcd_Init() 初始化物理 LCD（SPI + panel + 背光）。

- 再内部 App_Display_LvglInit() 用 lvgl_port_init + lvgl_port_add_disp() 把 LCD 注册给 LVGL。

- 最后 App_Display_CreateCompent() 创建：

- 顶部标题标签 title

- 中间内容标签 contentLabel

- 中间 emoji 标签 emojiLable

- 提供简单接口：

- App_Display_SetTitleText(char *datas)

- App_Display_SetContentText(char *datas)

- App_Display_SetEmojiText(char *emotion)：根据 emotion 字符串匹配到 emoji 表。

- App_Display_ShowQRCode(void* datas, size_t len) / App_Display_DeleteQRCode()：在 LVGL 上添加/删除二维码对象。

调用关系：

- 被 App_Application、App_Communication、App_OTA 调用，用于显示当前状态、内容和情绪。

- 调用 Inf_Lcd 完成真正的硬件初始化。

##### 2.3 App_Audio：音频处理流水线（录音 → 唤醒/VAD → 编码 → 上传 & 解码播放）

职责：用多个 Task + 环形缓冲区，把音频采集、唤醒检测、VAD、编码、解码/播放 串成有序流水线。

初始化 App_Audio_Init：

1. 调用 Inf_ES8311_Init()/Open()：初始化 I2C + I2S + codec，打开音频设备。

1. 调 Inf_SR_Init()：初始化语音前端（AFE + 唤醒 + VAD）。

1. 调 Inf_Encoder_Init() / Inf_Decoder_Init()：打开 Opus 编解码器。

1. 创建 5 个任务：

- App_Audio_ReadEs8311ToBufferTaskFunc

- 从 Inf_ES8311_Read() 取原始 PCM → 写入 es8311_to_sr_buffer。

- App_Audio_ReadBufferToSRTaskFunc

- 从 es8311_to_sr_buffer 取数据 → Inf_SR_Feed() 喂给语音模型。

- App_Audio_SRToBufferTaskFunc

- 从 Inf_SR_Fetch() 取结果，处理：

- 检测唤醒词 WAKENET_DETECTED → 置 is_wakup 并回调上层 wkCallback()（也就是 App_Application_Wakup）。

- 检测 VAD 状态变化 → 回调 vadStateCallback（App_Application_VadChange），驱动监听开始/结束。

- 在唤醒且处于“说话”时，将 res->vad_cache + res->data 放入 sr_to_encoder_buff。

- App_Audio_BufferToEncoderTaskFunc

- 从 sr_to_encoder_buff 取满一帧 PCM → Inf_Encoder_Process() 编为 Opus → 写入 encoder_to_ws_buff。

- App_Audio_BufferToDecoderTaskFunc

- 从 ws_to_decoder_buff 取来自服务器的 Opus 数据 → Inf_Decoder_Process() 解成 PCM → Inf_ES8311_Write() 播放到扬声器。

特点：

- App_Audio 自己不关心服务器、显示等，只负责“麦克风 ←→ 语音前端/编解码 ←→ 扬声器”流水线，并向上层发事件（唤醒 / VAD）。

##### 2.4 App_Communication：与云端的“会话层”

职责：管理 WebSocket 连接、控制 hello / listen / abort / TTS / LLM / IoT 指令等并与环形缓冲区对接。

主要功能：

- App_Communication_Init()：

- 调 Dri_Websocket_Init(websocket_url) 初始化 client。

- 注册回调 Dri_Websocket_RegisterCallback(App_Communication_WebsocketReceiveHandle, App_Communication_WebsocketFinishFunc);

- 创建两个 Task：

- App_Communication_UploadAudioTaskFunc：从 encoder_to_ws_buff 取编码音频，在状态为 LISTENING 时通过 Dri_Websocket_Send 发到服务器。

- App_Communication_ReconnectTaskFunc：监听 WEBSOCKET_CONNECT_ERROR/SUCCESS，重连成功后重新 SendHello。

- 连接和握手：

- App_Communication_ConnectServer()：

- 第一次连接时通过 Dri_Websocket_AppendHeader 设置 Authorization / Protocol-Version / Device-Id / Client-Id 请求头。

- 调 Dri_Websocket_Start() 并等待 WEBSOCKET_CONNECT_SUCCESS。

- App_Communication_SendHello()：构造 type=hello 的 JSON，声明音频格式等，并等待服务器 WEBSOCKET_HELLO_RESPONSE。

- 会话控制：

- App_Communication_SendWakup()：发送 listen detect 的唤醒词文本 “你好,小智”。

- App_Communication_StartLISTENING() / StopLISTENING()：发送 listen start/stop 控制服务器开始/结束听用户。

- App_Communication_Abort()：发送 type=abort 让服务器放弃说话（在唤醒后再次触发等情形）。

- WebSocket 收包处理 App_Communication_WebsocketReceiveHandle()：

- type == "hello"：拿到 session_id，设置事件标志，并把 Inf_IOT_GetDescriptors() / Inf_IOT_GetStatus() 的文本发给云端，让服务器知道本地 IoT 能力和当前状态。

- type == "tts"：TTS 状态：

- state=start：设置 communicationStatus = SPEAKING，标题显示“说话中…”。

- state=stop：置为 IDLE，标题显示“聆听中…”。

- state=sentence_start：读取 text 字段，调用 App_Display_SetContentText。

- type == "iot"：解析 IoT 命令，调用 Inf_ES8311_SetMute / Inf_Led_Open/Close / Inf_ES8311_SetVolume。

- type == "llm"：读出情绪 emotion，调用 App_Display_SetEmojiText。

- 非文本（二进制）数据：直接写入 ws_to_decoder_buff，供解码播放。

与 App_Application 的关系：

- App_Application_Wakup 在不同通信状态下调用 ConnectServer / SendHello / Abort / SendWakup。

- App_Application_VadChange 驱动 StartLISTENING / StopLISTENING。

- 音频上传任务与 App_Audio 共用缓冲区。

##### 2.5 App_OTA：激活/OTA 前置配置

职责：用 HTTP Client 向服务器上报当前固件和设备信息，拿回 WebSocket url 和 token 及可选的 activation_code。

关键点：

- 初始化：

- App_OTA_Init()：Dri_HttpClient_Init(SERVER_HTTP_URL, HTTP_METHOD_POST)，并注册 _App_OTA_HttpReceiveHandle。

- 响应处理：

- App_OTA_HttpReceiveHandle()：

- 解析 JSON，读取 websocket.url / websocket.token → 赋值给全局 websocket_url / token。

- 如果无 activation 字段 → 激活成功，设置 ACTIVATION_SUCCESS。

- 有 activation → 取出 code 保存到 activation_code，设置 ACTIVATION_FAIL。

- 请求构建：

- App_OTA_GetClientId()：用 NVS 存储/读取 client_id，不存在则生成 UUID v4。

- App_OTA_SetHeader()：设置 Content-Type / User-Agent / Device-Id / Client-Id。

- App_OTA_SetBodyAndRequest()：构造 JSON：

- application.version + elf_sha256

- board（类型、名称、WiFi 信息、MAC 等）

- 通过 Dri_HttpClient_SetBody + Dri_HttpClient_Request 发送。

- App_OTA_Activity()：循环：

- 设置头 + 构建请求 + 等待事件：HTTP_REQUEST_ERROR / ACTIVATION_DATA_ERROR / ACTIVATION_SUCCESS / ACTIVATION_FAIL。

- 成功激活则 break，失败时在屏幕上提示激活码，5 秒后重试。

------

#### 3. Dri/ 目录 —— 协议/系统驱动封装层

核心文件：

- Dri_WIFI.c：WiFi & 配网（BLE + QR）

- Dri_Websocket.c：WebSocket 客户端封装

- Dri_HttpClient.c：HTTP 客户端封装

##### 3.1 Dri_WIFI：网络配网 & 联网封装

职责：

- 初始化 NVS、TCP/IP、ESP 事件 loop。

- 启动 WiFi station。

- 通过 wifi_prov_mgr（BLE + QR）实现智能配网。

- 事件回调中：

- Got IP 时调用上层注册的 wifiCallback()（即 App_Application_WifiCallback），并在里面设置事件位 WIFI_CONNECTED。

- 生成配网用的 QR payload，调用上层的 qrcodeCallback(payload, len)（即 App_Application_ShowQRCode），最后由 App_Display_ShowQRCode 在屏上绘制。

对上层暴露的接口：

- Dri_WIFI_Init()

- Dri_WIFI_ResetProvisioning()

- Dri_WIFI_RegisterConnectedCallback()

- Dri_WIFI_RegisterShowQrCodeCallback()

##### 3.2 Dri_Websocket：WebSocket 客户端封装

职责：对 esp_websocket_client 做简单包装，暴露“连接/发送/注册回调”的统一接口。

- Dri_Websocket_Init(char *url)：创建 client，配置 SSL/证书、buffer、超时等，注册 websocket_event_handler。

- Dri_Websocket_Start() / Stop()。

- Dri_Websocket_IsConnected()。

- Dri_Websocket_Send(datas, len, type)：根据 type 选 send_text 或 send_bin。

- Dri_Websocket_AppendHeader(key, value)：在第一次连接前追加 header。

- Dri_Websocket_RegisterCallback(receive, finish)：把上层 App_Communication 的处理函数保存下来，事件回调中转发：

- WEBSOCKET_EVENT_CONNECTED → 置 WEBSOCKET_CONNECT_SUCCESS

- WEBSOCKET_EVENT_ERROR → 置 WEBSOCKET_CONNECT_ERROR

- WEBSOCKET_EVENT_DATA → 根据 op_code 判断文本/二进制调用上层 receiveHandleCallback

- WEBSOCKET_EVENT_FINISH → 调 finishCallback()（上层重置状态等）

##### 3.3 Dri_HttpClient：HTTP(S) Client 封装

职责：封装 ESP HTTP Client 的事件逻辑，聚合响应的数据并通知上层。

- Dri_HttpClient_Init(url, method)：

- 配置 crt_bundle_attach、method、transport_type=HTTP_TRANSPORT_OVER_SSL。

- HTTP 事件处理 _http_event_handler：

- HTTP_EVENT_ON_HEADER 时读 Content-Length，在 PSRAM 申请一块 buffer。

- HTTP_EVENT_ON_DATA：把分段数据 append 到 buffer。

- HTTP_EVENT_ON_FINISH：调用上层 receivecb(buffer, len)，清理 buffer。

- HTTP_EVENT_ERROR：释放 buffer，设置 HTTP_REQUEST_ERROR 事件位。

- 对上层开放接口：

- Dri_HttpClient_Request()

- Dri_HttpClient_SetHeader(key, value)

- Dri_HttpClient_SetBody(datas, len)

- Dri_HttpClient_RegisterHandleDataCallback(cb)

------

#### 4. Inf/ 目录 —— 硬件/能力抽象层

主要模块：

- 显示相关：Inf_Lcd

- 输入输出：Inf_Key、Inf_Led

- 音频相关：Inf_ES8311、Inf_SR、Inf_Encoder、Inf_Decoder

- IoT 描述：Inf_IOT

##### 4.1 Inf_Lcd：LCD + SPI + 背光

- 初始化 SPI bus、LCD Panel（以 st7789 为例），设置分辨率/颜色格式。

- 控制 LCD 电源、背光打开。

- 提供 io_handle / panel_handle 全局句柄，供 App_Display 在 lvgl_port_add_disp 中使用。

##### 4.2 Inf_Key：ADC 多键输入抽象

- 使用 iot_button_new_adc_device 通过 ADC 通道 + 电压区间来区分 Key1、Key2。

- 提供：

- Inf_Key_Init()

- Inf_Key_RegisterKey1Callback(event, cb, data)

- Inf_Key_RegisterKey2Callback(event, cb, data)

##### 4.3 Inf_Led：灯带抽象

- 基于 led_strip（WS2812）封装：

- Inf_Led_Init()

- Inf_Led_Open()：所有灯设为高亮白色。

- Inf_Led_Close()：清空。

##### 4.4 Inf_ES8311：音频 Codec + I2C + I2S

- 初始化 I2C / I2S / codec，将具体 GPIO、采样率、通道数、增益等硬编码配置。

- 提供：

- Inf_ES8311_Init() / Open() / Close()

- Inf_ES8311_Read(datas, len)：麦克风录音。

- Inf_ES8311_Write(datas, len)：扬声器播放。

- Inf_ES8311_SetVolume(int volume)

- Inf_ES8311_SetMute(bool is_mute)：静音/恢复。

##### 4.5 Inf_SR：语音前端 + 唤醒/VAD

- 基于 esp_afe_sr：

- Inf_SR_Init()：加载模型、配置 AFE（关闭 AEC/SE/NS，开启 VAD & WakeNet），分配内存（PSRAM）。

- Inf_SR_GetChunkSize() / Inf_SR_GetChannelNum()：告诉上层每次要喂多少 PCM。

- Inf_SR_Feed(datas)：喂一帧。

- Inf_SR_Fetch(res)：取出 AFE 结果（包括唤醒状态、VAD、缓存音频数据等）。

##### 4.6 Inf_Encoder / Inf_Decoder：Opus 编解码抽象

- Inf_Encoder：

- Inf_Encoder_Init()：配置采样率/通道/比特率/帧长等，打开 Opus 编码器。

- Inf_Encoder_GetSize(&pcm_size, &raw_size)：告诉上层编码一次需要多少 PCM、输出 buffer 要多大。

- Inf_Encoder_Process(in_frame, out_frame)：对 PCM 帧做编码。

- Inf_Decoder：

- Inf_Decoder_Init()：配置采样率等，打开 Opus 解码器。

- Inf_Decoder_Process(in_raw, out_frame)：对 Opus 数据解码为 PCM。

##### 4.7 Inf_IOT：IoT 能力描述

- 直接把 descriptors.txt / iot_status.txt 编译进程序，提供：

- Inf_IOT_GetDescriptors()

- Inf_IOT_GetStatus()

- 在 App_Communication 的 hello 互通成功后，发送给云端，后续云通过 type="iot" 消息来控制本地设备。

------

### 关键调用链 & 分层逻辑总结

#### 1. 启动 & 初始化主链路

1. app_main → App_Application_Start。

1. App_Application_Start 内部依次：

- 创建事件组。

- 调 App_Display_Init → Inf_Lcd_Init + LVGL 初始化 → 创建 UI 控件。

- 注册 Dri_WIFI 回调并 Dri_WIFI_Init → BLE 配网 + QR → 通过回调在 LCD 上显示二维码。

- 初始化按键 Inf_Key_Init 并注册应用层 key 回调。

- 等待 WIFI_CONNECTED 事件位（来自 Dri_WIFI 的 IP 事件）。

- 通过 App_OTA_Init / App_OTA_Activity 完成激活/获取 WebSocket URL & Token。

- 创建音频环形缓冲区，再 App_Audio_Init 启动音频采集 + 唤醒/VAD + 编/解码任务。

- App_Communication_Init 启动 WebSocket 客户端与上传/重连任务。

- 初始化 LED Inf_Led_Init。

#### 2. 语音交互链路（用户 → 麦克风 → 云 → 扬声器）

- 上行（用户说话 → 云）：

- 麦克风数据：Inf_ES8311_Read → App_Audio_ReadEs8311ToBufferTaskFunc → es8311_to_sr_buffer。

- 语音前端：App_Audio_ReadBufferToSRTaskFunc → Inf_SR_Feed。

- 唤醒 & VAD：App_Audio_SRToBufferTaskFunc → 检测 WAKENET_DETECTED / VAD 状态变化，通过回调通知 App_Application 去连服务器、进入 LISTENING 状态。

- 语音数据：App_Audio_SRToBufferTaskFunc 将有效语音写入 sr_to_encoder_buff。

- 编码：App_Audio_BufferToEncoderTaskFunc 从 sr_to_encoder_buff → Inf_Encoder_Process → encoder_to_ws_buff。

- 发送：App_Communication_UploadAudioTaskFunc 在 communicationStatus == LISTENING 时从 encoder_to_ws_buff→ Dri_Websocket_Send 发给云。

- 下行（云 → 扬声器 + UI）：

- 文本/控制类：

- WebSocket 文本帧 → Dri_Websocket 事件 → App_Communication_WebsocketReceiveHandle。

- tts 消息：驱动 communicationStatus、标题“说话中/聆听中”、句子文本显示。

- iot 消息：调用 Inf_Led / Inf_ES8311_SetVolume/SetMute 等控制本地设备。

- llm 消息：设置 emoji。

- 音频流：

- WebSocket 二进制帧 → 写到 ws_to_decoder_buff。

- App_Audio_BufferToDecoderTaskFunc：从缓冲取数据 → Inf_Decoder_Process → PCM → Inf_ES8311_Write 播放。

#### 3. 分层逻辑一句话概括

- App 层：只关心“要做什么业务流程”和“当前是哪个状态”，通过回调和事件与底层协作（配网、激活、唤醒、说话/聆听、IoT 控制、UI 显示）。

- Dri 层：封装具体“协议级”功能（WiFi 配网、HTTP、WebSocket），对上暴露简化接口 + 回调，对下使用 ESP-IDF 原生 API。

- Inf 层：封装“硬件和算法能力”，将 LCD、按键、LED、音频 codec、AFE、Opus、IoT 文本描述这些底层细节屏蔽起来，提供统一接口给 App 或 Dri 使用。

------

### 架构思路总结（可作为你后续扩展的指导）

- 分层清晰：App（业务） → Dri（协议） → Inf（设备/能力），上层通过函数/回调调用下层，下层通过事件标志和回调向上通知，基本遵守单向依赖。

- 解耦方式：

- 事件标志组：跨模块同步（WiFi 连接状态、HTTP 激活结果、WebSocket 连接与 hello 响应等）。

- 回调函数：按键、WiFi 成功、WebSocket 收包等方向的解耦。

- 环形缓冲区：音频流水线任务间的数据解耦。

- 扩展建议（按当前架构）：

- 新增业务功能，优先在 App_* 新建模块，通过现有 Dri/Inf 能力拼接逻辑。

- 新增硬件或底层能力（例如新传感器、更多灯效）放在 Inf_*，App 只通过抽象接口使用。

- 如需新增协议（MQTT、HTTP 其他服务等），与 Dri_Websocket/Dri_HttpClient 并列增加 Dri_* 模块，由 App_Communication 或新的 App_* 使用。





### 整体模块调用图（分层视角）

整体结构：

    ┌──────────────────────────────┐
    │          app_main            │
    └─────────────┬────────────────┘
                  │
                  ▼
    ┌──────────────────────────────┐
    │     App_Application(Start)   │
    └─────────────┬────────────────┘
      ┌───────────┼─────────────────────────┐
      ▼           ▼                         ▼
    ┌────────────────┐    ┌─────────────────────┐    ┌──────────────────┐
    │  App_Display   │    │     App_OTA         │    │   App_Audio      │
    │(UI 状态/二维码)  │    │(激活+获取 WS 信息)    │    │     (音频流水线)   │
    └───────┬────────┘    └──────────┬──────────┘    └─────────┬────────┘
            │                        │                        │
            ▼                        ▼                        ▼
    ┌──────────────┐       ┌────────────────┐       ┌──────────────────────┐
    │  Inf_Lcd     │       │ Dri_HttpClient │       │ Inf_ES8311 / Inf_SR  │
    │(LCD+LVGL底座)│        │   (HTTP 封装)  │        │ Inf_Encoder/Decoder  │
    └──────────────┘       └────────────────┘       └──────────────────────┘

---

### App_Application 其它关键调用

#### 按键 & LED

    App_Application
         │
         ├── Inf_Key_Init
         │      └── 注册回调 → App_Application_KeyCallback
         │
         └── Inf_Led_Init  （IoT 控灯能力）

#### WiFi 配网 & 二维码

    App_Application
         │
         ├── Dri_WIFI_RegisterConnectedCallback(App_Application_WifiCallback)
         │
         ├── Dri_WIFI_RegisterShowQrCodeCallback(App_Application_ShowQRCode)
         │           │
         │           └── App_Display_ShowQRCode → LVGL 二维码
         │
         └── Dri_WIFI_Init
                 ├── ESP-IDF WiFi / 事件循环 / 配网
                 └── Got IP → 调用 wifiCallback → App_Application_WifiCallback
                                        └── 设置 WIFI_CONNECTED 事件位

#### 通信 & 云端会话

    App_Application
         │
         └── App_Communication_Init
                 ├── Dri_Websocket_Init(websocket_url)
                 ├── Dri_Websocket_RegisterCallback(
                 │        App_Communication_WebsocketReceiveHandle,
                 │        App_Communication_WebsocketFinishFunc)
                 └── 创建任务:
                          - 上传音频: App_Communication_UploadAudioTaskFunc
                          - 重连任务: App_Communication_ReconnectTaskFunc
    
    App_Application_Wakup / App_Application_VadChange
         │
         └── 调用 App_Communication_ConnectServer / SendHello /
                    SendWakup / StartLISTENING / StopLISTENING / Abort

---

### Dri_Websocket 事件流向

    Dri_Websocket (内部 websocket_event_handler)
         │
         ├── WEBSOCKET_EVENT_CONNECTED → 设置 WEBSOCKET_CONNECT_SUCCESS
         ├── WEBSOCKET_EVENT_ERROR     → 设置 WEBSOCKET_CONNECT_ERROR
         ├── WEBSOCKET_EVENT_DATA
         │       └── 调 App_Communication_WebsocketReceiveHandle
         └── WEBSOCKET_EVENT_FINISH
                 └── 调 App_Communication_WebsocketFinishFunc

---

### App_Communication 收包分发

    App_Communication_WebsocketReceiveHandle
         │
         ├── type=="hello"
         │       ├── 保存 session_id
         │       ├── xEventGroupSetBits(WEBSOCKET_HELLO_RESPONSE)
         │       └── 调 Inf_IOT_GetDescriptors / GetStatus → Dri_Websocket_Send
         │
         ├── type=="tts"
         │       ├── start/stop → 更新 communicationStatus
         │       └── 调 App_Display_SetTitleText / App_Display_SetContentText
         │
         ├── type=="iot"
         │       └── 调 Inf_ES8311_SetMute / SetVolume / Inf_Led_Open / Inf_Led_Close
         │
         └── type=="llm"
                 └── 调 App_Display_SetEmojiText

---

### 音频全链路（用户 → 麦克风 → 云 → 扬声器）

#### 上行链路：用户说话 → 云端

用户语音 → 麦克风采集 → SR（唤醒+VAD）→ 编码 → WebSocket 发送。

    用户语音
       ↓
    Inf_ES8311_Read
       ▲
       │ (读麦克风数据)
    App_Audio_ReadEs8311ToBufferTaskFunc
       │
       ▼
    es8311_to_sr_buffer
       │
       ▼
    App_Audio_ReadBufferToSRTaskFunc
       │
       └── Inf_SR_Feed (喂给语音前端/唤醒模型)
    
    App_Audio_SRToBufferTaskFunc
       │
       ├─ 检测唤醒 WAKENET_DETECTED
       │      └─ wkCallback() → App_Application_Wakup
       │
       ├─ 检测 VAD 状态变化
       │      └─ vadStateCallback() → App_Application_VadChange
       │
       └─ 在唤醒且为“说话状态”时，把语音放入 sr_to_encoder_buff
    
    sr_to_encoder_buff
       │
       ▼
    App_Audio_BufferToEncoderTaskFunc
       │
       ├─ Inf_Encoder_GetSize / Inf_Encoder_Process
       └─ 编码好的 opus → encoder_to_ws_buff
    
    encoder_to_ws_buff
       │
       ▼
    App_Communication_UploadAudioTaskFunc
       │ （仅在 communicationStatus == LISTENING 时发送）
       ▼
    Dri_Websocket_Send (WebSocket 二进制帧发给服务器)


#### 下行链路：云端 → 扬声器 + UI

文本 / 控制类数据：

    WebSocket 文本帧
      ↓
    Dri_Websocket → App_Communication_WebsocketReceiveHandle

- tts：控制“小智说话/停止”，并更新标题、内容文字。
- iot：调用 Inf_Led_Open/Close、Inf_ES8311_SetVolume/SetMute 控制本地设备。
- llm：根据情绪字段设置 emoji（App_Display_SetEmojiText）。

音频数据：

    WebSocket 二进制帧
      ↓
    App_Communication_WebsocketReceiveHandle
      ↓
    ws_to_decoder_buff
      ↓
    App_Audio_BufferToDecoderTaskFunc
         │
         ├─ Inf_Decoder_Process (Opus → PCM)
         └─ Inf_ES8311_Write (PCM 播放到扬声器)