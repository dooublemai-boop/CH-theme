# Web 主题扩展设计基线

> 状态：设计已获批，尚未实现。当前 UI Theme 行为以 Codexhost 源码仓库中的 `docs/product/web-ui-extensions.md` 与实际源码为准；本文定义后续外置主题扩展的目标契约，不表示该契约已经发布。

## 目标

允许用户在 Codexhost Web/PWA 中安装并选择 Claude-inspired 等宿主主题。主题适用于当前浏览器中的全部 Harness 会话，由 Codexhost Web UI 统一呈现；切换 Harness 不切换主题。Codex Desktop 原生 UI、Harness 自己的 CLI/TUI/桌面 UI 不属于控制范围。

主题只改变 Codexhost 绘制的界面外观。消息文本、Markdown 结构、代码内容、工具/审批/提问状态、Harness 协议、执行语义和持久化数据不因主题而改变。

## 源码基线

以下是本地源码事实，不是未来能力声明：

- `apps/web-ui/packages/client/ui-theme/src/client/index.ts` 已有 `ThemeDefinition`、`ThemeRuntime.register()`、`setTheme()`、`overrideTokens()` 和 `theme/change`。注册 disposer 会移除主题；若被移除主题正处于活动选择，会把偏好重置为 `system`。
- `ThemeDefinition.colorScheme` 表示浅/深基础调色板；`ThemeRuntime.setTheme()` 现同时承载内置 `light | dark | system` 偏好和已注册主题 ID。`ThemeSettingsSchema` 的持久化字段目前只允许 `light | dark | system`，所以自定义 ID 不能经现有设置 schema 持久化恢复。
- `overrideTokens` 使用 `{ light, dark }` 成对值并进行运行时形状校验；当前名称仍可动态加入，不能据此推断已有稳定语义 token allowlist。
- `ui-settings` 已把 `ui-theme` 作为浏览器本地偏好 namespace，键为 `codexhost.web.preferences.v1:ui-theme`。保存 envelope 包含 `user` 和 `revision`，并经 namespace schema 校验。个人风格选择应继续留在浏览器，不写成 Web 服务全局用户设置。
- 客户端 `ThemePresenter` 从 `ThemeSnapshot` 把 scheme 与 token 投影到 `document`，并在 disposer 中撤回自己写入的 DOM 属性和变量。Web Server 的 `THEME_SCRIPT` 则在客户端模块运行前独立读取同一 localStorage 键，目前只识别浅/深/系统三种 preference。
- `WebAssets` 组合内置与已启用外置客户端包；`WebUiExtensions` 管理安装、启停和卸载；外置包在同进程、同页面权限下执行。Cordis 管理插件生命周期，但不提供代码沙箱。
- 外置包可带 DSH 原生 Host/client 入口，运行时由 `dsh-web-backend` 的 Loader 和 ClientModules 装配。插件禁用/卸载会等待 Cordis disposer，页面模块图通过现有 HMR 通道同步；当前没有外置主题专用的首屏风格描述或稳定公共 Theme API。

## 设计决定

### 两条独立偏好轴

主题选择由两个正交字段组成：

| 轴 | 值 | 所有者与持久化 |
| --- | --- | --- |
| 明暗模式 | `light | dark | system` | 沿用现有 `ui-theme` 浏览器本地偏好和操作语义 |
| 视觉风格 | 稳定主题 ID，例如 `default | claude` | 同一 `ui-theme` 浏览器本地偏好；不同浏览器/设备独立选择 |

明暗模式决定基础调色板；视觉风格在两个基础调色板上覆盖主题契约允许的语义 token。不能把 Claude 主题伪装成一种新的明暗模式，也不能让主题 ID 代表 Harness ID。

建议扩展 `ThemeSettings` 为 `preference: ThemePreference`、`styleId: string` 和现有 `fontSize`。`default` 是宿主内置风格。迁移时旧数据没有 `styleId` 则解释为 `default`；未知或本次未启用的主题 ID 不阻止页面启动。若该 ID 原本是浏览器保存的选择但当前服务端已不提供，当前浏览器将选择回退并持久化为 `default`。风格选择不写入 Host settings 文件或 Web 服务端共享设置。

