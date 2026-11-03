# ------------------------------------------------------------------
# README_AGENT — HMA-Random 的机器面文档
#
# 读者对象：LLM / coding copilot。人类读者请改用 README.md /
# README_ZH.md，本文件对人类没有阅读价值。
# 格式：YAML 方言。indent=2；键名是稳定 ID；`#` 注释是规范性
#       约束（normative），不是装饰。整个文件必须能被解析为
#       单个合法 YAML 文档（自检命令见 doc_protocol）。
# 阶段声明：当前为蓝图冻结阶段（2026-11-03），零代码。本文件
#       描述的全部是设计；实现进度只看 pending_work 看板。禁止
#       把计划写成现状；禁止重新提出 settled_decisions.rejected
#       中的任何条目——被否决即永久否决，除非 owner 以带日期的
#       新裁决显式重开。
#
# 第一规则（高于其余一切规则）：本项目的第一性原理是事实归一
# （truth normalization）：防御只发生在数据层，R 世界是系统里
# 唯一注册的事实。任何设计决策、任何代码变更、任何你（AI）想
# 提的新特性，都不得引入「对开放集合的查询拦截 / 结果过滤」；
# 翻译只允许发生在封闭集合（被隐藏应用自身 + 白名单）。推导不
# 出或与零拦截原则冲突的提案，默认拒绝，交 owner 裁决。
# ------------------------------------------------------------------

project:
  id: HMA-Random
  magisk_module_id: hma_random            # 小写加下划线；不得与 HMA-OSS 的 module id 冲突
  license: GPL-3.0                        # owner 裁决 2026-11-03；见 theory.rulings.2026_11_03_license_gpl3
  nature:
    - zygisk-module
    - privacy
    - identity-randomization
    - open-source
  core_metaphor: app == witness under protection   # 不是营销话术——三份文档的表格与术语都从它派生
  theory: truth-normalization                       # 宪法块，见 theory
  maturity: blueprint-frozen-zero-code              # 冻结于 2026-11-03；零代码；无发布；无安装
  ui_language: zh
  comment_language: zh-with-witness-metaphors       # 术语表见文件底部
  lineage: >-
    HMA-OSS 血统（同一份工作：别让其他应用知道我装了什么），不同的方式
    （拦截查询 → 改写真值）。零代码、零配置兼容是永久事实，不是待办——
    配置模型不同（身份映射 + 策略 vs 拦截规则），「导入 HMA 规则」永久
    不可实现。

doc_index:                                  # 三份文档的入口对齐
  zh: README_ZH.md
  en: README.md
  agent: README_AGENT.md                    # 本文件
  rule: >-
    人类文档讲原理、与 HMA-OSS 的关系、免责声明；本文件承载机器面全部
    设计事实。三份文档同步维护——人类文档与本文件冲突时，以本文件的
    裁决为准。

doc_protocol:                               # 本文件的维护规程
  update_triggers:
    - owner 裁决 → 追加 theory.rulings，带日期；历史裁决永不改写，只可被新裁决显式取代
    - pending_work 条目落地 → 事实移入对应 spec 块，从看板移除
    - 新暴露面发现或闭合 → 更新 exposure_register 状态行
    - 新的被否想法 → 追加 settled_decisions.rejected
    - 实现与本文件冲突 → 先改本文件（裁决），再改代码；顺序不可反
  yaml_selfcheck: "python3 -c 'import yaml; yaml.safe_load(open(\"README_AGENT.md\"))' — MUST succeed"

