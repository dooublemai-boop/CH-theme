# Web 主题扩展实施计划

> 状态：待实施。执行前须满足“实施 Gate”；本计划拆分 [Web 主题扩展设计基线](web-theme-extension-design.md) 的交付步骤，不表示公共 API、主题包或功能已经发布。实现行为以届时源码、产品文档和实际验证为准。

## 范围与前置条件

交付 Codexhost Web/PWA 的外置视觉风格扩展及一个 Claude-inspired 示例主题包。明暗模式和风格分别选择；风格在浏览器本地保存。同一浏览器中的全部 Harness 共用宿主外观，不改变消息、Markdown、工具和交互语义。不改 Codex Desktop 原生界面、Harness 自身 UI/TUI、Agent Runtime、任意 CSS 注入或插件沙箱。

开始功能实现前须完成以下 Gate，并把结论记录在设计文档或本计划的实施记录中：

1. 选定 browser-safe 公共契约包的名称、目录、公开导出、发布版本策略和主题包 peer 版本范围。包不得把 `@deepseek-ai/dsh-client-ui-theme`、`@codexhost/shared-contracts` 私有路径或 DSH 类型变成第三方主题的隐式公共 API。
2. 用一个真实外置客户端样包验证契约导入在源码构建和发行构建中解析到同一份宿主共享模块实例。确认包纳入 `apps/web-ui/scripts/build.ts` 的发现/构建方式，且外置模块在 `packages/web-server/src/external-client-assets.ts` 的可用模块图中可解析。仅有 workspace 依赖或 root `shared-contracts` 导出不算通过。
3. 把契约包纳入 `sourceClientRuntimeVersions()` 与 `client-runtime-versions.json` 发行快照、peer 校验和发行资源清单。验证缺失快照、缺失模块和不兼容版本有明确诊断，不运行不兼容主题。
4. 根据 `ui-theme/src/styles/design-platform.css`、组件实际消费点和默认 light/dark 调色板冻结最小 V1 语义角色 allowlist。先覆盖基础/分层背景、主要/次要文字、边框、强调与业务状态、侧栏、输入/消息表面、Markdown 行内和代码块；只公开使用证据充分的角色，不开放 `--dsw-*` 名称或全量变量表。
5. 明确启动脚本的单一数据来源及执行次序。现有 `web-server/src/web-assets.ts` 的 `THEME_SCRIPT` 读取浏览器明暗偏好；`ui-theme/src/boot-theme.ts` 通过 Web Server 注入 Host 明暗/字号。实施前画清普通首页、`/?extensions=off`、`--safe-mode` 的脚本来源和首屏顺序，不能留下互相覆盖的双重主题初始化。

## 分阶段实施

### 1. 冻结契约与纯校验

**负责位置**：新增选定的公共 browser-safe 契约包及其入口；`apps/web-ui/import-manifest.json`、`apps/web-ui/scripts/build.ts`、`packages/web-server/src/client-runtime-versions.ts`、`packages/web-server/src/external-client-assets.ts`、发行资源清单，以及对应的包清单和 boundary 规则。

- 定义 V1 版本常量、版本范围、保留 ID 规则、主题定义/静态启动描述、快照形状和稳定角色键；外部接口只暴露 `ctx.webTheme`，内部 `ctx.theme` 保留给现有 Host 消费者。
- 实现同一份无 DOM、无 Node 依赖的主题定义纯校验器，供 Web Server 静态 JSON 校验和浏览器运行时注册共同调用。校验必填/未知字段、主题 ID 唯一性、角色 allowlist、完整 light/dark 值及仅允许 3/4/6/8 位十六进制颜色；拒绝任意 CSS 表达式。
- 用 peer 版本范围和 Host 的精确 runtime snapshot 检查公共契约兼容性，错误说明请求范围与实际版本。确认新包在开发构建、源码 Web Server、打包 Web Server、发行静态资源中采用相同包身份和入口。

**覆盖与通过条件**：纯校验测试覆盖有效定义、缺失明暗值、未知/保留角色、保留/重复 ID、恶意 CSS 字符串、错误范围、范围不匹配及快照缺项。`npm run build:web:ui`、相关 Web Server Node 测试、`npm run test:web:distribution` 中对应发行测试通过；检查 `client-runtime-versions.json` 确实包含选定的 Host 模块版本。发布包布局、模块单例或快照任何一项未验证，不进入下一阶段。

### 2. 浏览器偏好、ThemeRuntime 与公开 API

**负责位置**：`ui-theme/src/theme-settings.ts`、`ui-theme/src/client/index.ts`、`ui-settings/src/client/browser-preferences.ts`；按 Gate 新增公共包的客户端 façade/Host 类型接线，并保留 `ui-layout/src/client/theme-presenter.ts` 和现有 `theme/change` 消费路径。

