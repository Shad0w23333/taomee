# Taomee 项目模块梳理

本仓库是淘米网（Taomee）一系列网页游戏 / 运维平台的服务端代码归档，包含
多个游戏项目的后端、若干公共基础库和运维支撑系统。各模块均基于自研的
`Libtaomee` / `Libtaomee++` 网络框架（仓库根目录的 `Libtaomee-0.7.5.tar.gz`
与 `Libtaomee++-0.7.2.tar.gz`）开发，普遍采用 "父子进程 + 共享内存队列
（shmq）+ 动态库（.so）热加载" 的服务模型。

下面按目录逐一说明每个模块的作用、内部组成与典型用途。

---

## 0. 仓库根

| 文件 | 作用 |
| --- | --- |
| `Libtaomee-0.7.5.tar.gz` | C 版基础网络/工具库源码包，所有服务的底层依赖 |
| `Libtaomee++-0.7.2.tar.gz` | C++ 版基础库源码包，封装了协议、容器、定时器等 |
| `README.md` | 仓库简介 |
| `MODULES.md` | 本文，模块梳理文档 |

约定：每个服务子目录下常见的文件
- `bench.conf` —— 主程序配置（日志、共享内存、绑定 IP/端口、子进程数等）
- `bind.conf` —— 端口/接口绑定配置
- `startup.sh` / `restart.sh` / `daemon.sh` / `stop.sh` —— 启停与守护脚本
- `Makefile` 或 `CMakeLists.txt` —— 构建配置
- `bin/` —— 编译产物或 so 库
- `conf/` —— 业务配置（xml/lua）
- `gen_proto/` —— 客户端协议生成脚本/工具

---

## 1. `c/` —— 公共 C/C++ 基础库与规范

提供所有服务公用的底层基础设施。

- `library/` 各类基础库源码：
  - `async-serv/` 异步网络服务框架（事件驱动核心）
  - `net-io-server/` 多进程 IO 服务端骨架
  - `net-client/` 异步网络客户端
  - `mysql-iface/` MySQL 同步/异步访问封装
  - `ring-queue/` 进程间共享内存环形队列（shmq 底层）
  - `mmtree-1.0.0/` 内存映射 B+ 树（持久化 KV）
  - `ini-file/` ini 配置文件解析
  - `serverbench/`、`newbench/`、`newbench-beta/` 服务程序基准框架（多进程 fork + so 热加载模型）
  - `doc/`、`doxyfile.txt` Doxygen 文档配置
- `specification/` 公司技术规范：C/C++ 代码规范、数据库应用规范、资源管理与代码模板。

> 这是整个仓库的 "标准答案" —— 所有 game/运维服务都通过引入这些库
> 来组装出 Login / Online / Switch / DB 这四大类后端进程。

---

## 2. `common-tools/` —— 通用支撑工具集

跨项目共用的中间件和工具。

### `linux/`
| 子目录 | 作用 |
| --- | --- |
| `main_login/` | 通用主登录服（账号鉴权、签发 session/票据） |
| `gameproxy/` | 游戏接入代理（接外网，转发到内网游戏服） |
| `proxy/` | 通用 TCP 代理服务 |
| `anticheat/` | 反外挂模块 |
| `advert_fliter/` | 广告/敏感词过滤 |
| `postcard_ser/` | 明信片/卡片业务服务 |
| `ip_postcard_ser_v3/` | IP 维度的明信片服务 v3 |
| `proto/` | 公共协议定义 |
| `shterm/` | 堡垒机/SSH 终端审计相关（打包件） |
| `tools/core_detect/` | core 文件巡检工具 |
| `tools/online_monitor/` | 在线人数/服务监控小工具 |

### `win/`
| 子目录 | 作用 |
| --- | --- |
| `mailer/` | Windows 端邮件发送器 |
| `netstat-checker/` | Windows 端 TCP 连接巡检 |

---

## 3. `dbser/` —— DB Server 公共表操作库

游戏 DB 服务（dbserver/dbproxy）共用的核心库，编译后产出
`libdbser`。封装了大量"按规则把数据切到不同物理表"的策略类：

- `Ctable` / `CtableDate` / `CtableDate_100` / `CtableMonth`
  —— 按日期、按月份分表
- `CtableRoute` / `CtableRoute10` / `CtableRoute100` /
  `CtableRoute100x1` / `CtableRoute100x10` / `CtableRoute10x10`
  —— 按 uid 路由分表（10/100/100×10 等不同分桶粒度）
