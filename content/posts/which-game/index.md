---
title: "Sparks of Weight-Space Programming, 36 networks later"
date: 2026-11-04
draft: false
summary: "Experiment 1 of Sparks of Weight-Space Programming, repeated on 40 networks and five models."
---

<!--
Draft skeleton. The notes in these comments are facts to use, not text; they don't show on the site.
Images go in this folder next to index.md and are linked as ![alt](figure.png).
Numbers: accuracy counts Freeway and Frogger as the same game, as the authors did.
-->

<!-- Intro (sections 1-3 of the outline). TODO: put the HN thread URL in the "Hacker News" link. -->

**TL;DR:** I repeated the "which game was this network trained on?" experiment from [Sparks of Weight-Space Programming](https://rikiyat.github.io/blog/weight-space-programming/)
on 36 new networks across 12 games. GPT-6 Astra goes from 4/4 accuracy on the original networks to 27/36 on mine. I find that some games consistently fool the LLMs, while others seem much easier to identify.

{{< svg-figure src="paper-vs-ours.svg" caption="Accuracy on the original 4 networks next to my 36." >}}

This week the research blog post [Sparks of Weight-Space Programming](https://rikiyat.github.io/blog/weight-space-programming/) by Rikiya Takehi et al. reached the front page of Hacker News. The question that set off debate was whether LLMs can understand a neural network's behaviour from looking at just its weights.

Their first experiment tests this question: Pick 4 games that are played on the same 10×10 grid, train a small neural network to play these games, feed the weights as plain numbers into an LLM and ask what game the network was trained on. The LLM had no access to tools or code execution. Astra got all four games from the original blog post correct: Breakout, Pong, Pac-Man, and Freeway.

A day later Simon Willison [tried the same prompts with Qwen 3.8 27B on his laptop](https://simonwillison.net/2026/Nov/3/qwen-can-interpret-model-weights/).
After roughly an hour of reasoning per prompt it named Breakout, Pong and Pac-Man, and only missed Freeway.

But both results rely on four classic games. That leaves some questions open:

- **Are these results consistent on a bigger sample size?**
- **Is the ability to interpret weights something only SOTA models are capable of?**
- **Does Astra understand how the network functions, or does it make an educated guess between a few famous games?**

So I rebuilt the setup and made it bigger:

- **12 games instead of 4**, including some games that look alike on a 10×10 grid (Freeway and Frogger, Snake and Maze, Pac-Man and Sokoban).
- **3 networks per game**, each trained from a different random seed: 36 new networks, plus the original experiment's 4.
- **The original prompt format, network size, and PPO hyperparameters**, applied to my own game implementations.
- **Five models:** GPT-6 Astra, GPT-6.1 Sol and GPT-6.1 Terra (through Codex, as the authors ran Astra), DeepSeek V4 Flash, and the same Qwen 3.8 27B that Simon used.

I published everything I used to derive these results [on GitHub](https://github.com/wohoef/name-the-game). This includes training code, the games, all 40 networks, and every model's answer.

## Findings

**Are these results consistent on a bigger sample size?** Mostly yes, for the best model. GPT-6 Astra named 27 of my 36 networks correctly (75%). The other models drop off quickly: GPT-6.1 Sol 24/36, GPT-6.1 Terra 17/36, DeepSeek V4 Flash 32/72 (two answers per network) and Qwen 3.8 27B 4/36. Every model did better on the original four networks than on mine.

**Is the ability to interpret weights something only SOTA models are capable of?** The strongest models performed better. During my experiments Qwen got only 1/4 of the original networks right, while Simon's run with the same model got 3/4 correct. Small samples are noisy: I asked DeepSeek Flash twice about each of my networks, and both answers named the same game for only 13 of the 36. Another difference is that I ran the model through OpenRouter and Simon ran it locally. My results did show a clear skill gap between Qwen and Astra (as expected): For 17 of my 36 networks, Qwen's answer did not map to any of my scoring labels. Many of these answers mentioned some variant of "grid world".

Here is the end of Qwen's answer for one of my Pong networks:

> … The architecture (MLP on flattened grid) is standard for simple discrete control tasks.\
> **Game:** Grid World Navigation (or Goal Seeking).\
> **Description:** The agent navigates a 10x10 grid from a starting position (likely on the top or left border) to a
> goal reward at the bottom-right corner (9,9), avoiding wall collisions.\
> **Confidence:** 90%

More output tokens did not translate into better results across these models. DeepSeek Flash wrote about 42,000 output tokens per answer, mostly reasoning, while Astra used only 2,300 on average.

**Is it reading the network, or guessing a famous game?** The mistakes show a pattern. Some games are consistently much harder to identify:

{{< svg-figure src="per-game.svg" caption="Answers correct per game on my networks, 3 networks per game (DeepSeek answered each twice), sorted from easiest to hardest. Freeway and Frogger are accepted interchangeably for accuracy." >}}


Every answer is in the confusion matrices below. Rows are the game a network was trained on, columns the game the model named; Freeway and Frogger are kept apart here (but they are equivalent for how I compute accuracy). Click a model to open its matrix.

<details class="cm"><summary>GPT-6 Astra <span class="cm-acc">27/36 correct</span></summary>

{{< svg-figure src="confusion-gpt-6-astra.svg" >}}

</details>
<details class="cm"><summary>GPT-6.1 Sol <span class="cm-acc">24/36 correct</span></summary>

{{< svg-figure src="confusion-gpt-6.1-sol.svg" >}}

</details>
<details class="cm"><summary>GPT-6.1 Terra <span class="cm-acc">17/36 correct</span></summary>

{{< svg-figure src="confusion-gpt-6.1-terra.svg" >}}

</details>
<details class="cm"><summary>DeepSeek V4 Flash <span class="cm-acc">32/72 correct (two answers per network)</span></summary>

{{< svg-figure src="confusion-deepseek-flash.svg" >}}

</details>
<details class="cm"><summary>Qwen 3.8 27B <span class="cm-acc">4/36 correct</span></summary>

{{< svg-figure src="confusion-qwen3.8-27b.svg" >}}

</details>

I find that the harder to identify games share a layout with another game: a Maze agent and a Snake agent both walk towards a target, and a Sokoban agent moves around walls much like Pac-Man. Astra's nine errors were concentrated in three games: it missed all three training seeds of Maze, Snake and Sokoban. These are different trained networks, rather than repeated attempts on the same network.

These findings make me wonder if the LLMs used the weights to find the layout of the game and a very general strategy. But they seem to lack the finer details. One possibility is that they then pick a familiar game that resembles this layout. The 4 games in the original experiment have distinct layouts, which may make the task easier. These experiments don't establish whether that is what the LLMs are doing.

<!-- Figures are made with `python scripts/blog_figures.py` in ~/pro/which_game. -->

## How I ran the experiment

I implemented twelve simplified games using the same 10×10 grid, five observation channels and five actions. Each network is a 500→32→32→5 MLP with tanh hidden layers. I trained three networks per game from different random seeds, for approximately one billion environment steps each, using the authors' PPO hyperparameters. I included all three seeds; the authors selected the best of three for each game.

Each model received the original prompt format: the observation and action definitions, followed by all 17,253 network parameters rounded to three decimals. It received no game name, filename or list of candidate games. For the four original networks, the prompts match the released prompts byte-for-byte. Each call used a fresh conversation without tools.

Most models answered once per network. I ran DeepSeek Flash twice on each of my networks and four times on each original one. The GPT models ran through Codex; DeepSeek through its API; and Qwen through OpenRouter, pinned to Alibaba's host. All used high reasoning effort. Full settings are in the repository.

I scored answers by mapping the final `Game:` line to game labels using keywords. Freeway and Frogger are accepted interchangeably, following the authors' treatment of Frogger answers for Freeway. Every answer can be inspected alongside its score.

There are limits to what this tests. The environments are my implementations, not the original game code. The Freeway networks learned an almost constant “up” strategy, with small but nonzero first-layer changes, so identifying them does not demonstrate recovery of traffic-avoidance behaviour. Three training seeds and mostly one answer per network also leave uncertainty about how consistently these results would repeat.

## Code and data

Everything is at [github.com/wohoef/name-the-game](https://github.com/wohoef/name-the-game): the games, the training code, all 40 networks and every model answer, with the notebook that makes the tables and figures.