# ------------------------------------------------------------------
# THEORY — 事实归一（本项目的宪法。机制可以死，公理不可以。）
# ------------------------------------------------------------------
theory:
  origin: >-
    拦截式隐藏（HMA 路线）的困境是拦截点不可枚举：binder 直调、native 层、
    system uid 直读文件，永远在补洞。本项目反转防御位置：不拦观察者，
    改写真值——随机身份 R 就是系统里唯一注册的事实，于是任何查询路径
    （包括今天未知的）返回的都是干净数据。被拦截的集合从「不可枚举的
    观察者」收敛为封闭小集合：被隐藏应用自己 + 白名单。对开放集合零拦截。
  axioms:                                   # 公理：全部设计的推导起点
    A1_truth_inversion:                     # 公理 I：事实归一
      statement: >-
        真值只有一个：R 世界。PMS 注册、packages.xml、/data/app 目录名、
        签名注册、label、installer 记录，全部以 R 为准。
      forbids:
        - 双重真值（按观察者身份返回不同结果）
        - 对非白名单进程的查询结果过滤
        - 「部分字段归一、部分字段放行」的半吊子实现
      generates:
        - 未知查询路径自动干净（无需枚举拦截点）
        - 验收矩阵的判定标准——任何格子不得漏出 P
    A2_zero_interception:                   # 公理 II：零拦截
      statement: >-
        拦截 / 翻译只允许发生在封闭集合：被隐藏应用自身进程 + 白名单。
        对开放集合（其余一切进程）零 hook、零过滤、零响应改写。
      only_exception: >-
        system_server 响应层的元数据字段替换（假组件类名 / authority）。
        原因：dex 真实类名无法源级重写（APK 有签名，动不得），而 A3 要求
        假世界自洽——组件类名以 P 开头是纯泄露。此豁免点是零拦截原则
        唯一的弯腰处，必须保持最小、逐字段登记于 channel_ledger。
      generates:
        - 自身翻译层（出界 P→R / 入界 R→P）
        - 白名单中介
        - mount namespace 属于视图提供而非结果过滤，不违反本公理
    A3_full_consistency:                    # 公理 III：做戏做全套
      statement: >-
        每个出境谎言必须自洽且经得起全量交叉验证：S′ 可真验签、诱饵是
        可解析的真 APK、label / installer / mtime / 体积互洽。观察者做
        最深的交叉验证（PMS 元数据 ↔ 文件内容 ↔ 签名验证），得到的也
        是一个自洽的世界。
      forbids:
        - 空 tmpfs 诱饵（「存在遮蔽」本身可被 stat 探测）
        - symlink / hardlink 共享（inode 痕迹：S_IFLNK、readlink、同 inode、同哈希、nlink>1）
        - 大小 ≈ 0 的假 APK（真实 APK 是 MB 级）
        - sparse 文件（st_blocks 与 st_size 对不上即穿帮）
        - R / S′ 生成出一眼假的值（com.x7f3k2… 会标记「该设备有反画像模块」）
      generates:
        - 诱饵 APK 构建器（mini-apksigner，S′ 真签名）
        - 签名门（对「P 是否安装」的一切侧信道答案必须与注册事实一致：否）
        - R 从可信常见包名池生成；S′ 用 RSA-2048 自签、合理 DN 与有效期
    A4_fixed_roll:                          # 公理 IV：身份掷一次
      statement: >-
        R、S′、假组件名、假元数据一次生成、持久化、跨启动稳定。重新随机
        = 摧毁一切按名字索引的系统状态（授权、快捷方式、账户、AppOps）。
        固定性不是实现妥协，是设计本身。
      forbids:
        - per-boot 重随机
        - 用户触发的静默 re-roll（唯一合法形态：完整卸载 + 重新隐藏流程）
      generates:
        - 映射的版本化与原子写
        - 「掷一次，终身不变」作为用户文档第一屏语义
    A5_device_boundary:                     # 公理 V：设备边界
      statement: >-
        翻译的势力范围到设备边界为止。设备外钉死项（assetlinks、Play 账务、
        应用自家后端）只能本地绕过或诚实接受破坏，永不假装可翻译。
      forbids:
        - 假装能翻译服务器侧身份
        - 把「从今往后不可见」偷偷说成「从未安装过」
      generates:
        - 承诺语义（security_narrative.promise_semantics）
        - C 类对策清单（本地绕过 / 接受破坏）
  channel_ledger:                           # 身份通道账本——完备性检验
    # 设计任何新特性前必须过这张表：你的改动让 P 在哪条新通道出现？
    # 每条通道四选一：源级归一 / 遮蔽+诱饵 / 响应替换 / 诚实声明为残余。
    - channel: binder 查询（包名 / 签名 / 组件 / label）
      treatment: 源级归一（R + S′ + 假组件 + 假 label）
    - channel: 组件类名 / authority
      treatment: 响应替换（A2 唯一豁免）+ AMS 解析还原
    - channel: /data/app APK 本体（manifest 写 P、签名块是 S）
      treatment: 遮蔽 + 诱饵（overlay，见 decoy_spec）
    - channel: packages.xml / /data/app 目录名 / /data/data
      treatment: 源级归一（目录名 = R）
    - channel: /proc 进程名（保留 P 供自身自洽）
      treatment: 遮蔽（hidepid=2）+ 残余声明（dumpsys 侧信道）
    - channel: 历史档案（usagestats / appops / 权限 / 账户）
      treatment: 迁移（源级归一，见 migration_spec）
    - channel: 安装试探（stub 试装探测存在性）
      treatment: 签名门（结果与「未安装」不可区分）
    - channel: installer 记录 / 应用来源
      treatment: 源级改写为可信本地安装器名
    - channel: 设备外（assetlinks / Play / 自家后端）
      treatment: 诚实声明（A5；本地绕过或接受破坏）
  derivation_map:                           # 推导表：每个机制从哪条公理来
    签名门: "A1 对偶：「P 是否安装」的一切侧信道答案必须与注册事实一致"
    诱饵_APK: "A3：假世界必须能通过全量交叉验证"
    诱饵体积填充: "A3：StorageStatsManager（真实统计）与观察者 stat() 必须互洽"
    hidepid: "A2：视图提供而非结果过滤；进程名留 P 使自身 /proc/self 天然自洽"
    白名单中介: "A2：封闭集合成员取回真值（label / icon）"
    固定随机: "A4"
    自身翻译层: "A2：封闭集合——证人永远认为自己是 P"
    多用户照抄系统: "A1：全局包身份一个 P 一个 R，每 user 状态原样——最不起眼的伪装"
    管理器零日志纪律: "A3：logcat 里出现 P 字样 = 假世界穿帮（system uid 可读 logcat）"
  rulings:                                  # 公理裁决记录（向后一致性的证据）
    2026_11_03_fixed_roll: "身份掷一次，终身不变。拒绝 per-boot 重随机——它摧毁一切按名索引状态。"
    2026_11_03_signature_gate: >-
      安装中介判据 = 密码学验签（incoming 签名 == S，或带有效 V3 rotation
      证明链到 S）。拒绝交互式确认：后台静默更新发生在无人时刻，且对静默
      安装者，确认环节引入的时延本身就是差分信号（无需安装即可探测）。
    2026_11_03_stub_must_install: >-
      验签失败的安装（stub 探测 / 撞名）必须走全新安装路径真实落地。
      阻塞或改写其结果 = 重新打开 oracle，永久拒绝。配套：本地探测告警
      （通知不影响安装结果，对探测者零差分）。stub 会占用 P 名——真实
      更新按签名匹配，不受占用影响，此为设计使然。
    2026_11_03_update_verify_real_S: "升级校验基准 = 映射内真证书 S，不是注册的 S′——安全语义不变（仍验「是否原开发者签名」），只换呈现层。"
    2026_11_03_refuse_to_hide: "v1 拒绝隐藏两类应用：sharedUserId 应用（多包共享 uid 的映射一致性是灾难）；持有 signature 级权限的应用（权限授予即证书匹配）。"
    2026_11_03_decoy_is_real_apk: >-
      诱饵必须是真 APK：模块持有 S′ 的私钥（自己生成的），因此能正规签名
      ——manifest 编译为二进制 XML 写 package=R，S′ 真签名，实占盘填充至
      ≈ 真实体积，非 sparse，结构同构（base.apk + oat/ + lib/<abi>/ +
      splits），inode 唯一，nlink == 1，SELinux label = data_app_file，
      mtime 对齐 firstInstallTime / lastUpdateTime。
    2026_11_03_no_symlink: "「多路径共享一份存储」的直觉正确但必须在存储层实现（overlay / bind 一份物理诱饵目录，映射进各观察者命名空间）；文件层共享留 inode 痕迹，永久拒绝。"
    2026_11_03_process_name_P: >-
      进程名全局保留 P（自身 /proc/self/cmdline 天然自洽，省一层 hook）。
      观察者命名空间以 hidepid=2 重挂 /proc，他 uid 进程整体不可见。
      已知残余：DUMP 权限持有者经 dumpsys activity 可见进程名——hidepid
      不覆盖 dumpsys；缓解方向为 system_server 侧输出翻译，实现方式待裁决。
    2026_11_03_multiuser_follow_system: "照抄系统多用户语义：全局包身份（一个 P 恒对应一个 R），每 user 启用状态 / 权限原样保留，克隆用户同此。"
    2026_11_03_translation_reach: >-
      翻译分类学：A 类单向替换（label、installer、dumpsys 输出、历史 XML
      迁移——机械劳动）；B 类往返契约（假组件名→拉起还原、安装中介、自身
      身份——每个出境谎言都是一张合同：谁拿到假值，谁日后拿它回来时必须
      认）；C 类设备外钉死（assetlinks / Play / 自家后端——只能本地绕过
      或接受破坏，永不假装可翻译）。
    2026_11_03_two_layer_narrative: "安全叙事两层，不得混写：身份反转（本模块、结构性、可承诺）+ 反检测（mountinfo / 注入痕迹——继承自 Magisk 生态，best-effort，随生态水平浮动）。威胁模型与对外说明按此分层。"
    2026_11_03_initial_hide_separate: "初始隐藏迁移（生成映射、改 packages.xml、迁移数据目录）与安装中介（后续到达的 manifest=P 请求）是两个流程，不共用代码——语义完全不同。"
    2026_11_03_naming: >-
      定名 HMA-Random（Hide My Applist — Randomized）。否决 HMA-Aggressive
      （描述力度不描述机制；单字母后缀 R 有 Android 版本字母歧义）。
      module id = hma_random。发布前必须向 HMA-OSS 维护者
      （github.com/frknkrc44/HMA-OSS）开 issue 打招呼——社区礼仪，目标是
      换取其 README「相关项目」链接。
    2026_11_03_license_gpl3: >-
      许可证定为 GPL-3.0（owner 裁决）。随之移除人类文档中的非商业声明
      ——GPL-3.0 明确允许商业使用，与非商业条款并存构成矛盾。若未来重设
      非商业定位，须更换许可证并以带日期的新裁决显式取代本条。
    2026_11_03_noncommercial_stance: >-
      恢复非商业声明（owner 裁决；本条取代 2026_11_03_license_gpl3 中
      「移除非商业声明」及「重设须换许可证」两个分句，其余仍有效）。
      定位澄清：声明是项目立场，不是对被许可人的附加限制——作者不商业
      化、不取分文、不设赞助渠道、不提供双许可；法律层面的全部规则由
      GPL-3.0 独自承担，商业使用只要完整遵守 GPL-3.0 即为合法。立场与
      许可并存不悖：许可管别人能做什么，声明管本项目自己会做什么。
      参照系：E2EE-Experiment 以 AGPL-3.0 搭配同型立场声明运行。
      措辞须为 HMA-Random 自己的声音（ruling 2026_11_03_doc_voice），
      禁止照抄参照项目的句子。
    2026_11_03_stance_wording_restraint: >-
      非商业声明的对外措辞必须克制（owner 裁决）：人类文档只陈述许可
      义务、不授权项与项目立场，不主动宣传许可证赋予的自由（如商业
      分发的合法性）——那会读成邀请，与立场相悖。法律事实（商业使用
      遵守 GPL-3.0 即合法）仅记录在本文件（机器面）供贡献者核对，
      不得进入面向用户的表述。两份人类文档的许可条目已按此重写。
    2026_11_03_doc_voice: >-
      人类文档仅借鉴 E2EE-Experiment 的骨架结构（双语入口、AI 文档分流、
      诚实边界、状态纪律），行文不得模仿其句式与修辞。文档要有本项目
      自己的声音——结构是公共的，嗓音是身份。

