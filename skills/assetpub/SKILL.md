# assetpub

品牌icon/feature graphic/截图改版后，级联同步到所有消费它的地方——App
自己的launcher icon、App内嵌的水印/二维码logo、静态法务页favicon、Google
Play Console的Default store listing（icon、feature graphic、手机截图）。
通用流程，不绑定任何具体项目/框架，但示例代码假设一个典型Flutter+Python
后端项目布局。

## 何时用这个skill

- 重新设计了App的品牌icon（新logo、新配色、新视觉方向），需要把这份改动
  同步到所有引用旧icon的地方，而不是只改了launcher icon就以为完事了。
- 需要把新的商店列表图形（icon 512×512、feature graphic 1024×500、手机
  截图）传到Google Play Console的Default store listing。
- 应用UI大改版之后，商店截图还是老版本UI，需要重新截图并上传。

## 第一步：枚举所有消费旧icon的地方，不要只改launcher icon

一个品牌icon改版，实际引用点往往比想象的多。改之前先搜一遍：

```bash
grep -rln "app_icon\|icon_512\|favicon\|feature_graphic" \
  --include="*.dart" --include="*.py" --include="*.yaml" \
  --include="*.html" --include="*.md" .
```

典型消费点（按项目实际情况增减）：

1. **平台launcher icon** —— Flutter项目用`flutter_launcher_icons`包
   （`pubspec.yaml`里配置`image_path`指向源图，跑
   `dart run flutter_launcher_icons`重新生成
   `android/app/src/main/res/mipmap-*/ic_launcher.png`各密度档位）。
2. **App内嵌logo** —— 比如邀请二维码中间嵌的品牌logo、分享卡片上的水印。
   如果这些地方跟launcher icon共享同一份源文件（`assets/icons/app_icon.png`
   这种），改一次就够；如果是独立文件，要单独找出来替换。
3. **静态页favicon** —— 如果法务页/落地页的favicon是内嵌base64（不额外起
   服务器路由的常见做法），源头也是这份品牌icon，需要重新生成不同尺寸
   （32×32、180×180 apple-touch-icon等）并重新嵌入：

   ```python
   import base64, io
   from PIL import Image

   src = Image.open("path/to/new_app_icon.png").convert("RGB")

   def b64_png(im, size):
       r = im.resize((size, size), Image.LANCZOS)
       buf = io.BytesIO()
       r.save(buf, format="PNG")
       return base64.b64encode(buf.getvalue()).decode("ascii")

   icon32 = b64_png(src, 32)
   touch180 = b64_png(src, 180)
   # 用正则把 <link rel="icon" ... href="data:image/png;base64,...">
   # 和 <link rel="apple-touch-icon" ...> 里的旧base64替换成新的
   ```

   **部署陷阱**：如果这些静态页是从`docs/`之类的仓库目录部署到生产服务器
   的，先确认生产环境Web服务器（Caddy/Nginx）实际serve的目录是不是就是
   git仓库checkout本身——很可能有一层`manage_deploy.py`之类的脚本把文件
   COPY到了另一个真正serve的目录（比如`/var/www/xxx-legal/`）。改完仓库
   文件、`scp`过去之后，**用`md5sum`核对本地文件和`/var/www/`里实际文件
   是否字节一致**，不要只核对仓库checkout目录——这两者可能是两份独立的
   拷贝。同时留意CDN/边缘代理（Cloudflare等）可能缓存旧内容，用
   `curl --resolve <domain>:443:<origin_ip>`绕过CDN直接查源站验证（注意
   源站防火墙可能只放行CDN节点IP，直连源站IP可能被拒；也可能因为没带
   真实浏览器User-Agent被WAF拦成403，加个`User-Agent: Mozilla/5.0 ...`
   请求头再试）。

4. **Google Play Console商店列表图形** —— 见下面专门一节，坑最多。

## 第二步：设计新icon（可选，纯本地SVG管线，不需要外部生图API）

如果新icon要重新设计（不只是替换现成文件），可以用SVG+光栅化的本地管线，
不依赖任何外部图像生成服务：

```bash
# 检查本机是否已有 librsvg 的 Python 绑定（gi.repository.Rsvg）
/usr/bin/python3 -c "
import gi
gi.require_version('Rsvg', '2.0')
from gi.repository import Rsvg
print('ok')
"
```

如果可用，用`cairo`把手写的SVG渲染成PNG：

```python
import gi
gi.require_version("Rsvg", "2.0")
from gi.repository import Rsvg
import cairo

handle = Rsvg.Handle.new_from_file("icon.svg")
size = 1024
surface = cairo.ImageSurface(cairo.FORMAT_ARGB32, size, size)
ctx = cairo.Context(surface)
handle.render_cairo(ctx)
surface.write_to_png("icon_1024.png")
```

