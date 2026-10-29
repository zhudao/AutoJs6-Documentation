# SpeechRecognitionResult

SpeechRecognitionResult 是 [speech.recognize](speech#m-recognize) 的返回值与 [speech.recognizeAsync](speech#m-recognizeasync) 的兑现值, 为普通 JavaScript 对象, 描述一段音频的识别结果. 麦克风或外部 PCM 会话的结果不使用此对象, 而是以 [SpeechSession](speechSessionType) 的 `final` 事件逐句到达.

```js
let result = speech.recognize('./meeting.wav');
console.log(result.modelId, result.durationMillis, result.text);
result.segments.forEach((s) => {
    console.log(`[${s.startMillis} - ${s.endMillis}]`, s.language || '', s.text);
});
result.warnings.forEach(w => console.warn(w.code, w.message));
```

---

<p style="font: bold 2em sans-serif; color: #FF7043">SpeechRecognitionResult</p>

---

## [@] SpeechRecognitionResult

**`6.8.0`**

- {{
    - text: [string](dataTypes#string);
    - segments: {{
        - startMillis: [number](dataTypes#number);
        - endMillis: [number](dataTypes#number);
        - text: [string](dataTypes#string);
        - language?: [string](dataTypes#string);
        - tags?: {{ emotion?: [string](dataTypes#string); event?: [string](dataTypes#string) }};
    - }}[];
    - language?: [string](dataTypes#string);
    - durationMillis: [number](dataTypes#number);
    - modelId: [string](dataTypes#string);
    - warnings: {{ code: [string](dataTypes#string); message: [string](dataTypes#string) }}[];
- }}

## [p] text

**`6.8.0`**

- { [string](dataTypes#string) }

整段音频的识别文字. `punctuation` 为 `true` (默认) 时保留模型输出的标点.

## [p] segments

**`6.8.0`**

- {{ startMillis: [number](dataTypes#number); endMillis: [number](dataTypes#number); text: [string](dataTypes#string); language?: [string](dataTypes#string); tags?: [object](dataTypes#object) }}[]

按时间排列的语音段. `vad` 为 `true` (默认) 时先按语音活动检测切段再逐段识别.

- **startMillis** / **endMillis** - 语音段在音频中的起止时间, 单位为毫秒
- **text** - 该段的文字
- **language** - 识别出的语种, 如 `'zh'`, `'en'`; 只有能识别语种的模型 (SenseVoice) 提供
- **tags** - SenseVoice 的附加标签: `emotion` (情感) 与 `event` (声音事件), 只在模型给出时存在

结果对象过大 (超过 256 KiB) 时, 插件省略各段的文字以保持在协议上限内 (此时各段 `text` 为空字符串, 整段文字仍在 `text` 中), 并在 `warnings` 中加入 `LIMIT_EXCEEDED`.

## [p] language

**`6.8.0`**

- { [string](dataTypes#string) | [undefined](dataTypes#undefined) }

整段音频的语种, 只在模型给出时存在.

## [p] durationMillis

**`6.8.0`**

- { [number](dataTypes#number) }

音频总时长, 单位为毫秒.

## [p] modelId

**`6.8.0`**

- { [string](dataTypes#string) }

实际使用的识别模型 ID.

## [p] warnings

**`6.8.0`**

- {{ code: [string](dataTypes#string); message: [string](dataTypes#string) }}[]

非致命问题, 没有时为空数组. 常见情形:

- `INVALID_OPTIONS` - 模型不支持热词而忽略了 `hotwords`, 或 SenseVoice 不认识 `language` 而改为自动识别.
- `LIMIT_EXCEEDED` - 结果过大, 省略了各段文字.
