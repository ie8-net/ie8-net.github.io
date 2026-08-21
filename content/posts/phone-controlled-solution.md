+++
title = "手机被控方案：基于 Worker 调度 + scrcpy 视频流的远程设备控制系统"
date = 2026-08-21T13:08:00+08:00
draft = false
tags = ["远程控制", "scrcpy", "自动化", "Qwen-UI-Agent", "Android"]
categories = ["工程实践"]
description = "把手机当成一个可被远程调度的执行节点：Worker 下发指令、scrcpy 回传画面、自动化任务交给 Qwen-UI-Agent。关键是设备只要有网络就能被控，不依赖 USB、不依赖内网穿透。"
+++

## 0. 这是什么

一句话：把一台手机当成一个**可被远程调度的执行节点**。你在一头（控制面）下指令，手机在另一头（被控端）真正去点屏幕、跑任务，中间用视频流把画面实时回传给你看。

为什么不直接用云手机？云手机是别人的机器，数据、环境、可控性都不在你手里。本方案要的是**真实物理设备**——自己的手机、自己的账号、自己的网络，只是「人不在旁边」而已。

核心约束有三个，后面每一节都是为它们服务的：

- 控制面（Worker）和被控手机之间**只走公网**，不要求 USB 常驻、不要求内网/VPN；
- 画面必须能实时回传，且能随时接管或交给自动化；
- 设备一旦上线就**不许版本碎片化**——强制自升级到统一版本。

## 1. 整体架构

<svg viewBox="0 0 680 400" width="100%" xmlns="http://www.w3.org/2000/svg" role="img" style="display:block;margin:16px auto;max-width:680px">
  <title>手机被控方案整体架构</title>
  <desc>控制面通过WebSocket与Worker通信，Worker下发语义指令给被控手机Agent，Agent内含scrcpy-server和Qwen-UI-Agent客户端</desc>
  <defs>
    <marker id="a" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="6" markerHeight="6" orient="auto">
      <path d="M2 1L8 5L2 9" fill="none" stroke="#5F5E5A" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
    </marker>
  </defs>

  <!-- 控制面 -->
  <rect x="40" y="100" width="150" height="56" rx="8" fill="#E6F1FB" stroke="#185FA5" stroke-width="0.5"/>
  <text x="115" y="122" text-anchor="middle" dominant-baseline="central" font-size="14" font-weight="500" fill="#042C53">控制面</text>
  <text x="115" y="142" text-anchor="middle" dominant-baseline="central" font-size="12" fill="#0C447C">（人 / 平台）</text>

  <!-- Worker -->
  <rect x="380" y="100" width="160" height="56" rx="8" fill="#EEEDFE" stroke="#534AB7" stroke-width="0.5"/>
  <text x="460" y="122" text-anchor="middle" dominant-baseline="central" font-size="14" font-weight="500" fill="#26215C">Worker（调度层）</text>
  <text x="460" y="142" text-anchor="middle" dominant-baseline="central" font-size="12" fill="#3C3489">设备管理 / 路由</text>

  <!-- 双向箭头 -->
  <line x1="190" y1="120" x2="375" y2="120" stroke="#5F5E5A" stroke-width="1.5" marker-end="url(#a)"/>
  <line x1="375" y1="136" x2="195" y2="136" stroke="#5F5E5A" stroke-width="1.5" marker-end="url(#a)"/>
  <text x="285" y="110" text-anchor="middle" font-size="12" fill="#444441">指令 / 状态 (WebSocket)</text>

  <!-- Worker -> 手机 -->
  <line x1="460" y1="156" x2="460" y2="203" stroke="#5F5E5A" stroke-width="1.5" marker-end="url(#a)"/>
  <text x="508" y="184" font-size="12" fill="#444441">下发语义指令</text>

  <!-- 被控手机容器 -->
  <rect x="260" y="210" width="340" height="170" rx="16" fill="#FAECE7" stroke="#993C1D" stroke-width="0.5"/>
  <text x="430" y="234" text-anchor="middle" dominant-baseline="central" font-size="14" font-weight="500" fill="#4A1B0C">被控手机（Agent A1.0.0）</text>

  <!-- scrcpy-server -->
  <rect x="280" y="252" width="140" height="56" rx="8" fill="#E1F5EE" stroke="#0F6E56" stroke-width="0.5"/>
  <text x="350" y="274" text-anchor="middle" dominant-baseline="central" font-size="14" font-weight="500" fill="#04342C">scrcpy-server</text>
  <text x="350" y="294" text-anchor="middle" dominant-baseline="central" font-size="12" fill="#085041">屏幕采集 + 触控注入</text>

  <!-- Qwen-UI-Agent -->
  <rect x="280" y="320" width="140" height="48" rx="8" fill="#EAF3DE" stroke="#3B6D11" stroke-width="0.5"/>
  <text x="350" y="344" text-anchor="middle" dominant-baseline="central" font-size="14" font-weight="500" fill="#173404">Qwen-UI-Agent 客户端</text>
  <text x="350" y="360" text-anchor="middle" dominant-baseline="central" font-size="12" fill="#27500A">看画面 → 规划 → 执行</text>

  <!-- 视频流输出 -->
  <line x1="420" y1="280" x2="535" y2="280" stroke="#5F5E5A" stroke-width="1.5" marker-end="url(#a)"/>
  <text x="480" y="270" text-anchor="middle" font-size="12" fill="#444441">视频流 (H.264)</text>

  <!-- Agent 内部箭头 -->
  <path d="M440 300 L440 310 L428 310" fill="none" stroke="#5F5E5A" stroke-width="1" marker-end="url(#a)"/>

  <!-- 右侧标注 -->
  <rect x="538" y="252" width="52" height="116" rx="6" fill="none" stroke="#B4B2A9" stroke-width="0.5" stroke-dasharray="4 3"/>
  <text x="564" y="314" text-anchor="middle" font-size="11" fill="#888780" writing-mode="vertical-rl">控制面回显</text>
