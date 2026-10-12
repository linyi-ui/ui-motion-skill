# 动效设计基础原则

不管做哪个具体动效,这些底线都要满足。

## 时长分级

不要一刀切用 300ms,根据元素大小和层级选择:

| 元素类型 | 推荐时长 | 例子 |
|---|---|---|
| 小元素、状态反馈 | 150–250ms | 按钮点击反馈、图标旋转、checkbox 打勾 |
| 中等面板、卡片 | 300–450ms | 弹出菜单、卡片展开、抽屉展开 |
| 整页转场、大容器 | 450–600ms | 页面切换、全屏模态框 |
| 任何动效 | 不超过 700ms | 再长就是卡顿了 |

## 曲线选择

**禁止 linear**。直线运动像机器人,必须用缓动曲线。

| 场景 | 曲线 | 值 |
|---|---|---|
| 进场(淡入、滑入) | ease-out | `cubic-bezier(0, 0, 0.2, 1)` |
| 退场(淡出、滑出) | ease-in | `cubic-bezier(0.4, 0, 1, 1)` |
| 双向(hover、按压) | ease-in-out | `cubic-bezier(0.4, 0, 0.2, 1)` |
| 轻微回弹(点赞、切换) | 弹簧曲线 | `cubic-bezier(0.34, 1.56, 0.64, 1)` 或 `spring(1, 80, 10)` |

**回弹分寸**: 超调不超过 8%。弹到 1.08 可以,弹到 1.2 就过了。

## 性能规则

**只动 transform 和 opacity**,避免触发重排(reflow)。

| ✅ 推荐 | ❌ 避免 | 原因 |
|---|---|---|
| `transform: scale()` | `width`, `height` | 改尺寸会触发重排,卡顿 |
| `transform: translate()` | `top`, `left`, `margin` | 改位置会触发重排 |
| `opacity` | `visibility` 切换时渐变 | visibility 要么全显示要么全隐藏,不能渐变 |

**确实需要改尺寸时用 FLIP 技巧**:
1. First: 记录元素当前位置和大小
2. Last: 直接切到最终状态,记录新位置和大小
3. Invert: 用 transform 把元素变回起始位置和大小
4. Play: 让 transform 回到 0,看起来就是流畅地从起始变到最终

## 层次感

**一组元素依次出现**,间隔 30–60ms,不要一起蹦出来。

```css
/* ❌ 错误:菜单项一起出现 */
.menu-item { animation: fadeIn 200ms; }

/* ✅ 正确:依次淡入 */
.menu-item:nth-child(1) { animation: fadeIn 200ms 0ms; }
.menu-item:nth-child(2) { animation: fadeIn 200ms 40ms; }
.menu-item:nth-child(3) { animation: fadeIn 200ms 80ms; }
```

**或者用 JS 循环**:
```js
items.forEach((item, i) => {
  item.style.animationDelay = `${i * 40}ms`;
});
```

## 有来源、可逆

- **有来源**: 元素从哪里来、回哪里去要讲得通。按钮变成菜单,就是按钮本身在变形,不是凭空弹出新的一层。
- **可逆**: 打开怎么来,关闭就怎么回去,路径一致。

```js
// ✅ 正确:菜单从按钮位置展开,关闭时缩回按钮
fab.onclick = () => {
  menu.style.transformOrigin = 'bottom right';
  menu.animate([
    { transform: 'scale(0)', opacity: 0 },
    { transform: 'scale(1)', opacity: 1 }
  ], { duration: 350, easing: 'ease-out' });
};

close.onclick = () => {
  menu.animate([
    { transform: 'scale(1)', opacity: 1 },
    { transform: 'scale(0)', opacity: 0 }
  ], { duration: 250, easing: 'ease-in' }); // 退场稍快
};
```

## 反馈

可点击的元素要有 **按下态** 和 **hover 态**:

- **按下态**: 缩放到 0.96–0.98,按下去 60ms,松开回弹 150ms。
- **Hover 态**: 轻微放大到 1.02 或加阴影,200ms 过渡。

