# 音频转写功能集成文档

## 概述

document_parser微服务已集成iFlytek ASR（自动语音识别）功能，支持将音频文件转写为文本。该功能支持异步转写，适合处理长音频文件。

## 支持的音频格式

- `.wav` - WAV音频文件
- `.mp3` - MP3音频文件
- `.pcm` - PCM音频文件
- `.aac` - AAC音频文件
- `.opus` - Opus音频文件
- `.flac` - FLAC音频文件
- `.ogg` - OGG音频文件

## 文件大小限制

- 最大文件大小：50MB
- iFlytek API支持最长5小时的音频转写

## 环境配置

### 必需的环境变量

在`env.dev`、`env.staging`、`env.prod`中配置：

```ini
# iFlytek (科大讯飞) 语音转写配置
XFYUN_APP_ID=your_xfyun_app_id
XFYUN_SECRET_KEY=your_xfyun_secret_key
```

这些环境变量会自动传递给`document-parser`容器。

## API使用方法

### 1. 异步转写（推荐）

上传音频文件，立即返回转写任务ID，然后轮询获取结果。

#### 上传音频文件

```bash
curl -X POST "http://localhost:8800/transcribe_audio" \
  -F "file=@audio.wav" \
  -F "wait_for_completion=false"
```

**响应示例：**

```json
{
  "success": true,
  "message": "音频转写任务已创建",
  "data": {
    "type": "audio",
    "text": "",
    "filename": "audio.wav",
    "file_size": 3136940,
    "text_length": 0,
    "orderId": "DKHJQ202512241648447034ABFl5ouLu58roKk",
    "transcription_status": "pending",
    "message": "Use /get_transcription_status endpoint to check progress."
  }
}
```

#### 查询转写状态

```bash
curl "http://localhost:8800/get_transcription_status?order_id=DKHJQ202512241648447034ABFl5ouLu58roKk"
```

**响应示例（进行中）：**

```json
{
  "success": true,
  "data": {
    "orderId": "DKHJQ202512241648447034ABFl5ouLu58roKk",
    "status": 3,
    "status_text": "in_progress",
    "text": "",
    "text_length": 0,
    "transcription_status": "pending"
  }
}
```

**响应示例（已完成）：**

```json
{
  "success": true,
  "data": {
    "orderId": "DKHJQ202512241648447034ABFl5ouLu58roKk",
    "status": 4,
    "status_text": "completed",
    "text": "转写后的文本内容...",
    "text_length": 486,
    "transcription_status": "completed"
  }
}
```

### 2. 同步转写（等待完成）

上传音频文件并等待转写完成（适合短音频）。

```bash
curl -X POST "http://localhost:8800/transcribe_audio" \
  -F "file=@audio.wav" \
  -F "wait_for_completion=true" \
  -F "max_attempts=60" \
  -F "interval=10"
```

**响应示例：**

```json
{
  "success": true,
  "message": "音频转写完成",
  "data": {
    "type": "audio",
    "text": "转写后的文本内容...",
    "filename": "audio.wav",
    "file_size": 3136940,
    "text_length": 486,
    "orderId": "DKHJQ202512241648447034ABFl5ouLu58roKk",
    "transcription_status": "completed"
  }
}
```

### 3. 从Redis缓存转写

如果文件已上传到Redis缓存，可以使用缓存ID进行转写：

```bash
curl -X POST "http://localhost:8800/transcribe_audio" \
  -F "cache_id=your-cache-id" \
  -F "filename=audio.wav" \
  -F "wait_for_completion=false"
```

### 4. 通过extract_text_with_redis端点（自动检测）

`/extract_text_with_redis`端点会自动检测音频格式，如果是音频文件会自动调用转写功能：

```bash
curl -X POST "http://localhost:8800/extract_text_with_redis" \
  -F "cache_id=your-cache-id" \
  -F "filename=audio.wav"
```

对于音频文件，该端点会：
1. 检测到音频格式
2. 自动创建转写任务
3. 返回转写任务ID（异步模式）或转写文本（如果后端实现了轮询）

## 转写状态说明

iFlytek API返回的状态码：

- `2` - 排队中（queued）
- `3` - 处理中（in_progress）
- `4` - 已完成（completed）

## 集成到Disk Scan功能

音频文件已集成到Disk Scan功能中：

1. **前端支持**：Disk Scan页面已添加音频格式到允许的文件类型列表
2. **后端处理**：文件上传后，`DocumentParserAdapter`会自动检测音频格式并调用转写功能
3. **异步轮询**：对于音频文件，后端会自动轮询转写结果，直到完成或超时

### 使用流程

1. 在Disk Scan页面上传音频文件
2. 文件上传到backend并缓存到Redis
3. backend调用`document-parser`服务的`/extract_text_with_redis`端点
4. `document-parser`检测到音频格式，创建转写任务
5. backend轮询转写状态，直到完成
6. 转写文本保存到文件记录中

## 错误处理

### 常见错误

1. **凭证未配置**
   - 错误：`iFlytek credentials not configured`
   - 解决：检查`XFYUN_APP_ID`和`XFYUN_SECRET_KEY`环境变量

2. **文件格式不支持**
   - 错误：`不支持的音频格式: .xxx`
   - 解决：使用支持的音频格式（.wav, .mp3, .pcm, .aac, .opus, .flac, .ogg）

3. **文件过大**
   - 错误：`音频文件太大: xxx bytes (最大: 52428800 bytes)`
   - 解决：文件大小不能超过50MB

4. **转写超时**
   - 错误：`Maximum polling attempts reached`
   - 解决：增加`max_attempts`参数或检查音频文件是否过长

5. **iFlytek服务时长不足**
   - 错误：`预处理服务时长不足`
   - 解决：在iFlytek开放平台充值

## 性能优化建议

1. **异步处理**：对于长音频文件，使用异步模式避免HTTP请求超时
2. **合理轮询**：根据音频长度调整轮询间隔，短音频可以设置较短的间隔
3. **结果缓存**：转写结果可以缓存，避免重复转写相同文件
4. **并发控制**：限制同时进行的转写任务数量，避免超出iFlytek API限制

## 测试

使用提供的测试脚本测试音频转写功能：

```bash
# 设置环境变量
export XFYUN_AUDIO_FILE=/path/to/audio.wav
export DOCUMENT_PARSER_HTTP_URL=http://localhost:8800

# 运行测试
python document_parser/test_audio_transcription.py
```

## 注意事项

1. **服务时长限制**：iFlytek API有服务时长限制，需要监控账户余额
2. **转写时间**：长音频转写可能需要较长时间（最长5小时），需要合理设置超时时间
3. **网络访问**：确保`document-parser`容器可以访问外部网络（调用iFlytek API）
4. **文件大小**：前端和backend都进行了50MB的文件大小验证
5. **异步处理**：音频文件转写是异步的，需要实现轮询机制获取结果

## 相关文档

- [iFlytek ASR API文档](https://www.xfyun.cn/doc/asr/ifasr_new/API.html)
- [转写结果格式说明](./xfyun-asr-result-format.md)
- [测试脚本使用指南](./xfyun-asr-test-guide.md)


