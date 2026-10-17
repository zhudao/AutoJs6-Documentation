# 语音 (Speech)

**`6.8.0`**

`speech` 与 `$speech` 是同一 Rhino 全局对象, 由兼容且已启用的 3-Macaw Speech 插件提供离线语音识别, 语音活动检测, 唤醒词和神经网络合成. 相关模型存放在插件私有目录. 宿主不提供缺少插件时的替代识别引擎.

本页对应当前宿主的 `Speech`, `SpeechModels`, `SpeechSession`, `SpeechWakeSession` 与协议 V1. Node.js 和 Bun 尚无对应的 `speech` 模块或全局对象, 不应根据 Rhino 的补全提示推断可用性.

麦克风由插件采集, 需要插件获得麦克风权限. 文件路径相对脚本工作目录解析, 也可使用绝对路径; 不接受 `content://`, `file://`, `http://` 等 URI. 错误由 `SpeechError` 表示.

## [m] isAvailable

### speech.isAvailable()

- <ins>**returns**</ins> { [boolean](dataTypes#boolean) } - 当前是否有兼容且已启用的插件应答.

该探测不抛出插件错误. 返回 `true` 不表示所有模型都已安装或麦克风权限已授予.

## [m] capabilities

### speech.capabilities()

- <ins>**returns**</ins> { [object](dataTypes#object) } - 当前插件能力快照.

返回字段包括 `contractVersion`, `features`, `catalogVersion`, `engineName`, `engineVersion`, `ttsEnginePackage`, `microphoneGranted`, `packageName`, `versionName`, `versionCode`. 可选插件信息可以为 `null`. `features` 可能包含 `asr`, `asrStreaming`, `vad`, `kws`, `tts`, `ttsSystemEngine`, `models`, 应根据实际返回值判断能力.

## [m] listen

### speech.listen(options?)

- **[ options ]** { [object](dataTypes#object) } - 听写选项, 见下表.
- <ins>**returns**</ins> { SpeechSession } - 持续到 `end` 事件的会话.

| 选项 | 类型 | 默认值与含义 |
| --- | --- | --- |
| `model` | string | 省略时使用相应默认模型 |
| `language` | string | 可选语言标签 |
| `source` | `'mic' \| 'external'` | `'mic'`, 麦克风或外部 PCM |
| `sampleRate` | number | 16000, 会话只接受此采样率 |
| `vad` | boolean | `true`, 使用语音活动检测 |
| `silenceEndMillis` | number | 1200, 范围 1..60000 |
| `maxDurationMillis` | number | 300000, 正整数且不超过 1800000 |
| `timeoutMillis` | number | 15000, 范围 0..1800000 |
| `hotwords` | string[] | 空数组, 最多 200 项 |
| `punctuation` | boolean | `true`, 启用标点处理 |

```js
let session = speech.listen({ language: 'zh', maxDurationMillis: 10000 });
session.on('final', event => console.log(event.text));
session.on('error', error => console.error(error.code, error.message));
session.on('end', event => console.log(event.reason, session.result()));
```

## [m] recognize

### speech.recognize(source, options?)

**`Non-UI`**

- **source** { [string](dataTypes#string) | ArrayBuffer | ArrayBufferView | Java byte[] } - WAV 路径或 PCM 数据.
- **[ options ]** { [object](dataTypes#object) } - `model`, `language`, `hotwords`, `punctuation`, `vad`, `sampleRate`.
- <ins>**returns**</ins> { [object](dataTypes#object) } - 识别结果.

PCM 采用单声道 PCM16 LE, `sampleRate` 默认 16000, 可为 8000..192000. `punctuation` 与 `vad` 默认 `true`. WAV 文件读取其文件头中的采样率. 不接受普通 JavaScript 数字数组代替 Java `byte[]`; 可传入 `Int16Array` 或 `files.readBytes` 得到的字节数组.

结果包含 `text`, `segments`, `durationMillis`, `modelId`, `warnings`. 每个语音段含 `startMillis`, `endMillis`, `text`, 并可能带 `language` 与 `tags` (`emotion`, `event`). `warnings` 为 `{ code, message }` 数组.

## [m] recognizeAsync

### speech.recognizeAsync(source, options?)

**`Async`**

参数与 `recognize` 相同, 返回解析为相同结果对象的 Promise. 在 UI 线程上应使用此形式, 错误通过 Promise 拒绝传递.

## [m] detectVoice

### speech.detectVoice(source, options?)

**`Non-UI`**

- **source** { [string](dataTypes#string) | ArrayBuffer | ArrayBufferView | Java byte[] } - 与 `recognize` 相同的音频来源.
- **[ options ]** { [object](dataTypes#object) } - `model`, `sampleRate`, `threshold`, `minSpeechMillis`, `minSilenceMillis`.
- <ins>**returns**</ins> { [object](dataTypes#object) } - `{ segments, durationMillis }`, 每段含 `startMillis`, `endMillis`.

`threshold` 为 0..1, 两个最小时间为 0..1800000 毫秒. 省略时由所选模型使用默认值. 该方法同步执行, 当前没有 `detectVoiceAsync` 入口.

## [m] wake

### speech.wake(keywords, options?)

- **keywords** { [string](dataTypes#string) | [object](dataTypes#object) | [Array](dataTypes#array) } - 一个关键词, 或由字符串及 `{ keyword, boost?, threshold? }` 组成的数组.
- **[ options ]** { [object](dataTypes#object) } - `model`, `minIntervalMillis`, `thenListen`.
- <ins>**returns**</ins> { SpeechWakeSession } - 唤醒会话.

关键词为 1..50 个且不可重复. `boost` 范围 -100..100, `threshold` 范围 0..1. `minIntervalMillis` 默认 1500, 范围 0..600000.

`thenListen` 可以为布尔值或听写选项. 为 `true` 时使用默认听写设置; 使用对象时只能选择麦克风来源, 不能设为 `'external'`. 每次 `wake` 事件的 `listen` 字段携带该次命中后的听写会话. 停止唤醒会话也会结束正在进行的听写轮次.

## [m] synthesize

### speech.synthesize(text, path, options?)

**`Non-UI`**

- **text** { [string](dataTypes#string) } - 非空文本, 最多 8000 字符.
- **path** { [string](dataTypes#string) } - 输出 WAV 文件路径.
- **[ options ]** { [object](dataTypes#object) } - `model`, `speaker`, `speed`, `format`.
- <ins>**returns**</ins> { [object](dataTypes#object) } - `{ path, sampleRate, durationMillis, modelId, speakerId, sizeBytes, warnings }`.

`speaker` 可以是说话人 ID 或名称; `speed` 范围 0.5..2, `format` 仅支持 `'wav'`. 插件负责合成, 宿主发布输出文件. `speakerId` 是数值 ID.

## [m] synthesizeAsync

### speech.synthesizeAsync(text, path, options?)

**`Async`**

参数与 `synthesize` 相同, 返回解析为相同文件结果对象的 Promise, 可在 UI 线程使用.

## [m] speak

### speech.speak(text, options?)

**`Async`**

- **text** { [string](dataTypes#string) } - 要播放的文本.
- **[ options ]** { [object](dataTypes#object) } - `model`, `speaker`, `speed` 及允许的 [TTS 选项](tts).
- <ins>**returns**</ins> { [Promise](https://developer.mozilla.org/docs/Web/JavaScript/Reference/Global_Objects/Promise) } - 与 `tts()` 相同的 [TtsResult](tts#ttsresult) 播放结果.

该调用使用 3-Macaw Speech 的系统 TTS 引擎. 不接受 `engine`, `voice`, `rate`, `speechRate`; 用 `model` / `speaker` 选择声音, 用 `speed` 选择语速. `speaker` 需要同时指定 `model`. 缺少插件时 Promise 以 `PLUGIN_UNAVAILABLE` 拒绝.

## [p+] models

`speech.models` 管理插件的模型目录和已安装模型. 模型类型为 `'asr'`, `'tts'`, `'kws'`, `'vad'`. 模型对象包含 `id`, `kind`, `modelType`, `name`, `languages`, `sizeBytes`, `installed`, `isDefault`, `recommended`, `version`, `source`, `license`; 已安装模型可带 `installedBytes`, TTS 模型可带 `speakers` 和 `sampleRate`.

### models.list(filter?)

返回模型对象数组. 可按 `kind`, `language`, `installed` 过滤.

### models.install(id, options?)

**`Non-UI`**

下载并安装目录中的模型, 返回模型对象. 选项包括 `acceptLicense` (默认 `false`) 和 `onProgress(done, total)`. 非商用模型需要明确接受许可; 此选项不改变模型许可本身. 进度回调在脚本线程执行.

### models.installAsync(id, options?)

**`Async`**

`install` 的 Promise 形式, 可在 UI 线程使用.

### models.importFile(path, options?)

**`Non-UI`**

导入脚本可读取的本地模型归档, 返回模型对象. 选项包括 `id`, `kind`, `modelType`, `name`, `languages`, `acceptLicense`, `onProgress`. 不支持 URI 路径.

### models.remove(id)

删除已安装模型, 返回是否实际删除的布尔值.

### models.setDefault(kind, id)

设置该类型的默认模型, 返回模型对象.

### models.getDefault(kind)

返回已安装的默认模型对象, 没有时返回 `null`.

### models.refreshCatalog()

**`Non-UI`**

刷新签名模型目录, 返回 `{ version, updated }`. 当前没有异步别名, 不应在 UI 线程调用.

## SpeechSession

该对象继承 EventEmitter. `on` / `once` 注册监听并返回会话本身. 属性包括 `id`, `modelId`, `startedAt`, `source`, `round`, `state`, `reason`, `isEnded`. `id` 和 `modelId` 在 `ready` 前为 `null`; `reason` 在结束前为 `null`; `state` 依次为 `'pending'`, `'active'`, `'ended'`.

- `isActive()`: 尚未结束时返回 `true`.
- `result()`: 返回目前已确定的文本; 会话结束后返回整段结果.
- `stop()`: 冲刷最终结果并结束, 返回会话, 可重复调用.
- `cancel()`: 不产生最终结果地结束, 返回会话, 可重复调用.
- `feed(pcm)`: 仅适用于 `'external'` 来源, 输入 16 kHz 单声道 PCM16 LE, 每次最多 1 MiB, 返回会话.

事件包括 `ready({ sessionId, modelId })`, `partial` / `final` (带 `text`), `endpoint` (无参数), `vad({ active })`, `warning({ code, message })`, `error(SpeechError)`, `end({ reason, text })`. `end` 的原因包括 `'stopped'`, `'timeout'`, `'maxDuration'`, `'cancelled'`, `'error'`.

## SpeechWakeSession

该对象继承 EventEmitter, 属性包括 `keywords`, `id`, `modelId`, `startedAt`, `state`, `reason`, `isEnded`. 提供 `stop()` 与 `isActive()`, 不提供听写会话的 `feed`, `cancel`, `result`.

事件包括 `ready`, `wake({ keyword, score, timestampMillis, listen? })`, `warning`, `error`, `end({ reason })`. 引擎未提供命中分数时 `score` 可以为 `null`. 设置 `thenListen` 后, 在 `wake` 回调中向 `event.listen` 注册该轮的听写事件.

## [p] SpeechError

### new speech.SpeechError(message?, code?)

构造继承 Error 的脚本错误对象, 也可不使用 `new`. `message` 和 `code` 转换为字符串, 缺省分别为空字符串和 `'INTERNAL'`. 可通过 `error instanceof speech.SpeechError` 与 `error.code` 判断错误.

常见错误码包括 `PLUGIN_UNAVAILABLE`, `PLUGIN_DISABLED`, `PLUGIN_INCOMPATIBLE`, `PERMISSION_DENIED`, `MODEL_MISSING`, `MODEL_DOWNLOAD_FAILED`, `MODEL_CHECKSUM_MISMATCH`, `MODEL_LICENSE_NOT_ACCEPTED`, `INVALID_OPTIONS`, `UNSUPPORTED_FORMAT`, `LIMIT_EXCEEDED`, `SESSION_BUSY`, `SESSION_CLOSED`, `SESSION_CANCELLED`, `ENGINE_FAILED`, `IO_FAILED`, `TIMEOUT`, `INTERNAL`.