- 在现有 `preference`、`fontSize` 之外增加 `styleId`，旧浏览器记录缺少该字段时解释为 `default`；schema 校验后继续使用 `ui-theme` namespace 的 `user/revision` envelope 和 `codexhost.web.preferences.v1:ui-theme` key。只写当前浏览器 localStorage，不写 Host 或服务端共享设置。
- 公开版本化 `register(definition)`（返回 Cordis disposer）、`getSnapshot()`、`setStyle(id)`、`subscribe(listener)`。快照包含 `colorMode`、`styleId`、风格元数据、可用主题和递增 revision。Web Theme façade 独立保存 styleId，只把已验证语义角色映射到 Host 受控 token；不得调用 `ctx.theme.setTheme()` 选择风格，也不得改变 `preference`。现有 `ThemeRuntime` 继续拥有 `light | dark | system` 与 Presenter 投影。
- 将注册主题定义转换为 Host 的受控 token 覆盖，拒绝超出 V1 allowlist 的角色。注册、选择、偏好恢复、OS 明暗变化、注销与失败回退均发出一致快照；释放时只清除该主题拥有的注册、token 和订阅，不覆盖其他来源。
- 若 localStorage 不可读/不可写，仍允许当前页使用默认外观继续运行；偏好迁移和清理不得让 Host settings scope 收到个人 styleId。

**覆盖与通过条件**：新增 Vitest 集成逻辑测试，放入 `tests/integration/` 并由 `tests/vitest.config.js` 收集；覆盖旧偏好迁移、浏览器 namespace 隔离、快照 revision、注册/注销、活动主题释放回退、重复注册、未知主题、明暗切换、存储失败和 Host settings 不写入 styleId。既有 `ThemePresenter` 对保留内部 `ctx.theme` 的消费者继续生效。

### 3. 唯一首屏启动路径

**负责位置**：`packages/web-server/src/web-assets.ts`、`packages/web-server/src/main.ts`、`ui-theme/src/boot-theme.ts`，以及客户端启动读取/ThemePresenter 接管逻辑。

- 依实施 Gate 决议合并现有 `THEME_SCRIPT` 与 `bootThemeInjections()` 的职责，明确唯一首屏生成器和唯一启动数据结构。浏览器在应用模块之前读取本地 preference/styleId；服务端只提供已启用且通过 V1 校验的静态主题启动描述及首次偏好的既定回退值，不执行主题包代码来发现主题。
- 页面启动数据只包含校验后的 ID、契约版本、显示元数据和 light/dark 语义角色 token。localStorage 只允许选已知 ID，不允许提供 CSS 字符串。启动脚本与运行时使用同一主题清单、色彩模式解析和默认回退规则。
- 确保初始 canvas、`color-scheme`、明暗属性、首屏 token 和字号在模块执行前就位；客户端 `ThemeRuntime` / `ctx.webTheme` 和 `ThemePresenter` 使用相同选择接管，不闪回默认风格。
- 普通首页、`/?extensions=off` 与 `--safe-mode` 走一致的内置初始化算法。恢复页面不包含外置主题描述；保存的主题当次不可用时展示并尝试持久化 `default`，localStorage 写入失败不阻断页面。

**覆盖与通过条件**：扩展 `packages/web-server/test/web-assets.test.ts`，断言普通首页包含已验证的启动数据、危险值被拒绝，恢复首页和 safe mode 不包含外置描述；验证已有 `THEME_SCRIPT` 行为与默认 light/dark canvas 未回归。`packages/web-server/test/ui-extensions.test.ts` 或 HTTP 测试覆盖禁用后首页立即回退。新增集成测试验证启动读取与首个运行时快照一致。不得留下两段对同一明暗/字号属性互相覆盖的初始化代码。

### 4. 外置包静态元数据与生命周期

**负责位置**：`packages/web-server/src/external-client-assets.ts`、`web-assets.ts`、`ui-extensions.ts`、`main.ts`；DSH Web 后端更新链 `pluginRuntime.update(externalPlugins, changedId)`；必要时扩展 `WebUiExtensions` 包元数据字段或其读取层。

- 在 Gate 决定的扩展包清单字段中声明 Web Theme 契约版本范围和一个包内静态 JSON 描述文件。仅通过受包目录限制的文件解析读取 JSON，复用阶段 1 校验器；不导入扩展模块来取得主题清单。
- 将启用扩展的主题描述与外置客户端图做同一版本更新。只用实际活动且校验通过的包生成首页清单；禁用/卸载/重组后同步移除启动描述。保留 `WebUiExtensions` 现有依赖检查、Cordis disposer、等待释放后清理包文件的生命周期。
- 客户端 `register()` 时核对实际 id、契约版本和规范化主题内容与静态 JSON 一致。任何不匹配、重复/保留 ID、版本不兼容或加载异常仅拒绝该主题，不破坏内置界面或其他扩展。
- safe mode 与 `/?extensions=off` 不导入或引用外置启动描述；不能为主题功能引入新的服务监听器、Agent Runtime 或全局主题市场。

**覆盖与通过条件**：在 `packages/web-server/test/web-assets.test.ts`、`ui-extensions.test.ts`、`dsh-web-hot-loading.test.ts`（按实际归属）扩展测试，覆盖包路径越界、JSON 无效/不在包内、主题定义与运行时清单不符、版本冲突、活动图移除、更新时描述同步、启停和卸载清理、依赖/加载失败时保留内置页面。Web Server 测试使用 Node runner；不得把测试放进该 runner 不发现的 `apps/web-ui/packages/**/test`。