- `CtableString` / `CtableWithKey` —— 字符串主键 / 复合主键表
- `Cbig_cache` —— 大对象缓存
- `Citem_change_log` —— 道具变更流水
- `Csync_user_data` —— 玩家数据同步
- `Cfunc_route_*` / `proxy_route` —— DB Proxy 路由分发的命令字映射
- `mysql_iface` —— 基于 `c/library/mysql-iface` 的二次封装

> 是 `mole/dbsvr`、`gf/db`、`pea/dbserver`、`mole2/mole2_db`、`monster/db-server`
> 等所有 DB 服务的公共底盘。

---

## 4. `reload/` —— 动态 so 热更新示例

`reload_so` 机制的 demo：演示如何在父进程不重启的情况下，
让子进程重新加载业务 `.so`（`reload.so` + `data.so`），保留共享数据。

包含 `reload.cpp`（业务 so）、`data.cpp`（数据 so）、
`reload_so.c`（驱动程序）、`startup.sh`、`bench.conf`。
这是后续所有"线上无缝热更"用法的最小可运行样例。

---

## 5. `boke/` —— 摩尔庄园·博克岛 游戏后端

围绕"博克岛"小游戏的一套精简服务集群。

| 子目录 | 作用 |
| --- | --- |
| `login/` | 登录服（账号校验、分配 online） |
| `online/` | 在线/玩法服（地图、精灵、活动、道具、聊天接入） |
| `switch/` | 转发/分发服（dispatcher + dbproxy，负责请求路由） |
| `db/` | DB 服（按 uid 路由、找地图、日表等） |
| `chat/` | 聊天服与聊天过滤（CChatCheck/CChatForbid/CChatString） |

---

## 6. `mole/` —— 摩尔庄园 1 游戏后端（完整体）

最完整的一个游戏后端，所有典型组件齐备。

| 子目录 | 作用 |
| --- | --- |
| `login/`、`new_login/` | 老/新版登录服 |
| `online/` | 玩法服（最厚的业务模块，含天使战、超进化、活动、随机事件等海量玩法 .c 文件） |
| `gamesvr/` | 小游戏服（卡牌等小游戏的 so 加载与统计） |
| `homeserv/` | 家园服 |
| `cache_serv/` | 玩法缓存服（如餐厅 dining_room 等） |
| `switch/` | 转发服（dispatcher + dbproxy + 组播） |
| `dbsvr/` | DB 服与 DB Proxy 容器（`ser/` + `com/`） |
| `register/` | 注册服 |
| `picserv/` | 图片服（CGI 鉴权、上传/下载入口） |
| `httppic/` | HTTP 图片服务（含 `libservice_so` + `tcp_http`） |
| `school_bar/` | 校园吧业务服 |
| `ip_counter/` | IP 计数/反作弊辅助服 |
| `errreport/` | 客户端错误上报 CGI |
| `flashpolicy/` | Flash 跨域策略文件服务（843 端口） |
| `downloadstat/` | 下载量统计 CGI |
| `tools/` | item_check / online_login_test / protocol_test 等测试工具 |

各服务自带 `readme`，详见 `mole/gamesvr/readme`、`mole/online/readme`、
`mole/dbsvr/readme`。

---

## 7. `mole2/` —— 摩尔庄园 2 游戏后端

第二代摩尔庄园后端，引入 lua 脚本和跨服系统。

| 子目录 | 作用 |
| --- | --- |
| `mole2_login_new/` | 新版登录服（含组播心跳、报警、密码限频） |
| `mole2_online/` | 玩法服（活动、战斗、神兽 beast 等） |
| `mole2_home/` | 家园服 |
| `mole2_switch/` | 转发服（接入 leveldb 缓存） |
| `mole2_db/` | DB 服（基于 `pubser` + `libpubser.so`） |
| `mole2_cross/` | 跨服服务（job_dispatcher，跨服匹配/对战） |
| `batrserv/` | 战斗服（lua-5.1 脚本驱动战斗 AI） |

---

## 8. `gf/` —— 功夫派 游戏后端

功夫派（Gongfu）是品类更复杂的回合制 + 战斗 + 跨服游戏，模块也最多。

| 子目录 | 作用 |
| --- | --- |
| `main_login/` | 主登录（异步主登录接口库 + 实现） |
| `login/`、`gongfu_login/` | 游戏内登录服 |
| `gongfu_online/` | 玩法主服（成就、大使、活动配置、ap 排行榜等） |
| `home_svr/` | 家园服 |
| `gongfu_switch/`、`btl_switch/` | 通用转发 / 战斗匹配转发（含战斗房间、对战、排行榜） |
| `battle_svr/` | 战斗结算服（光环 aura、效果 effect、AI） |
| `chat_svr/` | 聊天服（玩家/群组/对象） |
| `cache/`、`cache_svr/` | 通用缓存层 |
| `db/` | DB 服（含 PHP 工具与服务程序） |
| `trade_svr/` | 交易服 |
| `mail_project/` | 邮件系统（mail_server + mail_dbproxy + mail_dbserver） |
| `kf_common/`、`kf_home/` | 跨服公共代码与跨服家园 |
| `deluser_svr/` | 玩家删号服务 |

