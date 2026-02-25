### 第一步：修改 ESP32 代码 (发送清晰的 PCM)

我们需要改 3 个文件（都在 VS Code 里），别担心，都很简单。

#### 1. 修改 `App_Communication.c` (告诉服务器我要发 PCM 了)

找到 `App_Communication_SendHello` 函数（约 121 行），把 `opus` 改成 `pcm`。

C

```
// 原来的: cJSON_AddStringToObject(audio_params,"format","opus");
// 👇 修改后:
cJSON_AddStringToObject(audio_params,"format","pcm");
```

#### 2. 修改 `App_Application.c` (加大缓冲区，防止卡顿)

找到 `App_Application_CreateRingBuffer` 函数（约 98 行），把 `encoder_to_ws_buff` 的大小从 16k 加大到 64k。

C

```
// 原来的: encoder_to_ws_buff = xRingbufferCreateWithCaps( 16 * 1024 , ...
// 👇 修改后 (加大缓存):
encoder_to_ws_buff =  xRingbufferCreateWithCaps( 64 * 1024 , RINGBUF_TYPE_NOSPLIT, MALLOC_CAP_SPIRAM);
```

#### 3. 修改 `App_Audio.c` (最关键：跳过压缩，直接发数据)

找到 `App_Audio_BufferToEncoderTaskFunc` 函数（约 188 行）。 **我们要把里面的 `while(1)` 循环整个替换掉**，改成“直通模式”。

**请用下面这段代码替换掉原来的 `while(1)` 循环：**

C

```
    // 👇👇👇 用这段新的循环替换原来的 while(1) 👇👇👇
    while (1)
    {
        // 1. 从缓冲区取出 PCM 数据
        App_Audio_CopyRingBuffer(sr_to_encoder_buff, pcm_data, pcm_size);

        // 2. 【关键修改】不再调用 Inf_Encoder_Process，而是直接把 PCM 数据发出去
        // 注意：这里直接发送 pcm_data，跳过了编码步骤
        xRingbufferSend(encoder_to_ws_buff, pcm_data, pcm_size, 0);

        // 稍微延时，给 CPU 喘口气
        vTaskDelay(1);
    }
    // 👆👆👆 替换结束 👆👆👆
```

*(修改完这三个文件后，记得**重新编译并烧录** ESP32)*

![4323b89a-a376-4342-bd8e-64f2c35c9de2](C:\Users\Administrator\Desktop\截图日志\4323b89a-a376-4342-bd8e-64f2c35c9de2.png)

1. **堵车了**：你的 Python 服务器在执行“**听 (ASR)**”和“**想 (LLM)**”的时候，需要花费 1~3 秒的时间去请求云端 API。
2. **被挂电话**：在这个时间段里，Python 脚本是**阻塞**的（忙着等云端回复，没空搭理 ESP32 的心跳包）。
3. **耐心耗尽**：ESP32 的默认设置是“如果服务器 3 秒不理我，我就挂断”。

所以，每次你说完话，AI 正在思考的时候，ESP32 就以为网断了，自己断开了连接。

------

### ✅ 解决方法：给 ESP32 多一点耐心

我们需要修改 ESP32 的代码，告诉它：“服务器反应慢，多等一会儿，别急着挂电话。”

请在 VS Code 中修改 **`Dri_Websocket.c`** 文件（和你之前改 TCP 的是同一个地方）：

#### 🛠️ 修改步骤

1. 打开 `Dri_Websocket.c`。
2. 找到 `Dri_Websocket_Init` 函数里的 `websocket_cfg` 配置结构体。
3. 把 **`network_timeout_ms`** 从默认的 `3000` 改成 **`15000`** (15秒)。

**代码修改对照：**

C

```
    esp_websocket_client_config_t websocket_cfg = {
        .uri = url,
        .disable_auto_reconnect = false,
        .buffer_size = 3 * 1024,
        .transport = WEBSOCKET_TRANSPORT_OVER_TCP, // 之前改过的 TCP
        // .crt_bundle_attach = esp_crt_bundle_attach, 
        .reconnect_timeout_ms = 3000,
        
        // 👇👇👇 关键修改：把 3000 改成 15000 (给云端API留出15秒的处理时间) 👇👇👇
        .network_timeout_ms = 15000, 
    };
```