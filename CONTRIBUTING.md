# 插件市场投稿与上架流程

本页给**想把自己的插件放进 XO Launcher「市场」的作者**看，也给市场维护者看。本仓库（`rightry/xo-launcher-plugins`，分支 `main`）就是启动器「插件 → 市场」唯一的来源。

先记住一句实话：**SHA-256 只保证「下载到的包没被篡改」，不保证「这个包是谁写的」**。本仓库的写权限就是整个市场的信任锚点 —— 用户是按「这个插件在 XO 官方仓库里有个目录」来决定装不装的。所以投稿是「你提 PR，我们审，审过的人合」。

---

## 一、仓库布局

**一个插件一个目录，目录名就是插件 id；根目录没有索引文件。**

```
README.md
CONTRIBUTING.md
xo.server-tunnel/                        ← 目录名 = 插件 id
  meta.json                              ← 市场展示信息 + 这个 zip 的 sha256
  xo.server-tunnel-1.0.0.zip             ← 插件包本体，目录里正好一个 .zip
com.alice.demo/
  meta.json
  com.alice.demo-1.0.2.zip
```

启动器列一次仓库树（`GET /repos/<repo>/git/trees/main?recursive=1`）就能把市场清单拼出来：**第一段路径 = 插件 id**，每个目录里找一个 `*.zip` 和一份 `meta.json`。没有中央 `index.json`，所以新增插件不需要改任何公共文件 —— 多人投稿天然不冲突。

根目录的散文件（`README.md`、`CONTRIBUTING.md`）不会被当成插件；以 `.` 开头或名字不合 `^[A-Za-z0-9._-]{1,64}$` 的目录会被整个忽略并记一条日志。

## 二、目录里放什么

### 1. zip 包（插件本体）

