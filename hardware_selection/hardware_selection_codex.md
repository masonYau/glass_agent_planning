\# 确认眼镜硬件选型



> 请根据commercial\_plan中的商业计划，结合hardware\_selection中我和gemini的对话，再帮我确认一下眼镜硬件的选型



<details><summary>4 previous messages</summary>



> 我先把 `commercial\_plan` 和 `hardware\_selection` 里的材料读一遍，提炼出商业约束和你们已经讨论过的硬件取舍；如果涉及具体模块可买性或参数，我会顺手核一下当前资料，避免按过期印象拍板。

>

> 我看到工作区里正好有两个输入：商业计划是一个 HTML 提案，硬件讨论是一份文本。我现在把它们拆成可读内容，先看目标用户、定价、MVP边界，再看 Gemini 对话里的器件判断。

>

> 商业计划里已经有很明确的产品约束：先做 B 端现场记录，不追求全天候录像，眼镜端要轻量采集，手机/云端承担主要处理。接下来我会重点核对“Rokid 优先”这个结论是否还站得住，以及备选设备该怎么排。

>

> <details><summary>Searched the web</summary>

>

> Searched the web for `Rokid Glasses AI smart glasses developer SDK camera video stream CXR SDK official`

>

> Searched the web for `INMO Air 2 developer SDK official Android ADB camera smart glasses`

>

> Searched the web for `site:inmoglass.com INMO Air 2 developer SDK Android`

>

> Searched the web for `https://x-docs.rokid.com/docs/terminal-sdk/`

>

> Searched the web for `https://support.inmoxr.com/air2/guides/developer-mode/`

>

> Searched the web for `小米 AI 眼镜 官方 产品页 SDK 开发者 摄像头 接口`

>

> Searched the web for `Rokid Glasses official specs weight camera price Snapdragon AR1`

>

> Searched the web for `https://glasses.rokid.com/profile`

>

> Searched the web for `https://x-docs.rokid.com/docs/terminal-sdk/getting-started/%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97.html`

>

> </details>

</details>



结论：我建议确认 \*\*Rokid Glass3 / Rokid Sprite Enterprise 开发套件为主选\*\*。更准确地说，不要只买普通消费版“乐奇 AI 眼镜”就开始做 MVP，优先拿 \*\*带 SDK 权限、Demo、调试线/ADB、商务支持的企业/开发者版本\*\*。



