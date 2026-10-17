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

Bun 不自动获得 Node 的全部模块, 无障碍选择器, Java 互操作或 `rhino.run`. 截图需要独立能力开关及 Android 同意. SDK 的图像句柄也不是 Rhino 图片对象, 不自动注入 `cX`, `pinyin`, `opencc` 或 Rhino 的对象图.

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

会话提供 `root`, `isClosed`, `update(callback)`, `post`, `on/off`, `render(root)`, `flush`, `setTheme` 和 `close`. `render`, `flush`, `setTheme`, `close` 返回 Promise; `flush` 等待当前排队的状态更新被宿主接受. `ref.current` 在帧接受后设置, 移除节点或关闭窗口时清空. 组件提供 `get/set`, `append/insert/remove/replace/clear/slot`, `on/off` 和异步 `focus/blur/scrollTo/edit`. 读写组件属性使用 get/set; 原生对象的属性代理不属于此契约.

会话 `showSnackbar(message, options?)` 返回 Promise, 要求树中有使用默认 snackbarHost 的 Scaffold. options 支持 actionLabel, duration (short/long/indefinite), onAction 和 onDismiss, 每条消息至多触发一次终态回调. 悬浮会话还提供异步 `setPosition`, `setSize`, `getX/getY/getWidth/getHeight`, `requestFocus`, `disableFocus`, `setTouchable`; `session` 指向同一会话. 窗口旋转重建会保留组件状态, 已输入文本及事件路由.

该覆盖是可传输数据 API. Java View/Context 对象, 任意 Java 类构造器, 原生对象属性代理和原生悬浮窗调整框仍未纳入, 不应按同名模块推断这些入口可用. 未定义入口不会被模拟为成功. Node/Bun 仍不注入 Rhino 全局 `ui` / `compose`.

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

View 属于本次脚本执行, 可在 Node 主线程及 Worker 间传递标识, 但事件队列只允许一个读取者. AndroidView 借用 View; 关闭会话后 View 仍可复用. 有父节点或已被其他会话借用的 View 会被拒绝, 不能将包含容器的 View 再借给该容器. 只有 inflate 的根句柄能 dispose, 子句柄是别名. dispose 前须关闭借用会话; 释放容器会关闭其附着会话, 包括尚未显示的容器. attach 在同一容器成功接入新会话后关闭旧会话; 若要复用旧会话借用的同一个 View, 应先关闭旧会话. 旋转后重新连接生命周期, 保留 View 身份和状态.

Worker 退出只关闭它创建的窗口和附着会话; 已传回的 View 属于整个执行, 仍可由主线程使用和释放. 未交付的异步 View 创建结果会在 Worker 退出后释放. ui.showLayout(XML) 创建的根由窗口管理; ui.showLayout(handle) 借用已有根, 关闭窗口后由调用方 dispose. 整次脚本结束释放全部 View.

### 打包项目

Node.js/Bun 项目 APK 使用已安装的外部 Runtime 插件和 3-Block Compose 插件, 自己的应用版本号可以从 1 开始. runtimeVersionCode 表示内置宿主运行库版本, applicationVersionCode 表示应用声明版本. APK Builder 需要匹配的 Runtime Kit (本批为 6.8.0+5562).

Node 项目通过 node.permissions 声明 ui. Bun 项目使用 projectType=bun 或 bun 入口标记, 并在 project.json 明确声明 bun.hostCapabilities, 例如:

```json
{
  "projectType": "bun",
  "main": "main.bun.js",
  "bun": { "hostCapabilities": ["ui.layout", "ui.compose"] }
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
