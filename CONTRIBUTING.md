# 插件市场投稿与上架流程

本页给**想把自己的插件放进 XO Launcher「市场」的作者**看，也给市场维护者看。本仓库（`rightry/xo-launcher-plugins`，分支 `main`）是启动器「插件 → 市场」唯一的索引源。

先记住一句实话：**校验值只保证「拿到的字节没被篡改」，不保证「这个包是谁写的」**。`index.json` 的写权限就是整个市场的信任锚点 —— 用户是按「这条记录在 XO 官方仓库里」来决定装不装的。所以投稿不是「你传上去」，而是「你提，我们审，审过的人替你上」。

---

## 一、仓库布局

```
index.json                       ← 薄索引：每个插件一行，只写「在哪、钉哪个 ref」
plugins/
  xo.server-tunnel/              ← 一个插件一个目录，目录名 = 插件 id
    plugin.json                  ← 唯一的元数据来源（名称/版本/作者/描述/权限/entry）
    XO_Launcher.Plugin.ServerTunnel.dll
  com.alice.demo/
    plugin.json
    MyPlugin.dll
```

**插件的展示信息只写在目录内的 `plugin.json` 里，索引不再抄一遍。** 索引里出现 `name`/`version`/`sha256` 这类字段只有两种情况：历史 zip 条目（见第五节），或者维护者要覆盖显示 —— 正常投稿不要写，抄两份必然漂移。

## 二、投稿前自检（作者）

1. **版本号可解析**：`1.0.0`、`1.2.3` 这种 `Version.Parse` 认的形式。别写 `v1.0-beta`、`1.0.0-rc1`，写了比对会失效、用户收不到更新提示。
2. **改过内容就抬版本 + 重打 tag**：同版本号在用户侧不会提示「可更新」，等于发了个更不了的版本。
3. **权限如实且最小**：`plugin.json` 的 `permissions` 每一项都真用到；声明与实际行为不一致 = 骗授权。
4. **目录里只放**：`plugin.json` + 你的 DLL（+ 明确需要的第三方 DLL、必要的资源文件）。**不要**带 `XO_Launcher.PluginApi.dll`、`Microsoft.WinUI.dll`、WebView2 的 DLL —— 会让 `IXoPlugin` 类型判定失败。
5. **目录名 = 清单里的 `id`**，且 `id` 匹配 `^[A-Za-z0-9._-]{1,64}$`。
6. **`entry` 指向的 DLL 必须就在同一个目录里**，名字逐字对上（含大小写）；缺文件时安装会被拒（「目录里没有清单指定的入口程序集：xxx.dll」）。
7. **不含敏感物**：目录里不要有 token、密钥、账号数据、你替用户填的凭证。
8. **可辨识**：AI 工具名带插件前缀、全小写下划线，避免撞内置工具名（撞名会被静默过滤）。
9. **上限**：单插件最多 64 个文件、合计 64 MB、子目录只允许一层；有符号链接或子模块会被拒。
10. **源码可查**：`homepage` 指向你的仓库或说明页。闭源 + 申请 `process`/`gameWrite`/`replacePage` 任一项，会被拒（任意代码 + 高权限 + 无法审阅）。

## 三、目录里那份 plugin.json

```json
{
  "id": "com.alice.demo",
  "name": "示例插件",
  "version": "1.0.0",
  "author": "alice",
  "description": "一句话说明：做什么、适合谁。",
  "entry": "MyPlugin.dll",
  "permissions": ["network"],
  "minLauncherVersion": "1.0.0"
}
```

这份文件在市场卡片上的显示权重最高：名称、版本、作者、描述、权限**全部由启动器直接读它**，用户在安装确认框里看到的权限清单也是它（不是索引抄的那份）。

## 四、投稿三步

### 1. 提 PR：加你的目录

```
+  plugins/com.alice.demo/plugin.json
+  plugins/com.alice.demo/MyPlugin.dll
```

多人各改各的目录，互不冲突 —— 这也是不再让作者手改索引的原因。

### 2. 维护者合并后打 tag

```bash
git tag -a com.alice.demo-1.0.0 -m "示例插件 1.0.0"
git push origin com.alice.demo-1.0.0
```

tag 名就是 `<id>-<version>`。它钉住的是一个**不可变的 commit**，这是这套流程的完整性来源：分支头后面怎么动，用户装到的都是你审过的内容。

### 3. 索引加一行

```json
{
  "schema": 2,
  "updatedAt": "2026-10-31",
  "plugins": [
    {
      "id": "com.alice.demo",
      "dir": "plugins/com.alice.demo",
      "ref": "com.alice.demo-1.0.0",
      "homepage": "https://github.com/alice/xo-demo"
    }
  ]
}
```

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `id` | ✅ | 决定覆盖谁，必须与目录内 `plugin.json` 的 `id` 一致；不一致时启动器**拒绝安装**并显示「目录里的 id 与索引里的不一致」 |
| `dir` | ✅ | 插件目录（相对仓库根） |
| `ref` | 建议必填 | tag 或 commit SHA。**留空 = 跟随 `Branch` 分支头**，等于放弃发布点固定，只在新版本试水时这么写 |
| `homepage` | 建议 | 源码/说明页 |
| `updatedAt` | ✅ | 每次改动索引都更新成当天日期，界面会显示 |