### 版本化 Web Theme 契约

建立独立、browser-safe 的公共 Theme API/契约版本，不把 `@deepseek-ai/dsh-client-ui-theme` 的源文件、内部 store、插槽组件或 DSH 私有模块路径当作外部稳定 API。最终包名和 semver 范围在实现首个公共入口时确认，并通过现有 Host runtime version snapshot/peer 校验暴露不兼容原因。

契约至少定义：

- 契约主版本常量，以及由包清单声明的契约版本范围；Host 对不兼容主版本拒绝启用，清楚报告要求和实际版本。
- 稳定的主题 ID、显示名/本地化标签、描述、主题定义和首屏启动描述。ID 使用主题包命名空间，禁止占用 `default`、`light`、`dark`、`system` 等宿主保留 ID。
- 由 Host 发布且有文档的稳定语义角色 allowlist，以及每个角色的值类别；主题包使用角色键，不使用 `--dsw-*`、`--dsh-*` 或其他 CSS 自定义属性名称。
- `light` 与 `dark` 两套值必须齐全。角色名必须位于 Host 发布的 allowlist；首版值限制为经校验的 CSS 十六进制颜色（3/4/6/8 位形式），客户端和服务端共用同一纯校验函数，不接受函数、变量引用、URL、`@import`、选择器、字体或其他 CSS 表达式。
- 客户端通过 Host 提供的 `ctx.webTheme.register(definition)` 注册风格，并取得 Cordis 生命周期 disposer；通过 `getSnapshot()` 查询可用/当前选择和明暗模式，通过 `setStyle(id)` 选择风格。注册、选择、偏好恢复和注销均发布可观察变更。
- 外置包声明的 JSON 元数据和客户端实际注册的 `id`、契约版本、主题内容必须一致。宿主读取包元数据时只解析静态 JSON，不导入或执行扩展代码。

示例仅表达契约形状，具体导出名在实现时依本地包约定确定：

```ts
type WebThemeColor = string;
type WebThemeTokensV1 = Record<string, { light: WebThemeColor; dark: WebThemeColor }>;

interface WebThemeDefinitionV1 {
  id: string;
  label: string | { en: string; zh: string };
  description?: string | { en: string; zh: string };
  tokens: WebThemeTokensV1; // keys are stable semantic roles, never CSS variable names
}

interface WebThemeV1 {
  readonly contractVersion: 1;
  getSnapshot(): WebThemeSnapshot;
  setStyle(id: string): void;
  register(definition: WebThemeDefinitionV1): () => void;
  subscribe(listener: (snapshot: WebThemeSnapshot) => void): () => void;
}
```

`WebTheme` 的 `styleId` 是独立于现有 `ThemeRuntime.preference` 的选择轴：外置风格将语义角色映射到 Host 的受控 token，不调用 `ctx.theme.setTheme()` 来保存风格，也不改变 `light | dark | system`。内部 `ctx.theme` 仍是明暗模式、字体大小和 Presenter 的所有者；Web Theme façade 组合风格 token 并投影到该运行时。

### 首屏与运行时一致

刷新首屏不能先显示默认风格再闪到已保存风格。服务端构造首页时：

1. 从已启用外置包的声明元数据读取、验证静态主题描述；安全模式和 `/?extensions=off` 使用空外置主题集合。
2. 将允许的主题 ID 与 token 描述嵌入页面启动数据，不从 localStorage 获取任意 CSS 字符串。
3. 首屏脚本从浏览器偏好读取 `preference` 和 `styleId`，校验已知 ID，并在任何应用模块运行前设置明暗模式与语义 token；无存储、损坏存储、存储不可用、未知风格和校验失败均回退到 default/system，不阻断页面。
4. 客户端 ThemeRuntime/`ctx.webTheme` 使用同一份已组合主题清单和浏览器偏好恢复状态，Presenter 接管同一 token 集合，保证模块启动后不发生主题闪回。