---

## 9. `pea/` —— "豌豆"/赛尔号系战斗游戏后端

特点：登录/在线/战斗解耦明显，统一用 `bench.lua` / `dev.lua` 配置。

| 子目录 | 作用 |
| --- | --- |
| `login/` | 登录服 |
| `online/` | 在线服（含战斗接入 battle_switch） |
| `battle_server/` | 战斗主服（attack_obj、buff、bullet、battle_round 等完整回合战斗实现） |
| `battle_switch/`、`new_battle_switch/` | 战斗匹配/转发服（新旧两版） |
| `dbserver/` | DB 服（friends、extra_info 等） |
| `dbproxy/` | DB Proxy（带 `route.xml` 与 lua 配置） |
| `dirty/` | 脏字过滤服（dirty_agent） |
| `pea_common/` | 业务公共库（attr_config、effect_id、calculator、定时器等） |
| `gen_proto/` | 协议生成工具（gen_proto_app + plugin） |
| `bin/` | 公共可执行（AsynServ/ServerBench/reload_so 等） |
| `tools/` | 客户端模拟器、SQL 生成器 |

---

## 10. `monster/` —— 摩尔·小怪兽 游戏后端

对应文档 `monster/doc/小怪兽机器部署图.doc`。架构清晰，便于学习。

| 子目录 | 作用 |
| --- | --- |
| `login/` | 登录服 |
| `online/` | 在线/玩法服 |
| `multi-server/` | 多人玩法服 |
| `switch/` | 转发服（含 shop / everyday_fight 等业务路由） |
| `share-server/` | 共享数据服（c_server + cli_proto + db_proxy） |
| `db-server/` | DB 服（按业务子目录拆分：activity / badge / bag / day-restrict / denote …） |
| `db-cache-server/` | DB 缓存服 |
| `db-proxy/` | DB 路由代理 |
| `ucount-server/` | 计数服（埋点统计） |
| `dirty_server/` | 脏字过滤 |
| `thttpd/` | 内嵌 thttpd（32/64 位）做简单 HTTP 服务 |
| `common/` | 通用代码（pack、message、timer、mempool、stat 等） |
| `lib/` | `mysql-iface-1.0.1` 等第三方库副本 |
| `tools/` | gen_imitate / monitor / send_reload_conf / terminal 运维工具 |
| `test/` | 各模块对应的集成测试 |
| `doc/` | 文档与部署图 |

---

## 11. `pic_server/` —— 图片存储与服务集群

公司级图片云的全套服务，被各游戏与官网共用。

| 子目录 | 作用 |
| --- | --- |
| `fileserv/` | 文件存储服（FileServ，写入/读取原图，含 `change_thumb` 与 `fs_list`） |
| `thumbserv/` | 缩略图生成与读取服（ThumbServ） |
| `key_serv/` | 上传 key 鉴权服（`KeyServ`，发放上传凭证，mmap 维护索引） |
| `admin_serv/` | 管理服（`AdminServ`：删除/批量删除/改属性/集合操作） |
| `pic_cgi/` (`trunk/`) | 图片相关 CGI（cgi-bin / shell / src） |
| `pic_cgi_v2/` (`trunk/`) | 第二代图片 CGI |
| `FCGI_PIC/` (`trunk/`) | FastCGI 版本图片入口 |
| `2pic/` | 下一代 "2pic" 图片栈：`adminserv` / `fileserv` / `webproxy` / `include` |
| `pic_shell/` | 命令行客户端、批量迁移脚本 |
| `cgi_conf/` | CGI 配置（`bench.conf`、`crossdomain.xml`） |
| `include/` | 公共头（proto.h、error_nbr.h、fs_list.h、libpt 等） |

---

## 12. `itl/` —— OA 运维监控系统

公司内运维平台的全套后台。`itl/doc/` 中有完整设计文档：
《OA 后台服务概要设计》《OA-HEAD/NODE 模块细化与接口设计》
《OA 监控系统发布配置文档》《数据库管理模块协议设计文档》等。