改完索引记得同步把 `updatedAt` 改掉 —— 这是唯一还需要手改的字段。

## 五、旧的 zip 条目还能用吗

能，继续兼容，但不建议新增：

```json
{ "id":"…", "name":"…", "version":"…", "file":"packages/x-1.0.0.zip",
  "sha256":"…", "size":34434, "permissions":["network"], "homepage":"…" }
```

zip 条目缺 `sha256` 一律拒装；`file` 可以写绝对 https 直链（作者自己托管），也可以写仓库内相对路径。两种形式在界面上一眼能分：目录条目显示「来源 plugins/x@标签」，zip 条目显示「来源 packages/x.zip」。

## 六、维护者审核清单

审的是「**这个目录里的内容值不值得让用户以启动器权限跑**」，所以要**下载下来看文件本体**，不能只看 PR 描述。

- [ ] PR 只动了 `plugins/<他的 id>/` 目录，没顺手改别人的目录、没动 `index.json` 里别人的行；
- [ ] `plugin.json` 的 `id` 与目录名一致，且不撞已有条目（撞 `id` = 覆盖别人的插件，**直接拒**）；
- [ ] `entry` 指的那个 DLL 真在目录里；用 `git ls-tree` 或本地 checkout 核一遍文件清单与体积；
- [ ] DLL 确实是目标框架的类库，体积量级合理（几十 KB 骨架里塞了超大 DLL 要看是什么）；反编译或至少 `strings` 扫一遍有没有可疑域名、凭据、外传地址；
- [ ] `permissions` 与代码里实际用到的能力一致；声明了 `process` / `gameWrite` 却看不到用途的，要作者解释；
- [ ] 没有触碰账号数据（`accounts.json`、`settings.json` 的令牌）、没有静默往游戏目录外的位置写东西；
- [ ] 描述文案没有违规/误导（年龄提示、收费说明、不冒充官方）；
- [ ] 合并 → 打 tag → 索引里 `ref` 填**那个 tag**，`updatedAt` 改当天；
- [ ] 装一次跑通：市场能看到 → 安装确认框文件清单正常 → 首次授权弹框权限与声明一致 → 页面能打开 → 能删除且不报文件占用。

### 打回标准（直接拒）

- 目录里 `id` 与索引不一致，或撞已有 `id`；
- 目录里带了契约程序集副本（装上也加载失败，属于质量问题）；
- 权限声明与用途明显不符，或拒绝解释；
- 闭源 + 申请 `process` / `gameWrite` / `replacePage` 中任一项；
- 用符号链接、子模块、超深层目录绕开文件校验；
- 恶意行为：偷账号数据、静默写游戏目录外的位置、劫持启动参数却不说明。

## 七、更新与下架

| 情况 | 怎么做 |
| --- | --- |
| 发新版 | 提 PR 改你的目录 → 合并 → 打 `<id>-<新版本>` 新 tag → 索引里把 `ref` 换成新 tag。旧 tag 留在仓库里，已装用户不受影响 |
| 只改文案不改代码 | 也要抬版本 + 重打 tag + 改 `ref`，否则用户侧不会提示更新 |
| 发现恶意/严重缺陷 | 维护者直接把该条从 `index.json` 删掉并推送（用户下次刷新市场就看不到），开一条 issue 记录原因；已装的靠启动器「删除插件」，必要时在本体里加黑名单 |
| 作者想撤回 | 提 PR 删条目；目录可以留着也可以一起删 |
| 索引写坏（解析失败） | 市场页显示「市场索引加载失败：<原因>」并沿用旧缓存 —— 推之前本地 `JSON.parse` 一次，`dir` 与 `ref` 用 `git cat-file` 核一遍能不能取到 |

## 八、常见问题

**Q：为什么不开放直接推 `main`？**
用户看的是「这条记录在 XO 官方仓库里」，任何一次误推都会变成全体用户的风险。

**Q：启动器怎么知道目录里有哪些文件？**
用 GitHub 的 contents API 列目录（返回每个文件的 git blob SHA-1），再按 `ref` 从 raw 逐个取文件、**逐个重算 blob SHA-1 比对**。所以 `ref` 必须是 tag 或 commit SHA；填 `main` 时列完目录到下载之间分支可能已经变了，那种情况下会因校验不符而拒装。

**Q：匿名调用有速率限制吗？**
contents API 匿名限 60 次/小时。装一个插件 = 目录数 + 1 次；被限流时界面提示「GitHub API 拒绝访问…稍后再试」。raw 上的文件下载不吃这个额度。

**Q：一个 `id` 能同时挂多个版本吗？**
不能。市场按 `id` 比对本机版本，同 `id` 只有一条记录，安装会覆盖。

**Q：能不能让用户订阅多个仓库？**
目前本体把仓库与分支写死在 `Services/PluginMarketService.cs` 顶部常量（`Repo`、`Branch`），没有多源。要做需要改本体。

---

开发插件本身请看 wiki 的 [Home](https://github.com/rightry/xo-launcher-plugins/wiki/Home)、[03 工程搭建与打包](https://github.com/rightry/xo-launcher-plugins/wiki/03-工程搭建与打包)、[12 插件市场与分发](https://github.com/rightry/xo-launcher-plugins/wiki/12-市场与分发)。
