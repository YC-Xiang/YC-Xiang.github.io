---
title: CS61A Project1 Hog
date: 2024-01-25 22:20:28
tags:
- CS61A
categories:
- Project
---

Rule1: Sow Sad. 选择摇1-10次骰子，得分为点数之和，如果摇到1，那得分就为1。
Rule2: Boar Brawl. 可以选择不摇骰子，直接获得3*abs(对手得分的十位数-自己得分的个位数)的分数。
Rule3: Sus Fuss. 如果得分数有3或4个公因子，比如21有1,3,7,21，自动把分数加到下一个质数，21->23。