打包要求与本地导入完全一致（见 wiki [03 工程搭建与打包](https://github.com/rightry/xo-launcher-plugins/wiki/03-工程搭建与打包)）：

- `plugin.json` 在 zip 根目录，或在**唯一一个顶层目录**里；
- 只带 `plugin.json` + 你的 DLL（+ 明确需要的第三方 DLL / 资源）；
- **不要**带 `XO_Launcher.PluginApi.dll`、`Microsoft.WinUI.dll`、WebView2 的 DLL；
- 文件名建议 `<id>-<version>.zip`；**一个目录里只能有一个 `.zip`**，两个以上启动器不知道该装哪个，卡片会变灰。

### 2. meta.json（市场展示信息）

```json
{
  "name": "示例插件",
  "version": "1.0.0",
  "author": "alice",
  "description": "一句话说明：做什么、适合谁。",
  "sha256": "0000000000000000000000000000000000000000000000000000000000000000",
  "size": 34434,
  "permissions": ["network"],
  "homepage": "https://github.com/alice/xo-demo"
}
```

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `name` | 建议 | 卡片标题；留空回落成目录名 |
| `version` | ✅ | 必须能被 `Version.Parse`（`1.0.0`、`1.2.3`）。写成 `v1.0-beta`、`1.0.0-rc1` 会导致「可更新」判断失效，用户收不到升级提示 |
| `author` | 建议 | 显示 `by xxx`，留空显示「未署名」 |
| `description` | 建议 | 卡片与安装确认框上的一句话 |
| `sha256` | ✅ | **本目录那个 zip** 的 SHA-256，十六进制，大小写不限。缺失 = 卡片「不可安装：缺少 sha256，无法校验来源」 |
| `size` | 可选 | 字节数；不填启动器直接用仓库树里的实际大小 |
| `permissions` | ✅ | 与 zip 内 `plugin.json` 的 `permissions` 逐项一致（安装确认框按这里显示，装完首次授权弹框按包内那份显示） |
| `homepage` | 建议 | 源码页或说明页。闭源又申请高权限的，一般直接拒 |

生成校验值：

```powershell
(Get-FileHash .\com.alice.demo-1.0.0.zip -Algorithm SHA256).Hash.ToLower()
(Get-Item .\com.alice.demo-1.0.0.zip).Length
```

## 三、写 meta.json 之前先自查

1. **改过任何内容都要抬 `version` 并重算 `sha256`**：同版本号在用户侧不会提示更新；`sha256` 没跟着改则安装必然被校验挡下。
2. **权限如实且最小**：`permissions` 每一项都真用到；声明与实际行为不一致 = 骗授权。
3. **zip 里 `plugin.json` 的 `id` 必须等于目录名**。不一致时启动器按包内 `id` 安装并记一条 WARN —— 但效果是**顶掉另一个插件**，属于最严重的投稿错误之一，维护者会直接拒。
4. **AI 工具名**带插件前缀、全小写下划线（撞内置工具名会被静默过滤）。
5. **不含敏感物**：目录与包内都不要有 token、密钥、账号数据、你替用户填的凭证。
6. **别碰别人的目录**：PR 的 diff 只应该出现你自己那一层。

## 四、投稿三步

1. **开 PR**：新增 `plugins/<你的 id>/` 目录（`meta.json` + zip）。描述里带上源码地址、这个插件做了什么、为什么要这些权限、一句「本地实测过装 / 用 / 删」。
2. **维护者审 + 合并**：审核清单见下一节。合并即上线 —— 启动器列的是 `main` 的当前树。
3. **作者确认**：市场页刷新后能看到自己的条目、文件来源路径与体积正确。

> 没有 tag、也没有 ref 固定：这套模型追求的是「加个目录就上架」。要改成钉死发布点（按 commit/tag 安装）需要动本体，目前是跟随分支头 + 逐包 SHA-256 校验。

## 五、维护者审核清单

审的是「**这个 zip 值不值得让用户以启动器权限跑**」，所以要**下载下来看包本体**，不能只看 PR 描述。

- [ ] PR 只新增了你审查的那个目录，没顺手改别人的目录、没动根目录公共文件；
- [ ] 目录名合 `^[A-Za-z0-9._-]{1,64}$`，且**不与已有目录撞名**（撞名 = 覆盖别人的插件，直接拒）；
- [ ] 目录里正好一个 `.zip`，`meta.json` 存在；
- [ ] 用与 `meta.json` 一致的方式重算 zip 的 SHA-256 并比对；`size` 与实际相符；
- [ ] 解压看包：`plugin.json` 的 `id` == 目录名、`version` == `meta.json` 的 `version`、`entry` 与 DLL 文件名逐字对上；
- [ ] 包里**没有** `XO_Launcher.PluginApi.dll` / `Microsoft.WinUI.dll` / WebView2 的 DLL；
- [ ] DLL 体积量级合理；`strings` 扫一遍有没有可疑域名、凭据、外传地址；
- [ ] `permissions` 与代码里实际用到的能力一致；声明 `process` / `gameWrite` / `replacePage` 却看不到用途的，要作者解释；
- [ ] 没有触碰账号数据（`accounts.json`、`settings.json` 的令牌）、没有静默往游戏目录外的位置写东西；
- [ ] 描述文案没有违规/误导（年龄提示、收费说明、不冒充官方）；
- [ ] 装一次跑通：市场能看到 → 确认框权限清单与声明一致 → 首次授权弹框 → 页面能打开 → 能删除且不留文件占用。

### 打回标准（直接拒）

- 目录名或包内 `id` 撞已有插件；
- 包内 `plugin.json` 的 `id`/`version` 与 `meta.json` 不一致；
- 带了契约程序集副本（装上也加载失败，属于质量问题）；
- 权限声明与用途明显不符，或拒绝解释；
- 闭源 + 申请 `process` / `gameWrite` / `replacePage` 中任一项 —— 任意代码 + 高权限 + 无法审阅，三样凑齐就不收；
- 恶意行为：偷账号数据、静默写游戏目录外的位置、劫持启动参数却不说明。

## 六、更新与下架

| 情况 | 怎么做 |
| --- | --- |
| 发新版 | 提 PR：放进新 zip、更新 `meta.json` 的 `version` 与 `sha256`、**删掉旧 zip**（一个目录只留一个）。旧版本在 git 历史里，需要留档就另开 `releases/` 目录，别塞在插件目录里 |
| 只改文案不改代码 | 也要换 zip + 重算 `sha256`，否则用户侧不会提示更新 |
| 发现恶意/严重缺陷 | 维护者直接删该目录并推送（用户下次刷新市场就看不到），开一条 issue 记录原因；已装的靠启动器「删除插件」，必要时在本体加黑名单 |
| 作者想撤回 | 提 PR 删自己的目录 |
| `meta.json` 写坏（JSON 解析失败） | 卡片显示「meta.json 解析失败：<原因>」且按钮变灰，不影响其它条目 —— 但仍要改回来 |

## 七、常见问题

**Q：为什么根目录不放一个总 index.json？**
放一个公共文件，所有人投稿都要改它：数组末尾必然冲突，而且展示字段抄两份必然漂移（索引一份、包内一份）。现在展示信息只有一个来源，新增插件只是加一个目录。

**Q：`meta.json` 和包里的 `plugin.json` 不是一致的东西？**
是两处，但职责不同：`meta.json` 给**未安装**的市场卡片用（不能为了列个清单就把每个插件都下载解压一遍），`plugin.json` 是**已安装**插件的清单。所以审核时必须核对两者一致。

**Q：用户装的时候到底校验了什么？**
`meta.json` 里的 `sha256` 与下载到的 zip 逐字节比对，不符就删除并拒装；之后走和本地导入完全相同的 zip 安装流程（会先卸载全部插件再重载）。

**Q：GitHub API 有速率限制吗？**
列仓库树是一次调用，匿名限 60 次/小时；启动器把结果缓存 6 小时（`%LOCALAPPDATA%\XO_Launcher\plugin-market-cache.json`），市场页右上角的刷新按钮可强制重拉。zip 与 `meta.json` 走 raw，不吃这个额度。

**Q：一个插件能同时挂多个版本吗？**
不能。市场按目录名（= 插件 id）比对，一个目录只认一个 zip，安装会覆盖。

**Q：能不能让用户订阅多个仓库？**
目前本体把仓库与分支写死在 `Services/PluginMarketService.cs` 顶部的 `Repo` / `Branch` 常量，没有多源。要做需要改本体。

---

开发插件本身请看 wiki 的 [Home](https://github.com/rightry/xo-launcher-plugins/wiki/Home)、[03 工程搭建与打包](https://github.com/rightry/xo-launcher-plugins/wiki/03-工程搭建与打包)、[12 插件市场与分发](https://github.com/rightry/xo-launcher-plugins/wiki/12-市场与分发)。
