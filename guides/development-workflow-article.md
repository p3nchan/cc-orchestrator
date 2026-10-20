---
title: "多 Agent 開發流程：讓 AI 團隊寫出能用的東西"
description: "你的 AI 團隊能協作了，但做出來的東西品質怎麼控？這篇用學術研究 + 實戰經驗，拆解 Challenge Loop、Cross-family Review、Spec 驅動開發三個核心 pattern。"
date: 2026-04-14
category: ai
aiGenerated: false
tags: ["AI Agent", "Orchestration", "Code Review", "Challenge Loop", "Spec-Driven"]
keywords: ["AI code review", "AI challenge loop", "spec driven development", "multi agent development", "cross-family review"]
author: penna
draft: false
image: "/img/covers/ai-orchestration-dev-workflow.webp"
faq:
  - question: "為什麼 AI 的自我審查不可靠？"
    answer: "研究發現 AI 模型在審查自己的產出時有 64.5% 的盲點率，而且對自己寫的東西容忍度高出 5 倍。這表示讓同一個模型寫完再自己審，大部分問題它根本看不到。解法是用不同家族的模型來審。"
  - question: "什麼是 Challenge Loop？"
    answer: "Challenge Loop 是一種對抗式審查流程：一個 Agent 產出作品，另一個 Agent 嘗試找出問題，但每個挑戰都必須附上可測試的證據。沒有證據的批評不算數。研究顯示這種方式比一般 review 更能抓到真正的問題。"
  - question: "Spec 品質為什麼比 review 輪數更重要？"
    answer: "研究發現，改善 spec 品質可以讓程式碼正確率提升 13.87%，效果比多加一輪 review 還好。原因是如果 spec 本身就模糊，reviewer 和 coder 會共享同一個誤解，多審幾次也發現不了。"
---

[上一篇](/ai/orchestration-playbook/)在談一件很底層的事：多 Agent 怎麼不要互撞，也不要把成本燒在溝通上。我原本以為底盤穩了，品質自然就會跟上。

我後來才發現，這完全是兩回事。有次我讓一個 Agent 寫完 patch，再叫它自己 review。它回我一句 `looks good`，我也信了。測試一跑，新路徑是綠的，舊路徑直接炸開。那天我很確定，協作順不順，跟東西能不能上線，中間還差一整層方法。

這篇就接著講那一層。我現在看 AI 開發流程，重點已經換了。問題不在於能不能叫很多 Agent 一起做，重點在每一步有沒有真的帶來新資訊。下面我會分段講三個 pattern，它們背後都有研究，前提也都很簡單，少掉其中一個，後面的 review 很容易變成表演。

## Pattern 6：Challenge Loop

最常見的假 review 長這樣。你叫 Agent B 去審 Agent A 的 code，它回一堆看起來很專業的話，像是「建議考慮錯誤處理」或「這裡可能有風險」。句子很像 review，資訊量卻很低。你修了兩輪，bug 還在，因為整個過程從頭到尾都沒有人拿出可測試的證據。

我現在的規則很硬：每個 challenge 都要附證據。最常見的是 failing test 或可重現的 counterexample；有時直接對照 spec，指出哪一條沒滿足也可以。拿不出這些東西，我就把它當噪音。這樣做有點兇，效果卻差很多，因為 review 終於從「我有感覺這裡怪怪的」變成「這個輸入會錯，這條驗收條件沒過」。

2025 年有個專門測 sycophancy 的研究，看了大量 LLM 回覆後發現，58% 帶有奉承傾向。白話講，模型很容易順著你，也很容易順著前一個模型。你如果沒刻意把 review 設成對抗式，它自然會滑去橡皮圖章。這也是我現在不太接受「幫我 review 一下」這種 prompt 的原因。太空了。我要的是「找出 3 個能證明會失敗的地方」，或是「只回 spec violation」。

