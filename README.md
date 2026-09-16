[English](README.en.md) | **简体中文**

# play-console-assets

Claude Code skill：品牌icon/feature graphic/截图改版后，级联同步到App
内嵌logo、静态页favicon、Google Play Console的Default store listing图形
资源。通用skill，不绑定任何具体项目目录。内容默认简体中文；代码、CLI
指令、文件路径、技术术语保留原文。

## 安装

```
/plugin marketplace add mosjin/play-console-assets-skill
/plugin install assetpub@play-console-assets
```

实际调用用短名`assetpub`（如`/assetpub:version`、`/assetpub:upgrade`），
沿用`mosjin/android-voice-test-kit`的约定——描述性的marketplace/repo名下
配一个简短的插件命令名。

## 覆盖内容

- 品牌icon改版后要检查的完整消费点清单：平台launcher icon
  （`flutter_launcher_icons`）、App内嵌logo（二维码/水印）、静态法务页
  favicon（base64内嵌）、Play Console商店列表图形——不要只改了launcher
  icon就以为大功告成。
- 一条纯本地SVG→PNG光栅化管线（`gi.repository.Rsvg` + `cairo`），设计
  新icon不需要依赖任何外部图像生成API。
- **Play Console Assets编辑页的6个实测踩过的坑**，全部写进`assetpub`
  skill正文，包括：
  - 页面上有多个文字完全相同的"Add assets"链接（App icon/Feature
    graphic/Phone screenshots各一个），文本匹配必须限定作用域，否则会
    在错的字段里操作。
  - 多图字段（Phone screenshots）的文件选择器`<input type=file>`其实是
    单文件的，必须一张一张上传。
  - "上传成功/自动选中" ≠ "已加入字段"，还要再点一次"Add"，多图字段要
    每张单独走完整个"上传→Add"循环，不能攒到最后一次性多选提交。
  - "needs cropping"不可选中状态大概率是踩了作用域坑（在错误字段里
    操作），不是图片真的不满足宽高比——作者曾经在这个误判上反复折腾
    图片尺寸（2:1→16:9→1080×1920→1350×2400），全部无效，直到发现根本
    问题是操作错了字段。
  - 没点"Save as draft"之前，刷新页面会静默丢弃所有未保存的改动。
  - Save as draft之后，Play Console全局UI（右上角App切换器图标）不会
    立刻跟着换，要等Submit for review审核通过。
- 真机重新截图的坑：`adb shell input text`打不进中文（`NullPointerException`），
  推荐直接后端种数据代替UI文字输入。
- `upgrade`skill会直接对marketplace的git checkout跑`git pull`（用Bash
  执行，不用等你手动打指令）。cache快照刷新用`/plugin`菜单选"Update"
  即可，**但紧接着必须再跑一次`/reload-plugins`**——Update会更新磁盘上
  的registry，但不会自动同步进当前已经在跑的session，漏了这一步会误以
  为升级没生效。

## 相关

- [`play-console-app-publish`](https://github.com/mosjin/play-console-app-publish-skill)（playpub）——
  Chrome CDP连接方式、App创建、Internal testing上传、订阅商品创建。本
  skill专门补上playpub没覆盖的"商店列表图形资源"这一段，两者配合使用。
- [`flutter-release-aab-build`](https://github.com/mosjin/flutter-release-aab-build-skill)（aabbuild）——
  打包release AAB，同一条发布流水线上的另一环节。
- [`android-real-device-driver`](https://github.com/mosjin/android-real-device-driver-skill)（adbdrive）——
  真机adb截图/交互的通用坑，重新截图那一步引用。