# ------------------------------------------------------------------
# IDENTITY MODEL — 证人的两套身份
# ------------------------------------------------------------------
identity_model:
  witness: 被隐藏的应用（持有真身 P 与真签名 S）
  P: 原始包名——仅存在于模块私有映射与自身进程翻译中，系统里无任何残留
  R: 随机包名——注册事实；从可信常见包名池生成（不可一眼假），全局唯一，一次生成
  S: 真实签名证书——仅作升级验签基准（映射私有，永不外发）
  S_prime: 随机签名——RSA-2048 自签、合理 DN 与有效期；注册于 packages.xml；模块持有其私钥（诱饵签名用）
  fake_components: 假组件类名（R.a1、R.a2…）与假 authority——B 类往返契约，存于映射
  fake_metadata: 随机 label / icon、可信 installer 记录——与 S′ 同批生成
  map_storage: /data/adb/modules/hma_random/（root-only）；版本化、原子写（临时文件 + rename）
  map_scope: 全局（一个 P 恒对应一个 R）；每 user 应用状态不重映射

# ------------------------------------------------------------------
# ARCHITECTURE — 子系统与信任边界
# ------------------------------------------------------------------
architecture:
  substrate: >-
    Zygisk——zygote fork 出的每个进程在 specialize 前的注入窗口。system_server
    与厂商 app 同为 zygote 子进程，天然全覆盖（这是本方案可行性的地基）。
  subsystems:
    source_inversion: system_server 侧 PMS hook——注册名 R、签名 S′、假组件、packages.xml、目录名、label / installer、历史档案迁移
    self_translation: 被隐藏应用自身进程——出界 P→R（查询自身 / 组件解析 / PendingIntent / 自更新 / authority 字符串），入界 R→P（ApplicationInfo / cmdline / dataDir / 自身签名自检）；Java 层 + native 层（libc 级 /proc 自读）
    allowlist_mediation: 白名单进程（启动器 / 设置 / SystemUI）——进程内翻译取回真 label / icon
    fs_covering: 观察者进程 mount namespace——unshare(CLONE_NEWNS) + overlayfs + hidepid=2 /proc 重挂（见 mount_ns_spec）
    install_gate: system_server 侧安装中介——签名门（见 install_gate）
    decoy_builder: mini-apksigner——按映射为每个证人构建自洽诱饵 APK（见 decoy_spec）
    probe_alert: 本地通知——签名门捕获错误签名投递时提醒用户（不影响安装结果，零差分）
  trust_boundaries:
    closed_set:
      - 被隐藏应用自身进程
      - 白名单（启动器 / 设置 / SystemUI）
      - system_server（模块地盘）
    open_set: 其余一切进程——零 hook、零过滤，查询返回的就是注册事实