| 子目录 | 作用 |
| --- | --- |
| `itl-head/` | 头节点：汇聚所有 itl-node 上报的指标，对外提供查询 |
| `itl-node/` | 采集节点：在被监控机上跑，采集指标 + 接管 db_mgr |
| `switch/` | 监控数据转发服（采集 + 指标分发） |
| `switch-monitor/` | switch 自身的存活监控（带插件机制） |
| `alarm/` | 告警服（含 `check_host_alive`，对接告警接口） |
| `control/` | 控制服（下发动作、db_mgr） |
| `db/` | itl 自己的 DB 服（plugin_interface 可扩展） |
| `rrd/` | RRD 时序数据库写入/查询服（c_rrd_handler） |
| `metric-so/` | 指标采集 so 插件 |
| `gen_proto/` | 协议生成器（与游戏端共用风格） |
| `misc-server/alarm-server/` | PHP 的告警入口（Rmail 邮件告警） |
| `misc-server/auto-update/` | PHP 的自动升级服务 |
| `misc-server/download-update/`、`update-url-server/` | 升级包下载 / 升级 URL 派发 |
| `itl-speed/` | 网络测速子系统（`tst_spd` + `dbser`） |
| `tool/` | 部署工具：`install-script`、`oa-head-installer`、`server-monitor`、`xml_peeper`、`poster`、`client` |
| `script/` | 全套数据库初始化与升级 SQL（`db.sql`、`db_init.sql`、`db_itl_v2.sql`、触发器等） |
| `bin/all_server.sh` | 一键启动脚本 |
| `alarm/alarm.sh` 等 | 各服务的运行脚本 |

---

## 模块之间的典型调用关系

以一个游戏（mole / gf / pea / monster / mole2）为例，常见拓扑：

```
   客户端
     │
     ▼
  [Login]  ── 鉴权 ──▶ [main_login / common-tools/main_login]
     │
     ▼
  [Switch / Dispatcher] ──▶ [Online / Home / Battle / Chat / Trade ...]
     │                              │
     │                              ▼
     │                       [Cache / ShareServer]
     ▼                              │
  [DB Proxy] ──▶ [DB Server] ──▶ MySQL  ◀── dbser (libdbser)
                                  │
              [pic_server / itl] ◀┘ 横向支撑：图片、监控、告警
```

通用支撑：
- 协议：各项目自带 `gen_proto/`，由 `c/library` 中的协议工具生成。
- 热更：所有 `*serv` 都基于 `reload/` 演示的 so 热加载模型。
- 监控：线上机器跑 `itl-node`，统一由 `itl-head` 汇聚到 OA 平台。
- 图片：游戏内图片资源走 `pic_server`（FileServ/ThumbServ/KeyServ/AdminServ）。
- 反作弊/过滤：`common-tools/anticheat`、`*/dirty*` 脏字服、`advert_fliter`。

---

## 这份代码的可借鉴经验

1. **统一的服务骨架**。所有服务复用 `c/library/serverbench` / `newbench`：
   父进程负责监听 + 共享内存队列，子进程加载 `.so` 跑业务，
   通过 `reload_so` 实现不停机更新；运维成本低且一致。
2. **DB 分表策略沉淀成库**。`dbser/` 把按 uid / 日期 / 月份 / 路由分表
   抽象为 `CtableRoute*` / `CtableDate*` / `CtableMonth` 等模板类，
   新业务接入只需选择策略。
3. **协议代码自动生成**。各项目自带 `gen_proto/`，xml/lua 描述协议→
   生成 C++ 头与序列化代码，避免手写 pack/unpack。
4. **配置文件惯例统一**。`bench.conf` + `bind.conf` 几乎是所有服务的
   入口；`log_dir`、`log_level`、`shmq_length`、`run_mode`、`worker_num`
   语义跨项目一致，方便排障。
5. **运维平台与业务解耦**。`itl/` 单独成体系，靠 `itl-node` + `metric-so`
   插件采集，业务代码无感知。
6. **图片服务独立成栈**。`pic_server` 走自己的 KeyServ/FileServ/ThumbServ
   分层架构（v2 / 2pic 两代演进），适合作为自建图床的参考样板。
7. **跨服与战斗逻辑可热插拔**。`mole2/batrserv` 用 lua 跑战斗 AI，
   `pea/battle_server` 把回合战斗拆成 attack_obj / buff / bullet /
   battle_round 等独立单元，便于扩展技能与效果。

> 如果想从中挑一个最小可运行体来读，推荐顺序：
> `reload/` → `c/library/newbench` → `mole/online` 与 `mole/dbsvr` →
> `monster/`（结构最规整）→ `gf/`（业务最复杂）。
