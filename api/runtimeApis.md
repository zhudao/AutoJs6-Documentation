# 运行时 API (Runtime APIs)

<!-- type=misc -->

AutoJs6 的 Rhino, Node.js Runtime 和 Bun Runtime 使用独立的执行环境. 本文说明宿主 API 的导入方式, 异步返回值及编辑器声明边界. 安装运行时插件不会自动把全部 Rhino 全局对象注入 Node.js 或 Bun.

本文对应包含运行时 API 补齐的配套宿主与插件源码. 已安装的旧版本可能没有下列模块或能力; 文档更新不表示这些版本已经公开发布.

## 执行方式

| 环境 | 宿主 API 入口 | 返回值与限制 |
| --- | --- | --- |
| Rhino | 全局模块, 如 `pinyin`, `console`, `cX` | 按各模块页的同步或异步签名使用 |
| Node.js 主线程 | `require('pinyin')`, `require('screen_metrics')` 等公开模块, 或对应 ESM 导入 | 宿主调用通常返回 Promise; 不自动注入 Rhino 全局 |
| Node.js Worker | `opencc` / `autojs6:opencc`, `image` / `images`, `pinyin`, `pinyin4j`, `console`, `screen_metrics`, `media_projection`, `screen_recorder`, `recorder` 及其 `autojs6:` 入口; 其他能力可经 `autojs6:bridge` 转发 | 支持 CJS, ESM 和嵌套 Worker; 异步操作复用父线程封装, 仍受本次执行授权约束 |
| 原生终端 `node` / `npm` | 原生 Node.js 入口 | 不加载 AutoJs6 脚本执行器的模块与兼容全局 |
| Bun | 从 `process.env.AUTOJS6_HOST_CLIENT_MODULE` 导入客户端, 再调用 `createHostClient()` | 显式客户端, 异步调用, 按当前执行协商能力 |

独立脚本可以在首条字符串语句中使用 `"nodejs";` 或 `"bun";`. Node 也支持 `node` 标记. Node 的 `.node.js`, `.cjs`, `.mjs`, `.mts`, `.cts` 后缀和 Bun 的 `.bun.js`, `.bun.ts`, `.bun.tsx` 后缀用于选择对应环境. 普通 `.ts` / `.tsx` 默认仍按 Rhino 的 TypeScript 路径处理; 项目的 `type` / `projectType` 为 `node` 时可选择 Node. 当前 `type: 'bun'` 元数据本身不选择 Bun 引擎, 应使用 Bun 标记或后缀. 标记不能与固定引擎后缀冲突. 作为 Bun 入口的相对依赖导入的 `.mjs` 文件由 Bun 解释, 将该文件单独运行则按其后缀选择 Node.

## Node.js 拼音

**`Async`**

```js
"nodejs";

let pinyin = require('pinyin');
let pinyin4j = require('pinyin4j');

(async () => {
    console.log(await pinyin.simple('北京上海'));
    console.log(await pinyin.convert('重庆', { heteronym: true }));
    console.log(await pinyin4j.readings('重庆', { system: 'HANYU' }));
})();
```

公开模块调用宿主选择的外置 Pinyin / Pinyin4j 插件, 不再使用有限字典模拟完整转换. 未安装或不可用时, Promise 拒绝并保留桥接错误, 不把原文返回当作成功回退.

| 模块成员 | 返回值 |
| --- | --- |
| `pinyin(value, options?)`, `pinyin.convert(value, options?)` | `Promise<PinyinResult>` |
| `pinyin.simple(value, enableNumericTone?, enableSegment?)` | `Promise<string>` |
| `pinyin.compare(left, right)` | `Promise<number>` |
| `pinyin.compact(rows)` | `Promise<string[][]>` |
| `pinyin.fromCodePoint(codePoint)` | `Promise<string \| null>` |
| `pinyin.fromPhrase(phrase)` | `Promise<string[][]>` |
| `pinyin4j(value, options?)`, `pinyin4j.of(value, options?)`, `pinyin4j.as(value)` | `Promise<string>` |
| `pinyin4j.readings(value, options?)` | `Promise<string[][]>` |

`PinyinResult` 是二维字符串数组, 附有返回 `Promise<string[][]>` 的 `compact()` 方法. 需要先 `await` 转换, 再 `await result.compact()`. 风格和模式常量仍可同步读取; Node 的常量是数值, 不是 Rhino 的 Java 枚举对象.

原先按同步小字典结果编写的 Node 脚本需要增加 `await`. `Array#sort` 不接受异步比较函数, 因此不能直接把新的 `pinyin.compare` 传入 `sort`.

## Node.js 控制台

**`Async`**

`console.log`, `info`, `warn`, `error` 等日志方法保持原有输出方式. 下列窗口和配置方法实际调用宿主, 返回 `Promise<void>`, 应按顺序等待:

```js
"nodejs";

(async () => {
    await console.show();
    await console.setTitle('Node.js Runtime');
    await console.setSize(600, 400);
    await console.setPosition(20, 80);
    console.log('窗口已就绪');
})();
```

- 窗口: `show`, `hide`, `reset`, `expand`, `collapse`, `launch`, `clear`.
- 尺寸与位置: `setSize`, `setPosition`.
- 标题: `setTitle`, `setTitleTextSize`, `setTitleTextColor`, `setTitleBackgroundColor`, `setTitleBackgroundTint`, `setTitleBackgroundAlpha`, `setTitleIconsTint`.
- 内容: `setContentTextSize`, `setContentTextColor`, `setContentTextColors`, `setContentBackgroundColor`, `setContentBackgroundTint`, `setContentBackgroundAlpha`.
- 通用外观: `setTextSize`, `setTextColor`, `setBackgroundColor`, `setBackgroundTint`, `setBackgroundAlpha`.
- 行为与日志配置: `setExitOnClose`, `setTouchable`, `setGlobalLogConfig`, `resetGlobalLogConfig`.

需要悬浮权限的操作仍依赖宿主权限. 权限不足或参数无效时 Promise 拒绝. 不能再按旧空操作实现链式调用 `console.show().setTitle(...)`; 应分别 `await`. `build`, `input`, `rawInput` 等未实现入口不因窗口控制接通而变为可用.

## Node.js 坐标换算

**`Async`**

`screen_metrics` 模块提供与 Rhino 相同的坐标参数和基数规则, 每次调用读取当前屏幕尺寸, 返回 `Promise<number>`. 不应使用启动时缓存的 `device.width` / `height` 自行替代实时方向信息.

```js
"nodejs";

let screen = require('screen_metrics');

(async () => {
    console.log(await screen.getInfo());
    console.log(await screen.cX(360, 720));
    console.log(await screen.cY(0.5, true));
    console.log(await screen.cXy(0.5, '16:9', true));
    console.log(await screen.cYx(0.5, '16:9', true));
})();
```

| 方法 | 返回值 |
| --- | --- |
| `cX(value?, baseOrMode?, ratioMode?)` | `Promise<number>` |
| `cY(value?, baseOrMode?, ratioMode?)` | `Promise<number>` |
| `cXy(value, baseOrMode?, ratioMode?)` | `Promise<number>` |
| `cYx(value, baseOrMode?, ratioMode?)` | `Promise<number>` |
| `getInfo()` | `Promise<{ width, height, rotation, baseX, baseY }>` |
| `getScaleBases()` | `Promise<{ x, y }>` |
| `getScaleBaseX()`, `getScaleBaseY()` | `Promise<number>` |
| `setScaleBases(x, y)`, `setScaleBaseX(base)`, `setScaleBaseY(base)` | `Promise<void>` |

