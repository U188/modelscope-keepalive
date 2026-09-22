# ModelScope 创空间保活

每 10 分钟 ping 两个 ModelScope 创空间，防止「无人访问 → 休眠」。

## 保活对象

| 站点 | 空间 | 说明 |
|---|---|---|
| 国内版 `modelscope.cn` | `u18888/cm` | 独立站点、独立账号 |
| 国际版 `modelscope.ai` | `u1888888/cm` | 另一套账号/数据，**别和国内版混** |

> 国内版和国际版是**独立站点、独立账号、独立数据库**。查/保活哪个站就用哪个站的域名和接口，
> 绝不能拿一个站的判据去套另一个站。

## 为什么休眠能靠 ping 解决

ModelScope 免费创空间的官方口径（`/docs/studios/resource-selections`）：

> 若实例一段时间未使用后，将进入休眠，**重新访问后即会激活启动**。

即判定依据是**请求闲置** —— 有访问就保持/唤醒。所以外部定时 ping 就够，
不需要浏览器端保活。（Notebook 是另一套机制，会话级判定，本仓库不管。）

## 两个 workflow

### `keepalive.yml` —— 保活主体

- 每 10 分钟（UTC）跑一次，ping 两个空间 + 查询平台接口确认状态
- 有 URL 没返回 200 会**让这一步失败**，GitHub 会发通知（不然坏了你也不知道）
- **HTTP 200 不是有效判据**：这些站的 SPA 对不存在的地址也返 200，
  所以另加一步查 `…/api/v1/studio/<user>/<space>` 看 `Data.Status`

### `keep-repo-active.yml` —— 防「60 天无活动自动禁用」

**这是关键。** GitHub 会在仓库 **60 天没有任何活动**时**自动禁用**该仓库的
scheduled workflow（状态变 `disabled_inactivity`，且不会自动恢复）。
保活全靠 schedule，一旦被禁就等于彻底失效。

前车之鉴：`is361/modelscope-keepalivess` 曾经正常跑了 **744 次**，最后 push 是
`2026-05-29`，60 天后 `2026-07-28` 被自动禁用 —— 最后一次运行正是那天。

所以这个 workflow **每月 1 号提交一次 `heartbeat.txt`**，制造仓库活动，让 60 天计时器归零。
月度（≈30 天）< 60 天，能自我维持。

## 手动检查清单

```bash
# 1. workflow 有没有被禁用（罪魁祸首常在这）
gh api /repos/U188/modelscope-keepalive/actions/workflows \
  --jq '.workflows[] | .name + "=" + .state'
# 期望 active；出现 disabled_inactivity 就是被 60 天规则杀了

# 2. 最近运行记录（有没有按 10 分钟节奏在跑）
gh run list -R U188/modelscope-keepalive -L 10

# 3. 空间此刻是否活着（分域查，别跨站套判据）
curl -s -A 'Mozilla/5.0' "https://www.modelscope.cn/api/v1/studio/u18888/cm"   | python3 -m json.tool | head -20
curl -s -A 'Mozilla/5.0' "https://www.modelscope.ai/api/v1/studio/u1888888/cm" | python3 -m json.tool | head -20
```

## 加更多空间

编辑 `keepalive.yml` 的 `SPACE_URLS` 数组即可（数组是 bash 特性，`shell: bash` 不能删）。
别加太多 —— 每次运行串行跑 N 个 curl，而 cron 本来就是 10 分钟一次。

## 已知限制

- **GitHub cron 是尽力而为**：名义 10 分钟，高峰期实测会延迟到 15 分钟以上，偶尔会丢。
  要求更精准就换外部监控服务（cron-job.org / UptimeRobot 这类，5 分钟粒度）。
- 免费的 fork 仓库 schedule 触发不可靠，所以本仓库是**普通仓库**，不是 fork。
- ping 只能防「请求闲置」型休眠；平台写的硬性时长上限（如 Notebook 的单次运行上限）保活救不了。
