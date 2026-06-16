---
title: "基于 AC 自动机和贝叶斯方法的垃圾内容识别"
source: "https://zhuanlan.zhihu.com/p/25835417"
author:
  - "[[秦洛在火星上挖矿 (移民锦官城)]]"
published:
created: 2026-06-16
description: "背景 作为一个开放领域的知识社交平台，知乎为大家提供了「友善」、「理性」、「专业」的讨论氛围，吸引了大量用户参与，产生了很多优质内容。但同时也吸引了一些垃圾制造者，在知乎上生产了不少的垃圾内容，如「…"
tags:
  - "clippings"
---
编辑推荐

[

收录于 · 知乎技术专栏

](https://www.zhihu.com/column/hackers)

614 人赞同了该文章

**背景**

作为一个开放领域的知识社交平台，知乎为大家提供了「友善」、「理性」、「专业」的讨论氛围，吸引了大量用户参与，产生了很多优质内容。但同时也吸引了一些垃圾制造者，在知乎上生产了不少的垃圾内容，如「违法」、「广告」、「淫秽色情」、「人身攻击」等，严重影响了知乎用户的正常讨论交流，极大地影响了用户体验，同时也对社区管理造成了较大的干扰。

我们先来看看都有哪些真实的垃圾：

*非法内容* ，比如：1）传播、寻求或买卖情色，如「xxx 宾馆酒店小姐」、「xxx 一夜情小姐」、「xxx 全套小姐」、「xxx 红灯区小姐」；2）传播赌博，如 「澳\*新\*天地 xxx xxx 博人真钱赌 连赢 100 万」、「\*博 xxx xx 玩法多」、「xxx xx zzz」等等。这类内容违反了现行法律，危及了国家安全和社会和谐。

*不友善内容* ，比如：1）辱骂他人，如「你这个智障」、「艹你妈」；2）用侮辱、夸张的手法嘲讽他人，如「脑残」、「智商欠费」等等。这类内容表现为不尊重他人，用恶毒的言语刺激对方，使得讨论无法正常有效进行。

还有一些垃圾广告，如微商、代开发票。

这些垃圾内容严重影响了知乎用户的正常交流。此前我们的工程师们也尝试了一些方法去识别处理它们。如文本分类模型，准确率达到了 96%, 每天识别 300+ 条；利用 [DFA](https://zhida.zhihu.com/search?content_id=2520995&content_type=Article&match_order=1&q=DFA&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODE3NzA2NTIsInEiOiJERkEiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNTIwOTk1LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.d4702Wpth2jFnQSnbvJUhshiHT62LUxDHTifMIgRNhk&zhida_source=entity) 根据关键词大量召回。这些尝试虽然都取得了一定的效果，但是召回不够、或召回过多非垃圾内容、或者存在不少的误伤。为此我们引入人工审核，但不能快速处理，容易造成内容堆积，而且对管理员也是很大压力，平均每周要消耗 1 个人力。

前期的尝试虽然效果不是很理想，但积累了比较多的数据。对这些数据的分析，我们发现这些垃圾内容是有套路的。基于此，我们利用 [Aho-Corasick 自动机](https://zhida.zhihu.com/search?content_id=2520995&content_type=Article&match_order=1&q=Aho-Corasick+%E8%87%AA%E5%8A%A8%E6%9C%BA&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODE3NzA2NTIsInEiOiJBaG8tQ29yYXNpY2sg6Ieq5Yqo5py6IiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjUyMDk5NSwiY29udGVudF90eXBlIjoiQXJ0aWNsZSIsIm1hdGNoX29yZGVyIjoxLCJ6ZF90b2tlbiI6bnVsbH0.QvqOoPyRFF-MyWg3HuhB8vUVK8A2u5xV6V8tNk6Cy1s&zhida_source=entity) 实现多模匹配，在其基础上增加了过滤机制，实现了第一版的垃圾内容分析系统，取得了不错的效果。

**Aho-Corasick 自动机**

[AC 自动机](https://zhida.zhihu.com/search?content_id=2520995&content_type=Article&match_order=1&q=AC+%E8%87%AA%E5%8A%A8%E6%9C%BA&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODE3NzA2NTIsInEiOiJBQyDoh6rliqjmnLoiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNTIwOTk1LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.MeWXRdbzipRCXBiGYD28QE9iNup_NAI7bbwM5SDlK5Y&zhida_source=entity) 算法于 1975 年产生于贝尔实验室。该算法巧妙地将多模式串建成一个确定性有限状态机 (DFA)，以待匹配字符串作为该 DFA 的输入，使状态机进行状态转移，当到达某些特定的状态时，完成模式匹配， 能 $O \left(\right. n \left.\right)$ 时间内完成多模式匹配（其中 n 为待匹配字符串的长度）。下面以模式串「 he / she / his / hers 」构建一个 AC 自动机举例说明（如图一所示）

![](https://pic2.zhimg.com/v2-1af7cb53112eeecc2a19a198dfbaa251_1440w.png)

当输入一个字符串时「ushers」，该自动机从状态 0 开始进行状态转换，完整的状态转移路径如图二所示

![](https://pic2.zhimg.com/v2-3e411f4c181da4efa633be13d58e7a09_1440w.png)

当遇到 AC 中的红色节点时，说明发生了模式匹配，匹配到的模式有：「he」、「she」、「hers」。

具体可以用 [Double Array Trie](https://zhida.zhihu.com/search?content_id=2520995&content_type=Article&match_order=1&q=Double+Array+Trie&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODE3NzA2NTIsInEiOiJEb3VibGUgQXJyYXkgVHJpZSIsInpoaWRhX3NvdXJjZSI6ImVudGl0eSIsImNvbnRlbnRfaWQiOjI1MjA5OTUsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.Vp9UTCZDBL89etJ71ndY_7KbmzbC789viXoY7OGjeiw&zhida_source=entity) 实现 AC 自动机，在保持高效多模匹配的基础，进一步节省空间。

**[贝叶斯方法](https://zhida.zhihu.com/search?content_id=2520995&content_type=Article&match_order=1&q=%E8%B4%9D%E5%8F%B6%E6%96%AF%E6%96%B9%E6%B3%95&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODE3NzA2NTIsInEiOiLotJ3lj7bmlq_mlrnms5UiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNTIwOTk1LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.ql7er7Kgbf_NJcv4pZW6hVkif_HZB1uss7qFg8BR5iY&zhida_source=entity)**

虽然 AC 自动机能快速的从字符串中找到存在于词典中的关键词，但这仅仅能满足一小部分需求，即不顾准确率的大量召回，很显然会造成误伤，这对知友也是很不友好。肯定有知乎用户会问，直接用「贝叶斯」方法不就可以搞定吗？你看人家做 spam 邮件过滤，不也做得还不错，还用什么 AC ？对对对，你说的都是正确的。但是实验发现，单纯地用 「贝叶斯」方法直接进行过滤时，准确率和召回率都不是很理想。究其原因呀，有 1）知友们知识面广、思维发散，2）长尾，很多词语出现频次相对较低。

**AC + 贝叶斯 > max { AC, 贝叶斯 }**

考虑到上述问题，我们提出了利用 AC 自动机，根据设定的类别关键词圈定相应类别的内容，然后在每个类别里利用「贝叶斯」方法的思想准确过滤出垃圾内容。现在我们有了解决问题的思想（思想很重要），来看看我们具体是怎么利用 AC 和「贝叶斯」这两个神器，打造垃圾内容过滤的。直接上图，一图胜千言。

![](https://pica.zhimg.com/v2-031bcb2598eb5fa2a5725b7952334e52_1440w.png)

图三中「主关键词」就是利用 AC 自动机按关键词圈定相应类别的内容。圈定之后，利用「可有可无」这儿所配置的策略在每个类别里进行垃圾内容过滤，策略即是利用「贝叶斯」思想总结出来的。

下面以评论数据为例，介绍如何运用贝叶斯方法来总结策略。

首先，分析样本数据，提取每一个词，计算每个词在正常评论和垃圾评论中出现的频率。比如，我们假定「sb」这个词，在 1000 条垃圾评论中，有 500 条包含该词，那么它的出现频率就是 $0.5$ ；而在 1000 条正常评论中，只有 2 条包含该词，那么出现频率就是 $0.002$ 。那现在对一条新的评论，发现其中包含「sb」这个词，它是垃圾评论的概率可以通过式一计算。（此处用 $S$ 和 $H$ 分别表示垃圾评论和正常评论, $W$ 表示词 「sb」， $P \left(\right. S \left.\right)$ 表示垃圾评论的概率， $P \left(\right. W \left|\right. S \left.\right)$ 表示垃圾评论中 $W$ 出现的频率）

$P \left(\right. S \left|\right. W \left.\right) = \frac{P \left(\right. W \left|\right. S \left.\right) P \left(\right. S \left.\right)}{P \left(\right. W \left|\right. S \left.\right) P \left(\right. S \left.\right) + P \left(\right. W \left|\right. H \left.\right) P \left(\right. H \left.\right)}$ (式一）

在没有更多先验知识的情况下，我们通常假设 $P \left(\right. S \left.\right) = P \left(\right. H \left.\right) = 0.5$ 。那在前文的例子中，很容易计算出 $P \left(\right. S \left|\right. W \left.\right) = 0.996$ ，说明「sb」词很容易区分出垃圾评论。通过这样的方式去挖掘出词语，当然也可以从正面角度考虑，比如「我」这个词，在我们的数据中能较好地区分出不是垃圾评论。此外，还可以考虑多个词语联合共现，甚至词之间的空间结构关系。这些在目前的逻辑里都是支持的。

有了具体实现，我们来看看实际的效果，如图四所示。（对于不利于讨论的内容也会被处理）

![](https://pica.zhimg.com/v2-f74609deeb71f5b25322e5d0db71679e_1440w.png)

线上效果如图五所示。

![](https://pic3.zhimg.com/v2-55d28cd51be4801d2da8bd901b7ac89e_1440w.png)

这套逻辑已融入到了算法机器人「 [瓦力](https://zhida.zhihu.com/search?content_id=2520995&content_type=Article&match_order=1&q=%E7%93%A6%E5%8A%9B&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3ODE3NzA2NTIsInEiOiLnk6blipsiLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNTIwOTk1LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.mFXu1SrAL5JgJ2IEWaBrLimy6YaLzo_xQEGGreO-RV4&zhida_source=entity) 」的大脑中，在知乎的诸多场景下，如评论、私信、回答、提问等，以 99% 的准确率处理着每天产生的垃圾内容。 [每天处理掉 3000+ 条垃圾评论](https://zhuanlan.zhihu.com/p/23425975) ，上线后处理了 [站内上万条封建迷信提问](https://zhuanlan.zhihu.com/p/24641430) 、上千条代为完成个人任务、上千条求医问药等违规提问，帮助知友们维护起了一个「友善」的讨论环境。

此外，这套系统也十分方便运营人员实现一站式自助策略管理。首先通过样本制定策略，然后通过离线版本进行策略验证，评估其准确率和召回率，最后自助上线策略。整个过程均无需工程师的介入，大大提高了运营效率。

**总结和展望**

为了让知乎用户友善地讨论交流，我们踏出了这一小步，主动识别处理了诸多垃圾内容。但还有很长的路要走，后续我们将为机器人「瓦力」打造更加完善智能的大脑，如自动归纳策略，引入深度学习等，为其建立更加科学高效的识别能力，以全自动的方式准确地识别出所有内容。

虽以识别垃圾内容为出发点，建立了基于 AC 自动机和「贝叶斯」思想的内容识别系统，但这套系统还可以运用到其他场景，比如舆情、其他内容归类等，在这儿就不展开了。

作者： [秦洛](https://www.zhihu.com/people/qin-luo-68) ，张瑞，王璐

Reference:

\[1\] [Aho-Corasick Algorithm](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Aho%25E2%2580%2593Corasick_algorithm)

\[2\] [Finite State Machine](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Finite-state_machine)

\[3\] [Deterministic Finite Automaton](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Deterministic_finite_automaton)

\[4\] [AhoCorasickDoubleArrayTrie](https://link.zhihu.com/?target=https%3A//github.com/hankcs/AhoCorasickDoubleArrayTrie)

\[5\] [Bayesian Inference](https://link.zhihu.com/?target=http%3A//www.ruanyifeng.com/blog/2011/08/bayesian_inference_part_two.html)

我们正在招聘：

\- [数据挖掘工程师](https://www.zhihu.com/careers/167)

\- [算法专家](https://www.zhihu.com/careers/377)

欢迎投递简历到： [jobs@zhihu.com](mailto:jobs@zhihu.com)

「 [知乎技术日志](https://zhuanlan.zhihu.com/hackers) 」是知乎工程师运营的一个技术专栏，在这里我们会陆续将知乎在机器学习、人工智能、稳定性和安全管理、反作弊系统、微服务实践、Docker、自动化运维、移动端网络优化等领域的技术思考和实践分享给大家。希望各位大家给予关注，并提出你宝贵的意见和反馈。

编辑于 2017-04-07 20:47[豆包让小白也能玩转 AI 漫画图片生成](https://www.doubao.com/chat/?channel=dbweb_zhihu_xxl_pc_cpc_ty_shengt_stcj_dmtp_512&source=dbweb_zhihu_xxl_pc_cpc_ty_shengt_stcj_dmtp_512&keywordid=3734823&ad_platform_id=zhihu_feed_lead&ug_callback_url=https%3A%2F%2Fsugar.zhihu.com%2Fplutus_adreaper_callback%3Fsi%3D79ee0fa2-033b-4f15-b01a-a778bca950d8%26os%3D3%26zid%3D1629%26zaid%3D3744379%26zcid%3D3734823%26cid%3D3734823%26event%3D__EVENTTYPE__%26value%3D__EVENTVALUE__%26ts%3D__TIMESTAMP__%26cts%3D__TS__%26mh%3D__MEMBERHASHID__%26adv%3D784531%26ocg%3D0%26cp%3D0%26ocs%3D0%26aic%3D0%26atp%3D0%26ct%3D0%26ed%3DGiBNJgVzfCMmUW9XFyEvRA8xBGxJICwkOhh0FlwxKw1Gdx87VSAsMi9Cb0oDdj1dByRedwhlKy0iVm9XFyU5WQ94CH0Kcmt5eRFmUQVheANYdx8lViYzJHMVdAtEbXyrfWDZIhpJ6w%3D%3D&cb=https%3A%2F%2Fsugar.zhihu.com%2Fplutus_adreaper_callback%3Fsi%3D79ee0fa2-033b-4f15-b01a-a778bca950d8%26os%3D3%26zid%3D1629%26zaid%3D3744379%26zcid%3D3734823%26cid%3D3734823%26event%3D__EVENTTYPE__%26value%3D__EVENTVALUE__%26ts%3D__TIMESTAMP__%26cts%3D__TS__%26mh%3D__MEMBERHASHID__%26adv%3D784531%26ocg%3D0%26cp%3D0%26ocs%3D0%26aic%3D0%26atp%3D0%26ct%3D0%26ed%3DGiBNJgVzfCMmUW9XFyEvRA8xBGxJICwkOhh0FlwxKw1Gdx87VSAsMi9Cb0oDdj1dByRedwhlKy0iVm9XFyU5WQ94CH0Kcmt5eRFmUQVheANYdx8lViYzJHMVdAtEbXyrfWDZIhpJ6w%3D%3D&ug_semver=v1.0.0&spu=biz%3D0%26ci%3D3734823%26si%3D646014d4-75eb-411e-931a-95439e2aa6e7%26ts%3D1781597853%26zid%3D1629)

[

豆包让小白也能玩转 AI 漫画图片生成

](https://www.doubao.com/chat/?channel=dbweb_zhihu_xxl_pc_cpc_ty_shengt_stcj_dmtp_512&source=dbweb_zhihu_xxl_pc_cpc_ty_shengt_stcj_dmtp_512&keywordid=3734823&ad_platform_id=zhihu_feed_lead&ug_callback_url=https%3A%2F%2Fsugar.zhihu.com%2Fplutus_adreaper_callback%3Fsi%3D79ee0fa2-033b-4f15-b01a-a778bca950d8%26os%3D3%26zid%3D1629%26zaid%3D3744379%26zcid%3D3734823%26cid%3D3734823%26event%3D__EVENTTYPE__%26value%3D__EVENTVALUE__%26ts%3D__TIMESTAMP__%26cts%3D__TS__%26mh%3D__MEMBERHASHID__%26adv%3D784531%26ocg%3D0%26cp%3D0%26ocs%3D0%26aic%3D0%26atp%3D0%26ct%3D0%26ed%3DGiBNJgVzfCMmUW9XFyEvRA8xBGxJICwkOhh0FlwxKw1Gdx87VSAsMi9Cb0oDdj1dByRedwhlKy0iVm9XFyU5WQ94CH0Kcmt5eRFmUQVheANYdx8lViYzJHMVdAtEbXyrfWDZIhpJ6w%3D%3D&cb=https%3A%2F%2Fsugar.zhihu.com%2Fplutus_adreaper_callback%3Fsi%3D79ee0fa2-033b-4f15-b01a-a778bca950d8%26os%3D3%26zid%3D1629%26zaid%3D3744379%26zcid%3D3734823%26cid%3D3734823%26event%3D__EVENTTYPE__%26value%3D__EVENTVALUE__%26ts%3D__TIMESTAMP__%26cts%3D__TS__%26mh%3D__MEMBERHASHID__%26adv%3D784531%26ocg%3D0%26cp%3D0%26ocs%3D0%26aic%3D0%26atp%3D0%26ct%3D0%26ed%3DGiBNJgVzfCMmUW9XFyEvRA8xBGxJICwkOhh0FlwxKw1Gdx87VSAsMi9Cb0oDdj1dByRedwhlKy0iVm9XFyU5WQ94CH0Kcmt5eRFmUQVheANYdx8lViYzJHMVdAtEbXyrfWDZIhpJ6w%3D%3D&ug_semver=v1.0.0&spu=biz%3D0%26ci%3D3734823%26si%3D646014d4-75eb-411e-931a-95439e2aa6e7%26ts%3D1781597853%26zid%3D1629)