# ------------------------------------------------------------------
# INSTALL GATE — 签名门（B 类往返契约的核心实例）
# ------------------------------------------------------------------
install_gate:
  location: >-
    system_server 安装 / 扫描路径。覆盖 app 发起、adb install、pm install、
    一切第三方安装器——住在系统侧，不逐调用方适配。
  decision:
    - condition: incoming manifest 包名不在映射表
      action: 零介入，原生流程（对未隐藏名，模块等于不存在）
    - condition: 包名 == P（在映射表）且签名 == S，或带有效 V3 rotation 链到 S
      action: 翻译 P→R，以 S 验签，静默合并为 R 的更新
    - condition: 包名 == P（在映射表）且签名 != S
      action: 不翻译，按全新安装走（结果与「未安装」不可区分）+ 本地探测告警
  modes:
    auto: 推荐——真身更新自动翻译合并
    freeze: 拒绝一切合并，应用停在当前版本（失败仅对持有 S 签名方可见，不构成探测面）
  probe_alert_toggle: 独立开关，默认开
  residuals:
    - 会话元数据（appPackageName=P）在验签前窗口期对可枚举他人会话的特权观察者可见；commit 时重写，窗口留档为残余
    - stub 探测会真实落地并占用 P 名（ruling 2026_11_03_stub_must_install，设计使然）
  separation: 初始隐藏迁移不走本路径（ruling 2026_11_03_initial_hide_separate）

