---
layout: post
title: New Pre-Print (Inverse Filtering)
---

<p align="center">
    <img width="500" src="/img/if_preprint.png">
</p>

You can now find a pre-print of our latest work [*Inverse Filtering for Hidden Markov
Models with Applications to Counter-Adversarial Autonomous
Systems*](https://arxiv.org/pdf/2001.11809.pdf) on arXiv. In this paper, we provide
important extensions to the inverse filtering algorithms proposed in our earlier [NeurIPS
paper](http://papers.nips.cc/paper/7008-inverse-filtering-for-hidden-markov-models).

By observing, or intercepting, posterior distributions from a [Bayesian
filter](https://en.wikipedia.org/wiki/Recursive_Bayesian_estimation), we seek to estimate
*i)* the model of the [dynamic system](https://en.wikipedia.org/wiki/Markov_model),
*ii)* the accuracy of the sensors and *iii)* the measured observations. 

We also discuss the design of counter-adversarial systems. As in our [ICASSP'20
paper](https://rmattila.github.io/2019/10/18/preprint/), the setup is that an enemy:

1. measures our current state [via a noisy
   sensor](https://en.wikipedia.org/wiki/Hidden_Markov_model), 
2. computes a [posterior
   estimate](https://en.wikipedia.org/wiki/Recursive_Bayesian_estimation) (belief) and
3. [takes an
   action](https://en.wikipedia.org/wiki/Partially_observable_Markov_decision_process) that we can observe.

Based on observations of the enemy's actions and knowledge of our own state sequence, we
estimate the accuracy of the enemy’s sensors -- this forms a foundation for predicting,
and taking appropriate measures against, its future actions.

Have a look, and feel free to send me any comments you may have by email!


