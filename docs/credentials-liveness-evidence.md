# 凭证存活实测记录

> 用途：回复 issue #15 中 @xinbo888 关于「App 内切换登录 → 所有凭证立刻失活」的观察。
> 数据全部来自生产环境原始日志，未做删改；**不含手机号、app_token、jwt 等任何凭证**。

---

## 一、观测方案

2026-10-21 12:04 起在生产环境并行跑三个任务：

| 组件 | 频率 | 作用 | cron |
|---|---|---|---|
| `cmcc_keepalive.py` | 每 15 分钟 | 只做 auth 兑换，不碰业务接口 | `*/15 * * * *` |
| `cmcc_app_token_probe.py` | 每小时 | **纯 app_token、完全不带 jwt** | `13 * * * *` |
| `cmcc_sign.py` | 每日 07:00 | 主签到 | `0 7 * * *` |

**为什么探针要绕开 jwt**：主保活第一档在服务端放开时必然成功，会掩盖 app_token 自身状态。
只有纯 app_token 探测才能回答「app_token 本身还活着吗」。

签到侧采用三档凭证策略，逐级回退：

```
jwt + app_token  →  jwt 单独  →  纯 app_token 引导
```

## 二、方案期结果

观测区间 **2026-10-21 12:04:03 ~ 2026-10-22 10:45:00**

| 项目 | 值 |
|---|---|
| keepalive 运行次数 | **93** |
| keepalive 失败次数 | **0** |
| 探针探测次数 | **21** |
| 探针失败次数 | **0** |
| 探针连续时长 | **16:55:19** |
| 三档策略生效后签到次数 | **17** |
| 其中续期失败 | **0** |
| 三档期命中档位 | `jwt+app_token` ×17 |

**三档策略上线后，17 次签到续期全部成功，零失败**，全部命中 `jwt+app_token` 档。

## 三、基线对照（方案部署之前，同一账号）

这段时间只有每日一次签到会碰凭证，是「闲置多久会死」的直接对照：

| 日期 | 时间 | 结果 | 命中档位 | 凭证链龄 |
|---|---|---|---|---|
| 2026-10-10 | 09:50 | 成功 | 未使用 app_token | 0.0 天 |
| 2026-10-11 | 08:23 | 成功 | 未使用 app_token | 1.0 天 |
| 2026-10-12 | 07:00 | 成功 | 未使用 app_token | 1.9 天 |
| 2026-10-13 | 07:00 | 成功 | 未使用 app_token | 2.9 天 |
| 2026-10-14 | 07:00 | 成功 | 未使用 app_token | 3.9 天 |
| 2026-10-15 | 07:00 | 成功 | 未使用 app_token | 4.9 天 |
| 2026-10-16 | 07:00 | 成功 | 未使用 app_token | 5.9 天 |
| 2026-10-17 | 07:00 | 成功 | 未使用 app_token | 6.9 天 |
| 2026-10-18 | 07:00 | 成功 | 未使用 app_token | 7.9 天 |
| 2026-10-19 | 07:00 | 成功 | 未使用 app_token | 8.9 天 |
| 2026-10-20 | 07:00 | 成功 | 未使用 app_token | 9.9 天 |

- 连续 **11** 次日签到**全部成功**
- 相邻间隔 **22.5 ~ 24.0 小时**（约 24 小时）
- 链龄单调递增 **0.0 → 9.9 天**，跨度 9.9 天

**结论：统一认证会话被闲置约 24 小时并不会失效。** 这段时间走的还是「jwt 单独、不带 app_token」
那一档，说明纯 jwt 续期在服务端开口时，长时间闲置也能扛住。

## 四、10-21 07:07 那次失败的归因

失败发生在当日签到，间隔 24.0 小时 —— **与前 9 次成功的间隔完全相同**，
所以不是「闲置更久导致的」。同期 @yupaiLy 判断是「官方把 jwt 登录的口子给堵了」，
@jiangnanan 也报告「后台放开了」后又「又给关了」。我们 09:00 重新抓包换 token 后即恢复。

