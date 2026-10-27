---
layout: post
title: teaching a car to drive inside its own dream
date: 2026-10-27 10:00:00
description: a latent space that could predict but not drive, the structural fix that made it drive, and three ablations that each overturned my previous conclusion
tags: reinforcement-learning world-models digital-twins
categories: research-notes
og_image: https://fpoltronieri.github.io/assets/video/drive.png
_styles: >
  .clip {
    margin: 2rem 0;
    border: 1px solid var(--global-divider-color);
    border-radius: 6px;
    overflow: hidden;
  }
  .clip video {
    display: block;
    width: 100%;
    max-width: 100%;
    background: #2b2f2c;
  }
  .clip figcaption {
    font-size: 0.875rem;
    line-height: 1.5;
    color: var(--global-text-color-light);
    padding: 0.75rem 1rem;
    margin: 0;
    border-top: 1px solid var(--global-divider-color);
  }
---

Some months ago I started thinking about what to do next in my research career. I was at the ICIN
2026 conference in Athens, chatting with colleagues. Between teaching and everything else the
academic year brings, it is genuinely hard to find a moment to plan the steps ahead, and a
conference corridor is sometimes the only place where it happens.

I knew I wanted to do something around prediction and forecasting in the area of digital twins.

It was Prof. Stefano Secci who introduced me to the concept of world models. I had never looked into
them before — I only knew they were something Prof. Yann LeCun cared about a great deal.

## So what is a world model?

A world model is a neural network that learns to predict what happens next. Feed it what you
currently see and the action you are about to take, and it returns its guess about the next
observation and the next reward. Once you have one, something interesting becomes possible: you can
train a controller entirely against the model's predictions, never touching the real system. The
agent learns in a dream.

## World models in network and service management

I care about this for reasons that have nothing to do with cars. I work on orchestration across the
compute continuum, and the recurring obstacle there is that you cannot explore on a production
system. You cannot let a reinforcement learning agent try ten thousand bad autoscaling decisions on
a live cluster to find out which ones were bad. A world model learned from operational telemetry is,
in effect, a digital twin you can be reckless inside.

Before building that, I wanted to know whether I could make the machinery work at all on a problem
where I could see the answer with my own eyes. So: CarRacing, from pixels, five discrete actions
(nothing, left, right, gas, brake).

There was a second experiment running underneath the first one. I had no background in this area, so
I used generative AI throughout — LLM coding assistants, both to write the implementation and to
learn what I was writing. That was deliberate, and I come back to it at the end, because it shaped
the result and the mistakes in about equal measure.

The first version did not work, and I was wrong about why for a long time.

## The first attempt: a latent space that could predict but not drive

My first approach trained a recurrent state-space model on driving trajectories, froze it, and then
trained a small policy on top of its latent state. The idea was that the latent would contain
everything needed to drive, so a tiny controller reading it should suffice.

The latent was not empty. I probed it: a linear head on the frozen latent predicted the expert's
action with 0.788 accuracy, and recovered track geometry reasonably well — an R² of about 0.6 for
the curvature of the road ahead, 0.44 for how centred the car was in its lane.

And yet the policies I trained on it drove terribly. Average return hovered between −60 and +50,
against roughly 900 for a good driver. I tried more expert data (4 episodes, then 16, then 128).
Supervised accuracy improved. Driving did not. I tried auxiliary losses, mixed in random off-road
episodes for recovery supervision, added hand-written fallbacks. Each change moved the number a
little and none of them made a car that could get around a track.

The lesson I eventually took, and the one I would offer to anyone in the same hole: **prediction
quality and control usefulness are not the same thing.** A representation that lets you *read off*
the right action in states an expert visits can still be useless in the states your own imperfect
policy actually reaches. I had been treating this as a planner-tuning problem. It was a
representation problem.

I did not know that at the time, though, so I put the project in standby. There was no dramatic
moment of giving up: the results simply did not justify more of my time while other work was live,
and it sat untouched for a while. When I eventually picked it back up, I started from the diagnosis
instead of from the code.

## The fix was embarrassingly structural

What my latent space lacked was a reason to keep information it did not immediately need. Nothing in
the training objective forced it to represent the whole scene — only enough to predict the next
latent and the reward.