### 5. 设置中的风格选择

**负责位置**：`ui-theme/src/client/AppearanceRow.tsx`、对应 CSS、`settings-store.ts`、`locales.ts` 和 `client/index.ts`；新 UI 继续使用现有 General section slot 与 Cordis 插件 disposer。

- 在现有浅色/深色/跟随系统控件之外增加与之独立的风格选择控件，列出内置 `default` 和已注册且兼容的主题显示名；未启用主题不显示为可选项。
- 选择风格调用 `ctx.webTheme.setStyle()`，明暗控件仍走原操作。store 只镜像服务快照，不另建偏好存储；中文和英文 locale 都覆盖新增文案。
- 若活动主题在当前浏览器运行期间被禁用或卸载，设置选择同步回到 `default`，同时更新 service 和浏览器偏好。主题尚未加载、注册失败、可用集合更新或存储不可用时，设置页和内置主题保持可用。

**覆盖与通过条件**：Vitest 集成测试覆盖两个轴的独立选择、主题注册后选项出现、注销后选项消失/活动选择回退、快照同步及无效主题 ID。设置页不依赖具体主题包名，也不按 Harness ID 改变主题。

### 6. Claude-inspired 样包与 Web 验收

**负责位置**：新建独立外置主题包（目录按 Gate 确定）及其静态 JSON、客户端入口、文档；产品现有 UI 和多 Harness 行为不承担主题身份判断。

- 提供一个符合契约的 Claude-inspired 示例风格，分别为 light/dark 提供允许角色的十六进制 token；其包身份、作者说明与 Claude/Anthropic 品牌资产无依赖，不引用 PI 私有选择器、DOM、构建脚本或插件代码。
- 在至少两个隔离浏览器 profile 连接同一 Web 服务验收不同风格各自保存；刷新/重开恢复、切换 Harness 不改变风格；切换明暗时保持对应主题 palette；禁用和卸载后两端各自回退。
- 检查多 Harness 消息、Markdown/引用/代码块、工具/审批/提问状态、会话侧栏、输入区和窄屏。确认消息内容、状态、操作及持久化均不变。
- 检查 safe mode、`/?extensions=off`、扩展加载/静态描述失败、localStorage 不可用、浏览器原生 UI、Codex Desktop 原生界面与 Harness 自身 UI 边界。

**通过条件**：安装的外置包不要求在 Codexhost 仓库内重新编译；不启动第二套 Agent Runtime。按实际 Web/PWA 浏览器完成上述验收并记录版本/步骤/结果。浏览器视觉验收不能由 TypeScript 构建或单元测试替代。

### 7. 文档与交付门禁

- 更新 `docs/product/web-ui-extensions.md`，只说明现行安装、启停、恢复与主题扩展能力；更新现有 UI Theme 产品说明（如独立文档已存在则维护它）以反映 styleId、本地偏好、契约版本、安全范围和回退行为。未交付的阶段继续标待实现。
- 更新 `docs/proposals/web-theme-extension-design.md` 的已定案包名、角色 allowlist、启动职责和与实施一致的契约；更新 `docs/index.md` 状态。删除或改写已被实际接口取代的占位/未决描述，不留相互矛盾的契约。
- 以受影响 Web Server 测试为起点，执行实现验证：
  - `npm run build:web:ui`
  - `node --import tsx --test packages/web-server/test/web-assets.test.ts packages/web-server/test/ui-extensions.test.ts packages/web-server/test/dsh-web-hot-loading.test.ts`（先按最新测试文件核实名称及依赖；需要集成的 HTTP 路由测试时一并加入）
  - `npm run test:web:distribution`（改动发行清单、打包入口或 runtime snapshot 时）
  - `npm run typecheck`、`npm run lint`（公共包、跨包契约和构建边界改动完成时）
- 对照设计基线 1–7 验收项复核行为、文档、发行资源和未实施声明。不能以编译成功或单个主题包激活代替偏好隔离、首屏一致性、恢复与卸载、多 Harness 和浏览器验收。

## 阶段依赖与完成定义

顺序为 Gate → 1 契约 → 2 偏好/runtime → 3 首屏 → 4 静态元数据/生命周期 → 5 设置 UI → 6 示例主题和浏览器验收 → 7 文档交付。阶段 4 可并行开发服务端解析和主题包 fixture，但其接口只能基于阶段 1 已冻结的契约；阶段 5 依赖阶段 2 的公开快照 API。

交付完成须同时满足：公开 API 与版本诊断可由独立外置包使用；主题范围限于语义 token；每浏览器 styleId 与明暗偏好正交保存；正常和恢复首屏无风格闪回；无效/禁用/卸载主题可回退且不污染其他运行状态；指定自动化检查通过；多浏览器、多 Harness 和窄屏 Web/PWA 验收完成；产品文档只陈述已经实现的能力。