方案期唯一一次「jwt 续期失败」发生在 15:43，当时跑的还是**旧的两档代码**
（jwt 失败 → 回落 appTokenLogin 引导 → 活动会话建立成功 → 签到正常完成），
三档策略 15:45 才上线，之后再无失败。

## 五、探针逐条记录

```
{"ts": "2026-10-21 17:17:42", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 11.31, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261021T171742533<REDACTED>"}
{"ts": "2026-10-21 17:20:03", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 11.31, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261021T172003603<REDACTED>"}
{"ts": "2026-10-21 17:21:25", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 11.31, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261021T172125627<REDACTED>"}
{"ts": "2026-10-21 18:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 11.35, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261021T181301568<REDACTED>"}
{"ts": "2026-10-21 19:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 11.39, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261021T191301530<REDACTED>"}
{"ts": "2026-10-21 20:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 11.43, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261021T201301568<REDACTED>"}
{"ts": "2026-10-21 21:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 11.47, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261021T211301547<REDACTED>"}
{"ts": "2026-10-21 22:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.02, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261021T221301548<REDACTED>"}
{"ts": "2026-10-21 23:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.06, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261021T231301580<REDACTED>"}
{"ts": "2026-10-22 00:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.1, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261022T001301581<REDACTED>"}
{"ts": "2026-10-22 01:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.14, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261022T011301657<REDACTED>"}
{"ts": "2026-10-22 02:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.19, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261022T021301693<REDACTED>"}
{"ts": "2026-10-22 03:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.23, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261022T031301625<REDACTED>"}
{"ts": "2026-10-22 04:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.27, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261022T041301541<REDACTED>"}
{"ts": "2026-10-22 05:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.31, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261022T051301561<REDACTED>"}
{"ts": "2026-10-22 06:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.35, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261022T061301581<REDACTED>"}
{"ts": "2026-10-22 07:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.39, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261022T071301604<REDACTED>"}
{"ts": "2026-10-22 08:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.44, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261022T081301572<REDACTED>"}
{"ts": "2026-10-22 09:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.48, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261022T091301543<REDACTED>"}
{"ts": "2026-10-22 09:40:55", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.5, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261022T094055956<REDACTED>"}
{"ts": "2026-10-22 10:13:01", "mode": "app_token_only", "ok": true, "code": "SUCCESS", "msg": "成功", "jwt_issued": true, "chain_days": 0.52, "sid_prefix": "/appTokenLogin?sid=QWHDSSOD20261022T101301548<REDACTED>"}
```

`sid` 已打码：该值来自服务端 loginPath 查询串，每次请求重新生成（21 次探测 21 个不同值），不是客户端持有的凭证。

## 六、诚实的边界

- 我们**没有**在 App 内切换过登录账号，因此**没有复现** @xinbo888 描述的现象，
  本记录**不构成对其论断的否证**。
- 方案期保活间隔为 15 分钟，期间会话始终「热」，所以单看方案期无法区分
  「保活救了它」与「本来就不死」—— 闲置门槛的结论来自第三节基线数据。
- 探针 21 次全 SUCCESS 说明当前 app_token 链路健在，但只覆盖约 16:55:19 的窗口，
  更长窗口（数天/数周）尚无数据。

## 七、想请教 @xinbo888 的问题

1. 切换登录后是**全部凭证**立刻失效，还是只有部分（jwt 死但 app_token 还活）？
2. 三档策略在切换登录后是否还能命中任一档？我们的回退链是
   `jwt+app_token → jwt 单独 → 纯 app_token 引导`，如果切完登录后连纯 app_token 引导都失败，
   那就只有重新抓包一条路。
3. 切换登录后**多久**会失效？是当场，还是有延迟？
4. 你们是否观测到「云盘 token → SSO 置换 app token」这条路径可用？

## 八、复现方式

```bash
# 方案期统计（青龙）
0 7 * * *     task cmcc-auto-checkin/cmcc_sign.py --tasks --games --wed --delay 1800
*/15 * * * *  task cmcc-auto-checkin/cmcc_keepalive.py
13 * * * *    task cmcc-auto-checkin/cmcc_app_token_probe.py
```