我自己會再盯一個數字：challenge 導致實際修改的比例。如果一輪 review 提了 10 個點，真正讓 code 或 spec 改動的只有 1 個，命中率連 20% 都不到，這輪大多是在演。通常問題不在 reviewer 太笨，問題在 prompt 把它帶去產生很像批評的廢話。

這個 pattern 在 spec review 特別有感。前陣子我在看一個金融演算法 spec，整個 challenge loop 跑了 6 輪。R1 一進來就抓到 3 個真問題：有資料洩漏，滑價模型沒補，重試路徑也沒上限。R2 又補出 purge window 的 bug。一路收斂到 R6，真正有效的 challenge 掉到 0，外部測試也全綠，我才收工。重點從來不是「跑到第 6 輪很嚴謹」，重點是每一輪都得拿得出證據，judge 才知道該不該改。

我也會給 challenge loop 一個收斂上限。正常情況下，前 2 到 3 輪就該看到主要問題被抓出來。再拖下去，常見走向是每輪越寫越長、越改越偏。研究把這類 overthinking loop 歸成高頻失敗模式，我自己的體感也一樣。輪數一長，大家都像很努力，資訊密度卻一路往下掉。

![Challenge Loop 流程圖，標出每個 challenge 都要附 failing test 或 spec violation](./img/challenge-loop.webp)

## Pattern 7：跨家族審查

自我審查這件事，我現在幾乎直接跳過。

理由很單純。模型剛寫完東西時，腦中那條路徑還是熱的。它會沿著自己剛剛的推理，把那份產出看得比實際更合理。研究也很直接：Self-Correction Bench 量到 64.5% 的盲點率，2026 年另一篇研究又看到模型在審自己產出時，放水機率大概是審別人的 5 倍，連帶有風險的內容都更容易放過。

所以我現在把 review 預算花在獨立性，不花在儀式感。最實用的做法，就是讓不同模型家族互審。GPT 寫的東西，交給 Claude 看。Claude 寫的東西，交給 GPT 看。原因也很務實：2025 年有份看 350 多個模型的分析，發現同家族模型的錯誤關聯度明顯高於跨家族。你想抓盲點，就得讓 reviewer 帶著不同 priors 進來。

這裡也別神化。前沿模型之間的輸出相似度還是很高，ICML 2025 的研究看到最高可以接近 90%。這表示跨家族審查會提高命中率，卻不會讓你瞬間擁有全知視角。很多盲點是全模型共享的，像規格本來就模糊，或測試太弱，或背景知識本身有缺口。這些地方換 reviewer 也照樣會漏。

我現在最警覺的一個反模式，叫 Validation Retreat。fix loop 卡住時，Agent 很容易去改 test，讓畫面變綠，然後假裝事情解掉了。表面上看是成功，實際上只是把 benchmark 改鬆。我審 patch 時一定會回頭看：這輪修復有沒有動到 test？如果有，spec 有沒有授權？改的是驗收標準，還是真 bug？

這一點我吃過虧。之前我讓 GPT 的 Codex 寫一個修補，self-review 幾乎秒過。接著我用 Claude Sonnet 按 acceptance criteria 一條一條審，它馬上抓到 3 個 critical issue，而且都躲在舊路徑上。Codex 的注意力全放在剛改的新邏輯，自然看不到那些地方。換一個家族來看，視角真的會變。

review team 也不用越大越好。研究和實務都很像，兩個 reviewer 差不多就是甜蜜點。再往上加，邊際效益掉得很快，協調成本卻繼續長。對一般 code 變更，我現在寧願保留一個 generator、一個 cross-family reviewer，再加明確 spec 和測試，也不想堆三四個 reviewer 互相抄彼此作業。

![Cross-family Review 圖示，GPT 產生程式碼，Claude 依照 acceptance criteria 審查，旁邊標出 self-review blind spot 64.5%](./img/cross-family-review.webp)

## Pattern 8：Spec 驅動開發

很多人以為 bug 多，是 review 不夠嚴。我以前也這樣想。