# ------------------------------------------------------------------
# DECOY SPEC — 自洽诱饵（A3 的文件层兑现）
# ------------------------------------------------------------------
decoy_spec:
  what: 每个证人一份自洽诱饵 APK——不是「看起来像 APK 的文件」，是真 APK
  construction:
    - manifest（二进制 XML）写 package=R
    - S_prime 正规 v2 / v3 签名（mini-apksigner）
  size: 实占盘填充至 ≈ 真实体积（StorageStatsManager 通道互洽）；禁止 sparse
  structure: 镜像真实安装目录——base.apk + oat/（odex / vdex）+ lib/<abi>/*.so + split APK
  invariants:
    - inode 唯一
    - nlink == 1
    - SELinux label == data_app_file（读诱饵不得触发 avc denial——logcat 异常本身是信号）
    - mtime 对齐 firstInstallTime / lastUpdateTime
    - 权限位与属主对齐真实 APK（0644 system:system）
  storage: >-
    一份物理存储于模块私有目录，overlay / bind 映射进各观察者命名空间——
    共享发生在存储层，不在文件层（ruling 2026_11_03_no_symlink）。
  sync: 可见应用的安装 / 卸载由 lowerdir 透传自动跟随，零维护；仅隐藏应用的影子需维护

# ------------------------------------------------------------------
# MOUNT NS SPEC — 观察者的文件系统视图
# ------------------------------------------------------------------
mount_ns_spec:
  window: zygote fork 后、app 代码运行前（Zygisk specialize 前）
  operations:
    - unshare(CLONE_NEWNS)
    - overlayfs 盖 /data/app（lowerdir=真实目录透传可见应用；upperdir=仅隐藏应用影子诱饵）
    - hidepid=2 重挂 /proc（他 uid 进程整体不可见——进程名 P 的遮蔽）
  hygiene:
    - 挂载完成后卸载 / 匿名映射注入痕迹（观察者 /proc/self/maps 不得留陌生 so）
    - fd 全关（/proc/self/fd 不得指向模块路径）
    - logcat 零 P 字样（system uid 可读 logcat，日志卫生是硬要求）
  coverage: 所有 zygote 子进程（含厂商 system uid app）
  uncovered: adbd、非 zygote 原生守护进程（init 子进程）——入档为残余，写进威胁模型
  known_hard_problem: >-
    mountinfo：观察者读自己的 /proc/self/mountinfo 可见 /data/app 上叠着
    overlay——原生系统那里没有任何挂载条目。无干净解（unmount 少条目、
    overlay 多条目），只能指纹弱化。归入反检测层（two_layer_narrative），
    持续借鉴 Magisk / Shamiko / HMA-OSS 社区军备，不写成保证。

# ------------------------------------------------------------------
# MIGRATION SPEC — 初始隐藏与历史档案
# ------------------------------------------------------------------
migration_spec:
  initial_hiding:
    - 生成映射（R / S′ / 假组件 / 假元数据）
    - 改写 packages.xml
    - 迁移数据目录（/data/data/P → /data/data/R，restorecon）
    - 历史档案全量改名（P→R）
    - 构建诱饵
  historical_archives:
    - usagestats（暴露概率最高——读取权限常见）
    - appops 历史
    - runtime-permissions
    - package-restrictions（preferred activities）
    - shortcutmanager
    - AccountManager 账户类型
    - SyncManager adapters
    - taskAffinity 记录（默认值 = 包名）
  acceptance: 一份「按包名索引的系统库清单」——漏一个就是一个泄露点
  enumeration_method: >-
    对 AOSP manifest 字段表走查（ParsingPackage / AndroidManifest schema），
    每字段三问：从包名派生吗？谁读它？往哪个方向跨界？
    战利品示例：taskAffinity 默认值 = 包名；android:process=":x" 展开为 P:x。

# ------------------------------------------------------------------
# SECURITY NARRATIVE — 两层承诺与范围
# ------------------------------------------------------------------
security_narrative:
  layer_1:
    name: 身份反转
    property: 结构性、可承诺
    promise_semantics: 此后设备侧枚举不可见；不承诺「历史上从未安装」（服务器侧档案不召回）
  layer_2:
    name: 反检测（mountinfo、注入痕迹、Magisk 痕迹）
    property: 继承自生态、best-effort、随生态水平浮动
    rule: 不得写成保证；威胁模型与对外说明按此分层
  out_of_scope:
    - root 级对手（映射 root-only）
    - 物理 / 恢复模式访问
  c_class_consequences:
    app_links: 本地绕过——验证器在 system_server（模块地盘），直接标 verified
    play_billing: 无法绕过，接受破坏——隐藏应用不走商店内购
    peer_handshakes: 对端校验 getCallingPackage() 得到 R 的协议会断；中介可按策略双向翻译，逐案处理

# ------------------------------------------------------------------
# EXPOSURE REGISTER — 暴露面登记（诚实账本，验收矩阵的骨架）
# ------------------------------------------------------------------
exposure_register:
  F01_manifest_embedded_P:
    what: 组件类名（com.real.app.MainActivity）、authority——包名反转后类名仍以 P 开头（dex 类就住在那个包路径下，改不了）
    who: 任何普通 app，标准 resolver API，零权限
    treatment: 假组件名映射 + system_server 响应替换（A2 唯一豁免）+ AMS 解析还原
    status: 已定设计，待实现（S2）
  F02_mountinfo:
    what: /data/app 上的 overlay 挂载条目——原生无此条目
    who: 任何 app（读自己的 mountinfo 零权限）
    treatment: 指纹弱化，无完美解
    status: 继承问题，反检测层，永久 best-effort
  F03_process_name_dumpsys:
    what: 进程名保留 P；dumpsys activity processes 对 DUMP 权限持有者开放（hidepid 不覆盖）
    who: 持 DUMP 权限的厂商 app
    treatment: system_server 侧输出翻译，或回到进程名改 R 的老权衡（裸 syscall 绕过）
    status: 部分缓解，实现方式待裁决
  F04_historical_archives:
    what: 隐藏前以 P 记录的 usagestats / appops / 权限 / 账户历史
    who: 持对应读取权限的 app（usage-stats 常见）
    treatment: 初始迁移全量改名
    status: 已定设计，待实现（migration_spec）
  F05_label_icon_installer:
    what: label / icon 是与签名同级的一等身份因子；installer 写 com.android.vending 而 Play 目录查无此包是软矛盾
    who: 任何查询者
    treatment: label / icon 与 S′ 同批随机化 + 白名单中介取回真值；installer 改写为可信本地安装器名
    status: 已定设计，待实现（S1）
  F06_injection_traces:
    what: 观察者进程 maps 里的陌生 so、指向模块路径的 fd、logcat 里的 P 字样
    who: 反篡改 SDK（检测自己的 /proc/self/maps 与 fd 是标准动作）
    treatment: 挂载后卸载 / 匿名映射、fd 卫生、日志卫生
    status: 可缓解，工程纪律项
  F07_server_side_archives:
    what: FCM token、云备份历史、广告 ID 关联——隐藏之前就已知道 P
    who: 服务器侧（设备外）
    treatment: 范围外，写进承诺语义（A5）
    status: 永久接受
  F08_failure_path_equivalence:
    what: 所有「P 未安装」响应必须与原生 NameNotFoundException 逐字节、逐时延一致（错误码 / 异常类型 / binder 状态）
    who: 一切探测者
    treatment: 测试要求，非设计项
    status: 验收矩阵判据（S9）
  F09_multiuser:
    what: work profile / 克隆用户中同一 app 的另一份安装
    who: —
    treatment: 照抄系统语义（全局身份 + 每 user 状态原样）
    status: 已裁决（ruling 2026_11_03_multiuser_follow_system）
  F10_functional_breakage:
    what: app-links 验签失败、Play billing 断、对端验名握手断、被隐藏 app 自身遥测上报「环境异常」
    who: 用户可见；应用自家服务器
    treatment: C 类对策 + 自洽性翻译通道清单做全（B 类合同）
    status: 永久清单式维护
  F11_manager_visibility:
    what: 模块管理器 app 自身可见 = 观察者得知「该用户运行隐藏模块」
    who: 任何 app
    treatment: 待裁决——管理器自身可随机化身份或并入策略层
    status: 开放，owner 裁决

# ------------------------------------------------------------------
# ACCEPTANCE MATRIX — 验收矩阵（本项目的最终考卷）
# ------------------------------------------------------------------
acceptance_matrix:
  shape: 观察者类别 × 通道 × 实际看到什么
  observers:
    - 普通 app（untrusted_app）
    - system uid app（厂商预置）
    - shell / adb
    - 平台组件（system_server 进程内直调）
    - 非 zygote 原生守护进程（init 子进程）
  channels:
    - binder 查询（包名 / 签名 / 组件 / label）
    - 文件读取（/data/app、packages.xml、/data/data）
    - /proc（cmdline / mountinfo / maps / fd）
    - dumpsys
    - intent 探测（深链 / 分享目标存在性）
    - 安装试探（stub 试装）
    - StorageStats（体积对账）
  pass_criteria:
    - 无任何格子漏出 P
    - 「未安装」响应与原生逐字节、逐时延一致
    - 探测 stub 的落地结果与「未安装设备」上的行为不可区分
  method: 合成观察者套件（普通 + system uid 双档）+ 现成检测套件采样

# ------------------------------------------------------------------
# PENDING WORK — 实现看板（落地一条，移除一条，事实进 spec 块）
# ------------------------------------------------------------------
pending_work:
  S1_identity_generator:
    scope: 映射与身份生成——R 池、S′ 密钥对、假组件名、假 label / icon、installer 改写值
    depends: []
  S2_source_inversion:
    scope: system_server PMS hook——注册反转 / 持久化 / 假组件响应替换 / AMS 解析还原 / 历史迁移
    depends: [S1]
    risk_note: PMS 内部结构每大版本重构（11→14 Computer / PackagesDatabase）——最高维护成本中心；boot 一致性 + 安全模式验证是硬要求
  S3_self_translation:
    scope: 自身进程翻译层——Java + native（libc 级 /proc 自读）
    depends: [S1]
  S4_allowlist:
    scope: 白名单中介（启动器 / 设置 / SystemUI 取回真值）
    depends: [S1]
  S5_fs_covering:
    scope: mount namespace + overlay 遮蔽 + hidepid
    depends: [S6]
  S6_decoy_builder:
    scope: mini-apksigner + 诱饵目录构建与同步
    depends: [S1]
  S7_install_gate:
    scope: 签名门安装中介 + 探测告警 + auto / freeze 两档
    depends: [S1]
  S8_manager_ui:
    scope: 管理器与策略（隐藏列表、允许拉起粒度、告警开关）；含 F11 管理器自身可见性裁决
    depends: [S1]
  S9_acceptance_bed:
    scope: 验收矩阵测试床（合成观察者 + 逐字节等价断言）
    depends: [S2, S3, S5, S7]
  parallelism_note: S2 / S5 / S7 相互独立可并行；S6 依赖 S1；S9 依赖全部

# ------------------------------------------------------------------
# SETTLED DECISIONS — 已决事项（rejected = 永久拒绝，除非 owner 带日期重开）
# ------------------------------------------------------------------
settled_decisions:
  rejected:
    - per-boot 重随机 / 任意形式的静默 re-roll（A4：摧毁按名索引状态）
    - 阻塞或改写 stub 安装结果（重新打开 oracle）
    - 交互式安装确认（无人确认 + 时延差分）
    - symlink / hardlink 共享诱饵（inode 痕迹）
    - 空 tmpfs 诱饵 / 大小 ≈ 0 的假 APK / sparse 填充（可探测）
    - 隐藏 sharedUserId 应用 / signature 级权限应用（v1）
    - 假装可翻译 C 类设备外钉死项
    - 项目名 HMA-Aggressive（描述力度不描述机制；单字母 R 有 Android 版本字母歧义）
  kept:
    - 定名 HMA-Random，module id = hma_random
    - 许可证 GPL-3.0
    - 非商业声明（立场型）与 GPL-3.0 并存——许可管法律，声明管立场；对外措辞克制：只列义务与不授权项，不宣传许可证赋予的自由（ruling 2026_11_03_noncommercial_stance + 2026_11_03_stance_wording_restraint）
    - 签名门 + 探测告警
    - 诱饵为真 APK（S′ 真签名、体积填充、结构同构）
    - mount namespace + overlay + hidepid 的全量文件遮蔽
    - 多用户照抄系统语义
    - 两层安全叙事（结构性承诺 / 继承性 best-effort 分层书写）
    - 发布前向 HMA-OSS 维护者打招呼（github.com/frknkrc44/HMA-OSS）
    - 人类文档只借骨架不借嗓音（ruling 2026_11_03_doc_voice）

# ------------------------------------------------------------------
# GLOSSARY — 术语表（本文件与人类文档共用）
# ------------------------------------------------------------------
glossary:
  witness: 证人——被隐藏的应用
  P: 真身包名（仅存于映射与自身翻译）
  R: 注册事实的随机包名（掷一次，终身不变）
  S: 真签名（仅升级验签基准）
  S_prime: 随机签名（注册 + 诱饵签名；模块持其私钥）
  observer: 观察者——开放集合中任何非白名单进程
  handler: 接应人——白名单进程（启动器 / 设置 / SystemUI）
  probe: 探测——用 stub 安装试探某包名是否被隐藏
  oracle: 差分信号——任何能区分「已隐藏」与「未安装」的可观察差异
  decoy: 诱饵——观察者命名空间里的自洽假 APK
  signature_gate: 签名门——安装中介的验签判据（S / V3 rotation）
  translation_reach: 翻译势力范围——设备边界为止（A5）
  class_A: 单向替换（出口谎言，做完即闭合）
  class_B: 往返契约（谎言必须能回来；每个假值都是一张合同）
  class_C: 设备外钉死（不可翻译；本地绕过或接受破坏）
