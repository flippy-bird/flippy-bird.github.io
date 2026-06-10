---
title: 叮~，你有一个openclaw的手册待查收！(组内分享)
description: 当时openclaw热度达到了顶峰，所以就写了一个openclaw的简要介绍文档
date: 2026-03-06 14:05:16
cover: /covers/10.webp
tags:
    - agent
---

英文名：openclaw(主要名字)/moltbot/Clawdbot

中文名：小龙虾、大龙虾、龙虾

### 背景

为什么写这篇文档，最近关于小龙虾的热度非常高，同时也引爆了之前就在炒的一个概念：OPC(one person company),因此最近在小红书上和抖音上能看到：一人公司月入200w(而且是美金哦~)、我养了12个小龙虾帮我实现了工作自由、openclaw上门安装月入10万等等。

![image-20260424141038214](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260424141038214.png)

看到这些，你是否也被勾起了好奇心，看着每月到账的几个子儿，决定下海开创一番事业。别急，容我给你分析一番，再做决定也不迟。

这个也是我写这篇文档的初衷，简单介绍一下openclaw，预防一些"AI 诈骗", **珍爱生命，远离AI营销号 !, ! !**

### openclaw是什么？

一句话介绍：个人专属AI Agent，你可以在常用的IM软件中(比如飞书)发送一个消息，然后他帮你自动处理任务，7*24小时的为你工作。

有些使用过openclaw产品的同学，可能有一个感觉，使用的时间越长，你的openclaw越懂你，是因为openclaw会在本地记录你的偏好，每天干了什么，然后你每次提问时，都会根据历史(记忆)来帮你解决问题

为什么网上也提到了openclaw 是个token的吞金兽，原因就在于每次回答你的问题时，他都会加载你的历史信息到上下文中，因此token爆炸，非常费钱！！！（创业未半，而钱包中道崩殂）

![image-20260424140904164](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260424140904164.png)

因此openclaw 不存在技术上的创新，属于工程实践上的创意 ，因此你之前在使用agent产品上遇到的模型幻觉，回答不准确问题，openclaw并不能帮你解决；

那有一个问题，为什么openclaw这么火？原因在于他是第一个在本地运行的，开放了所有权限的本地agent，而且引入了网关(gateway),把聊天应用引入进来，和大家日常的app关联了起来，解锁了远程能力，有一定的使用体验；同时刚好踩中了个人agent流量风口，上一个是manus(当然manus也是工程上的创新 + 商业模式的创新，sandbox + 上下文管理 + 邀请码🐶)，同样的上一次manus的邀请码被炒到了十万。

### openclaw能干什么？

- AI 晨报：每天起床给你定制汇总昨天发生的AI 圈的事件，通过飞书的消息发送给你
- 邮件管家：4000封邮件两天清理完
- 会议纪要自动化：开完会后todo自动发送给对应的人
- 第二大脑：发条消息记住一切： 利用飞书，说：帮我记住这个xxx
- 习惯打卡教练：提醒你喝水，定时给你发送一个热门段子，给你提供情绪价值等等
- 帮你处理文件，代码，制作视频等等。。。

![image-20260424141129538](https://raw.githubusercontent.com/nashpan/image-hosting/main/image-20260424141129538.png)

### 你真的需要openclaw吗？

​          从上面的例子可以看到，日常提醒，打卡，定时任务，或者信息整合，我似乎通过闹钟、写一个定时功能即可完成，我还需要openclaw吗，确实，你不太需要openclaw，日常的GPT，豆包都可以帮你完成了，不过呢，由于是个人代理，用一用也无妨；

 个人觉得，对于那些重复的，可以复用成工作流(skills)的工作，可以使用openclaw来帮你完成，下面是我个人的一些想法，如有雷同，纯属巧合

|        |                                                              | 备注                                                  |
| ------ | ------------------------------------------------------------ | ----------------------------------------------------- |
| 程序员 | 飞书定时提醒喝水 (关注身体健康)定时给你讲一个段子（关注情绪健康）定时推送AI消息（保持大脑活跃）Openclaw review代码，并给出建议 |                                                       |
| HR     | 定期推送指定岗位的候选人，给出候选人的相关度，优势，评分，审核后自动推给相应的同学 | 需要装一个能够爬取岗位信息，分析信息，整合信息的skill |
| 运营   | 利用openclaw定时产出推广视频并发到指定平台                   | 需要安装如：视频生成skills、                          |
| 。。。 |                                                              |                                                       |

### Openclaw 不能干什么

因为OpenClaw开放了电脑权限，因此通过命令行的指令能完成的，OpenClaw都可以，但是OpenClaw不是GUIAgent或者AI 操作系统(豆包手机)，因此类似人类使用app的操作，OpenClaw都不可以完成，比如你让他帮你打开AE，剪辑一个视频；打开美团帮我点一杯奶茶，他就会懵逼；

另外一个问题：为什么我的OpenClaw没有网上说的好用，同上，OpenClaw一定要配合Skills使用，才会效果比较好，网上也很容易搜到很多适配OpenClaw的skills

### 不懂这个如何体验OpenClaw

上面也提到了，OpenClaw本质上是工程上的创新，不是技术上的，因此目前有很多复刻版本,大家看第一个阿里的CoPaw即可体验，不会安装的也可以找我，免费的，哈哈哈！

阿里：https://github.com/agentscope-ai/CoPaw

网易：https://github.com/netease-youdao/LobsterAI

云上：阿里，字节，腾讯各自的云上都推出了

<img src="https://raw.githubusercontent.com/nashpan/image-hosting/main/image_123.png" alt="image_123" style="zoom: 67%;" />

其它类似产品：

1. https://github.com/HKUDS/nanobot
2. https://github.com/zeroclaw-labs/zeroclaw
3. https://github.com/qwibitai/nanoclaw

结语，对于agent产品，往往具有很高的热度，AI 自媒体大都有夸大的成分，希望大家在谨慎AI买课的同时，也不用对技术的更新迭代速度太快而过于焦虑；保持好奇心，持续观望！