```css
.btn {
  transition: transform 200ms ease-out;
}
.btn:hover {
  transform: scale(1.02);
}
.btn:active {
  transform: scale(0.97);
  transition-duration: 60ms;
}
```

触屏上用 JS 加点击反馈:
```js
btn.addEventListener('touchstart', () => {
  btn.style.transform = 'scale(0.97)';
});
btn.addEventListener('touchend', () => {
  btn.style.transform = 'scale(1)';
});
```

## 无障碍

尊重系统设置,`prefers-reduced-motion: reduce` 时,动画改成 150ms 以内的淡入淡出:

```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.15s !important;
    transition-duration: 0.15s !important;
  }
}
```

或者直接关掉动画:
```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation: none !important;
    transition: none !important;
  }
}
```

## 页面切换与空间关系

切换不是"换一张图"，要让用户感觉页面在一个连续的空间里移动。

- **方向一致**: 点右边的 Tab，新内容从右边轻推进来；往左滑标签页，内容往左走。方向和操作反了，用户会迷路。
- **背后有反应**: 弹窗、抽屉出现时，背后页面缩小（约 0.92）、变圆角、压暗，像退后一步；关闭时同步恢复。背后一动不动，弹窗就像贴上去的。
- **指示器要流动**: 胶囊、下划线切换时，前沿先到、后沿慢半拍跟上（两个快慢不同的弹簧分别驱动左右边缘），途中自然拉长再收回，不要整块平移，更不要瞬移。
- **进度联动**: 可以跟手的切换（滑动标签、拖弹窗、滚动折叠标题），所有相关元素都按同一个进度值实时插值（位置、缩放、颜色、透明度），不要等到达后才突变。
- **不停在半路**: 折叠标题、弹窗、分页这类有明确"两个状态"的界面，松手后按速度和位置吸附到最近的状态。

```js
// 两个弹簧驱动指示器左右边缘，做出"水滴/毛毛虫"的拉伸感
const lead = spring({ stiffness: 420, damping: 26 });   // 前沿：快
const tail = spring({ stiffness: 190, damping: 20 });   // 后沿：慢半拍
function moveTo(i) {
  const goingRight = i > current;
  (goingRight ? rightEdge : leftEdge).animateWith(lead);
  setTimeout(() => (goingRight ? leftEdge : rightEdge).animateWith(tail), 60);
}
```

## 加载与等待

等待躲不掉，但可以让用户"看到东西在来"，而不是盯着一个转圈。

- **先占位再出场**: 结构已知时用骨架屏，形状、尺寸、位置和真实内容一致；图片框提前固定宽高比。加载前后布局一个像素都不跳。
- **内容分块出场**: 数据到了以后按块从上到下错开 40–60ms 浮现（淡入 + 小幅上移），不要一下子全部蹦出来。
- **能提前就提前**: 列表离底约一屏就预加载下一页，图片离进屏约 300px 才开始加载，用户正常操作基本感觉不到等待。
- **立刻有反馈**: 操作后 100ms 内必须有可见变化（气泡、占位、按钮状态），哪怕结果还没回来。
- **按时长选方式**: <300ms 不显示任何加载态，直接淡入结果；0.3–2s 用骨架屏或模糊占位；>2s 或进度可知时显示进度。
- **扫光克制**: 一页只有一道共用的光带，周期 1.2–1.8s，峰值透明度不超过 0.15；不要每块各自闪。

```css
/* 所有骨架共用一道光：按屏幕坐标铺渐变，而不是每块自己从 0 开始 */
.skeleton {
  background: #22262e linear-gradient(110deg, transparent 40%, rgba(255,255,255,.12) 50%, transparent 60%) 0 0 / 200vw 100% fixed;
  animation: shimmer 1.5s linear infinite;   /* 循环位移允许 linear，属于背景纹理不属于元素动效 */
}
@keyframes shimmer { from { background-position-x: 100vw; } to { background-position-x: -100vw; } }
```