理由很清楚：你的商业计划核心是 B 端现场记忆，不需要全彩 3D AR，而是需要第一视角采集、录音/录像、手机中继、云端入库、证据回放。商业计划里也已经把硬件路线定为 Rokid 优先，并要求验证 SDK、摄像头、录音、录像、推流和手机连接能力：\[ai\_glasses\_memory\_proposal.html](./commercial\_plan/ai\_glasses\_memory\_proposal.html:1453)。Rokid 官方 Glass3 SDK 也正好覆盖手机端/眼镜端协同、媒体采集、语音、AI、消息文件传输、P2P、视频预览等能力，和你的 MVP 链路贴得最紧。官方资料见 \[Rokid Glass3 SDK 概览](https://x-docs.rokid.com/docs/terminal-sdk/getting-started/%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97.html) 和 \[Demo 运行指南](https://x-docs-dev.rokid-inc.com/docs/downloads/demo-guide.html)。



\*\*推荐采购/借测组合：\*\*



| 优先级 | 硬件 | 用途 | 判断 |

|---|---|---|---|

| 主选 | \*\*Rokid Glass3 / Sprite Enterprise\*\* | 2 台，作为 MVP 主开发和试点机 | 最匹配：轻量、带摄像头/麦克风/HUD、SDK 文档和双端 Demo 完整 |

| 备选 1 | \*\*INMO Air2 / Air3\*\* | 1 台，验证标准 Android/全彩 AR 路线 | 可作为备胎，但 Air2 开启 ADB 有官方风险提示，且更偏全彩 AR 终端 |

| 备选 2 | \*\*RayNeo X3 Pro\*\* | 预算允许时借测 | 硬件强、支持 Unity/Android ARDK，但价格高，更偏 6DoF/SLAM/AR 应用 |

| 暂不主推 | 小米 AI 眼镜 | 只做消费级体验参考 | 产品硬件不错，但开发接口/眼镜端 SDK 开放度不如 Rokid 明确 |

| 暂不主推 | Brilliant Labs Frame | 算法/极客原型 | 极开放，但 720p、BLE、主机依赖强，不适合作为 B 端试点主硬件 |

| 排除 | XREAL Air 2 Ultra / Even G1 / INMO GO3 | 不适合当前主线 | XREAL 官方注明不支持录像拍照；Even/G1/GO3 无摄像头，不满足记忆采集 |



\*\*拍板条件：\*\*采购前让 Rokid 商务/开发者支持明确确认 5 件事：能否拿到 Glass3 企业版 SDK；能否远程触发拍照、录像、录音和实时预览；能否稳定通过 Android 手机中继取文件/流；是否支持物理按键或语音触发 30-120 秒事件片段；是否能采购或借测 2-3 台并提供调试资料。



一句话版：\*\*主硬件选 Rokid Glass3 Enterprise/Sprite Enterprise；INMO 或 RayNeo 作为第二供应商验证；小米、Frame、XREAL 不作为当前商业 MVP 主线。\*\*



> 我更像做的是一个生活全天候的记录，能够把生活和工作中的关键时段完整记录下来，也能把通勤居家等闲暇时段做轻度记录，转换成文本，并关联相关视觉/听觉记录。可能需要实现的重点功能包括：1、自动识别关键场景开始时刻；2、视频图片的频繁记录与压缩，并回传手机/云端，需要全天候低功耗实现这些功能；3、用户通过手机端/眼镜端与云端智能体对话，云端智能体通过RAG检索召回记忆文本回答用户问题，实现用户的贴身智能助理；4、常见场景包括：询问几日前就医时医生告诉的用药方法、询问耳机在哪里丢失、几天前的会议的具体数字细节等。



<details><summary>2 previous messages</summary>



> 这个补充很关键：你现在的目标已经从“B 端现场片段记录”变成了“个人全天候记忆层”。我会按这个新目标重新评估硬件，重点不再是 AR 显示炫不炫，而是低功耗采集、佩戴舒适、摄像头/麦克风开放、手机中继和隐私边界。

>

> <details><summary>Searched the web</summary>

>

> Searched the web for `Rokid Glasses official developer SDK camera audio video stream battery weight`

>

> Searched the web for `Meta Ray-Ban smart glasses official developer SDK camera access recording duration`

>

> Searched the web for `小米 AI 眼镜 官方 参数 电池 录像 时长 摄像头 骁龙 AR1`

>

> Searched the web for `Brilliant Labs Frame official specs battery camera audio developer SDK`

>

> Searched the web for `https://docs.brilliant.xyz/halo/`

>

> Searched the web for `site:docs.brilliant.xyz/halo Brilliant Halo SDK hardware developer docs camera audio`

>

> Searched the web for `Mentra smart glasses official SDK camera microphone battery open source`

>

> Searched the web for `Omi Glass Dev Kit official full-day camera recording battery open source specs SDK`

>

> </details>

</details>



这会明显改变硬件选型。你现在做的不是“现场记录设备”，而是 \*\*个人全天候记忆采集层\*\*。核心矛盾变成：眼镜不能一直高清视频录像，但必须一直低功耗感知，并在关键时刻升级采集强度。



我的更新结论是：



\*\*主线不应只押 Rokid。\*\*

更合理的是做“两条硬件线”：



| 路线 | 推荐硬件 | 角色 |

|---|---|---|

| 开发验证主线 | \*\*Rokid Glass3 Enterprise / Sprite Enterprise\*\* | 国内可落地、SDK 明确、验证采集链路 |

| 全天候记忆主线 | \*\*Mentra Live / Omi Glass Dev Kit\*\* | 更贴近 lifelogging、开源、低功耗持续采集 |

| 消费级参考 | 小米 AI 眼镜 / Ray-Ban Meta | 学习形态、续航、交互，但不作为首版开发主硬件 |



\*\*为什么不是简单选 Rokid？\*\*

Rokid 仍然适合第一版 MVP，因为官方 Glass3 SDK 明确支持手机控制眼镜、媒体采集、音视频预览、文件传输、语音和 AI 能力，且有 P2P 手机中继链路，和你要的“眼镜采集、手机回传、云端 RAG”很匹配：\[Rokid Glass3 SDK](https://x-docs.rokid.com/docs/terminal-sdk/getting-started/%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97.html)。硬件上也有 49g、12MP 相机、4 麦、Wi-Fi 6、BT 5.3：\[Rokid specs](https://global.rokid.com/products/rokid-glasses)。



但 Rokid 的电池只有 210mAh，适合 \*\*事件触发式记录\*\*，不适合真正全天候高频视频流。你这个产品必须设计成：



1\. 平时：音频/VAD/低频图像/手机传感器轻量记录

2\. 疑似关键场景：提高图片频率，开始 ASR，抽关键帧

3\. 明确关键场景：录 30 秒到数分钟视频，完整上传

4\. 闲暇场景：低频摘要，不保留大量原始视频



\*\*更贴近你目标的硬件是 Mentra Live 和 Omi Glass。\*\*

Mentra Live 官方定位就是带相机、扬声器、3 麦、开放 SDK 的开发/企业眼镜，43g，宣称 12+ 小时混合使用，260mAh 眼镜电池加 2200mAh 充电盒，支持 Android/iOS app 控制相机、麦克风、触控和按键：\[Mentra Live](https://mentraglass.com/live)。

Omi Glass Dev Kit 更直接打“full-day camera recording / lifelogging / AI memory”，会每隔几秒拍照并送到 Omi App 做记忆，但目前更像开发套件，需接受装配、供货和稳定性风险：\[Omi Glass Dev Kit](https://www.omi.me/products/omi-glass-dev-kit)。



\*\*我的采购建议：\*\*



第一批不要只买一种。建议：



| 数量 | 设备 | 用途 |

|---:|---|---|

| 2 台 | Rokid Glass3 Enterprise / Sprite Enterprise | 国内 SDK、手机中继、完整 MVP 主链路 |

| 1 台 | Mentra Live | 全天候轻量记录可行性验证 |

| 1 台 | Omi Glass Dev Kit | lifelogging / 开源记忆系统参考 |

| 可选 | 小米 AI 眼镜 | 消费级续航和形态参考，前提是能拿到开发接口 |



小米 AI 眼镜形态很好，40g、12MP、5 麦、263mAh、官方宣称 8.6 小时使用：\[Xiaomi AI Glasses](https://www.mi.com/global/discover/article?id=5172)。但开发接口不够确定，所以现在只能作为未来合作或消费化参考，不建议押它做首版。



一句话：\*\*Rokid 做可控 MVP，Mentra/Omi 验证全天候记忆范式，小米/Ray-Ban 做消费产品参考。\*\*

你的产品算法上应坚持“音频常开、视觉低频、事件升频、视频短段、证据可回放”，否则任何眼镜都会被续航、发热、隐私和存储成本拖垮。



> 针对我的需求对比乐奇、小米、Mentra Live的关键参数与开发者套件的功能支持



<details><summary>2 previous messages</summary>



> 我会把这三款按你的“全天候记忆”需求重新拉一张工程选型表，重点看 6 件事：佩戴/续航、摄像头、麦克风、连续/低频记录、手机/云回传、SDK 是否能让我们真正控制采集链路。

>

> <details><summary>Searched the web</summary>

>

> Searched the web for `Rokid Glasses official specs 49g 12MP 4 microphones battery SDK Glass3`

>

> Searched the web for `Mentra Live official specs 43g 12 hours camera microphones charging case`

>

> Searched the web for `site:mi.com/prod/xiaomi-ai-glasses 40g 263mAh 12MP 5 microphone 小米 AI 眼镜`

>

> Searched the web for `https://www.mi.com/prod/xiaomi-ai-glasses?spmref=MiShop\_PC.mihomepage.mtl\_68268f9e352df300011b9422.1`

>

> Searched the web for `https://www.mi.com/prod/xiaomi-ai-glasses?spmref=MiShop\_PC.mihomepage.mtl\_68268f9e352df300011b9422.1`

>

> Searched the web for `site:xiaoai.mi.com 小米 AI眼镜 开发者 SDK`

>

> Searched the web for `https://mentraglass.com/live`

>

> Searched the web for `https://github.com/Mentra-Community/Mentra-Bluetooth-SDK-Starter-Kit`

>

> Searched the web for `https://cloud-docs.mentra.glass/sdk/hardware-modules`

>

> Searched the web for `https://x-docs.rokid.com/docs/terminal-sdk/getting-started/%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97.html`

>

> Searched the web for `https://global.rokid.com/products/rokid-glasses?gad\_source=1`

>

> Searched the web for `https://global.rokid.com/en-jp/pages/academy`

>

> </details>

</details>



\*\*结论先行\*\*

针对“生活全天候记忆”这个方向，我会把三者排序为：



1\. \*\*Mentra Live：最适合验证全天候低功耗记忆采集\*\*

2\. \*\*乐奇 / Rokid Glass3 Enterprise：最适合做国内可控 MVP 和眼镜端交互\*\*

3\. \*\*小米 AI 眼镜：硬件形态很好，但目前不适合作为开发主硬件，除非拿到官方合作 SDK\*\*



\*\*关键参数对比\*\*



| 项目 | 乐奇 / Rokid Glasses | 小米 AI 眼镜 | Mentra Live |

|---|---|---|---|

| 定位 | 带 HUD 显示的 AI 眼镜 | 消费级无屏 AI 拍摄眼镜 | 开源开发者/企业 AI 眼镜 |

| 重量 | 49g | 40g | 43g |

| 显示 | 双目 Micro LED，单绿光，30° FOV | 官方资料未强调 HUD，偏无屏形态 | 无显示 |

| 摄像头 | 12MP Sony IMX681，最高 3K 视频 | 12MP Sony IMX681，支持视频/直播 | 119° FOV，1080p 视频，3264×2448 图片 |

| 麦克风 | 4 麦阵列 | 5 麦阵列，含骨传导 Mic | 3 麦 |

| 电池 | 210mAh；官方配件支持充电盒/胶囊电池 | 263mAh，官方称最高 8.6 小时使用 | 260mAh 眼镜 + 2200mAh 充电盒；12+ 小时混合使用 |

| 连续视频 | 更适合短视频/事件片段，不适合全天录像 | 官方支持录像/直播，但 SDK 不明 | 官方称视频录制 1+ 小时、直播 40+ 分钟 |

| 连接 | Wi-Fi 6、BT 5.3、P2P 手机中继 | 小米手机生态协同、跨设备相机 | Wi-Fi + Bluetooth，Android/iOS 均支持 |

| 价格 | 海外 $699；国内需看企业版报价 | 国内 1999 元起 | $349，但官方当前主要美国发货 |

| 开发开放度 | 高，尤其 Glass3 Enterprise SDK | 低/未知 | 很高，开源 SDK 明确 |



\*\*开发者套件能力对比\*\*



| 能力 | 乐奇 / Rokid Glass3 SDK | 小米 AI 眼镜 | Mentra Live SDK |

|---|---|---|---|

| 手机连接眼镜 | 支持，Phone SDK 负责蓝牙/P2P/消息/文件 | 消费级 App 和小米互联支持，但无明确开放 SDK | 支持，Android / iOS / React Native Bluetooth SDK |

| 眼镜端应用 | 支持眼镜端 Android 应用 | 未见公开眼镜端开发文档 | 主要由手机 App / MentraOS 控制眼镜，不强调眼镜端原生 App |

| 远程拍照 | 支持 | 消费功能支持，开发接口不明确 | 支持 `requestPhoto()` |

| 录像/视频预览 | 支持录像、音视频流、实时预览、P2P 传输 | 消费功能支持录像/直播，开发接口不明确 | 支持视频流、RTMP/SRT/WebRTC、本地媒体上传 Demo |

| 录音/音频流 | 支持录音、ASR/TTS、AI Chat | 5 麦硬件强，但原始音频接口不明确 | 支持麦克风事件、PCM/LC3、音频播放、本地转写 |

| 按键/触控 | 支持远程控制、按键自定义相关能力 | 用户手势支持，开发者控制不明确 | 支持按钮、触控、滑动事件 |

| 云端/RAG 接入 | 可通过手机中继到自有云，适合接你的 RAG 后端 | 很可能被小米生态限制，除非商务合作 | 非常适合接自有云/本地服务，官方强调可不依赖 Mentra 云 |

| 隐私/私有化 | 可做自有后端，但需企业版权限确认 | 不适合自定义隐私闭环 | 最友好，开源、可自托管、可离线控制 |



\*\*针对你 4 个核心需求的判断\*\*



| 需求 | 最适合 | 原因 |

|---|---|---|

| 自动识别关键场景开始 | Mentra Live / Rokid | 两者都能拿到摄像头、麦克风和按键事件；小米目前缺少公开开发接口 |

| 高频图片/短视频记录、压缩、回传 | Mentra Live | 更贴近 lifelogging，SDK 里媒体上传和流式能力更开放 |

| 手机/眼镜端与云端智能体对话 | Rokid | Rokid 有 HUD、ASR/TTS、AI Chat，眼镜端反馈体验最好 |

| RAG 记忆召回 | Mentra Live / Rokid | 这主要靠你的云端架构，硬件关键是能稳定采集和上传 |



\*\*我的建议\*\*

不要三选一，而是分工使用：



| 阶段 | 推荐硬件 |

|---|---|

| 第一阶段，验证全天候记忆采集 | \*\*Mentra Live\*\* |

| 第一阶段，验证眼镜端对话/HUD 体验 | \*\*Rokid Glass3 Enterprise\*\* |

| 消费级形态参考 | \*\*小米 AI 眼镜\*\* |

| 暂不建议作为主开发设备 | \*\*小米 AI 眼镜\*\*，除非小米商务明确开放摄像头/音频/文件/后台采集 SDK |



一句话：\*\*Mentra Live 更像你的“全天候记忆传感器”，Rokid 更像你的“可交互 AI 眼镜终端”，小米更像未来消费化对标产品。\*\*



参考来源：\[Rokid 产品参数](https://global.rokid.com/products/rokid-glasses)、\[Rokid Glass3 SDK](https://x-docs.rokid.com/docs/terminal-sdk/getting-started/%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97.html)、\[小米 AI 眼镜官方介绍](https://www.mi.com/global/discover/article?id=5172)、\[小米跨设备相机](https://hyperos.mi.com/continuity/abilities/ab0007)、\[Mentra Live](https://mentraglass.com/live)、\[Mentra Bluetooth SDK](https://github.com/Mentra-Community/Mentra-Bluetooth-SDK-Starter-Kit)。



> 再看看下面的关键点：低光场景下的图像采集，中国国内开发/测试的兼容性



<details><summary>2 previous messages</summary>



> 这两个点会把排序再拧一下：低光决定“耳机丢在哪儿、晚上回家路上发生了什么”这类问题能不能回答；国内兼容性决定你们能不能真的快速开发、联调、试点。我先补查官方资料，尽量把“硬件看起来强”和“实际能开发”分开说。

>

> <details><summary>Searched the web</summary>

>

> Searched the web for `Rokid Glasses low light night camera HDR official Sony IMX681 F2.25`

>

> Searched the web for `site:mi.com/prod/xiaomi-ai-glasses IMX681 F2.2 HDR 夜景 低光`

>

> Searched the web for `Mentra Live low light camera official HDR night`

>

> Searched the web for `https://mentraglass.cn/`

>

> Searched the web for `Xiaomi HyperConnect Xiaomi AI glasses camera video call livestream official`

>

> Searched the web for `MentraOS 中国 国内 官网 mentraglass.cn 开发者 文档`

>

> </details>

</details>



这两个关键点加进来后，我会把判断改成：



\*\*低光 + 国内开发综合最优：乐奇 / Rokid Glass3 Enterprise\*\*

\*\*低光消费体验潜力强但开发不确定：小米 AI 眼镜\*\*

\*\*开发开放最好但中国落地和低光不确定：Mentra Live\*\*



| 维度 | 乐奇 / Rokid | 小米 AI 眼镜 | Mentra Live |

|---|---|---|---|

| 低光硬件 | 12MP Sony IMX681，F2.25，109° | 12MP Sony IMX681，F2.2，105° | 12MP / 3264×2448，1080p 视频，未公开光圈/传感器型号 |

| 低光算法 | 官方提到 HDR、降噪、稳定，以及 Low-Light HDR | 小米影像算法强，但 AI 眼镜低光实测评价一般，暗部细节会弱 | 官方没有明确低光 HDR/夜景能力 |

| 开发可控性 | SDK 可控拍照、录像、音视频流、P2P 回传 | 没看到面向 AI 眼镜的公开完整 SDK | SDK 最开放，可控相机/麦克风/触控/上传 |

| 中国国内开发 | \*\*最友好\*\*：中文文档、Gitee Demo、国内商务/生态 | \*\*购买售后最友好，但开发最不确定\*\* | \*\*风险较高\*\*：主要海外生态，发货/Google Play/GitHub/npm/云服务都要验证 |

| 适合你的全天候记忆 | 国内 MVP 主力 | 消费形态参考/商务合作候选 | 海外/开源原型验证 |



\*\*低光场景判断\*\*



如果你的关键问题是“晚上耳机丢哪了”“餐厅里医生/朋友说了什么”“下班路上发生了什么”，低光不能只看像素，要看三件事：镜头进光、ISP/夜景算法、开发者能否控制曝光/帧率/补采样。



乐奇和小米的基础相机接近，都是 Sony IMX681，乐奇官方参数是 F2.25、109°，并明确写了 HDR、降噪和稳定能力；Rokid 近期教程还提到 Low-Light HDR。小米是 F2.2，理论进光略好，小米影像调校也强，但目前公开能力更多是消费功能，不等于你能在自己的 App 里调夜景、多帧降噪或后台低频拍摄。



Mentra Live 的开放性非常好，但低光这里资料不够硬：官方只写了 1080p 视频、3264×2448 图片、119° FOV，没有公开光圈、传感器型号、夜景/HDR 算法。它适合你验证“全天候低功耗记录架构”，但不应默认它夜间画质最好。



\*\*中国国内开发/测试判断\*\*



乐奇明显最稳。它有中文 Glass3 SDK、手机端/眼镜端 Demo、Gitee 仓库、P2P 文件/音视频流链路，开发方式也贴合你要的“眼镜采集 → 手机中继 → 国内云/RAG”。这对中国团队非常重要，因为联调时少掉一堆网络、账号、商店、发货、售后问题。



小米最适合国内消费者测试，但不适合作为第一开发主硬件。官方 HyperOS 有“跨设备相机”，可以把小米 AI 眼镜作为视频通话/直播摄像头；小米开发者平台也有相机生态引擎 SDK，支持 HDR、夜景、多帧降噪等能力。但这些目前更像小米手机/系统相机能力，没有明确证明“第三方可以完整控制小米 AI 眼镜的摄像头、音频、后台采集和文件回传”。



Mentra Live 最大问题不是开发能力，而是国内落地摩擦。官方页面写过主要美国发货，虽然有中文 MentraOS 页面，但实际采购、App 安装、Google Play、GitHub/npm、Mentra 云、售后都要逐项验证。若走 Mentra，建议一开始就做“自有 Android App + 国内云 + 不依赖 Mentra 云”的验证。



\*\*更新后的建议\*\*



第一批硬件我会这样配：



| 用途 | 推荐 |

|---|---|

| 国内主开发/MVP | \*\*Rokid Glass3 Enterprise，2 台\*\* |

| 小米生态和低光消费体验参考 | \*\*小米 AI 眼镜，1 台\*\* |

| 开源全天候记录架构验证 | \*\*Mentra Live，1 台，但作为辅助线\*\* |



优先级一句话：\*\*如果你要在中国快速做出可测试产品，先押 Rokid；小米要等官方/商务确认 SDK；Mentra 做技术灵感和开放架构验证，不做国内首发主线。\*\*



参考：\[Rokid 参数](https://global.rokid.com/products/rokid-glasses)、\[Rokid Glass3 SDK](https://x-docs.rokid.com/docs/terminal-sdk/getting-started/%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97.html)、\[小米 AI 眼镜](https://www.mi.com/global/discover/article?id=5172)、\[小米跨设备相机](https://hyperos.mi.com/continuity/abilities/ab0007)、\[Mentra Live](https://mentraglass.com/live)、\[Mentra Bluetooth SDK](https://github.com/Mentra-Community/Mentra-Bluetooth-SDK-Starter-Kit)。



> 乐奇支持自开发的手机app端吗



支持，而且这是乐奇比较适合你的重要原因之一。



按 Rokid 官方 \*\*Glass3 SDK / Sprite Enterprise\*\* 文档，它支持你开发自己的 \*\*Android 手机 App\*\*，不是只能用 Rokid 官方 App。手机端 SDK 可以做：



| 能力 | 是否支持 |

|---|---|

| 扫描、配对、连接眼镜 | 支持 |

| 蓝牙 + Wi-Fi P2P 通道 | 支持 |

| 手机端远程控制眼镜拍照/录像 | 支持 |

| 实时音视频预览 | 支持 |

| 眼镜与手机收发消息、文件、图片、视频 | 支持 |

| 眼镜媒体同步到手机 | 支持 |

| 手机转发网络请求到你的云端 | 支持 |

| ASR / TTS / AI Chat 集成 | 支持，但在线能力通常需要 API Key |

| 自开发眼镜端 App | 也支持 |



官方链路就是：



`自开发手机 App ⇄ Rokid 眼镜 ⇄ 自有云/RAG 服务`



更具体地说，Rokid 文档写得很清楚：Phone SDK 负责扫描连接、蓝牙/P2P 通道、网络转发、消息和文件收发、控制眼镜能力；眼镜不是直接访问公网，通常是 \*\*眼镜 → P2P → 手机 → 云端服务\*\*。这正好适合你的“眼镜低功耗采集，手机中继压缩上传，云端做记忆/RAG”的架构。



需要注意两点：



1\. 当前公开文档主要指向 \*\*Android 手机端开发\*\*，iOS 是否同等开放要单独问 Rokid 商务/开发者支持。

2\. 最好拿 \*\*Glass3 Enterprise / Sprite Enterprise 开发套件\*\*，不要只买普通消费版。要确认 SDK 权限、Demo、API Key、调试线和商务支持是否一起给。



参考：\[Rokid Glass3 SDK 概览](https://x-docs.rokid.com/docs/terminal-sdk/getting-started/%E6%8E%A5%E5%85%A5%E6%8C%87%E5%8D%97.html)、\[Demo 运行指南](https://x-docs.rokid.com/docs/en/downloads/demo-guide.html)。



> 请给我Glass3 Enterprise / Sprite Enterprise的购买链接



<details><summary>2 previous messages</summary>



> 我来核一下官方入口。企业/开发者版这类设备很多时候不是公开购物车购买，而是通过 Rokid 商务或开发者渠道开通 SDK 权限，所以我会把“能直接买”和“需要联系申请”的入口分开给你。

>

> <details><summary>Searched the web</summary>

>

> Searched the web for `Rokid Glass3 Enterprise purchase Sprite Enterprise buy official`

>

> Searched the web for `site:rokid.com/zh-CN 乐奇AI眼镜 购买 Rokid Glasses 官方 商城`

>

> Searched the web for `https://x-docs.rokid.com/docs/downloads/apps.html`

>

> Searched the web for `https://vip.rokid.com/desk/home`

>

> Searched the web for `https://www.rokid.com/zh-CN`

>

> </details>

</details>



我查到的情况是：\*\*Glass3 Enterprise / Sprite Enterprise 没有公开电商购买链接\*\*，更像是企业/开发者渠道开通。官方企业版 App 下载页也写了：进入 Rokid AI 企业版登录后，\*\*账户信息需联系销售申请\*\*。



你可以用这些入口：



| 用途 | 链接 |

|---|---|

| \*\*Glass3 / Sprite Enterprise SDK 文档\*\* | \[Rokid Sprite Enterprise SDK](https://x-docs.rokid.com/) |

| \*\*企业版 App / 资源下载页\*\* | \[Rokid AI 企业版下载页](https://x-docs.rokid.com/docs/downloads/apps.html) |

| \*\*乐奇 AI 眼镜官网产品页\*\* | \[glasses.rokid.com](https://glasses.rokid.com/) |

| \*\*Rokid 中文官网购买入口\*\* | \[rokid.com/zh-CN](https://www.rokid.com/zh-CN) |

| \*\*官方京东购买链接，消费版\*\* | \[京东 Rokid Glasses](https://item.jd.com/100189431904.html) |

| \*\*Rokid 开放平台\*\* | \[open.rokid.com](https://open.rokid.com/) |



建议你直接联系 Rokid 销售/开发者渠道，说明要的是：



`Rokid Glass3 Enterprise / Sprite Enterprise 开发套件 + Glass3 SDK 权限 + Rokid AI 企业版账号 + 调试线 + Demo/API Key`



官方公开邮箱里，最相关的是：



\- 国内 / 汽车业务：`Glasses.BD@rokid.com`

\- 生态合作 / 开发者：`developer@rokid.com`

\- B 端渠道合作：`bd@rokid.com`

\- 售后电话：`400-113-0099`



注意：京东/官网能买到的大概率是\*\*消费版乐奇 AI 眼镜\*\*，不一定自带 Enterprise SDK 权限。你要做自研手机 App 和眼镜端采集，最好先让销售确认系统版本、SDK 授权和调试支持。



> 个人开发者如何购买



<details><summary>2 previous messages</summary>



> 我再按“个人开发者”这个具体身份确认一下：企业套件通常走销售申请，但个人开发者可能有两条路，一条是买消费版打开开发者模式，一条是向销售申请企业版测试机/账号。我会把能落地的路径讲清楚。

>

> <details><summary>Searched the web</summary>

>

> Searched the web for `Rokid Glass3 consumer developer mode enable developer mode Glass3 SDK personal developer`

>

> </details>

</details>



个人开发者建议走两条路：



\*\*路线 A：先买消费版做原型\*\*

你可以先买 \*\*乐奇 AI 眼镜 / Rokid Glasses 消费版\*\*，再开启开发者模式做基础开发验证。Rokid FAQ 里提到：消费系统使用消费版 Rokid AI App，并在眼镜上开启 developer mode。



购买入口：



\- \[乐奇 AI 眼镜官网](https://glasses.rokid.com/)

\- \[Rokid 中文官网](https://www.rokid.com/zh-CN)

\- \[京东消费版购买页](https://item.jd.com/100189431904.html)



这条路适合先验证：手机 App 连接、拍照录像、录音、P2P/文件传输、眼镜端 Demo 是否能跑通。



\*\*路线 B：申请 Enterprise / Sprite 开发套件\*\*

如果你要完整的 \*\*Glass3 Enterprise / Sprite Enterprise SDK 权限、企业版账号、API Key、企业 Demo、调试线\*\*，目前没有公开电商购买链接，需要联系 Rokid 销售或开发者渠道申请。



联系时可以发给：



\- `developer@rokid.com`

\- `Glasses.BD@rokid.com`

\- `bd@rokid.com`

\- 海外/全球：`dev@rokid.com`、`enterprise@rokid.com`



邮件标题建议：



`个人开发者申请购买 Rokid Glass3 Enterprise / Sprite Enterprise 开发套件`



正文直接写清楚：



```text

你好，我是个人开发者，计划开发一款基于智能眼镜的个人记忆记录产品。

希望购买/申请 Rokid Glass3 Enterprise / Sprite Enterprise 开发套件。



需要确认：

1\. 是否支持个人开发者购买或申请测试机

2\. 是否包含 Glass3 SDK 权限

3\. 是否包含 Rokid AI 企业版账号/API Key

4\. 是否包含眼镜端调试线

5\. 是否可运行官方 phone-side / glasses-side Demo

6\. 是否支持自开发 Android 手机 App 控制眼镜拍照、录像、录音、媒体同步和 P2P 回传



项目用途：个人全天候记忆记录，眼镜采集音视频，手机中继上传，云端做 ASR、摘要和 RAG 问答。

预计采购数量：1-2 台。

```



我的建议：\*\*先买 1 台消费版快速验证硬件佩戴和基础采集，同时向 Rokid 申请 Enterprise 开发套件。\*\* 因为消费版能快，企业版才更适合你后续自研 App 和云端链路。



> 查一下消费版可能会缺少那些关键开发功能



<details><summary>2 previous messages</summary>



> 我会按“官方能确认”和“需要销售确认/实机验证”分开看，避免把企业版文档里的能力默认套到消费版上。重点查：系统版本、企业账号/API Key、企业 App、眼镜端 Demo、调试线、后台采集和自启动这些对你项目最要命的点。

>

> <details><summary>Searched the web</summary>

>

> Searched the web for `Rokid Glass3 consumer system developer mode limitations enterprise SDK FAQ`

>

> Searched the web for `https://x-docs.rokid.com/docs/en/faq/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98.html`

>

> Searched the web for `https://x-docs.rokid.com/docs/en/openapi/ApiKey.html`

>

> </details>

</details>



查下来，消费版不是完全不能开发，但它\*\*可能缺的是企业版那套“完整可控的开发/部署权限”\*\*。对你这种全天候记忆产品，风险点主要在这些地方：



| 可能缺失/受限点 | 对你项目的影响 |

|---|---|

| \*\*企业账号与 API Key\*\* | Rokid OpenAPI 明确要求企业账号，API Key 需销售/OA 审批。消费版个人账号可能拿不到平台 OpenAPI、设备管理、智能体管理等能力。 |

| \*\*Rokid AI 企业版 App 账号\*\* | 企业版 App 登录账号需联系销售申请。消费版只能用普通 Rokid AI App。 |

| \*\*企业系统固件\*\* | 官方 FAQ 说系统版本含 `e` 通常是企业/工作版；消费版要开启 developer mode。企业 Demo/企业能力未必在消费版完整可用。 |

| \*\*企业 Demo 与 -E SDK 能力一致性\*\* | 最新 SDK 包和 OTA 推荐版本带 `-E` / `e`，消费版是否完整兼容要实机验证。 |

| \*\*GB/RTSP 视频监控能力\*\* | 文档里的视频监控依赖“灵眸 AR 企业平台”的设备信息，消费版大概率不能直接用。 |

| \*\*企业后台配置能力\*\* | 如设备绑定、行业 App 配置、AI 助手 Prompt 下发、车牌/工单类业务配置，偏企业平台能力。 |

| \*\*自启动/后台保活/长期运行权限\*\* | 全天候记忆很依赖后台采集。消费版即使能跑 App，也要验证熄屏、锁屏、长时间待机后是否被系统杀掉。 |

| \*\*按键/语音指令深度拦截\*\* | 新版 SDK 有乐奇语音指令拦截、设备事件监听等能力，但部分说明偏企业 SDK；消费版能否稳定覆盖系统默认行为需验证。 |

| \*\*调试线/ADB/系统权限\*\* | 消费版包装不一定带企业调试配件；能否稳定安装眼镜端 APK、抓日志、调后台行为，需要确认。 |



\*\*但这些能力不一定缺：\*\*



消费版 FAQ 里明确提到：消费系统可用普通 Rokid AI App，并在眼镜上开启 developer mode。文档也显示 SDK 有拍照、录像、实时预览、P2P、文件同步、手机端 SDK、眼镜端 SDK 等能力。所以消费版很可能能做“开发原型”，但不一定适合直接做“稳定全天候产品”。



\*\*对你的项目，我会这样判断：\*\*



| 需求 | 消费版风险 |

|---|---|

| 手机 App 连接眼镜 | 中低，应该可验证 |

| 拍照/录像/文件同步 | 中低到中，需跑 Demo |

| 实时音视频流/P2P 回传 | 中，需实机压测 |

| 长时间后台低频采集 | 高，必须测试 |

| 自动场景识别/语音触发 | 中高，涉及后台音频和系统指令 |

| 云端 RAG 问答 | 低，如果完全走你自己的手机 App + 云端 |

| Rokid 官方智能体/OpenAPI | 高，可能要企业账号/API Key |



\*\*建议验证顺序：\*\*



先买消费版可以，但第一天就验证这 5 项：



1\. 能否开启 developer mode 并安装眼镜端 APK

2\. 官方 Demo 的手机端和眼镜端能否跑通

3\. 手机端能否远程触发拍照/录像/录音并拿到文件

4\. P2P 视频预览连续 30-60 分钟是否稳定、发热如何

5\. App 退到后台/锁屏后，采集和传输是否还能持续



如果第 4、5 项不稳，就必须转 Enterprise / Sprite Enterprise。



参考：\[Glass3 SDK FAQ](https://x-docs.rokid.com/docs/en/faq/%E5%B8%B8%E8%A7%81%E9%97%AE%E9%A2%98.html)、\[Rokid OpenAPI Key](https://x-docs.rokid.com/docs/en/openapi/ApiKey.html)、\[企业版 App 下载说明](https://x-docs.rokid.com/docs/downloads/apps.html)。



> 如果能稳定连接外网Mentra Live会是更好的选择吗



<details><summary>2 previous messages</summary>



> 这个假设会让 Mentra Live 的分数上升不少。我会重新按“外网不再是主要阻碍”的前提判断，但还要保留低光、无显示、售后和最终中国市场落地这些硬约束。

>

> <details><summary>Searched the web</summary>

>

> Searched the web for `Mentra Live official SDK camera microphone buttons offline direct Android iOS app full control`

>

> Searched the web for `https://mentraglass.com/blogs/blog/making-mentra-live`

>

> </details>

</details>



是的，\*\*如果外网、App 安装、GitHub/npm、Mentra 云或你自己的海外服务都能稳定访问，Mentra Live 会更像你这个产品的“主开发硬件”\*\*，尤其是做全天候个人记忆采集。



我会这样改判断：



| 目标 | 更优选择 |

|---|---|

| 全天候低功耗记录、频繁拍照/音频采集 | \*\*Mentra Live\*\* |

| 自开发手机 App 完整控制硬件 | \*\*Mentra Live\*\* |

| 开源、少厂商锁定、可自托管 | \*\*Mentra Live\*\* |

| 中国国内快速采购、售后、商务联调 | Rokid |

| 眼镜端 HUD 显示、字幕、提词、可视反馈 | Rokid |

| 低光画质确定性 | Rokid / 小米更有参数优势，Mentra 需实测 |



Mentra Live 对你最关键的优势是：官方说它有 open-source SDK，开发者可以写自己的 Android/iOS App 直接连接眼镜，并控制 camera、speakers、microphone、touchpad、buttons，而且可以离线工作、不依赖 Mentra 托管云。这比 Rokid 消费版/企业版账号体系更自由。来源：\[Mentra Live](https://mentraglass.com/live)、\[Mentra Bluetooth SDK](https://github.com/Mentra-Community/Mentra-Bluetooth-SDK-Starter-Kit)。



另外，Mentra 的设计理念更贴近你的“生活记忆传感器”：无显示、更轻、43g、260mAh 眼镜电池 + 2200mAh 充电盒、12+ 小时混合使用；官方还强调为了持续音频做了 BLE 低功耗麦克风通道，让 App 在手机播放音乐/通话等情况下仍能拿到眼镜麦克风数据。来源：\[Making Mentra Live](https://mentraglass.com/blogs/blog/making-mentra-live)。



但它不是全方位碾压 Rokid。两个风险还在：



1\. \*\*低光图像没有 Rokid/小米确定。\*\* Mentra 官方没有明确公布传感器型号、光圈、HDR/夜景算法。你的“耳机丢哪了”“晚上路上发生了什么”这类问题必须实拍验证。

2\. \*\*无显示。\*\* Mentra 更适合“采集 + 手机/语音交互”，不适合眼镜里直接显示字幕、证据、提醒。Rokid 在 HUD 交互上仍然更强。



我的最终建议会变成：\*\*如果你能稳定用外网，第一主线买 Mentra Live；同时保留 1 台 Rokid 做中国本地兼容和 HUD 交互验证。\*\* Mentra 负责记忆采集架构，Rokid 负责国内落地和显示型交互。



> 把这个对话的关键内容进行整理用于说明硬件采购需要考虑的关键点，使用场景要重点强调。



> 把这个对话的关键内容整理成html用于说明硬件采购需要考虑的关键点，使用场景要重点强调。



<details><summary>4 previous messages</summary>



> 我会做成一份可直接打开给合伙人/采购方看的 HTML 说明页，结构上先讲使用场景，再讲这些场景反推出来的硬件采购指标，最后给出乐奇、小米、Mentra Live 的采购判断和验证清单。

>

> 我准备在 `hardware\_selection` 下新增一份独立 HTML，命名为 `hardware\_purchase\_key\_points.html`。内容会避免只做参数堆叠，而是用“场景 -> 采购指标 -> 设备判断 -> 实测清单”的顺序，让采购决策更贴近产品真实使用。

>

> <details><summary>Created a file</summary>

>

> - Wrote `./hardware\_selection/hardware\_purchase\_key\_points.html` (+998 -0)

>

> </details>

>

> HTML 已经生成好了。我再快速检查一下文件是否落盘、标题和关键章节是否完整，顺手确认没有明显编码或结构问题。

</details>



已整理成 HTML 文件： \[hardware\_purchase\_key\_points.html](./hardware\_selection/hardware\_purchase\_key\_points.html)



里面重点按你的真实使用场景组织了内容：就医用药回忆、物品遗失回溯、会议数字细节、通勤居家轻记录。然后从这些场景倒推出硬件采购指标，并对 \*\*Rokid 乐奇 / Glass3、小米 AI 眼镜、Mentra Live\*\* 做了对比和采购建议。



我也加了样机验证清单，覆盖低光图像、会议录音、低频视觉记录、关键场景自动触发、手机中继、后台保活、佩戴发热和隐私控制。静态 HTML，可以直接用浏览器打开。