`cX()` / `cY()` 返回当前宽度 / 高度. `ratioMode` 为 `true`, `false` 或 `'auto'`, 第三个参数表示比例模式. 交叉换算接受 `width:height` 比例字符串; 数值模式, 默认基数, 有限数值与合法基数规则参见 [全局坐标换算](global#m-cx). 每次执行的每个方向只能设置一次自定义基数; `setScaleBases` 必须两个方向都成功才应用.

`require('autojs6:compat')` 也提供上述换算和基数成员, 返回值同样为 Promise. 只有显式安装兼容全局后才能省略该对象; 兼容安装不改变异步语义.

## OpenCC 转换

**`Async`**

Node.js 的 `autojs6:opencc` 提供固定的宿主模块入口. 无前缀的 `opencc` 可被已安装的同名 npm 包遮蔽; 需要本节契约时应使用带前缀的入口:

```js
"nodejs";

let opencc = require('autojs6:opencc');
(async () => {
    console.log(await opencc.s2twi('程序员使用鼠标'));
    console.log(await opencc.convert('程序员使用鼠标', 'S2TWI'));
})();
```

模块自身可调用, `opencc(text, type)` 与 `opencc.convert(text, type)` 等价. `convert` 和以下 32 个转换方法均返回 `Promise<string>`, 不再抛出旧版占位实现的固定不支持错误:

| 分类 | 方法 |
| --- | --- |
| 基础转换 | `hk2s`, `hk2t`, `jp2t`, `s2hk`, `s2t`, `s2tw`, `s2twp`, `t2hk`, `t2s`, `t2tw`, `t2jp`, `tw2s`, `tw2t`, `tw2sp` |
| 惯用词别名 | `s2twi`, `twi2s` |
| 组合转换 | `s2jp`, `t2twi`, `hk2tw`, `hk2twi`, `hk2jp`, `tw2hk`, `tw2twi`, `tw2jp`, `twi2t`, `twi2hk`, `twi2tw`, `twi2jp`, `jp2s`, `jp2hk`, `jp2tw`, `jp2twi` |

转换类型不区分大小写, 使用与 [Rhino OpenCC](opencc) 相同的转换规则和组合链. 文本可以为空字符串, 但 `null` / `undefined` 不作为空字符串处理. 缺少插件, 没有授权或参数不合法时 Promise 拒绝, 不静默返回原文.

Bun 客户端对应 `host.opencc`, 同样可调用且提供 `convert` 和全部 32 个方法. 需要本次执行具有 `text.opencc` 能力, 该能力默认启用, 仍要求可用的 OpenCC 插件. Node/Bun 都不自动注入 Rhino 的 `opencc` 全局对象.

## 图像句柄与处理

Node.js 通过 `require('images')` 或 `require('image')` 使用图像接口; Bun 使用显式客户端的 `host.images`. 图像由宿主持有, 脚本拿到带 `id`, `width`, `height` 的普通对象, 不能当作 Rhino `ImageWrapper` 或 Android `Bitmap` 使用.

每个句柄只属于创建它的执行. `copy`, 缩放, 旋转和滤镜操作返回新句柄, 不替换原图. 不再使用时调用并等待 `recycle(handle)`, 执行结束也会清理该执行持有的图像. 不应复用已回收句柄或把它传给另一次执行.

两端共同提供以下接口. 除 `getSize` 同步读取句柄尺寸外, 表中方法均返回 Promise:

| 方法 | 结果与主要参数 |
| --- | --- |
| `fromBase64(text)`, `fromBytes(bytes)`, `readImage(path)` | 图像句柄; 输入为编码后的图像, Base64 可带 data URL 前缀 |
| `toBase64(image, options?)`, `toBytes(image, options?)` | Base64 字符串或字节; `format` 为 `png`, `jpg`, `jpeg`, `webp`, `quality` 为 0-100 |
| `saveImage(image, path, options?)` | 保存文件, 返回 `void` |
| `pixel(image, x, y)`, `readPixels(image, options?)` | ARGB32 数值或像素数组与尺寸; `readPixels` 支持 `region` |
| `getSize(image)` | 同步返回 `{ width, height }` |
| `copy(image)`, `clip(image, x, y, width, height)` | 独立图像句柄 |
| `resize(image, width, height)`, `scale(image, fx, fyOrOptions?, options?)` | 新尺寸的图像; `scale` 支持 OpenCV `interpolation` 名称或数值 |
| `rotate(image, degrees, options?)` | 旋转并扩展边界; 选项为 `centerX`, `centerY`, `filter` |
| `flip(image, direction)` | `horizontal`, `vertical`, `both`, `none`; 也接受两个布尔参数指定水平和垂直翻转 |
| `grayscale(image)`, `threshold(image, options?)`, `invert(image)` | 灰度, 阈值化, 反色; 反色保留 alpha |
| `blur(image, size)`, `gaussianBlur(image, size, options?)`, `medianBlur(image, size)` | 滤波; 普通和高斯核可为数值或 `[width, height]`, 中值核为正奇数 |
| `findImage(image, template, options?)`, `matchTemplate(image, template, options?)` | 匹配位置或带分数的结果数组 |
| `findColor(image, color, options?)`, `findMultiColors(image, color, points, options?)` | 首个匹配点或 `null` |
| `detectsColor(image, color, x, y, options?)`, `detectColor(...)` | 布尔结果, 支持 `threshold` 与 `algorithm` |
| `findAllPointsForColor(image, color, options?)`, `findPointsByColor(...)` | 匹配点数组, 支持 `threshold` 与 `region` |
| `recycle(image)` | 回收句柄, 返回 `void` |

`readPixels` 返回 `{ data, width, height }`, `data` 是按行排列的有符号 ARGB32 数值数组. 单个像素与编码字节不是同一种数据: 像素整数描述颜色, PNG/JPEG/WebP 字节描述编码后的文件.

Bun 的 `readPixels` 每次最多返回 16,384 个像素, 可使用 `region: [x, y, width, height]` 缩小区域; Node 的公开 `readPixels` 经二进制通道读取, 不受该 JSON 像素上限影响. 两端的 `findAllPointsForColor` 最多返回 4,096 个点, 超限时拒绝而不是截断. 颜色搜索返回图像自身的像素坐标, 不应用屏幕换算基数. `detectsColor` 默认使用 `diff` 算法, 还支持 `rgb`, `rgb+`, `hs`, `h`, `equal`, `oklab`; `findColor` 和 `findAllPointsForColor` 则按 RGB 各通道阈值匹配不透明像素.

编解码, 像素读取, 复制, 裁剪, 旋转, 翻转及 `detectsColor` 不依赖 OpenCV. 缩放, 滤镜, 模板和颜色搜索需要可用的 OpenCV 插件. 两端也提供 `read` / `save`, `findColorEquals`, `findPointByColor` / `findPointByColors`, `getWidth` / `getHeight` 和 `isRecycled` 等便捷入口. `isRecycled` 只跟踪当前线程客户端的回收操作, 不跟踪其他线程或绕过封装的原始桥接调用.

Node 的 `toBytes` 保留原有 PFD 字节传输和 Buffer 生命周期, `format: 'rgba'` 表示紧密排列的直通 alpha sRGB RGBA8, 与 PNG 等编码格式不同. Bun 的 `fromBytes` / `toBytes` 使用 Base64 JSON 通道与 `Uint8Array`, 不提供 PFD 或原始 RGBA 输出. Bun 继续受 64 KiB 请求和 256 KiB 响应上限约束, Base64 膨胀也计入限制. 大图宜使用已授权的宿主文件读取/保存路径, 不能据 Node 的字节能力推断 Bun 的上限.

Bun 将 `image.process` (默认启用), `image.files` (默认关闭) 与 `screen.capture` (默认关闭) 分开授权. 它们分别控制图像处理, 宿主文件读写和请求 Android 录屏授权. 下列高级接口也使用这些授权, 不向运行时直接暴露 Java Bitmap 或 Mat 对象.

## 高级图像 API

**`Async`**

Node.js 的 `image` / `images` 与 Bun 的 `host.images` 已提供宿主 images 注册的 81 个函数入口, 另提供 `readImage` 别名和 `ObjectFrame` 构造器. Node.js Worker 使用相同入口, 支持 CJS/ESM/嵌套执行. 这表示函数入口和下述可传输参数已覆盖, 不表示 Java 对象原型, Java 枚举及所有 Rhino 自动转换重载完全相同.

除 `getSize`, `getWidth`, `getHeight`, `isRecycled`, `buildRegion`, `__buildRegion` 和 `ObjectFrame` 构造器外, 下列宿主操作均返回 Promise. 参数中的区域和坐标使用图像自身的整数像素, 不自动应用屏幕基数或 Rhino 的比例区域重载. OpenCV 枚举参数可使用常量名字符串或数值; 颜色使用 ARGB 数值或颜色字符串.

| 操作 | 接口与返回值 |
| --- | --- |
| 初始化与载入 | `initOpenCvIfNeeded()`; `load(url)` / `loadAsync(url)` 返回图像或 null; HTTP/HTTPS/file URL 均可使用, 无法解码的响应返回 null |
| 矩阵桥接 | `imread(path)` 返回带 id/width/height/channels/depth 的 Mat 句柄; `matToImage(mat)` 创建独立图像. imread 使用 OpenCV 默认的 BGR 文件解码, matToImage 转换为 RGB 图像 |
| 颜色与滤镜 | `isGrayscale(image)`; `inRange(image, lower, upper)`; `interval(image, color, threshold)`; `adaptiveThreshold(image, options)`; `bilateralFilter(image, options)`; `cvtColor(image, code, dstCn?)` |
| 几何 | `findCircles(image, options?)` 返回 `{ x, y, radius }[]`; `concat(a, b, direction?)` 返回新图像; 方向为 left/right/start/end/top/bottom, 默认 right |
| 多点颜色 | `detectMultiColors` / `detectsMultiColors(image, color, points, x, y, options?)` 返回 boolean; `findPointsByColors(image, color, points, options?)` 返回点数组; `countPointsByColor` 返回数量; `getMeanColor` 返回平均 ARGB32 |
| 查找别名 | `findPointByColorExactly`, `findColorInRegion`, `findPointByImage`, `findImageInRegion`; 区域形式接受 x/y/width/height/threshold, 未提供的宽高延伸到图像边缘 |
| 区域辅助 | `buildRegion(image, region?)` 返回 `{ x, y, width, height }`; `__buildRegion(region?, width?, height?)` 在缺少图像尺寸时返回 null; 数组可省略后部字段 |
| 相似度 | `psnr`, `ssim`, `mssim`, `hist`, `mse`, `ncc` 返回 number; `isEqual` 返回 boolean; `getSimilarity(a, b, { metric })` 默认 mssim, 兼容 type 和 pnsr 拼写别名 |
| 压缩 | `compress(image, format?, quality?)` 返回新图像; `compressToBytes(...)` 返回独立编码字节; 默认 PNG/quality 60; PNG quality 非 100 时使用量化插件 |
| 下采样 | `downsample(source, width, height, withAlpha?)` 返回新图像, source 可为图像句柄, 路径/URL 或编码字节; withAlpha 默认 true |
| 截图辅助 | `requestScreenCaptureAsync` 是异步授权入口的别名; `getScreenCaptureOptions()` 返回当前 width/height/orientation, 无会话时返回 null |

`adaptiveThreshold` 可使用 maxValue/adaptiveMethod/type/blockSize/C 选项, 也支持对应的位置参数. 默认值为 255/MEAN_C/BINARY/3/0; 输入先转换为灰度, blockSize 必须为大于 1 的奇数. `bilateralFilter` 支持 d/sigmaColor/sigmaSpace/borderType, 默认 0/40/20/DEFAULT, 同时支持位置参数. RGBA 输入会保留原 alpha. `cvtColor` 的结果保留原生通道信息, 例如 RGBA2RGB -> RGB2HSV -> HSV2RGB -> RGB2RGBA 可以连续调用. `findCircles` 支持 region/dp/minDst/param1/param2/minRadius/maxRadius, 返回原图坐标.

相似度比较使用当前矩阵尺寸和通道, 比较前应统一尺寸及颜色空间. 部分指标可能返回 Infinity 或 NaN; 公开封装恢复这些数值, 不将其误记为 0 或 null. Mat, 特征描述符和普通图像共享已有的执行句柄上限. `recycle` 接受这些句柄, 也支持数组或多个句柄; 执行结束会统一释放.

### 特征提取与匹配

`detectAndComputeFeatures(image, options?)` 返回特征句柄, 含 id/count/method/scale/region. options 支持 `method: 'SIFT' | 'ORB'`, region, scale 和 maxFeatures. 默认 SIFT, 自动缩小大图; 显式 scale 不大于 8, maxFeatures 为非负整数. grayscale 选项用于兼容, 两种算法均在灰度强度上检测特征. 句柄不暴露 Java 对象或原生指针.

`matchFeatures(scene, object, options?)` 可以接收两个特征句柄, 或直接接收图像/路径并临时提取特征. options 支持检测选项, matcher, threshold, ransacThreshold, minInliers 和 drawMatches 文件路径. 默认比例阈值为 SIFT 0.7 / ORB 0.8, RANSAC 阈值 3, 最少内点 4. 找不到有效单应矩阵时返回 null.

非空结果为 `ObjectFrame`, 包含 topLeft/topRight/bottomLeft/bottomRight, center/centerX/centerY, width/height, angle, bounds, matches, inliers, points 和 summary 方法. 坐标已还原到场景原图, 包括检测时的区域偏移和缩放. Worker 会在本地恢复结果的方法与构造器. 可以显式使用 `new images.ObjectFrame(topLeft, topRight, bottomLeft, bottomRight, matches?, inliers?, points?)` 创建几何结果.

```js
"nodejs";
let images = require('autojs6:images');

(async () => {
    let scene = await images.read('/sdcard/Pictures/scene.png');
    let object = await images.read('/sdcard/Pictures/object.png');
    let sceneFeatures;
    let objectFeatures;
    try {
        sceneFeatures = await images.detectAndComputeFeatures(scene, { method: 'ORB', maxFeatures: 700 });
        objectFeatures = await images.detectAndComputeFeatures(object, { method: 'ORB', maxFeatures: 700 });
        let frame = await images.matchFeatures(sceneFeatures, objectFeatures, { minInliers: 6 });
        if (frame) console.log(frame.center, frame.angle, frame.inliers);
    } finally {
        await images.recycle([sceneFeatures, objectFeatures, scene, object].filter(Boolean));
    }
})();
```

### PNG 量化与文件输出

`quantize(image, options?)` 返回 `{ bytes, size, quality, quantizationError, peakWorkingMemoryBytes }`. Node 主线程与 Worker 的 bytes 是独立 Buffer, Bun 为 Uint8Array. 需要可用的 Image Quantization 插件及资源控制/取消 API. options 支持 quality/maxColors/speed/minQuality/maxQuality/ditheringLevel/posterizeBits/preserveAlpha/maxPixels/maxMemoryBytes, 延续宿主量化规则; 默认最多 256 色, speed 8, 最大质量 100, 保留 alpha, 1600 万像素和 256 MiB 额外原生工作内存上限.

`quantizeToFile(image, path, options?)` 使用文件描述符输出 PNG, 返回相同度量和 path, 不返回 bytes. 完成后再替换目标文件; 量化失败时保留原文件并清理临时输出. `compress` / `compressToBytes` 的 PNG quality 也可使用量化选项对象. JPEG/WebP 仍按编码质量参数处理.

Node 的 quantize/compressToBytes 使用 PFD, Worker 恢复 Buffer 类型. Bun 量化字节仍经过 Base64 JSON, 受既有请求/响应体积限制; 大图应使用 quantizeToFile. 质量不足, 资源预算不足, 插件不可用或 Android 拒绝文件访问时明确失败, 不返回未处理图像冒充成功.

Node 的 load 需要 image + network 执行能力. Bun 的文件读取, URL 载入, imread, 保存, quantizeToFile 和 drawMatches 均需要 image.files; quantizeToFile 还需要 image.process. 只开启 image.files 时仍可回收读取的图像/Mat. 原始桥接同样检查这些边界, 内存图像处理授权不能绕过文件授权.

## Bun 截图与录屏授权

**`Async`**

在插件中心启用 Bun 的屏幕截图能力后, 使用显式客户端:

```js
"bun";

/** @type {typeof import('autojs6:bun-host')} */
let sdk = await import(process.env.AUTOJS6_HOST_CLIENT_MODULE);
let host = await sdk.createHostClient();

await host.images.requestScreenCapture({ timeoutMs: 45000 });
try {
    let image = await host.images.captureScreen();
    try {
        console.log(host.images.getSize(image));
        console.log(await host.images.pixel(image, 0, 0));
    } finally {
        await host.images.recycle(image);
    }
} finally {
    await host.images.stopScreenCapture();
}
```

| 方法 | 返回值与行为 |
| --- | --- |
| `requestScreenCapture(options?)` | `Promise<boolean>`, Android 同意后返回 `true`; 拒绝, 超时或未授予 `screen.capture` 时拒绝 Promise |
| `captureScreen()` | `Promise<ImageHandle>`, 从本次执行已授权的屏幕会话取得一帧; 不会静默弹出授权或改用其他截图来源 |
| `stopScreenCapture()` | `Promise<boolean>`, 停止已授权会话或取消待处理授权; 没有持有请求/会话时返回 `false` |

`options` 支持 `orientation: 'auto' | 'portrait' | 'landscape'`, 同时指定的 `width` / `height` (各为 1-8192), `requireExistingPermission` 以及 `timeoutMs` (1-120000 毫秒, 默认 120000). 默认方向为自动. `requireExistingPermission: true` 在本次执行没有有效会话时直接失败, 不显示授权对话框. 同一客户端的并发申请共用一个 Promise, 已生效的会话可复用; 停止或系统撤销后再次申请会重新经过 Android 授权流程.

权限开关只允许请求系统授权, 不代替 Android 的明确同意. 关闭开关影响下一次执行. 运行结束或取消时会释放会话及待处理授权, 已产生的图像句柄仍按执行作用域清理. 视频文件使用下述 `host.screenRecorder` 接口, 另需 `screen.record` 授权.

授权等待使用短状态调用, 不延长普通宿主调用的 10 秒上限. 整体脚本超时仍可提前结束等待; Bun 默认执行限时为 60 秒, 需要更长时间时应在执行配置中调整 `bun.timeoutMillis`. 截图返回句柄, 不把整张图片放入 JSON; 随后的字节和像素读取仍遵循前述传输上限.

## Bun 视频录制与文件输出

`host.screenRecorder` 将已授权的屏幕录制为 H.264/MP4 视频, 默认无声. 需要同时启用 `screen.capture` 和 `screen.record` (均默认关闭), 并先完成 `host.images.requestScreenCapture()`. `screen.record` 允许宿主输出 MP4 文件, 不替代系统授权, 也不依赖图像文件的 `image.files` 开关.

| 方法 | 返回值 | 行为 |
| --- | --- | --- |
| `start(path, options?)` | `Promise<ScreenRecordingActive>` | 开始录制, 路径必须以 `.mp4` 结尾, 已存在的目标文件会被拒绝 |
| `getStatus()` | `Promise<ScreenRecordingStatus>` | `state` 为 idle, recording 或 failed; 活动状态包含 id, path, width, height, frameRate, bitRate, elapsedMs, audio, audioSource 和 error |
| `stop()` | `Promise<ScreenRecordingResult \| null>` | 正常收尾并输出文件; 当前未录制时返回 null |
| `cancel()` | `Promise<boolean>` | 丢弃未完成视频, 保持截图会话; 当前未录制时返回 false |

`options.width` 和 `options.height` 必须同时指定, 为 2..8192 内的偶数; 省略时使用截图会话尺寸. `frameRate` 为 1..120 的整数, 默认 30; `bitRate` 为正整数, 单位 bit/s, 默认 8000000. 尺寸, 帧率和码率还必须受到当前 Android H.264 编码器支持. 不支持的配置拒绝 Promise, 不自动降级. 音频通过 audio 选项单独启用.

`options.audio` 选择音频来源, 默认 `"none"`, 也可用 false 表示不录音:

| audio | 音频 | 额外能力授权 |
| --- | --- | --- |
| `"none"` / false | 无音轨 | 无 |
| `"microphone"` | 麦克风 | screen.record.microphone |
| `"playback"` | 允许捕获的应用播放声音 | screen.record.playback |
| `"mixed"` | 麦克风和应用播放声音, 每路 0.5 增益混合 | 上述两项 |

两个音频开关默认关闭, 修改对下一次执行生效. 所有有声模式都需要宿主的 Android `RECORD_AUDIO` 运行时权限, 包括只捕获应用播放声音的模式. 缺少权限会明确拒绝, 不偷偷降级为无声录制; 可在 Android 的 AutoJs6 应用权限设置中授权. 麦克风模式还使用带停止按钮的麦克风前台服务; Android 14+ 的使用中权限规则要求从允许的前台状态启动, 不能用插件能力开关绕过后台限制. 通知中的停止按钮会取消使用该服务的录制并丢弃未完成视频.

音轨采用 AAC-LC, 48 kHz, 单声道, 128 kbit/s. 播放捕获使用 Android AudioPlaybackCapture, 只包含同一用户配置下允许捕获的 MEDIA/GAME/UNKNOWN 用途音频. 来源应用禁止捕获时, 该来源保持静音; 系统麦克风隐私开关或音频共享规则也可能静音输入. `audio: true` 的结果字段表示文件存在音轨, 不表示一定有可听见的声音. 参阅 [Android 音视频捕获规则](https://developer.android.com/media/platform/av-capture) 与 [麦克风前台服务要求](https://developer.android.com/develop/background-work/services/fgs/service-types#microphone).

输出路径属于宿主, 相对路径按脚本工作目录解析, 仍受 Android 文件访问规则限制. 录制先写临时文件, `stop()` 成功后才生成目标文件. 结果包含 `id`, `path`, `size` (字节), `durationMs`, `width`, `height`, `mimeType: 'video/mp4'`, `videoCodec: 'h264'`, `audio` 和 `audioSource`. 有声录制还返回 `audioCodec: 'aac'`, `audioSampleRate: 48000` 和 `audioChannels: 1`; 无声时这三个字段为 null. 没有编码出有效视频帧时停止会失败, 不生成一个冒充成功的空视频.

每次执行只允许一个活动录制. 录制期间可以继续调用 `host.images.captureScreen()`, 截图和编码从同一投影帧输入取得数据, 不额外申请授权或暂停录制. 截图尺寸遵循 requestScreenCapture 的设置: 自动方向跟随当前屏幕, 固定尺寸保持指定值, 不被视频输出尺寸替换. 停止或取消录制后截图仍然可用, 也可复用同一授权开始下一段录像. 视频输出尺寸在一段录像内保持固定, 横竖屏变化由 Android 等比缩放居中显示, 可能出现留边. 需要交换宽高时应停止当前录像后以新尺寸开始下一段. `stopScreenCapture()`, 执行结束/取消和系统撤销授权会丢弃未完成文件, 因此保留视频需要显式等待 `stop()` 完成.

```js
"bun";
let { createHostClient } = await import(process.env.AUTOJS6_HOST_CLIENT_MODULE);
let host = await createHostClient();
await host.images.requestScreenCapture();
try {
    await host.screenRecorder.start(`/sdcard/Movies/autojs6-${Date.now()}.mp4`, {
        width: 720, height: 1280, frameRate: 30, bitRate: 4000000,
        audio: 'playback', // 需要额外授权和 Android 录音权限.
    });
    let frame = await host.images.captureScreen();
    try { console.log(host.images.getSize(frame)); }
    finally { await host.images.recycle(frame); }
    await Bun.sleep(3000);
    let video = await host.screenRecorder.stop();
    console.log(video.path, video.durationMs, video.size);
} finally {
    await host.screenRecorder.cancel();
    await host.images.stopScreenCapture();
}
```

目标文件名应保持唯一. 较长录制须同时配置足够的 `bun.timeoutMillis`; 编码期间不占用一条长时间宿主调用, 开始和停止仍受普通调用超时约束. 音视频按同一单调时钟写入一个 MP4, 停止时无需重新合并整段文件. frameRate 是目标帧率; 高频大尺寸截图的像素读回会占用图形处理时间.

## Node.js 录屏与音频

**`Async`**

`screen_recorder` / `autojs6:screen_recorder` 在主线程和 CJS/ESM/嵌套 Worker 中提供同一组方法. 先通过 `images.requestScreenCapture()` 或 `media_projection.requestScreenCapture()` 完成 Android 授权. `screen_capture` 是必要的执行能力; 有声录屏另需 `media` 和 `media.recording`, 并要求宿主具有 Android RECORD_AUDIO 权限. 原始 `autojs6:bridge` 调用同样执行这些检查.

| 方法 | 返回值 | 行为 |
| --- | --- | --- |
| `start(path, options?)` | `Promise<ActiveRecording>` | 启动本次执行唯一的录屏, 拒绝覆盖已有文件 |
| `stop(options?)` | `Promise<RecordingResult \| null>` | 完成并保存 MP4, 空闲时返回 null |
| `cancel(options?)` | `Promise<boolean>` | 丢弃未保存文件, 保留截图授权; 空闲时返回 false |
| `getStatus(options?)` | `Promise<Status>` | 返回 idle 或 recording/failed 状态及活动录制信息 |

`start` 的 width/height 是 2..8192 内的偶数, 默认采用授权会话尺寸; frameRate 为 1..120, 默认 30; bitRate 默认 8000000. `audio` 可为 `none`, `microphone`, `playback` 或 `mixed`; 省略或 false 表示无声, 不接受 true. 有声模式输出 AAC-LC 48 kHz 单声道, 目标 128 kbit/s. mixed 对两路音源等比例混合. playback/mixed 要求 Android 10+, 尊重源应用的播放捕获策略; 音轨存在不代表音轨内必有声音. 麦克风模式受 Android 前台服务和使用中权限约束, 并显示可停止录制的通知.

活动结果含 id/path/width/height/frameRate/bitRate/elapsedMs/audio/audioSource/error, state 为 recording 或 failed. `stop` 的非空结果含 id/path/size/durationMs/width/height/mimeType/videoCodec/audio/audioSource/audioCodec/audioSampleRate/audioChannels. 视频为 H.264/MP4; 无声音轨时最后三个音频格式字段为 null.

`stop` / `cancel` 的 options 可含 `id`, 仅操作指定录制; ID 已失效时返回 null/false, 不影响新录制. 所有方法接受 timeoutMs/signal. `start` 在取消与编码器启动同时发生时会等待启动结果并清理该录制. 主线程与 Worker 共享会话, 可以跨线程停止或取消. Worker 退出会丢弃它及其后代启动而尚未保存的录屏; 整次执行结束或系统撤回授权也会清理. 保存文件必须显式等待 stop, 再退出 Worker.

```js
"nodejs";

let images = require('autojs6:images');
let recorder = require('autojs6:screen_recorder');

(async () => {
    await images.requestScreenCapture();
    try {
        let active = await recorder.start(`/sdcard/Movies/node-${Date.now()}.mp4`, {
            width: 720, height: 1280, audio: 'playback',
        });
        let image = await images.captureScreen();
        try { console.log(images.getSize(image)); } finally { await images.recycle(image); }
        await new Promise(resolve => setTimeout(resolve, 2000));
        console.log(await recorder.stop({ id: active.id }));
    } finally {
        await recorder.cancel();
        await images.stopScreenCapture();
    }
})();
```

录制期间, 主线程和 Worker 都可调用 images.captureScreen 或 ScreenCapturer.nextImage. 截图尺寸随授权会话和方向变化, 视频保持 start 时的输出尺寸. 二者共用一次 MediaProjection, 不重复弹出授权. Worker 的 media_projection 返回对象保留可调用的 nextImage/stop 方法; 整个对象含方法, 不应直接 postMessage, 可以传递其产生的图像句柄.

原有 `recorder` / `autojs6:recorder` 是单独的音频录制模块, 现在也支持 Worker 的 start/getStatus/stop. 它仍要求 media + media.recording 与 Android RECORD_AUDIO, 按原有 maxDurationMs 和执行生命周期管理, 不使用 screen_recorder 的 Worker 退出丢弃规则. 不承诺设备支持同时启动多个麦克风采集器.

## Node.js Worker 模块

Worker 可直接 `require('autojs6:opencc')` / `require('images')`, 也可使用对应 ESM 导入. OpenCC 的可调用入口和 33 个成员, 以及图像模块的 82 个函数和 ObjectFrame 构造器, 使用与主线程相同的参数和返回契约. 裸名 `opencc` 仍优先解析已安装的同名 npm 包.

异步图像操作由父线程执行, 因而大图上传仍使用既有 PFD 通道. 返回 Worker 的字节经过 Node 结构化克隆, 是该 Worker 独立持有的 Buffer, 不承诺跨线程零复制. `getSize`, `getWidth`, `getHeight` 和 `isRecycled` 同步读取本地信息.

图像, Mat 和特征描述符句柄属于整个脚本执行, 可通过 `postMessage` 交接并在创建它的 Worker 退出后继续使用. 调用方应明确安排回收, 避免一个线程回收另一个线程正在使用的图像. Worker 已退出时才完成创建且无法交付的图像, Mat 和特征描述符会回收, 整次执行结束也会清理剩余句柄. 其他未列出的主线程高级模块仍需单独核查.

Worker 还提供 `pinyin` / `autojs6:pinyin`, `pinyin4j` / `autojs6:pinyin4j`, `screen_metrics` / `autojs6:screen_metrics` 和 `console` / `node:console` / `autojs6:console`, 支持 CJS, ESM 和嵌套 Worker. 拼音常量保持数字值, 可调用入口等价于 convert/of; `await pinyin(...)` 的二维数组保留异步 `compact()` 方法. 文本在进入结构化克隆之前转换, Buffer 和自定义 toString 对象不会被误解释为字节列表或普通对象.

坐标换算每次读取当前屏幕信息, `cX/cY/cXy/cYx` 的重载与主线程一致. 基数属于整个执行, 主线程和所有 Worker 共享同一个状态, 每个轴仍只能设置一次. 原生 Worker stdout/stderr 和自定义 Console 流保持在线程内运行; 窗口控制方法异步作用于父执行的控制台, 全局日志配置方法仍影响宿主全局. Worker 的全局 console 同样提供这些控制方法, 不把普通日志输出转为异步宿主调用. Rhino 的 input/rawInput/build 仍明确报不支持.

## Bun 客户端

**`Async`**

```js
"bun";

/** @type {typeof import('autojs6:bun-host')} */
let sdk = await import(process.env.AUTOJS6_HOST_CLIENT_MODULE);
let host = await sdk.createHostClient();

console.log(await host.pinyin.simple('北京上海'));
console.log(await host.pinyin4j.readings('重庆'));
console.log(await host.coordinates.cX(360, 720));
console.log(await host.coordinates.getInfo());
```

`autojs6:bun-host` 在声明与 Ace 中只作为类型别名使用. 可执行模块必须从环境变量给出的路径加载; 不应编写 `import ... from 'autojs6:bun-host'` 作为值导入.

| 能力 | 客户端成员 | 授权 |
| --- | --- | --- |
| `ui.toast` | `host.toast(text, duration?)` | 既有能力, 每次执行最多 4 条 |
| `device.info` | `host.device.info()` | Build 品牌, 型号, 系统与 ABI 信息 |
| `text.pinyin` | `host.pinyin.convert/simple/compare/compact/fromCodePoint/fromPhrase` | 默认启用, 需要可用 Pinyin 插件 |
| `text.pinyin4j` | `host.pinyin4j.of/as/readings` | 默认启用, 需要可用 Pinyin4j 插件 |
| `ui.layout` | `host.ui` 原生 JSON 布局与窗口事件 | 默认关闭 |
| `ui.compose` | `host.compose` 数据组件, 窗口与状态更新 | 默认关闭, 另需 Compose 插件 |
| `text.opencc` | `host.opencc` 的 `convert` 与 32 个转换方法 | 默认启用, 需要可用 OpenCC 插件 |
| `image.process` | `host.images` 的内存图像操作 | 默认启用, 受 JSON 传输上限约束 |
| `image.files` | `host.images.readImage/saveImage` | 默认关闭, 宿主文件访问独立授权 |
| `screen.capture` | `host.images.requestScreenCapture/captureScreen/stopScreenCapture` | 默认关闭, 另需 Android 录屏授权 |
| `screen.record` | `host.screenRecorder.start/stop/cancel/getStatus` | 默认关闭, 另需 screen.capture 与系统授权, 输出 MP4 文件 |
| `screen.record.microphone` | audio: microphone / mixed | 默认关闭, 另需 Android 录音权限和麦克风前台服务 |
| `screen.record.playback` | audio: playback / mixed | 默认关闭, 另需 Android 录音权限, 遵循来源应用捕获策略 |
| `screen.metrics` | `host.coordinates` 的四个换算及基数成员 | 默认启用, 与 Node 坐标语义相同 |
| `console.control` | `host.console` 的窗口与配置方法 | 默认关闭, 需要明确授权 |

先通过 `host.capabilities` 或 `host.hasCapability(name)` 检查当前执行是否协商到所需能力. Bun 的基数 setter 和控制台命令返回 `Promise<void>`. 普通 `console.log` 等仍由 Bun 原生控制台处理, `host.console` 是显式宿主窗口控制入口.

Bun 不自动获得 Node 的全部模块, Java 互操作或 `rhino.run`. 无障碍选择器通过下述显式 `host.accessibility` 接口使用. 截图需要独立能力开关及 Android 同意. SDK 的图像句柄也不是 Rhino 图片对象, 不自动注入 `cX`, `pinyin`, `opencc` 或 Rhino 的对象图.

## Bun 无障碍, 应用与剪贴板

配套 Bun Runtime 0.3.0 和宿主 Android build 5572, 显式客户端提供 `host.accessibility`, `host.app`, `host.clipboard`. 参数转换, 选择器描述符, Intent 构造及公开别名与 Node.js 对应模块保持一致; Android 操作复用同一宿主实现. 不注入 Rhino 全局对象.

| 执行能力 | 默认值 | 范围 |
| --- | --- | --- |
| `app.control` | 关闭 | 应用查询, 启动页面/服务, 发送广播, 文件查看/编辑和宿主应用信息 |
| `accessibility.control` | 关闭 | 无障碍状态, 节点查找, 点击/长按, 输入, 滚动和系统导航 |
| `accessibility.gesture` | 关闭 | 在 `accessibility.control` 之外单独允许 `swipe` 和 `gesture` |
| `clipboard.control` | 关闭 | 读取, 判断和替换系统剪贴板文本 |

在插件中心的 Bun 能力桥设置中分别开启. 撤销影响下一次执行. 打包项目需在 `bun.hostCapabilities` 中显式声明相同 ID; 无障碍服务还需要在 Android 设置中启用, 每个打包应用独立授权. 系统剪贴板访问受 Android 前台窗口等条件限制, 不因脚本运行于 Bun 插件进程而改用插件的剪贴板身份.

```js
"bun";
let { createHostClient } = await import(process.env.AUTOJS6_HOST_CLIENT_MODULE);
let host = await createHostClient();
let enabled = await host.accessibility.isEnabled();
if (!enabled) throw Error('请先开启运行此脚本的应用的 Android 无障碍服务.');
let target = host.accessibility.text('示例按钮');
let node = await host.accessibility.findOne(target);
if (node) await host.accessibility.click(target);
```

### host.accessibility

| 方法 | 返回值 | 说明 |
| --- | --- | --- |
| `isEnabled(options?)` / `ensureEnabled(options?)` | `Promise<boolean>` | 查询或等待宿主服务可用. `ensureEnabled` 不替代 Android 授权 |
| `findByText(text, options?)` / `findOne(selector, options?)` | `Promise<UiNodeSnapshot \| null>` | 返回节点数据快照, 不返回 Java 节点对象 |
| `findAll(selector, options?)` | `Promise<readonly UiNodeSnapshot[]>` | 返回匹配节点快照 |
| `click(selector, options?)` / `clickText(text, options?)` / `longClick(selector, options?)` | `Promise<boolean>` | 操作匹配节点 |
| `setText(selector, text, options?)` | `Promise<boolean>` | 设置可编辑节点文本 |
| `scrollForward(selector, options?)` / `scrollBackward(selector, options?)` | `Promise<boolean>` | 对目标执行滚动操作 |
| `back(options?)` / `home(options?)` / `recentApps(options?)` | `Promise<boolean>` | 请求系统导航动作 |
| `swipe(x1, y1, x2, y2, durationMs?, options?)` | `Promise<boolean>` | 坐标滑动, 需独立手势授权 |
| `gesture(durationMs, points, options?)` | `Promise<boolean>` | 按 `[x, y]` 数组执行手势, 需独立手势授权 |

本地选择器构造器包含 `text`, `desc`, `id`, `className`, `textContains`, `descContains`, `textMatches`, `descMatches`, `idMatches`, `classNameMatches`, `clickable`, `enabled`, `scrollable`, `depth`, `boundsInside`, `boundsContains`. 正则参数可用字符串, `RegExp` 或 `{pattern, flags}`. 可通过对象展开组合描述符; 构造器本身不访问 Android, 不需要授权.

当前宿主的 `click` 实际操作支持选择器目标. 坐标点击可使用相同起终点的短 `swipe` 或 `gesture`, 需要 `accessibility.gesture`. 与 Node.js 相同, 此接口不提供 Rhino 的链式 Java 节点对象或任意节点反射. 手势时长最多 10000 ms; 返回值表示完成或取消结果. 查找操作查询当时的节点树, 不把 `timeoutMs` 解释为持续重试直到节点出现.

### host.app

`packageName`, `versionName`, `versionCode` 为当前宿主应用或打包应用的只读信息, 通过 `app.control` 授权获取, 不表示 Bun 插件自身的包信息.

| 方法组 | 说明 |
| --- | --- |
| `getPackageName`, `getAppName`, `isInstalled`, `getInstalledApps`, `getPackageInfo` | 查询 Android 对当前应用可见的包信息. `getInstalledApps({includeSystem:false})` 排除系统应用 |
| `launchPackage`, `launchApp`, `launch` | 启动应用. `launch` 为 `launchPackage` 别名 |
| `openAppSetting`, `openAppSettings`, `launchSettings`, `launchAppDetailsSettings` | 打开应用详情设置, 缺省目标为当前宿主/打包应用 |
| `startActivity`, `openUrl` | 请求页面或 URI. Intent 可指定 action, data/type, packageName/className, categories, flags 和 JSON extras |
| `sendBroadcast`, `startService` | 发送广播或启动普通服务, 仍受 Android 导出规则和后台限制约束 |
| `sendEmail`, `uninstall` | 请求邮件编辑页或系统卸载确认页, 不自动发送邮件或确认卸载 |
| `viewFile`, `editFile` | 使用宿主允许的路径与文件授权请求查看/编辑 |
| `intent`, `parseUri`, `intentToShell` | 同步构造 JSON 描述符, 检查 URI 或生成命令文本. `intentToShell` 不执行命令 |

宿主操作返回 Promise. `getUriForFile`, `kill`, root/Shizuku 和双开相关入口继续明确拒绝, 不返回伪造的成功值. `startService` 不自动升级为前台服务. 包查询遵循 Android 包可见性, 大结果也遵循 Bun 返回体积上限.

### host.clipboard

| 方法 | 返回值 |
| --- | --- |
| `getText(options?)` | `Promise<string>` |
| `setText(text, options?)` | `Promise<void>` |
| `hasText(options?)` | `Promise<boolean>` |

读取与写入均使用宿主/打包应用的 `ClipboardManager`. `hasText` 判断读到的文本是否非空. Android 拒绝后台读取时, 可能读不到文本; 不能用一次空结果证明剪贴板本来为空.

### 等待, 取消与限额

调用选项支持 `timeoutMs` (整数, 1..120000) 和 `signal` (`AbortSignal`). 默认等待时间沿用 Node.js 对应方法, 通常为 5000 或 10000 ms; `ensureEnabled` 为 30000 ms. 长手势和服务等待通过短轮询完成, 不扩大 Bun 的 10 秒单次 RPC 上限. 同时最多四项操作, SDK 会排队后续调用. 取消停止等待和回收任务记录, 无法撤回已经交给 Android 的点击, Intent 或手势. 原有单次执行 1024 次桥调用及 JSON 体积限制仍适用, 轮询也计入调用次数.


## UI 与 Compose

**`Async`**

以下接口对应当前工作区的开发版本.

Node.js 主线程和 CJS/ESM/嵌套 Worker 可使用 `require('ui')` / `require('compose')`, 也可从 `autojs6:ui` / `autojs6:compose` 导入. 两个模块使用 `ui` 能力声明. Bun 使用 `host.ui` / `host.compose`, 在插件设置中分别开启 `ui.layout` / `ui.compose`, 默认均关闭. Compose 还需要已安装, 启用并授权且契约兼容的 3-Block Compose 插件. Android 悬浮窗权限仍由系统判断.

`ui.showLayout(descriptor)` 返回 Promise, 完成后取得窗口句柄. 描述树支持 Text, Button, Input, Image, Column/Row, ScrollView, List, Switch, Checkbox, Progress 和 Tabs 等现有原生布局组件. `ui.state(initial)` 与 `ui.bind(state, key)` 在运行时本地创建状态和绑定. 窗口提供 `on/once/off`, `update`, `batchUpdate`, `close` 和 `closed`; 更新和关闭返回 Promise. 事件回调在创建窗口的 JavaScript 线程执行. 此入口支持 JSON 描述树, XML 字符串或本次执行的 View 句柄; E4X 语法和 Java View 对象不跨运行时传输.

`compose` 提供 54 个 V1/V2 数据组件构造器及 AndroidView 互操作构造器, 包括 Text, Button, Column, TextField, LazyColumn, Scaffold, DatePicker, TimePicker 和 NavigationBar. 组件, `state`, `ref`, `modifier`, `batch`, `memo`, `theme` 都在本地处理. `isAvailable()` 与 `version()` 返回 Promise; `sessions` 是当前打开会话的快照. `mount`, `dialog`, `floaty` 和直接调用 `compose(root)` 都返回 `Promise<Session>`, 并等待窗口实际接入渲染器. 每个窗口有独立 Activity 或悬浮窗口生命周期.

```js
// Node.js 主线程或 Worker; Bun 改为 let compose = host.compose.
let compose = require('autojs6:compose');
(async () => {
let count = compose.state(0);
let window = await compose.mount(() => compose.Column({ padding: 16 },
    compose.Text({ text: 'Count: ' + count.value }),
    compose.Button({ onClick() { count.value++; } }, compose.Text({ text: 'Add' }))
));
// 完成后: await window.close().
})().catch(error => console.error(error));
```

组件参数遵循宿主组件目录, 颜色使用字符串或 ARGB 数字, 尺寸使用 dp/sp 数字或相应后缀字符串. Image.src 接受本次执行的图像句柄或 `@drawable/name`; 渲染器持有独立像素副本. 悬浮窗口选项 x/y/width/height 使用物理像素, width/height 的 -1/-2 表示 match_parent/wrap_content. `dialog` 的 type 为 alert 或 bottomSheet, cancelable 默认 true.

事件使用一个数据对象, 例如 `onValueChange(event)` 中的 `event.text`, `event.selectionStart`, `event.selectionEnd`, `event.editSeq`, `event.value`, 或 `onCheckedChange(event)` 中的 `event.checked`, 具体字段由组件决定. 回调中的 this 为对应组件. 跨渲染期间仍然有效的节点事件路由到当前已接受树的处理器. 文本框状态更新使用带原生编辑序号的编辑命令, 不直接覆盖原生输入缓冲区.

会话提供 `root`, `isClosed`, `update(callback)`, `post`, `on/off`, `render(root)`, `flush`, `setTheme` 和 `close`. `render`, `flush`, `setTheme`, `close` 返回 Promise; `flush` 等待当前排队的状态更新被宿主接受. `ref.current` 在帧接受后设置, 移除节点或关闭窗口时清空. 组件提供 `get/set`, `append/insert/remove/replace/clear/slot`, `on/off` 和异步 `focus/blur/scrollTo/edit`. 读写 Compose 数据组件属性使用 get/set; 下文另列原生 View 句柄的异步属性代理.

会话 `showSnackbar(message, options?)` 返回 Promise, 要求树中有使用默认 snackbarHost 的 Scaffold. options 支持 actionLabel, duration (short/long/indefinite), onAction 和 onDismiss, 每条消息至多触发一次终态回调. 悬浮会话还提供异步 `setPosition`, `setSize`, `getX/getY/getWidth/getHeight`, `requestFocus`, `disableFocus`, `setTouchable`; `session` 指向同一会话. 窗口旋转重建会保留组件状态, 已输入文本及事件路由.

该覆盖是可传输数据 API. Java View/Context 对象, 任意 Java 类构造器, 任意 Java 属性反射和原生悬浮窗调整框仍未纳入, 不应按同名模块推断这些入口可用. 未定义入口不会被模拟为成功. Node/Bun 仍不注入 Rhino 全局 `ui` / `compose`.

窗口及回调属于创建它们的执行. Worker 终止时关闭该 Worker 创建的窗口, 包括终止后才返回的创建结果, 主线程的其他窗口继续使用. 脚本结束关闭全部所属窗口. Bun 仍遵守 64 KiB 请求, 256 KiB 返回, 4 个并发请求和每次执行 1024 次调用的桥接限制; UI 事件读取也计入调用次数. 长时间界面脚本应观察失败事件并显式关闭, 不能把此次 API 扩展理解为取消现有运行时配额.

### 原生 View 与 XML compose 容器

以下互操作接口需要宿主构建 5562 及配套 Node.js 1.8.0/Bun 0.2.7 客户端. Bun 的 AndroidView 工厂和 XML attach 同时需要 ui.layout 与 ui.compose.

| 方法 | 异步结果与用途 |
| --- | --- |
| ui.context() | Promise<UiContext>, 包含执行标识, density/scaledDensity, packageName, runtimeVersionCode/applicationVersionCode |
| ui.inflate(layout, context?) | Promise<NativeViewHandle>, 从 XML 字符串或 JSON 描述树创建未显示的原生 View |
| ui.createView(type, props?, context?) | Promise<NativeViewHandle>, 创建受支持的原生组件, 可使用 TextView, Button, FrameLayout 和 Compose 等别名 |
| ui.findView(root, id) | Promise<NativeViewHandle \| null>, root 是 View 句柄或窗口 ID |
| ui.view(token) | 当前线程的句柄包装, 用于恢复 Worker structuredClone 传回的纯数据标识 |
| compose.AndroidView({ view }) | 本地组件; view 是句柄或返回句柄/Promise 的工厂函数 |
| compose.attach(container, root, options?) | Promise<Session>, container 必须是本次执行的 XML compose/Compose 容器 |

UiContext 提供异步 inflate/createView. View 句柄提供 get(), get(name), set(patch), findView(id), dispose() 和本地 on/once/off 监听. get 返回快照, 包含 id/type, width/height, enabled/visible/attached, contentDescription, 文本组件的 text 和勾选组件的 checked. set 不替换 type 或 children. XML 接受 vertical/horizontal/frame/compose/text/button/input 等受支持标签; w/h, layout_width/layout_height 和 android: 属性名会规范化. 尺寸默认 dp, px 后缀表示物理像素, `*`/match_parent 表示填满, auto/wrap_content 表示按内容计算. XML 不接受 DTD/实体, 最大 64 KiB, 深度 16, 节点 5000, 每层子节点 128.

AndroidView 工厂在创建组件的 JavaScript 线程执行, 接收数据 context, 通过其异步 inflate/createView 返回原始句柄标识. 可用 ui.view 包装该标识后读写属性. 工厂不是 Android Context 或 Android 主线程回调. 同一个组件节点保留工厂结果; 重新创建节点会重新调用工厂, 需要稳定身份时使用已有 View 句柄或 memo.

```js
// Node.js 主线程或 Worker; Bun 改为 let ui = host.ui, compose = host.compose.
let ui = require('ui'), compose = require('compose');
(async () => {
    let root = await ui.inflate('<vertical><text text="XML"/><compose id="content" h="300"/></vertical>');
    let window = await ui.showLayout(root);
    let native = await ui.createView('Button', { text: 'Native button' });
    let container = await root.findView('content');
    let session = await compose.attach(container, compose.AndroidView({ view: native }));
    native.on('click', event => console.log(event.type));
    // 完成后依次 await session.close(), native.dispose(), window.close(), root.dispose().
})().catch(error => console.error(error));
```

View 属于本次脚本执行, 可在 Node 主线程及 Worker 间传递标识, 配套宿主 build 5569 及客户端支持独立事件订阅, 主线程与 Worker 可分别读取事件. AndroidView 借用 View; 关闭会话后 View 仍可复用. 有父节点或已被其他会话借用的 View 会被拒绝, 不能将包含容器的 View 再借给该容器. 只有 inflate 的根句柄能 dispose, 子句柄是别名. dispose 前须关闭借用会话; 释放容器会关闭其附着会话, 包括尚未显示的容器. attach 在同一容器成功接入新会话后关闭旧会话; 若要复用旧会话借用的同一个 View, 应先关闭旧会话. 旋转后重新连接生命周期, 保留 View 身份和状态.

Worker 退出只关闭它创建的窗口和附着会话; 已传回的 View 属于整个执行, 仍可由主线程使用和释放. 未交付的异步 View 创建结果会在 Worker 退出后释放. ui.showLayout(XML) 创建的根由窗口管理; ui.showLayout(handle) 借用已有根, 关闭窗口后由调用方 dispose. 整次脚本结束释放全部 View.

### 常用 XML 组件与 View 属性代理

本节对应宿主构建 5563, Node.js Runtime 1.9.0/Bun Runtime 0.2.8 及配套客户端. Node.js 主线程与 CJS/ESM/嵌套 Worker 使用同一接口; Bun 通过 host.ui 调用, 仍需 ui.layout 授权. Compose 混合布局另需 ui.compose.

| XML 标签 | 原生组件 | 常用属性 |
| --- | --- | --- |
| hscroll | HorizontalScrollView | width, height, padding |
| space | Space | width, height |
| relative | RelativeLayout | 子项的 below/above, toLeftOf/toRightOf/toStartOf/toEndOf, alignTop/alignBottom/alignStart/alignEnd 和 alignParent/center 规则 |
| card | CardView | cardCornerRadius, cardElevation, backgroundColor |
| spinner | Spinner | entries/items, selectedIndex |
| seekbar | SeekBar | min, max, progress |
| ratingbar | RatingBar | numStars, stepSize, rating, isIndicator |
| radiogroup | RadioGroup | orientation, checkedId/checkedButton |
| radio | RadioButton | text, checked |
| toggle | ToggleButton | checked, textOn, textOff |
| checkedtext | CheckedTextView | text, checked |
| numberpicker | NumberPicker | min, max, value, wrapSelectorWheel, displayedValues |

这些标签及原生类名可用于 ui.inflate 的 XML/JSON 描述和 ui.createView. img 对应 ImageView, linear 对应线性布局. 原有 vertical/horizontal/frame/compose/text/button/input/image/scroll/list/switch/checkbox/progressbar 等标签继续可用. 相对布局规则只引用同一父容器中具有唯一 id 的兄弟组件, 同时接受 layout_below 等 layout_ 前缀. XML id 不可重复. 子项的宽高, 间距与 weight 会保留; weight 要求线性布局父容器, layoutGravity 要求线性/Frame/Drawer/Toolbar 父容器.

View 句柄现在支持直接属性赋值, 读取返回 Promise:

```js
// 位于 async 函数中; Node.js 使用 let ui = require('ui'), Bun 使用 let ui = host.ui.
let root = await ui.inflate('<vertical><spinner id="choice" entries="One|Two"/><seekbar id="level" max="100"/><text id="label" text="Ready"/></vertical>');
let window = await ui.showLayout(root);
let choice = await root.findView('choice');
let label = await root.findView('label');
choice.items = ['Alpha', 'Beta', 'Gamma'];
choice.selectedIndex = 2;
label.text = 'Queued';
label.text = 'Ready';
await choice.flush();
console.log(await choice.selectedIndex); // 2.
console.log(await label.text); // Ready, 等待此前该句柄的赋值完成.
// 完成后依次 await window.close(), root.dispose().
```

属性代理覆盖 46 个可写名称:

| 类别 | 属性 |
| --- | --- |
| 通用状态 | enabled, visible, visibility, alpha, contentDescription, clickable, focusable, backgroundColor |
| 布局 | width, height, padding, margin, gravity, layoutGravity, weight, orientation |
| 文本与输入 | text, hint, textSize, textColor, textStyle, singleLine, maxLines, inputType |
| 选择与数值 | checked, min, max, progress, indeterminate, rating, numStars, stepSize, isIndicator, items, selectedIndex, checkedId, textOn, textOff, value, wrapSelectorWheel, displayedValues |
| 卡片与图像 | cardCornerRadius, cardElevation, src, scaleType, adjustViewBounds |

另外提供 attached, layoutWidth, layoutHeight, drawableWidth, drawableHeight, xmlId 六个只读 Promise 属性. 原有 id/type/context 是同步句柄元数据; id 是执行所属句柄标识, xmlId 是 XML 中声明的 id. 属性是否适用取决于原生组件, 例如 text 属于文本控件, checked 属于 Checkable 控件, selectedIndex 属于 Spinner/ListView. 在不支持的组件上读写会拒绝, 不会模拟成功. get()/get(name)/set(patch) 继续可用, set 拒绝未知属性及身份/子树修改.

赋值依 JavaScript 线程内该句柄的顺序提交. 对象与数组在赋值时复制. `await view.flush()` 或下一次 await 属性读取会等待此前写入, 并抛出待报告的原生赋值错误; 报告后仍可继续修正属性. set(patch) 本身返回 Promise, 直接 await 即可接收错误. 非 JSON 值会同步拒绝. dispose 会先等待已排队写入, 即使某次赋值失败也会尝试释放原生根, 然后报告错误. flush 表示属性已提交, 不等于 Android 已完成下一次布局/绘制. 多个 Worker 同时操作同一个句柄时, 需要由调用方协调顺序和事件读取.

width/height/padding/margin/cardCornerRadius/cardElevation 的数字或 dp 字符串按 dp 写入, px 后缀表示物理像素; width/height 支持 match_parent/-1 和 wrap_content/-2. 几何, 间距及卡片尺寸的读取值是物理像素. textSize 按 sp 写入和读取, 写入也可指定 px. padding/margin 支持 1 项, 2 项 (水平/垂直) 或 4 项 (左/上/右/下). visibility 取 visible/invisible/gone; visible=false 等价于 gone. 颜色写入接受 Android 颜色字符串或 ARGB 数字, 读取返回 ARGB 整数; 非纯色背景可返回 null. textStyle 取 normal/bold/italic/bold|italic. inputType 支持原生整数及 text/textPassword/number/numberDecimal/phone/textEmailAddress. scaleType 接受 fitCenter/centerCrop 等 ImageView 缩放名称或原生大写名称.

entries 字符串使用 `|` 分隔; items 也可使用数组. selectedIndex 从 0 开始, 同时接受表示无选择的 -1, 原生读取也可能为 -1. ProgressBar/SeekBar 的 min 写入要求 Android API 26 及以上. NumberPicker 的 displayedValues 为 null 或长度等于 max-min+1 的字符串数组. 原生 widget 约束仍决定最终显示状态. 数值必须有限, alpha 为 0..1, numStars 为 1..100, stepSize 为 0.01..100, textSize 为 0..1000, maxLines 为 1..10000, 布局尺寸不超过 1000000, items 不超过 1024 项; 仍受桥接字节预算限制. 不合法的属性值在变更前校验.

ImageView.src 支持本次执行的图像句柄, @drawable/name, @android:drawable/name 或 null. 图像句柄使用独立像素副本, 原图释放后 View 仍可显示. 此入口不隐式打开宿主文件或 URL. 资源名称不存在及非图像句柄对象会被拒绝.

Spinner 通过 select 事件报告 selectedIndex/text. SeekBar 的 change 事件包含 progress/value/fromUser, 另有 startTracking/stopTracking. RatingBar 的 change 包含 rating/value/fromUser. RadioGroup 通过 change 报告 checkedId; ToggleButton/CheckedTextView 使用 checked/value; NumberPicker 使用 value/oldValue. Spinner 的初始化及程序设置也可能产生选择事件. ToggleButton 保留脚本提供文字的大小写.

Ace 默认运行时模式下会提示属性的 Promise 读取类型, 以及 ui.inflate 的普通 XML 字符串和无插值模板字符串中的标签/属性名. 其他对象的同名 inflate 方法不启用该提示. 这些扩展仍使用显式组件和属性目录, 任意 Java 反射及完整 Rhino XML 表达式绑定仍需后续实现.

### WebView, Drawer, Toolbar 与 ViewPager

本节需要宿主构建 5564, Node.js Runtime 1.10.0/Bun Runtime 0.2.9 及配套客户端. Node.js 主线程与 CJS/ESM/嵌套 Worker 使用同一 View 句柄方法, Bun 使用 host.ui, 需要 ui.layout. 所有方法返回 Promise, 与同一包装对象的属性赋值按调用顺序提交.

| XML 标签 / JSON 类型 | 属性与操作 |
| --- | --- |
| webview / WebView | url, html, javaScriptEnabled, domStorageEnabled; loadUrl/loadHtml, evaluateJavascript, goBack/goForward, reload/stopLoading |
| drawer / Drawer / DrawerLayout | drawerLockMode; openDrawer/closeDrawer; 子项使用 layout_gravity="start" 或 "end" |
| toolbar / Toolbar | title, subtitle, navigationIcon, navigationContentDescription, menu; showOverflowMenu/hideOverflowMenu |
| viewpager / ViewPager | currentItem, offscreenPageLimit; setCurrentItem |

这批增加 12 个可写属性, 合计 58 个. canGoBack/canGoForward, drawerOpenStart/drawerOpenEnd 和 pageCount 是新增的只读 Promise 属性. title 对 Toolbar 可写, 对 WebView 只读; progress 对 WebView 只读, 不改变原有 ProgressBar/SeekBar 的可写契约. 不支持的组件/操作组合会拒绝 Promise.

#### view.loadUrl(url)

**`Async`**

- **url** { [string](dataTypes#string) } - 交给 Android WebView 加载的地址.
- <ins>**returns**</ins> { [Promise](dataTypes#promise) } - 加载请求已提交.

完成页面加载应监听 pageFinished, 不能把本方法的完成当作页面加载完成. WebView 的网络, 明文 HTTP, 文件和内容访问由当前应用的 Android 权限, 网络安全配置及系统 WebView 决定. 不添加 Java 对象桥, 不自动批准网页摄像头/麦克风权限或绕过证书检查. 下载, 文件选择和自定义同步 WebViewClient/WebChromeClient 回调不属于本批接口.

#### view.loadHtml(html, baseUrl?)

**`Async`**

- **html** { [string](dataTypes#string) } - HTML 文本, MIME 为 text/html, 编码为 UTF-8.
- **[ baseUrl = null ]** { [string](dataTypes#string) | [null](dataTypes#null) } - 相对资源的基准地址.
- <ins>**returns**</ins> { [Promise](dataTypes#promise) } - 加载请求已提交.

html 属性赋值等同于不指定 baseUrl 的加载. url 与 html 不能在同一次 set 中同时出现. html 读取最后提交的 HTML 文本, 不读取当前 DOM; 页面导航后可为 null. url/title 读取当前 WebView 状态, 页面未准备好时可为 null.

#### view.evaluateJavascript(source, options?)

**`Async`**

- **source** { [string](dataTypes#string) } - 在当前页面执行的 JavaScript.
- <ins>**returns**</ins> { [Promise](dataTypes#promise) } - WebView 返回值解码后的 JSON 值.

javaScriptEnabled 默认 false, 需先显式赋值 true. domStorageEnabled 默认 false. 返回对象/数组/字符串等 JSON 值, 不返回包着 JSON 的字符串. 页面异常和不可序列化值遵循系统 WebView 的返回语义, 此接口不把网页 Promise 自动转换为宿主 Promise. 配套 Node 1.11.0/Bun 0.2.10 客户端使用异步原生任务, options.timeoutMs 默认 30000, 可设为 100..120000 毫秒; 超时或 View 销毁时拒绝. 用户等待对话框不会占住 Bun 的单次 RPC. goBack/goForward/reload/stopLoading 无参数, 返回 Promise; 无可用历史时遵循 WebView 行为.

WebView 事件包括 pageStarted/pageFinished (url), titleChanged (title), progressChanged (progress), webError (message, 可含 url/mainFrame/errorCode/didCrash) 和 httpError (url/mainFrame/statusCode). 根 View 释放或执行结束会停止并销毁所有 WebView, 包括 ViewPager 中未显示的页面; 渲染进程退出时报告 webError 并释放该 View.

#### view.evaluateJavascriptAsync(source, options?)

**`Async`**

- **source** { [string](dataTypes#string) } - 在当前页面求值的表达式, 可返回 Promise.
- **[ options ]** { [Object](dataTypes#object) } - timeoutMs 默认 30000 毫秒, 范围 100..2147483647.
- <ins>**returns**</ins> { [Promise](dataTypes#promise) } - 等待表达式及其 Promise 完成后的 JSON 值, undefined 转为 null.

需要 AutoJs6 build 5568, Node.js Runtime 1.14.0 或 Bun Runtime 0.2.13, 并启用 javaScriptEnabled. Node.js 主线程, CJS/ESM/嵌套 Worker 和 Bun 使用相同接口. 页面异常保留错误名称和信息; BigInt/循环引用等 JSON 序列化错误也会拒绝. DOM 对象, MediaStream 和 MediaRecorder 留在网页中, 只传递 JSON 数据或 blob URL.

等待页面结果不会阻塞 UI 命令队列, 因而权限回调, 暂停/停止录制和其他求值可以并行进行. 导航, 销毁 View, Worker 退出或超时会结束对应等待并清理结果槽. 普通 Promise 不具备通用取消能力: 超时不会撤销页面已执行的副作用, 应在 finally 中停止 MediaStreamTrack/MediaRecorder, 或销毁属于当前执行的 WebView. Worker 在借用其他线程的 View 时也应显式管理网页资源的生命周期. 关闭承载已获媒体授权网页的窗口时, WebView 导航到 about:blank 以释放摄像头/麦克风; 句柄仍可复用, 再次显示时需重新加载页面. Worker 退出关闭其窗口时也执行这一清理, 不关闭其他线程的窗口.

```js
let value = await web.evaluateJavascriptAsync('Promise.resolve({ answer: 42 })');
console.log(value.answer);
```

#### view.openDrawer(side?, animate?) / view.closeDrawer(side?, animate?)

**`Async`**

- **[ side = 'start' ]** { [string](dataTypes#string) } - start 或 end, 遵循布局方向.
- **[ animate = true ]** { [boolean](dataTypes#boolean) } - 是否播放开合动画.
- <ins>**returns**</ins> { [Promise](dataTypes#promise) } - 开合请求已提交.

Drawer 要求恰好一个不带侧栏 gravity 的内容子项, 两侧各至多一个侧栏子项. drawerLockMode 为 unlocked/lockedClosed/lockedOpen, 默认 unlocked. drawerOpenStart/drawerOpenEnd 读取原生开合状态. drawerOpened/drawerClosed 事件含 drawerId/side, drawerSlide 另含 offset, drawerStateChanged 含原生 state (0 空闲, 1 拖动, 2 动画中).

#### Toolbar 菜单与导航

navigationIcon 接受 @drawable/name, @android:drawable/name 或 null. menu 是数组, 每项必须有唯一非空 id 与 title; 可含 icon, enabled/visible (默认 true) 和 showAsAction (never/ifRoom/always, 默认 never). 赋值会替换原菜单. navigationClick 表示导航图标点击, menuClick 包含 itemId/title. showOverflowMenu()/hideOverflowMenu() 返回 Promise<boolean>, 由 Android 报告是否接受操作; 菜单需要已附着的可见 Toolbar. Toolbar 也接受普通子 View, layoutGravity 决定位置. 它不会自动成为 Activity 的 ActionBar.

#### view.setCurrentItem(index, smoothScroll?)

**`Async`**

- **index** { [number](dataTypes#number) } - 从 0 开始的页面索引, 必须小于 pageCount.
- **[ smoothScroll = true ]** { [boolean](dataTypes#boolean) } - 是否平滑滚动.
- <ins>**returns**</ins> { [Promise](dataTypes#promise) } - 切换请求已提交.

ViewPager 的直接子项组成固定页面集合, 页面填满容器. currentItem 赋值使用无动画切换, offscreenPageLimit 默认为 1, 可设为 1..128. findView 可查找未显示页面及其后代; set 不动态替换页面集合. pageSelected 包含 position/pageId, pageScrolled 包含 position/offset/offsetPixels, pageScrollStateChanged 包含 state. 通过 ui.showLayout(XML/handle) 显示的受管窗口在旋转时保留页面, WebView DOM 和事件路由.

```js
let ui = require('ui'); // Bun 改为 host.ui.
(async () => {
    let root = await ui.inflate('<vertical><toolbar id="bar" title="Pages"/><viewpager id="pages" h="180"><text text="One"/><webview id="web" javaScriptEnabled="true"/></viewpager></vertical>');
    let window = await ui.showLayout(root);
    let bar = await root.findView('bar'), pages = await root.findView('pages'), web = await root.findView('web');
    bar.menu = [{ id: 'next', title: 'Next', showAsAction: 'always' }];
    bar.on('menuClick', () => pages.setCurrentItem(1).catch(console.error));
    web.on('pageFinished', () => web.evaluateJavascript('document.title').then(console.log).catch(console.error));
    await web.loadHtml('<title>Portable WebView</title><p>Hello</p>');
    // 完成后依次 await window.close(), root.dispose().
})().catch(console.error);
```

Ace 同步提供上述方法, 属性, XML 标签与属性名提示. 这些组件不要求 JSOX 原型扩展.

### WebView 下载, 文件选择与客户端回调

需要宿主构建 5565, Node.js Runtime 1.11.0/Bun Runtime 0.2.10. Node 主线程, CJS/ESM/嵌套 Worker 和 Bun host.ui 使用相同的 NativeViewHandle 方法. 本批提供 19 个 WebViewClient 回调, 20 个 WebChromeClient 回调及 onDownloadStart, 具体字段和答复类型见配套声明.

| 方法 | 返回值与用途 |
| --- | --- |
| setWebViewClient(client, options?) | Promise<void>; 设置页面, 导航, 资源, 认证和安全回调; null 移除 |
| setWebChromeClient(client, options?) | Promise<void>; 设置标题, 对话框, 文件, 权限, 弹窗和全屏回调; null 移除 |
| setDownloadListener(callback) | Promise<void>; 设置下载通知; null 移除, 不自动开始下载 |
| chooseFiles(options?) | Promise<string[] \| null>; 系统文件选择, 取消返回 null |
| cancelFileChooser() | Promise<void>; 取消当前选择 |
| exitFullscreen() | Promise<boolean>; 退出受管 WebView 全屏 |
| download(options) | Promise<{id, url, fileName}>; 显式提交 Android DownloadManager 下载 |
| getDownload(id) | Promise<DownloadState \| null>; 查询当前应用的下载任务 |
| readDownload(id, offset?, length?) | Promise<Uint8Array>; 读取已下载文件, 默认最多 32768 字节 |
| cancelDownload(id) | Promise<boolean>; 移除任务和对应文件 |

回调接收 JSON 数据, this 是本地 View 包装器. 设置客户端会替换同类回调, 取消其尚未答复的请求. 自定义答复支持 Promise; 普通回调超时默认 15 秒, onShowFileChooser 默认 120 秒, callbackTimeoutMs 自宿主 5567 起可设为 100..2147483647 毫秒. 异常, 无效答复, 超时和取消会采用拒绝/取消结果; 异常和超时通过 webCallbackError 报告. 过期答复不会影响后续请求.

WebViewClient 回调如下. 通知回调无需返回值; 请求回调可直接返回表中的答复或相应 Promise.

| 回调 | 主要数据或答复 |
| --- | --- |
| onPageStarted, onPageFinished, onPageCommitVisible, onLoadResource | 通知 url/title |
| doUpdateVisitedHistory | 通知 url/title/reload |
| onScaleChanged | 通知 oldScale/newScale |
| onReceivedError | 通知请求信息, errorCode/message |
| onReceivedHttpError | 通知请求信息, statusCode/reasonPhrase/mimeType/responseHeaders |
| onReceivedLoginRequest | 通知 realm/account/args |
| shouldOverrideUrlLoading, shouldInterceptRequest | 通知请求信息与 blocked/intercepted |
| shouldOverrideKeyEvent, onUnhandledKeyEvent | 通知 keyCode/action, 前者另含 blocked |
| onFormResubmission | 返回 boolean/null, true 重新提交 |
| onReceivedHttpAuthRequest | 接收 host/realm, 返回 {username,password}/null |
| onReceivedSslError | 接收 url/primaryError, 返回 cancel/proceed/null |
| onReceivedClientCertRequest | 接收 host/port/keyTypes/principals, 返回 {alias}/ignore/cancel/null |
| onSafeBrowsingHit | 接收请求信息/threatType, 返回 backToSafety/proceed/showInterstitial/null |
| onRenderProcessGone | 通知 didCrash/rendererPriorityAtExit, 宿主释放该 WebView |

请求信息包含 url, method, mainFrame, hasGesture, redirect 和 headers. 设备未提供请求头时 headers 为空对象. 资源响应的数据流不跨进程传给回调.

| WebChromeClient 回调 | 主要数据或答复 |
| --- | --- |
| onProgressChanged, onReceivedTitle | 通知 progress 或 title |
| onReceivedIcon | 通知最多 64x64 的 PNG 缩略图 pngBase64/width/height |
| onReceivedTouchIconUrl | 通知 url/precomposed |
| onConsoleMessage | 通知 message/sourceId/lineNumber/level |
| onJsAlert, onJsConfirm, onJsBeforeUnload | 接收 url/title/message/defaultValue, 返回 boolean/null |
| onJsPrompt | 接收同上, 返回 string/null |
| onPermissionRequest | 接收 origin/resources, 返回允许的资源数组, `{ resources, requestAndroidPermissions? }` 或 null; 仅对象的显式选项会申请缺少的 Android 媒体权限 |
| onPermissionRequestCanceled | 通知已取消的 origin/resources |
| onGeolocationPermissionsShowPrompt | 接收 origin, 返回 boolean/{allow,retain?}/null |
| onGeolocationPermissionsHidePrompt | 通知定位请求已隐藏 |
| onShowFileChooser | 接收 acceptTypes/multiple/mode/captureEnabled/title/filenameHint/timeoutMs, 返回 URI 数组/null |
| onCreateWindow | 接收 url/dialog/userGesture, 返回空白 WebView 句柄/null |
| onCloseWindow, onRequestFocus | 通知网页请求关闭/聚焦 |
| onShowCustomView, onHideCustomView | 通知全屏开始/结束, 原生 View 由宿主管理 |
| getVisitedHistory | 返回已访问 URL 数组/null |

setDownloadListener 对应 Android 的 onDownloadStart, 仅接收通知数据.

Android 必须同步决定导航, 资源和按键处理. 这些决策通过 setWebViewClient 的 navigationRules, resourceRules 和 blockedKeyCodes 预先提交. 规则按顺序匹配 url/urlPrefix, 可进一步限定 method/mainFrame, 首条命中生效. 导航 action 为 allow/block; 资源 response 可提供 mimeType, encoding, statusCode, reasonPhrase, headers, body 或 bodyBase64. shouldOverrideUrlLoading/shouldInterceptRequest/shouldOverrideKeyEvent 只通知实际决策, 不能通过回调返回值改变它. onConsoleMessage 同样只通知, 使用 consumeConsoleMessages 选项控制原生返回值. 桥接不会为等待 JavaScript 而取消导航再调用 loadUrl.

onJsAlert/onJsConfirm/onJsBeforeUnload 返回 boolean, onJsPrompt 返回 string 或 null. 未配置处理器时, 已显示的 WebView 使用原生对话框. HTTP 认证返回 {username,password} 或 null; SSL 默认取消, 只有显式 'proceed' 才继续. 客户端证书可返回已授权 KeyChain 的 {alias}; Web 权限只允许请求资源的子集, 并仍须拥有 Android 摄像头/麦克风/定位权限. onCreateWindow 可返回新的空白执行内 WebView 或 null; 关闭借用窗口后再 dispose. 全屏可通过 fullscreen:false 禁用.

默认 HTML 文件输入使用系统文档选择器. chooseFiles 的 acceptTypes 是 MIME/扩展名数组, multiple 控制多选, initialUri 是 Android 8+ 的可选初始目录提示. onShowFileChooser 可 await this.chooseFiles(event), 或返回已有读取授权的 content URI 数组/null. 仅接受其他提供器的可读 content URI, 不接受任意路径或本应用任意私有提供器 URI, 下文专门登记的临时媒体捕获结果除外. 新获得的持久读取授权在 View 释放时回收, 已有用户授权保留. 自定义回调提供的临时授权按 Android 生命周期生效. 宿主 5565 支持 open/multiple; 5566 扩展拍照, 保存和目录选择, 详见下一节. captureEnabled 等信息传给自定义回调.

下载通知包括 url, userAgent, contentDisposition, mimeType, contentLength 和 Android 推断的 suggestedFileName. HTTP(S) 默认下载由 DownloadManager 保存到公共 Downloads; fileName 只能是文件名. 可配置 headers, mimeType, title, userAgent, includeCookies (默认 false). 任务在 View 关闭后保留, 通过 getDownload/readDownload/cancelDownload 管理. readDownload 的 offset 默认 0, length 为 1..32768. 宿主 5566 扩展 blob:/data: 及所选文档来源和目标, 详见下一节.

```js
let ui = require('ui'); // Bun 使用 host.ui.
(async () => {
    let web = await ui.createView('WebView', { javaScriptEnabled: true });
    let window = await ui.showLayout(web);
    await web.setWebViewClient({
        onPageFinished(e) { console.log(e.url); },
        shouldOverrideUrlLoading(e) { console.log(e.url, e.blocked); }
    }, { navigationRules: [{ urlPrefix: 'https://example.invalid/blocked', action: 'block' }] });
    await web.setWebChromeClient({
        onJsConfirm(e) { return e.message === 'Continue?'; },
        onShowFileChooser(e) { return this.chooseFiles(e); }
    });
    await web.loadHtml('<title>Files</title><input type="file" multiple>');
    // 完成后 await window.close(); await web.dispose().
})().catch(console.error);
```

窗口/会话关闭会取消待处理的原生 UI 请求, View 释放还会回收文件授权. Worker 退出清理自己注册的客户端和任务, 不覆盖其他线程的新注册; 配套宿主 build 5569 及客户端为共享句柄提供独立事件订阅. 原始 Java View/Bitmap 返回值, 已废弃数据库/配额回调及任意 Java 子类不属于本契约. 原生语义可参阅 Android 的 [WebViewClient](https://developer.android.com/reference/android/webkit/WebViewClient) 和 [WebChromeClient](https://developer.android.com/reference/android/webkit/WebChromeClient).

### WebView 拍照, 保存/目录选择与 blob/data 下载

以下能力需要宿主 5566, Node.js Runtime 1.12.0 或 Bun Runtime 0.2.11, 包括 Node CJS/ESM/嵌套 Worker.

#### view.chooseFiles(options?) 的扩展选项

**`Async`**

- **[ options.mode ]** { [string](dataTypes#string) | [number](dataTypes#number) } - open/0, multiple/1, directory/2, save/3. 未指定时根据 multiple 选择 open 或 multiple.
- **[ options.captureEnabled = false ]** { [boolean](dataTypes#boolean) } - image 接受类型优先调用系统相机拍摄完整图片. 宿主 5567 起扩展 video/audio 捕获, 详见下一节.
- **[ options.fileName ]** { [string](dataTypes#string) } - 保存模式的建议文件名, 其次使用原生 filenameHint, 默认 download.
- **[ options.writable ]** { [boolean](dataTypes#boolean) } - 目录默认请求读写权限, 可显式设为 false; 保存模式始终请求读写权限.
- **[ options.permissionMode ]** { [number](dataTypes#number) } - Android 37+ 原生回调提供的 0 (读取) 或 1 (读写), 未显式指定 writable 时采用此值.
- <ins>**returns**</ins> { [Promise](dataTypes#promise)<[string](dataTypes#string)[] | [null](dataTypes#null)> } - 所选文档/目录树 URI, 拍照 URI, 或取消结果.

save 使用系统创建文档流程, 同名文件由提供器处理为新的文件, 不覆盖原文件. directory 返回一个目录树 URI. initialUri 只是初始位置提示, 系统或提供器可以忽略它; Android 对存储根目录, Download 根目录和 Android/data/obb 的限制仍适用. 网页是否发出原生保存/目录请求取决于 WebView 版本, 显式 chooseFiles 不依赖该网页能力.

拍照使用系统相机和专用临时 JPEG 输出 URI, 不取缩略图. 如果应用已声明 CAMERA 权限却尚未获得授权, 会请求系统授权; 拒绝或取消返回 null. 拍照文件在所属 View 和使用它的下载任务释放后删除, 不自动发布到相册. 除此类登记过的拍照结果, 本应用其他私有提供器 URI 仍不接受. 自定义处理器可继续使用 `onShowFileChooser(e) { return this.chooseFiles(e); }`.

#### view.readDocument(uri, offset?, length?)

**`Async`**

- **uri** { [string](dataTypes#string) } - 有效读取授权的 content URI, 或尚未释放的拍照结果.
- **[ offset = 0 ]** { [number](dataTypes#number) } - 非负字节偏移.
- **[ length = 32768 ]** { [number](dataTypes#number) } - 本次最多读取的字节数, 1..32768.
- <ins>**returns**</ins> { [Promise](dataTypes#promise)<Uint8Array> } - 实际读取的字节. 继续读取时增加实际返回长度, 返回空数组表示文件结尾.

#### view.download(options) 的扩展来源和目标

**`Async`**

- **options.url** { [string](dataTypes#string) } - HTTP(S), data:, 当前页面可读取的 blob:, 或有读取授权的 content: URI.
- **[ options.destinationUri ]** { [string](dataTypes#string) } - 可写的空文档 URI, 通常来自 save 模式.
- **[ options.directoryUri ]** { [string](dataTypes#string) } - 可写目录树 URI, 在其中用 fileName 创建子文档. 与 destinationUri 互斥.
- **[ options.timeoutMs = 120000 ]** { [number](dataTypes#number) } - 页面/本地/SAF 传输的总时限, 100..2147483647 毫秒; 不改变 DownloadManager 的超时策略.
- <ins>**returns**</ins> { [Promise](dataTypes#promise) } - 下载任务 {id, url, fileName}.

任务 id 现在可能是 DownloadManager 的数字或以 web: 开头的不透明字符串. 将收到的 id 原样传给 getDownload/readDownload/cancelDownload, 不自行计算. 本地传输失败时状态额外包含 error, reason 保持数字; managed=false 表示已完成的所选 SAF 输出由用户保留.

blob 由来源页面按最多 32 KiB 分块读取, 有背压, 不注入 Java 接口. 需要启用 JavaScript 并保留来源页面; 导航或关闭来源页面会取消未完成的 blob 读取. data URL 支持百分号编码的二进制内容和 base64, 字面加号不变为空格. 大型内联 data URL 仍受运行时请求大小限制, 大数据应使用页面 blob URL.

传输先写入宿主临时文件, 完成后发布到公共 Downloads 或所选位置. destinationUri 已有内容时拒绝且保持原内容. 取消未完成任务会清理临时/不完整输出; 取消已完成的默认下载会删除其受管文件, 已完成的显式 SAF 文件则保留. 新持久读写授权按 View/传输引用回收, 既有授权保持不变. SAF 文件在 View 释放后的读取仍须拥有有效 Android 授权.

HTTP(S) 默认目标继续使用 DownloadManager; 其他本地/页面/SAF 传输由宿主进程执行, 不支持进程终止后续传. 已完成文件不会因来源 View 关闭而删除. Worker 使用相同的参数, 任务标识和字节结果; Worker 退出会关闭其所属窗口, 从而取消依赖该页面的未完成 blob 读取.

```js
let ui = require('ui'); // Bun 使用 host.ui.
(async () => {
    let web = await ui.createView('WebView', { javaScriptEnabled: true });
    let window = await ui.showLayout(web);
    try {
        let files = await web.chooseFiles({ mode: 'save', fileName: 'report.txt', acceptTypes: ['text/plain'] });
        if (!files) return;
        let job = await web.download({ url: 'data:text/plain;charset=UTF-8,' + encodeURIComponent('Hello'), destinationUri: files[0] });
        while (true) {
            let state = await web.getDownload(job.id);
            if (state.status === 'failed') throw Error(state.error || state.reason);
            if (state.status === 'successful') break;
            await new Promise(resolve => setTimeout(resolve, 100));
        }
        console.log(new TextDecoder().decode(await web.readDocument(files[0])));
        await web.cancelDownload(job.id); // 清理任务记录, 已保存的 SAF 文档保留.
    } finally {
        await window.close();
        await web.dispose();
    }
})().catch(console.error);
```

参阅 Android [存储访问框架](https://developer.android.com/training/data-storage/shared/documents-files) 和 [FileChooserParams](https://developer.android.com/reference/android/webkit/WebChromeClient.FileChooserParams). 视频/音频捕获见下一节. 任意文件系统路径不属于文件选择接口.

### WebView 视频拍摄与录音捕获

宿主 5567 配套 Node.js Runtime 1.13.0/Bun Runtime 0.2.12, Node.js 主线程和 CJS/ESM/嵌套 Worker 使用相同接口.

#### view.chooseFiles(options?) 的媒体选项

**`Async`**

- **[ options.captureEnabled = false ]** { [boolean](dataTypes#boolean) } - 启用媒体捕获. acceptTypes 属于单一 image/video/audio MIME 家族时自动选择对应流程.
- **[ options.captureKind ]** { [string](dataTypes#string) } - image/video/audio, 在 captureEnabled 为 true 时明确选择捕获类型, 可用于混合或通配 acceptTypes. save/directory 模式仍使用 SAF.
- **[ options.videoQuality ]** { [number](dataTypes#number) } - 视频相机提示: 0 为低画质, 1 为高画质. 未指定时采用相机默认值.
- **[ options.durationLimitSeconds ]** { [number](dataTypes#number) } - 正整数秒, 1..2147483. 视频传给系统相机, 录音用于停止录制并等待确认.
- **[ options.sizeLimitBytes ]** { [number](dataTypes#number) } - 正安全整数字节数. 视频传给系统相机, 录音按原生编码器限制停止; 最后编码帧和容器开销可略超出该值.
- **[ options.timeoutMs = 120000 ]** { [number](dataTypes#number) } - 文件选择等待时限, 100..2147483647 毫秒. 到期取消选择并清理未确认的录制.
- <ins>**returns**</ins> { [Promise](dataTypes#promise)<[string](dataTypes#string)[] | [null](dataTypes#null)> } - 已确认的临时媒体 URI 数组, 或取消结果.

视频调用系统相机, 提供应用私有临时输出 URI. 画质/时长/大小均为相机提示, 各厂商处理方式可能不同. 相机若返回单独的可读 content URI, 宿主将内容复制为临时捕获结果; 不删除外部相机保存的原件. 本应用声明 CAMERA 而尚未获授权时, 先请求系统授权.

录音使用宿主前台界面, 提供开始, 停止, 使用录音和取消操作, 输出 AAC/M4A (audio/mp4, 48 kHz 单声道), acceptTypes 不用于选择其他音频编码. 不依赖系统是否安装录音 Intent 处理程序. 点击开始后才打开麦克风, 离开前台即停止录制, 返回后可确认已录内容; 旋转时保留同一录制并重建界面. 打包项目需要在 permissions 中声明 android.permission.RECORD_AUDIO, 并取得 Android 授权. Node 使用 ui 权限, Bun 使用 ui.layout 授权; 这是用户操作的文件捕获流程.

默认 HTML video/audio 文件输入和自定义 onShowFileChooser 均支持此能力. WebChromeClientOptions.callbackTimeoutMs 的范围为 100..2147483647 毫秒, 文件选择默认 120000, 其他异步回调默认 15000. 原生文件请求携带 timeoutMs, 可通过 this.chooseFiles(event) 转发. 调长文件选择等待时, 还应按需调整 Node/Bun 项目的总执行期限.

确认结果后, 可用 readDocument 分段读取, 或用 download 的 content 来源复制到 Downloads/已选 SAF 位置. 临时文件随 View 和正在使用它的传输引用释放. 取消, 页面跳转, 窗口关闭及 Worker 退出会关闭未完成的捕获; 录音不会转为隐藏的后台任务.

```js
let ui = require('ui'); // Bun 使用 host.ui.
(async () => {
    let web = await ui.createView('WebView', { javaScriptEnabled: true });
    let window = await ui.showLayout(web);
    try {
        let video = await web.chooseFiles({
            captureEnabled: true, captureKind: 'video',
            acceptTypes: ['video/*'], videoQuality: 1,
            durationLimitSeconds: 30, timeoutMs: 180000,
        });
        let audio = await web.chooseFiles({
            captureEnabled: true, captureKind: 'audio',
            acceptTypes: ['audio/*'], durationLimitSeconds: 15,
        });
        if (audio) console.log(await web.readDocument(audio[0], 0, 1024));
    } finally {
        await window.close();
        await web.dispose();
    }
})().catch(console.error);
```

参阅 Android [视频捕获 Intent](https://developer.android.com/reference/android/provider/MediaStore#ACTION_VIDEO_CAPTURE) 和 [MediaRecorder](https://developer.android.com/media/platform/mediarecorder). 这组文件捕获接口与网页 getUserMedia/MediaRecorder 流式 API 及屏幕录制接口分别使用各自的契约.

### 打包项目

宿主 5566 起, Bun 项目可在 project.json 设置 bun.timeoutMs, 默认 60000 毫秒, 按现有运行时范围限制为 1..600000 毫秒. 显式执行参数 bun.timeoutMillis 优先于项目设置. 打包应用采用同一设置; 拍照/文件选择和下载的等待时限不会延长整个项目的执行预算.

Node.js/Bun 项目 APK 使用已安装的外部 Runtime 插件和 3-Block Compose 插件, 自己的应用版本号可以从 1 开始. runtimeVersionCode 表示内置宿主运行库版本, applicationVersionCode 表示应用声明版本. APK Builder 需要匹配的 Runtime Kit (本批为 6.8.0+5567).

Node 项目通过 node.permissions 声明 ui. Bun 项目使用 projectType=bun 或 bun 入口标记, 并在 project.json 明确声明 bun.hostCapabilities, 例如:

WebView 网络加载和下载还需要打包项目的 Android INTERNET 权限. 在项目的 permissions 数组中保留 android.permission.INTERNET; Runtime 插件自身的联网权限不能替代打包应用的权限. HTTPS/明文流量仍遵循应用的 Android 网络安全配置.

Node.js/Bun 项目的执行总预算与 WebView 回调超时相互独立. 需要等待用户选择文件或操作界面时, 在 node 配置中显式设置足够的 timeoutMs, 例如 `"node": {"permissions": ["ui"], "timeoutMs": 120000}`. 默认项目预算可能先于 WebView 的等待期限耗尽; evaluateJavascript 的 timeoutMs 不会延长整个项目的执行预算.

```json
{
  "projectType": "bun",
  "main": "main.bun.js",
  "permissions": ["android.permission.INTERNET"],
  "bun": { "hostCapabilities": ["ui.layout", "ui.compose"], "timeoutMs": 180000 }
}
```

示例只列出运行时相关字段, 应合并到完整的项目配置. Bun 打包应用不读取 AutoJs6 应用的授权开关, 未列出的能力不提供. 能力名称必须有效且不重复. 本批没有新增打包配置界面; 通过 project.json 配置. SDK 模块应随项目打包, 可从配套插件源码的 app/src/main/assets/autojs6/host-client.mjs 复制并相对导入. 普通宿主仍使用原有受保护服务; 打包应用通过独立的普通权限服务调用外部 Bun, 执行及取消按调用方 UID 隔离. 运行时代码仍是插件 UID 下的可信代码, 不是不受信任脚本沙箱. Android 权限及插件自身的运行条件仍需满足.

## 声明与 Ace 智能提示

`AutoJs6-TypeScript-Declarations` 提供三个独立入口:

- `declarations/autojs6/index.d.ts`: Rhino.
- `declarations/autojs6-node/index.d.ts`: Node 插件的公开宿主模块.
- `declarations/autojs6-bun/index.d.ts`: Bun 显式客户端的类型别名; 原始模块声明为同目录 `host-client.d.mts`.

同一类型检查项目应选择实际执行环境的入口. Node 自身标准库需要对应 Node 类型包; Bun 原生 API 需要对应 Bun 类型包. 这两套宿主声明不表示完整原生运行时类型已被收录.

Ace 在默认设置下根据脚本标记, 文件名和项目类型加载独立声明. Node 的 `require` / ESM 导入以及 Bun 示例中的 JSDoc / TypeScript 类型注解可获得成员提示. Node/Bun 不再混入 Rhino 的 `selector`, `ui.layout` 或 Java 全局; 已启用的 Android 声明选项也不会改变该边界.

Rhino 的 `jsox.extendAll()` 可按既有规则启用原型扩展提示. Node 使用 `require('jsox.mathx')` 等模块成员, 向显式目标安装扩展不等于修改全局 `Math` / `Array` / `Number`. Bun 客户端没有 JSOX 扩展入口.

### WebView getUserMedia / MediaRecorder 流式音视频

这组能力运行在系统 WebView 网页内. Node.js/Bun/Worker 通过 `evaluateJavascriptAsync` 控制网页的原生 `navigator.mediaDevices.getUserMedia` 和 `MediaRecorder`, 并通过已有的 `download` / `readDownload` 接收录制分片或最终文件. Node.js/Bun 全局没有因此增加 DOM 或 MediaRecorder 构造器.

前提是网页处于安全上下文 (可信 HTTPS 或本机 loopback), javaScriptEnabled 已启用, 系统 WebView 支持相应接口/编解码器, 宿主应用或打包应用已声明 CAMERA/RECORD_AUDIO. 打包应用需在项目权限中显式选择这两项. Chromium 麦克风采集还需要 MODIFY_AUDIO_SETTINGS 普通权限; APK Builder 1.2.6 起在选择 RECORD_AUDIO 时自动保留此配套权限, 使用其他打包器时需自行声明. 仅麦克风流不需要 CAMERA. MODIFY_AUDIO_SETTINGS 不替代 RECORD_AUDIO 的系统授权. Web 权限与 Android 权限分别裁决; 不安装信任证书, 不绕过 TLS 错误或安全上下文检查.

`onPermissionRequest` 返回数组时沿用只使用已有 Android 权限的行为. 返回 `{ resources, requestAndroidPermissions: true }` 时, 会为所选音频/视频资源申请缺少的 Android 权限. resources 必须是本次请求的子集; 请先核对 origin, 再明确选择需要的资源. 空数组/null, 未注册回调, 系统拒绝或清单缺少权限均不能取得流. 系统授权默认等待 120000 毫秒, 可通过 Chrome client 的 callbackTimeoutMs 调整; 导航, View 释放和客户端替换会取消尚未完成的授权. 已取得的流需调用 track.stop() 结束, 更换回调不撤回已经授予网页的权限.

下例假设 web 已加载自己的可信网页并收到 pageFinished. Node.js 使用 `require('ui')`, Bun 使用 `host.ui`; Worker 使用同一 ui 接口. `trustedOrigin` 必须来自脚本的可信配置, 不能无条件接受任意网页报告的 origin.

```js
let trustedOrigin = 'https://your-media-site.example';
await web.setWebChromeClient({
    onPermissionRequest(event) {
        if (new URL(event.origin).origin !== trustedOrigin) return null;
        return {
            resources: event.resources.filter(resource =>
                resource === 'android.webkit.resource.AUDIO_CAPTURE' ||
                resource === 'android.webkit.resource.VIDEO_CAPTURE'),
            requestAndroidPermissions: true,
        };
    },
}, { callbackTimeoutMs: 120000 });

try {
    await web.evaluateJavascriptAsync(`(async () => {
        if (!isSecureContext || !navigator.mediaDevices || !window.MediaRecorder) {
            throw new Error('Web media is unavailable');
        }
        window.stream = await navigator.mediaDevices.getUserMedia({ audio: true, video: true });
        window.parts = [];
        window.recorder = new MediaRecorder(stream);
        recorder.ondataavailable = event => { if (event.data.size) parts.push(event.data); };
        window.stopped = new Promise((resolve, reject) => {
            recorder.onstop = resolve;
            recorder.onerror = event => reject(event.error);
        });
        recorder.start(1000);
        return true;
    })()`, { timeoutMs: 120000 });

    // 录制过程中取得一个分片, 可循环读取更多分片.
    let part = await web.evaluateJavascriptAsync(`new Promise(resolve => {
        let timer = setInterval(() => {
            if (parts.length) {
                clearInterval(timer);
                resolve({ url: URL.createObjectURL(parts[0]), size: parts[0].size });
            }
        }, 50);
    })`);
    let job = await web.download({ url: part.url, fileName: 'part.webm' });
    try {
        while (true) {
            let status = await web.getDownload(job.id);
            if (status.status === 'successful') break;
            if (status.status === 'failed') throw new Error(status.error || String(status.reason));
            await new Promise(resolve => setTimeout(resolve, 50));
        }
        let bytes = await web.readDownload(job.id, 0, 32768);
        console.log(bytes.length); // Uint8Array; 大分片按 offset 继续读取.
    } finally {
        await web.cancelDownload(job.id);
        await web.evaluateJavascript('URL.revokeObjectURL(' + JSON.stringify(part.url) + ')');
    }

    await web.evaluateJavascriptAsync('new Promise(r => { recorder.onpause = () => r(true); recorder.pause(); })');
    await web.evaluateJavascriptAsync('new Promise(r => { recorder.onresume = () => r(true); recorder.resume(); })');
    let recorded = await web.evaluateJavascriptAsync(`(async () => {
        recorder.requestData();
        recorder.stop();
        await stopped;
        stream.getTracks().forEach(track => track.stop());
        let blob = new Blob(parts, { type: recorder.mimeType });
        return { url: URL.createObjectURL(blob), size: blob.size, mimeType: blob.type };
    })()`);
    // 使用 download({url: recorded.url, destinationUri}) 输出到已获授权的保存位置.
    console.log(recorded);
} finally {
    await web.evaluateJavascript(`
        if (window.stream) stream.getTracks().forEach(track => track.stop());
        if (window.recorder && recorder.state !== 'inactive') recorder.stop();
    `).catch(() => {});
}
```

`timeslice` 是目标分片间隔, 不是精确计时保证. `requestData()` 产生分片, `pause()`/`resume()` 控制录制, 最终 `stop` 事件发生在最后的 dataavailable 之后. 多个分片可能只有合并后才能独立播放, 应按产生顺序拼接并保留实际 mimeType; 扩展名应匹配该类型, 不应假设系统一定输出 MP4. 需要固定格式时先使用 `MediaRecorder.isTypeSupported()` 检查, 再传入 mimeType. 这些行为由 [MediaStream Recording 标准](https://www.w3.org/TR/mediastream-recording/) 和设备 WebView 实现决定.

持续录制应及时读取/上传分片并释放不再需要的 Blob 引用和 object URL. 示例为便于最终合并而保留 parts, 会随录制时长占用内存. 编码后的分片走 blob 下载, 每次 readDownload 最多 32768 字节, 不把整段录音/视频塞入一次 JSON 结果. 下载仍需在源网页有效期间完成. 页面跳转/销毁结束其媒体流及未完成的页面下载; 已完成输出沿用下载接口的保留规则. 具体设备的编解码器和远端 WebRTC 通话兼容性需另行验证. 屏幕共享与后台录制使用下一节的显式适配/原生接口.


### WebView 屏幕共享与后台录制

这组接口需要宿主 build 5569, Node.js Runtime 1.15.0 或 Bun Runtime 0.2.14. Node.js 主线程, CJS/ESM/嵌套 Worker 和 Bun 使用相同的 WebView 句柄方法.

#### view.enableDisplayCapture(options)

**`Async`**

- **options.origin** { [string](dataTypes#string) } - 当前可信网页的 HTTP(S) origin, 不含路径或查询参数.
- **[ options.maxWidth = 1280 ]** { [number](dataTypes#number) } - 编码帧的最大宽度, 1..8192.
- **[ options.maxHeight = 720 ]** { [number](dataTypes#number) } - 编码帧的最大高度, 1..8192. 保持宽高比.
- **[ options.frameRate = 5 ]** { [number](dataTypes#number) } - 最大送帧率, 整数 1..60. 实际帧率取决于设备负载.
- **[ options.timeoutMs = 120000 ]** { [number](dataTypes#number) } - 每次 Android 屏幕授权的等待时限, 100..120000 毫秒.
- <ins>**returns**</ins> { [Promise](dataTypes#promise) } - `{ id, installed: true, source: 'android-media-projection', audio: false }`.

在当前页面没有原生 getDisplayMedia 时, 显式安装兼容适配器到 `navigator.mediaDevices.getDisplayMedia`. 已有原生实现不会被替换. 页面必须处于安全上下文, origin 必须匹配; 每次调用都需要可见页面的用户激活, 并重新显示 Android MediaProjection 授权. 脚本求值或定时器不能代替点击手势, 不缓存或复用上次授权.

适配器将 Android MediaProjection 的实时帧传入页面 canvas, 返回可用于 MediaRecorder 的真实视频 MediaStream. 它不是系统 WebView 原生的屏幕采集实现, 不包含系统音频轨道. `audio: true` 也只返回视频; 如需麦克风, 可另行获得明确授权的 getUserMedia 音频流并组合轨道. 支持 video 的 width/height/frameRate 数字或 ideal/max 提示, 不接受 video:false 或 min/exact 约束. 屏幕输出保持初始 canvas 尺寸, 旋转后的画面按比例放入该尺寸. 当前一次安装只持有一个屏幕会话, 再次申请前先停止全部轨道及其克隆.

```js
let registration = await web.enableDisplayCapture({
    origin: 'https://your-media-site.example',
    maxWidth: 1280, maxHeight: 720, frameRate: 5,
});
// 下列代码属于该网页. button 必须由用户实际点击.
await web.evaluateJavascript(`
    document.getElementById('share').onclick = async () => {
        window.shared = await navigator.mediaDevices.getDisplayMedia({ video: true });
        document.getElementById('preview').srcObject = shared;
        window.recording = new MediaRecorder(shared);
        recording.start(1000);
    };
`);
```

最后一个视频轨道及其克隆停止时释放 Android 采集. Android 撤销授权时结束轨道; 页面导航, 关闭窗口, 销毁 View 或结束执行会取消共享. Worker 退出清理其安装, 并以安装 id 避免删除其他线程后续替换的安装. 兼容层一次只处理一帧, 不累积待传输截图队列. 页面 CSP 的 img-src 需要允许 data: 图像, 以便解码本地传入的 JPEG 帧. 它不保证低延迟远程 WebRTC 通话或高帧率游戏采集.

#### view.disableDisplayCapture(id?)

**`Async`**

- **[ id ]** { [string](dataTypes#string) } - 可选安装 id. 指定旧 id 时不影响新的安装.
- <ins>**returns**</ins> { [Promise](dataTypes#promise) } - `boolean`, 是否移除了匹配的安装.

停止待处理授权和活动共享, 移除当前页面的适配器. 完成 MediaRecorder 的 stop/dataavailable 及输出保存后再调用; 页面资源关闭后不能重新读取尚未保存的 Blob.

#### 后台网页录音

WebChromeClient 的 `onPermissionRequest` 对象回复增加 `backgroundCapture: true`. 在前台取得 CAMERA/RECORD_AUDIO 授权后, 为所选资源启动带停止操作的 Android 前台服务, 然后将资源授予网页. 麦克风流可在应用退到后台后继续录制, 仍需显式停止轨道及释放服务.

```js
await web.setWebChromeClient({
    onPermissionRequest(event) {
        if (new URL(event.origin).origin !== trustedOrigin) return null;
        return {
            resources: event.resources.filter(value => value === 'android.webkit.resource.AUDIO_CAPTURE'),
            requestAndroidPermissions: true,
            backgroundCapture: true,
        };
    },
});
```

前台服务不会覆盖 Chromium 的页面可见性策略. 已验证的 WebView 111/130 在页面隐藏时仍会将 getUserMedia 摄像头轨道置为 muted; 需要后台摄像头录像时使用下述原生文件录制 API. 不修改 document.hidden, 不伪装页面可见. 后台启动相机/麦克风服务仍可能被 Android 拒绝, 应从可见界面发起. 应用退到后台不等于关闭其窗口; 关闭窗口或停止执行会释放其资源.

#### view.startBackgroundRecording(options?)

**`Async`**

- **[ options.facing = 'back' ]** { [string](dataTypes#string) } - 'back' 或 'front'.
- **[ options.cameraId ]** { [string](dataTypes#string) } - 原生 Android 相机 id, 优先于 facing.
- **[ options.width = 1280 ]** / **[ options.height = 720 ]** { [number](dataTypes#number) } - 尺寸提示, 1..8192; 选择设备支持的相近录制尺寸.
- **[ options.frameRate = 30 ]** { [number](dataTypes#number) } - 帧率提示, 1..60; 返回实际支持的帧率.
- **[ options.bitRate = 4000000 ]** { [number](dataTypes#number) } - 正整数视频码率.
- **[ options.audio = false ]** { [boolean](dataTypes#boolean) } - 是否同时录制麦克风 AAC 音频.
- **[ options.durationLimitSeconds ]** / **[ options.sizeLimitBytes ]** { [number](dataTypes#number) } - 沿用媒体捕获的时长/大小限制, 达到限制后保存结果.
- **[ options.timeoutMs = 120000 ]** { [number](dataTypes#number) } - 权限及相机启动时限, 100..120000 毫秒.
- <ins>**returns**</ins> { [Promise](dataTypes#promise) } - `{ id, state: 'recording', cameraId, width, height, frameRate, audio }`.

使用 Camera2/MediaRecorder 输出 H.264/MP4, 可选 AAC 48 kHz 单声道音频. 这条路径独立于网页 getUserMedia/MediaRecorder 对象. 从可见 WebView 发起, 必要时请求应用已声明的 CAMERA/RECORD_AUDIO, 并启动相机/麦克风前台服务. 每个 View 同时持有一个录像, 活动期间可退到后台. 相机被其他应用占用, 权限拒绝或原生编码器失败时拒绝 Promise, 不返回空文件冒充成功.

#### view.getBackgroundRecording() / view.stopBackgroundRecording() / view.cancelBackgroundRecording(id?)

**`Async`**

- getBackgroundRecording 返回 idle/starting/recording/stopped/cancelled/failed 状态.
- stopBackgroundRecording 停止并保存结果, 返回 `{ id, state: 'stopped', reason, uri, mimeType: 'video/mp4', size, durationMs, cameraId, width, height, frameRate, audio }`. 达到限制或点击前台服务的停止操作也会保存可用结果.
- cancelBackgroundRecording 丢弃正在录制的临时文件并释放相机, 返回是否取消了活动录像. 可选 id 用于避免取消后来替换的录像.

完成后的 uri 是当前 View 持有的临时 content URI. 使用 readDocument 分段读取, 或用 download 复制到持久位置, 然后再销毁 View. 页面导航不影响独立的原生录像; 关闭所属窗口, 销毁 View 或结束执行会取消尚未保存的录像. Node Worker 退出时会取消其启动的原生录像, 包括借用父线程 View 的情况, 不取消其他线程后来启动的替代录像.

```js
let active = await web.startBackgroundRecording({ audio: true, facing: 'back' });
try {
    await new Promise(resolve => setTimeout(resolve, 10000));
    let movie = await web.stopBackgroundRecording();
    let job = await web.download({ url: movie.uri, fileName: 'recording.mp4' });
    // 等待 getDownload(job.id) 成功后再释放 View.
    console.log(active.id, movie.durationMs, job.id);
} finally {
    await web.cancelBackgroundRecording(active.id);
}
```

#### view.stopMediaCapture()

**`Async`**

- <ins>**returns**</ins> { [Promise](dataTypes#promise) } - `true`.

取消活动媒体, 原生未保存录像及屏幕共享, 释放前台服务, 并导航到 about:blank. 完成并保存网页 MediaRecorder 的结果或原生 stopBackgroundRecording 结果后再释放; 此方法本身不负责合并网页录制分片.

#### 打包权限与平台边界

屏幕共享项目需声明 FOREGROUND_SERVICE_MEDIA_PROJECTION; APK Builder 1.2.7 会保留其配套 FOREGROUND_SERVICE. 选择 CAMERA/RECORD_AUDIO 时, Builder 同时保留对应 FOREGROUND_SERVICE_CAMERA/FOREGROUND_SERVICE_MICROPHONE; RECORD_AUDIO 仍会保留 MODIFY_AUDIO_SETTINGS. 这些普通权限不会替代 Android 的相机/麦克风授权或每次屏幕共享授权.

本批后台行为指应用退到其他应用之后继续运行的受保护前台服务. 不承诺开机启动, 系统强停后恢复, 绕过屏幕锁定时的撤销, 厂商省电限制或无限时录制. Android 和 WebView 的限制仍然适用. 参阅 [Android 前台服务类型](https://developer.android.com/develop/background-work/services/fgs/service-types) 与 [Screen Capture 标准](https://www.w3.org/TR/screen-capture/).