</svg>

控制面不直接碰屏幕像素，Worker 不直接连手机——中间全靠 Agent 这一个进程兜住。下面拆开讲四块。

## 2. Worker 控制层

Worker 是整套系统的**调度大脑**，职责很纯粹：

- **设备管理**：维护在线设备清单，每台用 `device_id` 唯一标识，心跳保活；
- **任务编排**：接收上层任务（比如「在抖音发一条视频」），拆成可下发的步骤；
- **指令下发**：把步骤翻译成手机能执行的语义指令，通过长连接推给对应设备；
- **状态回收**：收集执行结果、截图、报错，回填给上层。

指令协议走 JSON，不传坐标裸操、只传语义动作，这样换设备分辨率也不崩：

```json
{
  "device_id": "ac_81600373",
  "action": "tap",
  "target": "text=发布",
  "timeout_ms": 5000
}
```

Worker 和手机之间用 WebSocket 长连接，断线自动重连。一台 Worker 可以管 N 台设备，靠 `device_id` 路由，互不串台。这一层**不依赖 USB、不依赖内网**——只要设备能出网，连上来就行。

## 3. scrcpy 视频流

scrcpy 负责两件事：**屏幕镜像 + 触控注入**。它原本跑在 adb 之上，但本方案的关键改造是——**不要求 USB 线**。

做法：被控端起一个 `scrcpy-server`，Worker 在设备上线时通过反向隧道把 scrcpy 的 socket 映射到控制端。于是控制面看到的是一路实时 H.264/H.265 视频流，延迟很低，并且可以直接往里注入触控事件（`tap` / `swipe` / `input text`）。

几个工程要点：

- **弱网自适应**：码率随带宽动态降档，宁可糊一点也不要卡死；
- **手动/自动切换**：人想接管时点开画面就能手动操作，松手后交还给自动化；
- **回传即校验**：每一帧画面都是「这一步到底成没成」的最强证据，自动化任务靠它做视觉断言。

## 4. 版本号 A1.0.0 与强制自升级

被控端 Agent 当前版本 **A1.0.0**，语义化版本，主版本 `A` 代表「被控端大架构代际」。

之所以要做**强制自升级**，是因为这个系统最怕设备碎片化：十台手机九个版本，scrcpy 桥接层和控制协议对不上，排错能让人疯。强制策略如下：

1. Worker 在下发任务前先比对 `min_supported_version`；
2. 设备心跳时若发现自身版本低于阈值，**无条件拉取新包**；
3. 升级走 HTTPS，包体带签名校验，校验不过不装；
4. 升级失败自动回滚到上一可用版本，绝不让设备「变砖失联」。

强制升级只在「主版本/协议不兼容」时触发硬阻断，日常小版本是静默热更。一句话：**设备上线即统一，不允许长期落后。**

## 5. 设备接入要求：只要有网络

这是方案能不能规模化的命门。要求只有一条——**设备能出网即可**，具体讲：

- 不要求 USB 调试线常驻，线只是首次引导用，之后拔掉照常工作；
- 不要求内网、不要求 VPN、不要求端口映射；
- WiFi / 4G / 5G 任意一种都行，设备用 Worker 下发的注册 token 上线；
- 网络层用**反向隧道 / 中继**，控制面不需要知道设备真实 IP，设备也不需要公网 IP。

上线流程极简：设备装好 Agent、用 token 注册一次，之后只要联网就自动出现在 Worker 的设备列表里。断网恢复后靠心跳自动重连，无需人工插手。

## 6. 自动化任务接入 Qwen-UI-Agent

多步、需要「看屏幕判断」的任务，交给 [Qwen-UI-Agent](https://tongyi-mai.github.io/Qwen-UI-Agent/#top) 来跑。它干的事本质上是把「人看着手机操作」这件事自动化：

1. **感知**：拿 scrcpy 回传的当前帧作为视觉输入；
2. **规划**：Qwen-UI-Agent 根据任务目标拆解下一步动作；
3. **执行**：把动作翻译成 Worker 指令（tap / swipe / input），经 scrcpy 注入到真机；
4. **校验**：再看一帧画面，确认这一步成了没，没成则重试或换路。

这就是一个完整的「观察 → 决策 → 执行」闭环，跑在真实设备上、用真实账号。典型适用场景：

- 批量内容发布（选题、生成、定时排程后的「最后一公里」执行）；
- 日常巡检（打开 App 看状态、截图回收）；
- 表单填写、信息核对这类可脚本化但需要视觉判断的活。

把「规划智能」和「真机执行」解耦：Qwen-UI-Agent 负责想，Worker + scrcpy 负责做，手机负责真跑。

## 7. 小结

这套方案落地的核心就三句话：

- **控制面不管屏幕、只下语义指令**，设备兼容性靠这一层抽象兜住；
- **scrcpy 把真机变成一路可接管、可注入的视频流**，拔掉 USB、跨公网照样工作；
- **强制自升级 + 注册 token 上线**，让设备规模化管理不再依赖人工维护版本和网络环境。

自动化那部分交给 Qwen-UI-Agent，人只需要在关键节点打开画面看一眼，或者完全放手让闭环自己跑。
