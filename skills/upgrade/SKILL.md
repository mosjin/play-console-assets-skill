# assetpub upgrade

拉取`play-console-assets`插件的最新版本并核实。

参考`aabbuild`自己的`upgrade`skill模式：直接对git checkout跑`git pull`，
而不是干等用户去点`/plugin`菜单——`/plugin install`在插件已安装时是
no-op（报"already installed globally"，不会刷新任何内容）。

## 步骤

1. **找到plugin的marketplace git checkout路径**（按顺序，取第一个存在的）：
   - `~/.claude/plugins/marketplaces/play-console-assets/`
     （Claude Code marketplace安装位置——真正的git checkout，能`git pull`）
   - 如果这个目录不存在，说明用户还没跑过
     `/plugin marketplace add mosjin/play-console-assets-skill`，
     告诉用户先跑这条，然后停止。

   **为什么是`marketplaces/`不是`cache/`？** Claude Code的插件管理器把
   git checkout存在`~/.claude/plugins/marketplaces/<name>/`，然后把每个
   版本的快照单独拷一份到
   `~/.claude/plugins/cache/<marketplace-name>/<plugin-name>/<version>/`
   （这份没有`.git`）。`git pull`只对`marketplaces/`那份checkout管用。

2. **记录旧版本号**（这样后面能看出到底有没有变化）：
   ```bash
   OLD=$(grep -oP '"version":\s*"\K[^"]+' ~/.claude/plugins/marketplaces/play-console-assets/.claude-plugin/plugin.json)
   OLD_SHA=$(git -C ~/.claude/plugins/marketplaces/play-console-assets rev-parse --short HEAD)
   ```

3. **拉取+快进到最新`main`**（用Bash直接执行，不用等用户跑任何指令）：
   ```bash
   git -C ~/.claude/plugins/marketplaces/play-console-assets fetch origin main
   git -C ~/.claude/plugins/marketplaces/play-console-assets pull --ff-only origin main
   ```
   如果`pull`失败（本地有改动、分支分叉），中止并汇报，不要force——
   告诉用户这个目录理论上不该有本地改动，如果真有，先处理掉再重跑
   `/assetpub:upgrade`。

4. **记录新版本号**：
   ```bash
   NEW=$(grep -oP '"version":\s*"\K[^"]+' ~/.claude/plugins/marketplaces/play-console-assets/.claude-plugin/plugin.json)
   NEW_SHA=$(git -C ~/.claude/plugins/marketplaces/play-console-assets rev-parse --short HEAD)
   ```

5. **刷新Claude Code的cache快照——这一步必须用户自己操作。** `git pull`
   只更新了marketplaces那份git checkout。Claude Code此时仍然从
   `cache/play-console-assets/assetpub/<旧版本>/`加载旧的固定快照。

   打开插件管理器选**Update**（不是install，install已装时是no-op）：
   ```
   /plugin            # 打开管理器 → 选中assetpub → 选"Update"
   ```
   这一步会更新`~/.claude/plugins/installed_plugins.json`里
   `assetpub@play-console-assets`的`installPath`并把新版本拉进
   `cache/play-console-assets/assetpub/<新版本>/`——**但这个变化
   只写进磁盘上的registry，不会自动同步进当前这个已经在跑的session**。

   **必须紧接着再跑一次`/reload-plugins`**：
   ```
   /reload-plugins
   ```
   **验证**：跑`/assetpub:version`，如果显示的还是旧版本号，先别急着
   下结论"升级没生效"——去查一下
   `~/.claude/plugins/installed_plugins.json`里
   `assetpub@play-console-assets`的`installPath`字段实际指向哪个
   版本（`grep -A2 "assetpub@play-console-assets" ~/.claude/plugins/installed_plugins.json`）。
   如果那个文件已经指向新版本，说明Update其实成功了，只是这个session
   还没`/reload-plugins`过——再reload一次就好，不是Update本身失败。

6. **展示一份checklist**：
   ```
   ## assetpub 升级结果
   - [x] Marketplace路径：~/.claude/plugins/marketplaces/play-console-assets
   - [x] git pull：<OLD_SHA>..<NEW_SHA>（main分支）
   - [x] 版本：<OLD> → <NEW>
   - [x] cache已刷新到<NEW>（/plugin→Update + /reload-plugins后确认）
   ```
   成功用`[x]`，失败用`[ ]`，数字/版本号照实填，不要编。

7. **提醒用户reload**，让新的SKILL.md真正生效：
   ```
   /reload-plugins
   ```

## 幂等性

如果`git pull`报`Already up to date.`：
- `OLD == NEW` → checklist显示"版本 <NEW>（未变化）"。
- 不用提醒reload——用户已经是最新的了。

## 这个skill做不到什么

第5步的cache刷新只能由用户在Claude Code客户端里手动点"Update"完成，
这个skill没有触发它的权限；`git pull`本身可以由这个skill直接执行（已经
在第3步做了），但cache刷不刷新、reload有没有真的生效，最终仍需要用户
自己用`/assetpub:version`核实一次，不要替用户断言升级成功。
