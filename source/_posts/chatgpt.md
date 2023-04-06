---
title: ChatGPT的X种用途
tags:
  - ChatGPT
description: 这是我最近几个月使用ChatGPT干的一些事情，和以后想做的一些事情
cover: 'http://images.ashery.cn/img/image-20230406210518731.png'
abbrlink: 6918edaf
date: 2023-03-20 15:06:59
categories:
---


### 日常百科
当做知识图谱最基本的用法，可以省去很多信息聚合的工作 。

![image-20230406204356342](http://images.ashery.cn/img/image-20230406204356342.png)

### 写代码
目前ChatGPT还不能帮我们写业务代码，但是在一些其他规则比较固定的地方能有比较大的作用


1. 脚本代码：bash、python、正则表达式、cronjob之类的脚本可以完全给chatgpt来做，比如格式转换、批量操作等，这类操作ChatGPT基本不会出错，并且不用去搜索引擎找各种类库
2. 前端页面：如果一个前端页面没有什么花里胡哨的效果，只有几个框的展示和简单按钮的展示，也可以交给ChatGPT去做，尤其是在GPT-4下，可以输入图片，直接把UI图喂给 ChatGPT，基本就能画一个页面（自己还没有资格用GPT-4 ）。但是如果涉及到复杂的前端交互和后端的数据交互可能会比较麻烦。
3. 功能单一的小函数：比如format数字、公式计算。主要是效率高，自己写可能也不会很麻烦。
4. 写单测：对于上面提到的这种小函数，可以顺便让GPT生成单测代码

![image-20230406205200539](http://images.ashery.cn/img/image-20230406205200539.png)

### 解释代码


1. 读开源代码：自己硬读开源代码是很难的，网上带着读源码的博客也比较少。自己面对一大段代码时，往往无法快速识别出主要内容以及作用，这时就可以直接问chatgpt：关键代码是那几行，每一行的作用是什么。
2. leetcode：对着答案解析也看不懂的可以试试，也可以一步步的问他解题思路，并让它引导你，当做一个老师。

![image-20230406205211920](http://images.ashery.cn/img/image-20230406205211920.png)

![image-20230406205221009](http://images.ashery.cn/img/image-20230406205221009.png)

![image-20230406205228916](http://images.ashery.cn/img/image-20230406205228916.png)

### 改bug
自己写了一大段代码后，可以让Chatgpt识别出有哪些潜在风险，尤其对于一些lint难以检测出的问题，ChatGPT能给出一些“经验之谈”

（不完全准确，有时候大家的经验并不怎么足够，下面是智障现场）![image-20230406205251844](http://images.ashery.cn/img/image-20230406205251844.png)

![image-20230406205325186](http://images.ashery.cn/img/image-20230406205325186.png)







### 学英语

1. **改语法：**搭配grammarly更香

![image-20230406205405022](http://images.ashery.cn/img/image-20230406205405022.png)

2. **练对话**：搭配插件Voice Control for ChatGPT (https://chrome.google.com/webstore/detail/voice-control-for-chatgpt/eollffkcakegifhacjnlnegohfdlidhn)。只能让它帮你改口语表达和听力，并不能纠正你的发音。

![image-20230406205419349](http://images.ashery.cn/img/image-20230406205419349.png)

3. **文章润色**：由于训练集是英文的原因，应该会更native

![image-20230406205424622](http://images.ashery.cn/img/image-20230406205424622.png)

主要是目前这些功能在和市面上的收费服务差不多的情况下 **免费**

### 技术调研
不仅能帮你扒出有哪些组件可以用，还能分析每个组件的优缺点

![image-20230406205441234](http://images.ashery.cn/img/image-20230406205441234.png)

### 智能客服

以后必然的趋势

用户的反馈更快的得到更准确的回复

推销机器人越来越像人了



### AIGC

**主题：**

- 手动指定
- 爬热点
- 直接让GPT生成 （可以结合2）

**制作：**


- 图文：直接生成图片（mid journey）+文字就OK了
- 简单的视频：给出脚本，素材，网上捞一捞现有的资源
- 复杂的视频：给出每一段的分镜然后自己拍视频

**剪辑：**

- 剪辑风格
- 软件使用
- 一键成片


**发布：**

- 标题，内容

- 账号自动运营



### 私人助理
最近ChatGPT开放了**第三方接口的调用**能力，可以用来干更多事了

[ChatGPT Plugin](https://platform.openai.com/docs/plugins/introduction)



> 更多关于ChatGPT的内容可以参考[哈尔滨工业大学-ChatGPT调研报告.pdf](https://github.com/YHsilver/awesome-ChatGPT-resource-zh/blob/main/pdfs/230311-%E5%93%88%E5%B0%94%E6%BB%A8%E5%B7%A5%E4%B8%9A%E5%A4%A7%E5%AD%A6-ChatGPT%E8%B0%83%E7%A0%94%E6%8A%A5%E5%91%8A.pdf)