## 表单输入反馈

输入框是用户"在说话"的地方，反馈要像一个耐心的人：等你说完再回应，说错了指给你看，说对了马上点头。

- **校验时机**: 用户还没输完时不报错。停止输入约 400ms（debounce）或离开输入框（blur）后再校验；已经报错的框，一改对就立刻变绿（晚报错、早夸奖）。密码强度这类"引导"可以每个字符实时更新，但不显示红色错误。
- **就近反馈**: 对勾、错误提示、强度条都紧贴在对应输入框旁边或下方，不要堆在页面顶部，更不要用 alert 弹窗。
- **三重提示**: 错误要有动作 + 颜色 + 文字（摇头 + 变红 + 一句说清错在哪的话），不能只靠变红。色弱用户分不清红绿，文字也要告诉用户怎么改。
- **布局不跳**: 标签浮起、对勾出现、错误提示出现都不能改变输入框高度，也不能把下面的表单往下推。提示文字的位置提前预留，或用 FLIP 平滑收放；边框变色用 `box-shadow` / `outline` 加粗，不改 border 宽度。
- **标签常在**: 用真正的 `<label>`，不要把 placeholder 当标签；输入时标签留在上方，用户随时知道这格在填什么。
- **键盘和读屏**: 出错的框加 `aria-invalid="true"`，用 `aria-describedby` 关联提示文字；动态变化的提示放在 `aria-live` 区域；评分、验证码这类自定义控件要能用键盘操作。
- **手机输入**: 用对 `type` / `inputmode` / `autocomplete`（邮箱 `type="email"`、验证码 `inputmode="numeric" autocomplete="one-time-code"`），输入文字至少 16px，避免 iOS 聚焦时自动放大页面。
- **尊重系统设置**: `prefers-reduced-motion: reduce` 时取消摇头、弹跳和波浪，只保留颜色变化和淡入淡出，信息一个都不能少。

```css
/* 浮动标签：只动 transform，输入框高度不变 */
.field label {
  transform-origin: left center;
  transition: transform 200ms cubic-bezier(0, 0, .2, 1), color 200ms;
}
.field input:focus + label,
.field input:not(:placeholder-shown) + label {   /* input 写 placeholder=" " 占位 */
  transform: translateY(-28px) scale(.8);
}

/* 输错摇头：幅度递减，只用 translateX；减少动效时只变红 */
@keyframes shake {
  0%, 100% { transform: translateX(0); }
  15% { transform: translateX(-10px); } 30% { transform: translateX(10px); }
  45% { transform: translateX(-6px); }  60% { transform: translateX(6px); }
  75% { transform: translateX(-3px); }  90% { transform: translateX(3px); }
}
.field.is-error { animation: shake 400ms cubic-bezier(.36, .07, .19, .97); }
@media (prefers-reduced-motion: reduce) {
  .field.is-error { animation: none; }
}
```

## 粒子动画

粒子是"仪式感"的放大器，用对了是惊喜，用多了就是廉价特效。核心是：粒子要有来源、有物理、有退场。

- **有来源**: 粒子从触发它的东西里来——彩纸从按钮喷出，沙粒就是被删的那条消息本身（取原来的颜色），Logo 粒子最后要拼回 Logo。不要凭空满屏飘。
- **有物理**: 至少有重力或阻力其中一种，让速度"先快后慢"；加一点随机（速度、角度、大小、寿命各自不同），但整体方向一致（向上喷、往右上吹、往中心聚）。完全随机 = 噪点，完全一致 = 机械。
- **各自退场**: 每个粒子寿命随机，到了各自淡出、变小；绝不能整批同时消失。
- **只在关键时刻**: 一个页面最多一处粒子效果，只给"完成 / 庆祝 / 删除 / 品牌亮相"这类时刻；日常点击用缩放和颜色反馈就够。
- **性能**: 所有粒子画在一个 canvas 上（不要一个粒子一个 DOM 元素）；同色粒子批量 `fill`；按时间差 dt 计算位置；手机上数量减半；页面不可见时（`document.hidden`）暂停。
- **不挡操作**: 粒子 canvas 设 `pointer-events: none`，动画期间界面照常可点。
- **尊重系统设置**: `prefers-reduced-motion: reduce` 时不放粒子，保留状态变化（按钮变"已完成"、消息淡出、Logo 直接显示）。

