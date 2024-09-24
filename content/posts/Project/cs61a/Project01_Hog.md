---
date: 2024-01-24T22:20:27+08:00
title: CS61A Project1 Hog
tags:
  - CS61A
categories:
  - Project
draft: true
---

`python3 hog_hui.py`: 调用 gui 界面。

Rule1: Sow Sad. 选择摇 1-10 次骰子，得分为点数之和，如果摇到 1，那得分就为 1。  
Rule2: Boar Brawl. 可以选择不摇骰子，直接获得 3\*abs(对手得分的十位数-自己得分的个位数)的分数。  
Rule3: Sus Fuss. 如果得分数有 3 或 4 个公因子，比如 21 有 1,3,7,21，自动把分数加到下一个质数，21->23。