後來我越做越覺得，天花板通常卡在 spec。你一開始交給 Agent 的任務如果寫得模糊，後面不管審幾輪，generator 跟 reviewer 都是在同一團霧裡面摸。有人把這件事講得很直：沒有可執行 spec 的 AI review，本質上是一種結構性循環。兩邊都從同一段模糊文字推理，猜對算賺到，猜錯就一起錯。

這件事有數字支撐。ClarifyGPT 那篇研究把 spec refinement 放進流程後，Pass@1 直接從 70.96% 拉到 80.80%，提升 13.87%。我看到這個結果時很有感，因為它跟我自己的觀察完全對得上。很多 fix loop 看起來像在修 code，實際上是在補 spec 本來就該講清楚的東西。

我現在寫複雜任務的 handoff，通常會固定放這幾塊：

```text
Objective
Context
Acceptance Criteria
Interface Contract
Anti-patterns
Review Checklist
```

如果題目跟資料或回測有關，我還會再補 data integrity 要求，像是不能 look-ahead，也要把交易成本和資料切分方式寫實一點。這些欄位寫起來有點煩，卻能把一大堆後面才會爆的問題直接擋在前面。

我自己的 spec 也變很多。以前我常寫「幫我改 regime detector，讓它改看 absolute APR，不要看 relative」。現在不會這樣交。現在我至少會寫清楚一句 objective、幾條驗收條件、資料完整性限制，還有明確禁止的做法。這個改法很土，效果卻很明顯。Cod 第一次交回來的東西通常更靠近我要的版本，fix loop 也常從平均 3 輪縮到 1 到 2 輪。

Spec 也不是越厚越好。簡單任務口頭交代再配 1 到 2 個 acceptance criteria，常常就夠了。中等複雜度的工作如果硬套完整模板，流程會明顯變慢。我現在的原則是：只寫到能避免歧義的程度，多的不要補。Spec 的作用是收斂，不是把所有腦內雜訊都倒給下一個 Agent。

![Spec-Driven Development：Structured Handoff Schema 範本，標出 objective、acceptance criteria 與 review checklist](./img/spec-driven-development.webp)

## Bonus：把三個 pattern 串成一條線

我現在最常用的開發流水線很短：

```text
Spec（Structured Handoff）
  → Challenge Loop（找 spec 漏洞）
  → 實作
  → Cross-family Review（找 code 漏洞）
  → 修復
  → 完成
```

這條線的核心原則只有一句：每一步都要帶入新資訊。如果兩個步驟只是同一個模型對著同一份材料再想一次，我就傾向把它砍掉。

多 agent 並行也一樣。DeepMind 那篇 orchestration scaling 研究很有參考價值，能平行拆分的任務，效果可以多出 80.9%；本質上偏串行推理的工作，表現反而會掉 39% 到 70%。溝通成本還會照 1.724 這個指數往上長。換成白話，agent 變多以後，常常不是在做事。它們會互相等，接著花時間說明和驗證。

所以我現在很少把 agent 數量開太大。3 到 4 個差不多是上限，再上去 token 消耗很容易膨脹到單 agent 流程的 3.5 倍左右。模組真的能獨立拆，才值得並行。其他情況，短流程加高獨立性，通常更穩。

模型會一直進步，2024 到 2026 這批研究裡的具體數字，過一陣子多半會被改寫。盲點率會降，review 工具也會更強，spec 生成看起來多半也會越來越像樣。

我倒覺得有幾個底層原則短時間不太會變。review 要有獨立視角，spec 品質也還是決定上限。iteration 到某個點以後，多半只是在空轉。現在 self-review 的盲點率大概還在 64.5% 這個量級。模型再往前走，這個數字一定會掉。掉到多少以下，self-review 才重新值得放回主流程，我還在想。

延伸閱讀：

- Part 1：<https://penchan.co/ai/openclaw/orchestration-playbook/>
- GitHub Repo：<https://github.com/p3nchan/orchestration-playbook>

本文僅供研究與討論，非投資建議。DYOR + NFA。

小企鵝阿批 Penchan