内置、恢复和普通首页路径共享同一首屏初始化实现。内置或恢复页面不加载扩展主题元数据；浏览器之前选择的扩展风格若本次不可用，当前页面用 default 显示并将浏览器偏好更新为 default，不重新执行或恢复已卸载包。

### 禁用、卸载和失败回退

- 外置主题被禁用/卸载后，Web 服务的活动插件图和主题启动描述同时移除该主题；现有 Cordis disposer 撤回注册、token 和监听。卸载仍沿现有生命周期等待插件释放后删除文件。
- 当前浏览器的 `styleId` 若指向已不提供的主题，UI 和首屏立即使用 `default`，并将浏览器本地偏好更新为 `default`；刷新后不重新执行或恢复已卸载包。写入失败不阻断当前页面回退。
- 主题注册重复 ID、保留 ID、契约版本不兼容、客户端/静态元数据不一致、非法 token 或加载异常均拒绝该主题激活并记录扩展错误；其余扩展和内置 UI 保持可用。
- 首屏只使用通过静态验证的元数据；若扩展运行时随后失败，内置 default 必须可立即应用。清理只撤回该主题拥有的变量和注册，不覆盖其他扩展或 Host 样式。
- 主题扩展和现有 Web 扩展一样是可信同进程代码，不承诺沙箱。支持风格不会开放通用 CSS 注入权限。

## 范围与非目标

首版只支持 Codexhost 自己绘制的 Web/PWA，单个浏览器的风格作用于该服务连接的全部 Harness 会话。Claude-inspired 是一个外置主题包样例，不将 Anthropic/Claude 品牌资产或 PI 专属实现作为必要依赖。

不支持 Codex Desktop 原生窗口主题、Harness 自身 UI、Harness 身份联动、任意用户 CSS、整套页面替换、依赖私有 DOM/CSS selector、主题市场评分/自动更新、动态执行或主题沙箱。无 OpenCode 主题源码基线时不把 OpenCode 风格纳入本轮兼容标准。

## 验收标准

1. 用户可独立选择 `default` 或已启用的 Claude-inspired 风格，并独立选择 `light | dark | system`；刷新、浏览器重开后恢复。
2. 同一服务的两个浏览器可以选择不同风格；单浏览器切换 Harness 不改变风格。
3. 首屏和客户端 Presenter 使用相同偏好，无默认风格闪回；浅色/深色切换后主题各自 token 保持可读。
4. 多 Harness 会话的消息、Markdown、代码块、工具状态和窄屏布局只改变视觉表现，不改变内容和交互语义。
5. 禁用/卸载主题后，页面和刷新均回到 default；重复 ID、非法 token、静态元数据与运行时不一致、加载失败都不会破坏内置页面。
6. safe mode 和 `/?extensions=off` 不引用外置主题；安装/启用主题时不启动另一套 DSH Agent Runtime。
7. Codex Desktop 原生 UI 与 Harness 自己的 UI 不受主题选择影响。

## 实施前确认

- 在确定公共 API 包位置和包名时，核对其能被 `apps/web-ui/scripts/build.ts` 发现、被 Web Runtime version snapshot 记录、能作为宿主共享浏览器模块被外置客户端解析，并纳入相应构建与发行清单；不能假设 `packages/shared-contracts` 会自动进入 DSH Web 插件图。
- 从 `design-platform.css` 和各 token 消费点整理最小语义 allowlist，优先覆盖背景、表面、文本、边框、强调/状态、Markdown 代码与侧栏；不得直接把所有 `--dsw-*` 变量开放为公共契约。
- 按 Web UI 实际主题注入顺序统一 `ui-theme/src/boot-theme.ts` 与 `web-server/src/web-assets.ts` 的首屏初始化责任，避免双重 source of truth。
