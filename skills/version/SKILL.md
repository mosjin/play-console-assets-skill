# assetpub version

大多数用户运行这个skill时从没clone过这个repo——只是在别的项目里
`/plugin install`过它。不要假设当前工作目录就是这个repo，也不要靠
搜索/猜测去找"那个"plugin目录。

**权威来源，无可替代：** 每次这个skill加载时，Claude Code都会在context
前面插入一行，说明这个skill自己的base目录，例如：

```
Base directory for this skill: ~/.claude/plugins/cache/play-console-assets/assetpub/1.0.0/skills/version
```

只有这一行——也只有这一行——说明了本次运行的session实际加载的是哪个
安装副本。plugin的cache目录会累积每个曾经安装过的版本各一个子目录
（Claude Code不会删除被替代的旧版本）。自己去列这个目录、然后挑一个
版本——不管是挑最高semver、最新mtime、字母序最后，还是任何别的启发式
方法——都可能悄悄报出一个过期的兄弟版本，而不是本session实际加载的
那个。不要这样做。

## 步骤

1. 从那一行原样取出base-directory路径。去掉结尾的`/skills/version`得到
   plugin根目录——例如上面例子里的
   `~/.claude/plugins/cache/play-console-assets/assetpub/1.0.0`。
   这是本次调用唯一正确的plugin根目录。
2. 用Read工具读取`<plugin root>/.claude-plugin/plugin.json`的`version`
   字段，锚定到这个确切路径——绝不用搜索或猜测出来的plugin根目录。
3. 如果那行base-directory信息碰巧没出现在context里（正常情况下一定会
   出现——如果缺失，如实说明，不要瞎猜），就没有其它可靠来源；如实
   报告版本号无法确定，不要凭空推断一个。
4. 只把这个版本号报告给用户。如果他们还想知道改了什么，读
   `<plugin root>/CHANGELOG.md`最上面那条entry。

## 这个skill能告诉你什么、不能告诉你什么

这里报告的是**本session实际加载运行**的代码版本——不代表这是不是
marketplace上发布的最新版本。刚跑完`/assetpub:upgrade`之后，运行这个
skill就是用来确认reload真的加载了新内容（一次没生效的reload在这里会
显示成旧版本号）。想知道"我是不是最新发布版本"，用`/assetpub:upgrade`——
这个skill只回答"我现在运行的是哪个版本"这一个问题。

**如果刚Update完看到的还是旧版本号，先别断言升级失败**：这里报告的
版本可能落后于磁盘上registry真实指向的版本——参考`aabbuild`skill在这个
问题上踩过的坑，先查`~/.claude/plugins/installed_plugins.json`里
`assetpub@play-console-assets`的`installPath`字段指向哪个版本号，跟这里
报告的对比：如果磁盘上已经是新版本，只需要再跑一次`/reload-plugins`，
不是Update本身出了问题。