设计上给用户看效果时，**同时导出小尺寸（如96px）和圆形裁切预览**——很多
反馈（"看起来像灯泡不像笔头""背景不干净"）只有在真实尺寸/真实裁切形状下
才会暴露，1024px大图上看着没问题的设计缩小到48px可能完全不可读。

SVG里的注释不能包含连续两个连字符`--`（`<!-- a -- b -->`），rsvg会解析
报错`Double hyphen within comment`，写icon设计相关的说明性注释时留意。

## 第三步：Google Play Console商店列表图形上传

### 前置：连接到用户自己登录的Chrome

复用`play-console-app-publish`（playpub）skill里记录的CDP连接方式——
非默认`--user-data-dir`、用户自己跑`google-chrome --remote-debugging-port=9222`、
`agent-browser --cdp <port> open <url>`接管。不要重复发明这一段，直接
参考那个skill。

### 陷阱1：同一页面上有好几个文字完全相同的"Add assets"链接——不要盲目文本匹配

Default store listing的Assets编辑页里，App icon、Feature graphic、Phone
screenshots……每个图形字段下面都有一条**文字完全一样**的"Add assets"链接。
如果用`document.querySelectorAll("*")`找`textContent.trim() === "Add assets"`
再取第一个匹配，大概率抓到的是页面上第一个字段（比如App icon）的那个，
而不是你真正想操作的字段（比如Phone screenshots）。

**后果**：这个bug不会报错，只会让你在错的图形选择器里传错的文件——比如
想给"手机截图"传4张图，结果把App icon的裁剪对话框打开了，还傻乎乎地
在"App icon"这个1:1裁剪预设下把截图裁成了512×512正方形塞进App icon
文件库（不会真的覆盖已保存的App icon，但会让排查过程充满误导）。

**正确做法**：先精确定位到目标字段独有的元素（比如"Remove Phone
screenshots"这个aria-label在整个页面上只属于Phone screenshots字段），
从它往上找共同祖先节点，再把"Add assets"的查询限定在这个祖先节点内部：

```javascript
const removeBtns = Array.from(document.querySelectorAll("button"))
  .filter(b => (b.getAttribute("aria-label")||"").includes("Remove Phone screenshots"));
let ancestor = removeBtns[0];
while (ancestor && !removeBtns.every(b => ancestor.contains(b))) {
  ancestor = ancestor.parentElement;
}
// 再往上走几层，直到这个祖先节点内部只有恰好1个"Add assets"匹配
let scope = ancestor, found = null;
for (let i = 0; i < 8 && scope; i++) {
  const matches = Array.from(scope.querySelectorAll("*"))
    .filter(e => e.textContent.trim() === "Add assets" && e.children.length === 0);
  if (matches.length === 1) { found = scope; break; }
  scope = scope.parentElement;
}
```

**每次DOM刷新后（比如点了一次"Add"提交完一张图之后），这个scope引用会
失效**——不要缓存一次就复用到底，每完成一张图的上传+Add流程，就重新
执行一次上面这段定位逻辑。

### 陷阱2：文件选择器`<input type=file>`是单文件的，即使字段本身允许多图

Phone screenshots字段允许2-8张图，但打开的原生文件选择器
（`<input type=file>`）**没有`multiple`属性**：

```javascript
document.querySelector("input[type=file]").multiple  // false
```

传一个文件数组给它只会生效第一个（或者干脆报错/静默失败，取决于工具
链）。**必须一张一张上传**：点"Add assets" → 点"Upload" → 传1个文件 →
（见陷阱3）→ 重复。

### 陷阱3："Save as copy"/上传成功 ≠ 已经加入这个字段——还要点一次"Add"

一张图上传到资源库（右侧的Asset picker面板）之后，它会**自动被勾选/
选中**（`check_circle`图标），但这时它只是"在库里、当前被选中"，还
**没有真正进入你正在编辑的那个字段的图片列表**。必须再点一次面板底部
的**"Add"按钮**（不是"Add assets"，是选中资源之后才出现的那个"Add"），
才会真正追加进字段。

对Phone screenshots这种多图字段，正确的单张循环是：

```
1. 定位scope（陷阱1的方法）
2. scope内点"Add assets"
3. 点"Upload"
4. 传1个文件（陷阱2）
5. 等上传完成（2-4秒，轮询/sleep，直到该资源不再是skeleton占位）
6. 点"Add"（把这张已选中的图追加进字段）
7. 回到步骤1，处理下一张
```

**已经上传过、但当前处于"未选中"状态的资源，点击它的缩略图/tile
不一定能重新选中它**——实测无论是`element.click()`还是完整的
`pointerdown/mousedown/pointerup/mouseup/click`事件序列，都无法可靠
把一个已经失去选中状态的资源重新选中。不要在这上面浪费时间调试；
可靠路径就是"每上传一张就立刻点一次Add"，不要指望上传完4张之后一次性
多选提交。