So I rebuilt the thing properly, following [DreamerV3](https://arxiv.org/abs/2301.04104): a
convolutional encoder, a recurrent state-space model with categorical latents, and crucially **a
decoder that has to reconstruct the frame** from the latent state. Then an actor and a critic trained purely on imagined rollouts —
sequences the model generates internally, never in the real environment.

Training then runs in two phases. First the world model and the controller are trained offline on a
fixed set of 150 bootstrap episodes, each recorded on a different track. Then the agent drives 300
fresh tracks itself, and every episode it drives goes straight back into the replay buffer — so by
the end, most of what the world model has learned from is the agent's own experience rather than the
episodes it started with. Those starting episodes are a seed, never the whole diet.

The decoder changed everything. The same control approach that had been producing −15 on the old
latent produced a car that drove.

<figure class="clip">
  <video controls muted loop playsinline preload="metadata" poster="{{ '/assets/video/drive.png' | relative_url }}">
    <source src="{{ '/assets/video/drive.mp4' | relative_url }}" type="video/mp4">
    Your browser does not support the video tag.
  </video>
  <figcaption><strong>The trained agent on a track it has never seen.</strong> Every action is chosen by a policy that was trained entirely inside the world model, never in the real environment.</figcaption>
</figure>

One more fix mattered more than I expected. CarRacing's reward is spiky: you get about +3 for each
new piece of track, and a small negative every step. A regression head trained with mean-squared
error on such a target learns something close to the *average*, which systematically underestimates
the spikes. A quick sanity check: on a synthetic target that is +3 half the time and −0.2 the rest
(true mean 1.45), a symlog-MSE head learns 0.72, while a distributional "twohot" head — predicting a
distribution over 255 reward bins instead of a single number — learns 1.42.

In the real system that mattered: the model's prediction of a 15-step imagined return went from 8.3
to 14.0, against a true value of 13.0. Final driving performance improved by 18%. An agent optimizing
a reward model that under-reports progress drives timidly, which is exactly what it had been doing.

<figure class="clip">
  <video controls muted loop playsinline preload="metadata" poster="{{ '/assets/video/sees.png' | relative_url }}">
    <source src="{{ '/assets/video/sees.mp4' | relative_url }}" type="video/mp4">
    Your browser does not support the video tag.
  </video>
  <figcaption><strong>Left: the 64&times;64 frame the model receives. Right: the decoder&rsquo;s reconstruction of its internal state.</strong> The latent keeps the geometry &mdash; the curve, the car, even the HUD bars &mdash; and throws the texture away.</figcaption>
</figure>

And the part that still delights me — the agent closing its eyes. Below, the left panel is what the
model *imagines* while it drives; the right panel is reality receiving the same actions. The agent is
not looking at the right panel. For roughly 90 decisions it steers through a track it is only
dreaming, and the dream stays on the road.

<figure class="clip">
  <video controls muted loop playsinline preload="metadata" poster="{{ '/assets/video/dream.png' | relative_url }}">
    <source src="{{ '/assets/video/dream.mp4' | relative_url }}" type="video/mp4">
    Your browser does not support the video tag.
  </video>
  <figcaption><strong>Left: imagination. Right: reality, receiving the same actions, unobserved.</strong> 90 blind decisions, no run off the road, and the real track kept paying out reward the whole time.</figcaption>
</figure>

## Then the ablations, which took my conclusions apart three times

At this point I had a working agent at 781 average return over 20 unseen tracks, against 531 for the
DQN expert whose trajectories I had been learning from. Good enough to stop. So naturally I asked the
question that ruins afternoons: **how much of this comes from the expert at all?**

**First ablation.** Replace the expert's 150 episodes with 150 episodes of *random* driving, and
switch off behaviour cloning entirely. Pure reinforcement learning inside the world model, bootstrapped
from noise. I expected a degradation. It scored **867** — better than the version that had learned
from the expert, and better than the expert itself by a wide margin. My conclusion, which I wrote
down with far too much confidence: the teacher is unnecessary and actively harmful.

**Second ablation.** That conclusion had an obvious hole: I had removed two things at once, the
expert's *data* and the imitation loss. So: expert data, no cloning. This scored **899**, the best of
the lot. Which means my previous conclusion had been wrong, because it had conflated two effects
pulling in opposite directions. The expert's trajectories *help*; copying the expert's action choices
*hurts*, and hurts more than the data helps. Remove both and the net looks like a win.

The corrected version is a sentence I actually like: **use the expert's data, not its policy.** The
traces of a mediocre controller are valuable precisely because they cover the region of the state
space where the system operates. Its action choices are the thing you are trying to beat.

**Third ablation, which was about me rather than the method.** Every number above came from a single
training run. So I ran three seeds of each configuration on a GPU, and evaluated every checkpoint on
the same machine to avoid confounds:

| configuration | data | cloning | mean over 20 unseen tracks |
|---|---|---|---|
| random actions | — | — | −47 |
| DQN expert (model-free, 1M+ env steps) | — | — | 531 |
| world model + cloning | expert | yes | 781 |
| world model | random | no | 863 ± 29 |
| world model | expert | no | **899 ± 6** |

The headline gap survived: +36 for expert data, almost exactly what the single run had suggested.
But at three seeds per configuration that is t = 2.46, p ≈ 0.08 — real-looking, not significant. I am
not allowed to write "significantly better", and neither is anyone else with n = 3.

What *did* come out unambiguously is a different claim, and a better one. Paired track by track, the
expert-data version wins on 17 of 20 tracks, but the **median** track difference is only +12 against
a mean of +36. The mean is carried by a handful of tracks where the random-bootstrap version
collapses. Counting tracks scoring under 800, per individual run:

- random bootstrap: **1, 5, 0, 3** — and a *different* track each time
- expert data: **0, 0, 0** — no run, no track, no seed

So the expert's data does not make the agent drive faster. It makes it **stop having catastrophic
tracks**. That is a reliability result, it is much stronger than the mean, and I would never have
seen it from one seed.

## Four things I would tell my past self

**Two bugs that silently changed the experiment, not the output.** Twice I shipped a run that trained
fine, logged plausible losses, and answered a different question than the one I asked. Both times the
cause was a loss weight hardcoded in the pretraining loop while the command-line flag only governed a
later phase — so "reinforcement learning without a teacher" ran with *no actor gradient at all*
(the car sat still and scored −93), and later "expert data without cloning" ran 13,000 steps of
cloning. The second bug was invisible in the earlier experiment, because random episodes carry no
expert action labels, so the cloning loss was multiplied by an empty mask and read 0 regardless of
its weight. A silent bug that only manifests under one configuration is the worst kind.

**Evaluate on enough tracks, every time.** On a 10-track internal set the last two configurations
looked *tied* (876 vs 878). On the 20-track held-out set the same checkpoints differ by 36. I had
written this rule down months earlier, after an "improvement" from 586 to 737 collapsed back to 586
on more seeds — and then I broke my own rule and briefly announced that the +36 was noise.

**Floating point is a random seed.** The identical configuration with the identical seed, trained on
my laptop's GPU versus a CUDA card, scored 867 and 821 — a 46-point spread, *larger* than the
variation across seeds. Training is chaotic enough that a different arithmetic order diverges the run.
Which is clarifying, actually: it means hardware is not a separate confound to worry about, it is
seed noise, and should be pooled as such.

**The negative results were the useful ones.** The failed first attempt is what located the actual
problem (the representation), and two of the three ablations overturned the conclusion I had just
written down. If I had stopped at any point where the numbers looked good, I would have published something
wrong and felt fine about it.

## The other experiment: learning a field with generative AI

I should say plainly that I did not write all of this code by hand, and that this was the point. I
had no background in world models, no spare weekends to acquire one, and I wanted to know whether
LLM coding assistants are good enough to carry someone into an unfamiliar field — not to produce
code for a field they already know, which is the usual question, but to *learn* with.

My honest assessment, with the caveat that it is one project and one person: it moved the bottleneck
rather than removing it, and it moved it somewhere more interesting.

What it did extremely well was the accidental complexity. KL balancing, free bits, symlog, twohot
encoding, straight-through gradients — a long tail of small technical details that are easy to get
subtly wrong, miserable to debug, and mostly unexplained in the papers that use them. Having them
written correctly and, more importantly, *explained on demand at the exact moment I needed them*, is
a real acceleration in learning and not only in typing. The first attempt in this post was also
assistant-written, and it failed for a reason no amount of coding speed would have fixed.

What it did not do is protect me from being wrong. Both of the silent bugs above came out of code
that was generated quickly and reviewed too lightly: plausible, well-named, and answering a
different question than the one I had asked. And the conclusion I had to retract twice was not a
coding failure at all — it was an experimental design failure, too few seeds and too small an
evaluation set, and for a while the assistant and I were confidently wrong about it together. A tool
that is fluent in your domain will agree with you fluently.

So the division of labour I would suggest to anyone learning a field this way: let the tool carry
the implementation and the explanation, and keep the experimental protocol entirely for yourself.
Decide what each number has to be measured against *before* you look at it. That is the part where
being fast is worth nothing, and it is the part that decides whether you have learned the field or
merely produced its code.

## Where this goes next

Back to the thing I actually care about. What transfers to network and service management is not the
racing car — it is the shape of the finding. Production systems already run heuristic policies:
threshold-based autoscaling, ECMP, admission control. Those heuristics have been generating traces
for years, and nobody had to take a risk to produce them.

Whether that is enough to learn a useful twin of a real system, with real telemetry and real failure
modes, is the experiment I am setting up now. A racing car is not a Kubernetes cluster, and I expect
the honest answer to take a while. But I now have a piece of machinery I understand well enough to be
suspicious of, which is more than I had when I set this aside.

---

*The implementation is a from-scratch DreamerV3-style world model in PyTorch, about 1,200 lines,
trained on a MacBook (5.2 hours per run) and an RTX 5090 (0.85 hours per run). Every number above is
from a held-out set of 20 tracks never used for training or model selection.*