```js
// 粒子通用写法：一个 canvas + 按 dt 推进；阻力用 e^(-k·dt)，各自寿命
function step(p, dt) {
  const drag = Math.exp(-p.k * dt);       // 空气阻力：越落越慢
  p.vx *= drag; p.vy = p.vy * drag + G * dt;
  p.x += p.vx * dt; p.y += p.vy * dt;
  p.age += dt;
  p.alpha = 1 - Math.max(0, (p.age - p.life) / 0.5);  // 到寿命后 0.5s 淡出
}
```

## 滚动触发动效

滚动动效分两类，写法完全不同，先分清再动手：

- **进场类（播一次）**：依次浮现。元素露出约 10–15% 时才触发（IntersectionObserver 的 `threshold` 或 `rootMargin`），同一组依次晚 50–80ms，播完就 `unobserve`，往回滑不重播。首屏内容不要藏起来等滚动，直接显示。
- **进度类（跟手）**：视差、逐字点亮、钉住换图、进度条。动画进度 = 滚动位置的函数（0–1），手停画面就停、往回滑倒着走；绝不用定时器或 CSS transition "自己播完"。
- **平滑但不拖**：电脑滚轮一格 100px 会"一格一跳"，用平滑跟随（每帧追上剩余距离的 10–15%）；手机原生惯性已经很顺，不要再劫持触摸滚动。
- **钉住（sticky）**：优先用 `position: sticky`，外层容器的高度决定钉多久；钉住期间的进度 = 已滚距离 / (容器高 − 视口高)。
- **性能**：scroll 事件里只读 `scrollY`，在 `requestAnimationFrame` 里统一写 transform / opacity；不在滚动里改 width、top、font-size；视差层加 `will-change: transform`，层数控制在 5 层以内。
- **尊重系统设置**：`prefers-reduced-motion: reduce` 时取消视差和浮现，逐字点亮直接全亮，钉住换图改成普通上下排列。

```js
// 进度类通用写法：只读滚动位置，rAF 里写样式；平滑跟随 12%
let target = 0, cur = 0;
addEventListener('scroll', () => { target = scrollY; }, { passive: true });
(function tick() {
  cur += (target - cur) * 0.12;                    // 平滑跟随（电脑滚轮）
  const p = clamp((cur - start) / (end - start));  // 0–1 进度，完全由滚动决定
  layers.forEach(l => l.el.style.transform = `translateY(${-cur * l.speed}px)`);
  requestAnimationFrame(tick);
})();
```

## 参数集中管理

把动效参数写成 CSS 变量或 JS 常量,方便用户微调:

```css
:root {
  --dur-small: 200ms;
  --dur-panel: 400ms;
  --dur-page: 500ms;
  --ease-out: cubic-bezier(0, 0, 0.2, 1);
  --ease-in: cubic-bezier(0.4, 0, 1, 1);
  --spring: cubic-bezier(0.34, 1.56, 0.64, 1);
}

.menu { animation: fadeIn var(--dur-panel) var(--ease-out); }
```

或者 JS:
```js
const DUR = { small: 200, panel: 400, page: 500 };
const EASE = {
  out: 'cubic-bezier(0, 0, 0.2, 1)',
  in: 'cubic-bezier(0.4, 0, 1, 1)',
  spring: 'cubic-bezier(0.34, 1.56, 0.64, 1)'
};

menu.animate([...], {
  duration: DUR.panel,
  easing: EASE.out
});
```