### 陷阱4："needs cropping" / `unselectable-icon`（灰色禁止图标）大概率不是宽高比问题

如果某张图在资源面板里显示为不可选中状态（`class="unselectable-icon"`，
一个灰色禁止符号图标，`aria-label`说"To select this image, crop it so
that it's the correct aspect ratio or dimensions..."），**先怀疑是不是
踩了陷阱1**（其实操作的是别的字段，比如App icon的1:1要求，而不是
Phone screenshots的16:9/9:16要求），而不是急着去改图片的实际像素宽高比。

本skill的作者在这个坑上折腾了大量时间：以为是截图宽高比不满足16:9
上限，依次尝试了pad到2:1、pad到精确16:9、crop到标准1080×1920、pad到
跟"成功案例"一模一样的1350×2400——每次都失败，直到发现所谓"成功案例"
其实是在错误的字段（App icon）里操作的，跟截图字段的真实宽高比要求根本
无关。**如果同一张图片、同样的裁剪操作，在换到正确字段后立刻就能选中，
说明之前纠结的宽高比从来都不是问题。**

真正需要检查宽高比的场景（先用一张**未经任何裁剪加工的原始截图**上传
测试，如果它在正确字段下直接可选，说明宽高比合规，不用再处理）：
- Phone screenshots: 16:9 或 9:16，每边320px~3840px。
- Feature graphic: 1024×500，精确尺寸，不是范围。
- App icon: 512×512，精确尺寸。

### 陷阱5：编辑内容在没点"Save as draft"之前，刷新页面就没了

删除旧截图、上传新截图、选择裁剪……这些操作在Assets编辑页里都是**未持久化
的本地状态**，只有点了底部的**"Save as draft"**（提交"Draft saved"提示）
才会真正保存到服务器。**页面刷新（包括`location.reload()`或手滑导航
到别的URL再回来）会静默丢弃所有未保存的改动**，不会有任何警告弹窗。

建议：完成一个字段（比如App icon+Feature graphic）就点一次"Save as
draft"，不要攒到最后一次性保存——即使只是想验证"刚才的改动生效了没"
而刷新页面调试，也要先保存。

### 陷阱6：Save as draft之后，Play Console页面顶部/全局的icon不会立刻更新

Store listing存成draft之后，Console里所有页面右上角的App切换器小图标、
以及App列表页的icon，仍然显示**旧的、已批准上线**的icon，不是刚存的
草稿。这是预期行为，不是bug——草稿要"Submit for review"、Google审核
通过之后，才会成为真正上线的版本，全局UI才会跟着换。不要因为顶部图标
没变就怀疑上传失败。

## 第四步：真机重新截图（如果UI也改版了）

如果这次品牌改版伴随着UI大改版，商店截图也需要重新拍。

**已知坑：`adb shell input text`打不进中文**——很多设备的默认输入法
（包括厂商自带的、以及部分第三方输入法如WeType）在通过`adb shell input
text`注入非ASCII字符时会抛`NullPointerException: Attempt to get length
of null array`，切换输入法也不一定能解决。可靠的替代方案（按优先级）：

1. **后端直接种数据**（推荐，如果App连的是本地/可控后端）：跳过UI输入，
   直接往数据库/API插入中文示例数据，然后只是"导航到展示页 + 截图"，
   不涉及任何文字注入。
2. `com.android.adbkeyboard/.AdbIME` + `am broadcast -a ADB_INPUT_TEXT`
   （需要提前在设备上装好这个IME apk）。
3. 纯ASCII内容可以正常用`adb shell input text`，只有CJK等非ASCII字符
   会触发这个bug。

截图分辨率如果是设备原生分辨率（比如很多现代手机是1080×2400，20:9），
天然满足Play的16:9/9:16要求（2400/1080=2.22，超过老式2:1认知但这其实
是陷阱4提到的"先怀疑操作错字段"场景——不要一遇到"needs cropping"就去
改像素）。

## 相关

- [`play-console-app-publish`](https://github.com/mosjin/play-console-app-publish-skill)（playpub）——
  Chrome CDP连接方式、App创建、Internal testing上传、订阅商品创建。这个
  skill（assetpub）专门补上playpub没覆盖的"商店列表图形资源"这一段。
- [`flutter-release-aab-build`](https://github.com/mosjin/flutter-release-aab-build-skill)（aabbuild）——
  打包release AAB，跟本skill是同一条发布流水线上的不同环节。
- [`android-real-device-driver`](https://github.com/mosjin/android-real-device-driver-skill)（adbdrive）——
  真机adb截图/交互的通用坑（坐标缩放、BACK键退出App等），第四步引用。
