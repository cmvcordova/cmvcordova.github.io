---
layout: post
title: "What do you mean you can't run the notebook?"
date: 2026-02-25
description: "On research engineering, agentic workflows, and why the boring infrastructure work is the work that compounds."
---

In January 2024, Fabio Viola — a lead research engineer at DeepMind — gave a talk at Mila that changed how I think about my own work. The core idea was deceptively simple: faster and cheaper experimental loops mean more answers per unit time. Not more GPUs. Not a better architecture. Just: can you go from hypothesis to result and back again without everything falling apart?

At the time I was a fresh PhD student. I had aspirations about research in geometry and biology, not systems engineering. But I wrote that line down — *faster and cheaper loops* — and it's been load-bearing in my work ever since.

Two years later, I'm building frameworks where an AI agent can read a one-sentence experiment description, design a 300-configuration sweep, submit it to a GPU cluster, and pull analyzed results into a report. Had I not shifted that mindset, I'd likely still be waiting for the notebook cell to finish running.

This is the story of how I got there, and why I think it matters.

## The research engineering tax

Here's the lived experience of a new computational science grad student:

**Weeks 1–2:** Get authorized on the cluster. Learn SLURM. Pass the Mila quiz.
**Weeks 3–4:** Write experiment scripts. Debug. Discover Python is 0-indexed the hard way. Fix an off-by-one error. Re-run the whole thing.
**Weeks 5–6:** Wire up logging. Parse results. Wonder why runs are duplicated.
**Weeks 7–8:** Change one parameter. Repeat everything. Ask yourself if you really need to re-run the whole pipeline for one hyperparameter.
**Month 3 onwards:** Finally think about the science.

Did you sign up to be a systems engineer? No. You signed up because neural networks seemed cool, or because you wanted to make sense of multiscale biological datasets, or because geometry felt like the right language for biology or neural networks, or both, or whatever was on your mind at the time. But here you are, debugging SLURM scripts at 2 am on a Tuesday.

This is the research engineering tax. Everybody pays it. Almost nobody talks about it in an actionable way. It's just implicitly accepted as a sort of hazing ritual for anyone who's ever built on HPC clusters. Stories from another time at Mila go all the way back to writing your job down on a mythical excel sheet before it was to be submitted. Still, I believe the people who figure out how to minimize it are the ones who ship, often to positive developments in their careers.

## Instruments of thought

There's a line I keep coming back to. Thinking about my time at Mila, the parallels are easy to draw. Mila's very own Yoshua Bengio, often seen as one of the machine learning pioneers by trajectory and continuous excellence, had [Theano](https://github.com/Theano/Theano) (since defunct), come out of his lab. Not to publish a systems paper, but I believe, to prototype theories faster. The tools we use are not afterthoughts. They are instruments of thought.

Better code doesn't just mean fewer bugs. It means more time spent on thoughts that matter, which means better questions, which means more impactful science. The causal chain is direct.

From Fabio's talk, from my time as a research engineering intern at Mila, and from building [manylatents](https://github.com/latent-reasoning-works/manylatents), I landed on four properties that separate research code that compounds from research code that rots:

**Reproducibility.** Seed everything — CPU, GPU, dataloader. In multithreaded systems, that's not enough: you need stateful RNGs passed explicitly. The cost of one non-reproducible result is astronomical. Not just wasted time — wasted trust. If you can't reproduce your own numbers from last Tuesday, why would a reviewer believe your paper? The actual state of this in ML is another matter entirely, and perhaps better saved for another day.

**Velocity.** Fabio's core insight. The goal isn't impeccable, unbreakable and absolutely elegant code (which you likely won't be pulling off your first iterations, or, ever - I know I didn't, probably aspire to on my good days and then quickly forget, and still have ways to go!). The goal is to shrink the inner loop: launch to result, in minutes. Cheap runs mean more hypotheses tested. It's okay to be scrappy — iterate, fail fast, learn fast. Better an ugly prototype today than a perfect pipeline next quarter.

**Modularity.** You cannot own the full stack. That's not a failure, it's a fact. You don't have the bandwidth. So: own one module deeply, trust the rest. Hydra configs become lego bricks — swap a model, swap a dataset, swap a trainer, without rewriting `main()`. Focus your cognitive budget where your novelty lives.

**Traceability.** Every result needs a lineage. A single command — `python -m src.main experiment=phate_sweep` — plus a saved config file and a git SHA should be enough for anyone to replay your run. Future you will need a time machine. Build one.

These aren't abstract principles. They're the design decisions behind [manylatents](https://github.com/latent-reasoning-works/manylatents): a framework for applying dimensionality reduction techniques across diverse datasets, built on PyTorch Lightning, Hydra, and WandB. Pick an algorithm, pick a dataset, pick what to measure, pick where to run it. Everything is a config. Nothing is hardcoded.

One command, one run:

```
algorithm=phate data=embryoid_body metrics=topology cluster=mila seeds=3
```

One command, hundreds of SLURM jobs:

```
algorithm=pca,tsne,umap,phate data=swissroll,blobs,torus,cells
k=5,10,15,20,25 seeds=1,2,3 cluster=mila
```

The day I saw the full pipeline fire for the first time — every component talking to each other, results uploading to WandB automatically — was the happiest day I'd had in a while. Not because the science was solved, but because the instrument was finally built. Now I could start playing it.

## But who's driving?

Here's the thing nobody told me when I started investing in this infrastructure: I was accidentally building something agent-legible.

A well-structured codebase with config-driven behavior, clean entry points, and documented contracts isn't just easier for humans. It's easier for AI agents to operate. And that realization changed everything.

Consider the skeptic's response to all the config machinery: "I don't care for this stuff. This codebase is ugly and unwieldy. Why would I learn it instead of just doing Jupyter notebook development?"

Fair. All they really want to do is say: "Run PCA, UMAP, PHATE, and t-SNE across multiple synthetic datasets and measure local intrinsic dimensionality across values of k=15, 50, 100. Three seeds each, logged to WandB, on the Mila SLURM GPU cluster."

So I made it possible to do exactly that. You type that sentence into Claude Code, and provided you have a battle tested `MEMORY.md` (or equivalent provider's stateful md file) for your local development machine here's what happens while you're still running your notebook cell:

**Step 1 — Read (~seconds).** The agent reads `CLAUDE.md`, config files, existing experiments, cluster docs. It understands what algorithms exist, what datasets are available, how the config system works, which cluster to target, and what went wrong last time.

**Step 2 — Design (~minutes).** It writes experiment config YAML. 4 algorithms × 5 datasets × 5 neighborhood sizes × 3 seeds = 300 configurations. It chooses metrics, resources, QOS. It knows to use `gpu_light` instead of `cpu_light` because it learned from past failures.

**Step 3 — Verify (~seconds).** Dry run. Checks config resolution. Catches stale files, wrong paths, missing dependencies. Fixes config shadowing before it becomes a silent failure.

**Step 4 — Launch (~seconds).** Submits to SLURM. 300 jobs fan out across GPUs. Results stream to WandB, or perhaps more importantly, to your local outputs folder, where agents can read it.

**Step 5 — Analyze (~minutes).** Pulls results from the WandB API. Deduplicates. Aggregates. Generates figures and tables. Identifies key findings.

The whole thing from sentence to WandB dashboard takes about five minutes. Setting this up manually would take a new grad student a week.

## Three things that have to be true

This is not magic. Three things have to be true for agentic workflows to actually work on a research codebase.

**The codebase must be legible.** A `CLAUDE.md` file explains entry points, core abstractions, data flow, the config system, how to extend it, naming conventions, and non-obvious breaks. The agent needs to understand the codebase the way a new collaborator would — except it reads the documentation every single time, which is more than most humans do.

**Guardrails must be in place.** Clean contracts ensure transparent results. Intermediate consolidation files give the agent something structured to parse. When the agent generates a results report, it writes down the source file and how it parsed it alongside every number. No hallucinated statistics. The values are traceable or they don't ship.

**The agent must have memory.** Working on a cluster is dynamic. Resource conditions change. Configs that worked yesterday break today. The agent adapts through a `MEMORY.md` file — a local developer settings file that accumulates lessons. "cpu_light gets congested on Mila — use gpu_light even for CPU-bound work." "Config shadowing caused silent failures — always verify resolution before sweeps." Mistakes happen once. The agent writes them down. Next session, it knows better.

## The step change

Here's what actually shifted in my workflows, roughly:

|  | Before | After |
|---|---|---|
| **Bottleneck** | Infrastructure | Ideas |
| **Driving question** | "I know what to run, but it takes weeks to set up" | "I can test anything in hours. I need better questions" |
| **Rate limiter** | Grad student hours | Scientific taste |
| **Cost of being wrong** | Weeks | Hours |
| **Iteration speed** | ~2 experiments per month | Many per day |

When infrastructure is no longer the bottleneck, the question changes from "can we run this?" to "should we run this?"

That second question is harder. And more interesting. And it's the one you actually signed up to answer.

## Words of caution

I want to be careful here, because this is where people get it wrong.

**Code was never the real bottleneck. Ideas were always the bottleneck.** The infrastructure just obscured that fact. Now that agents can handle the plumbing, you're confronted with the actual hard problem: which experiments are worth running?

It's easy to fall into the trap of "just one more prompt and see what happens" — which is exactly the undisciplined approach you were trying to escape. Being effective does not mean maximizing churn. Prototyping isn't banned; doing it without an end goal in mind is.

**Slop is technical debt.** An agent can produce a lot of code very fast. That doesn't make it good code. As Olexa Bilaniuk — of eponymous legend fame — once told me: "Consider the potential it may have to be too clever by half."

And think about your experiment design before you run it. I learned this the hard way: a few words missing in a prompt meant Claude designed an experiment that only computed three of the data points I needed, when a single forward pass would have given me all of them for free. A small omission in a spec can cost you hours. Think of the points in the plot beforehand. Be very explicit.

## The thesis

None of this is groundbreaking. Hydra, Lightning, WandB, Claude Code — these are all existing tools. I didn't invent anything. What I did was invest the time to wire them together into something I could trust, make it legible enough for an agent to operate, and then build on top of that trust to ask questions I couldn't have asked before.

An investment in your code today ensures you're building on solid foundations tomorrow. The research engineering tax doesn't go away. But it can be paid once, up front, in infrastructure that compounds — and that eventually lets something other than you handle the plumbing while you focus on the science.

The notebook really wasn't meant to be passed around. Build something that is — and maybe build it so an agent can drive it, too.

---

[manylatents](https://github.com/latent-reasoning-works/manylatents) is open source. If you work with dimensionality reduction, embeddings, or geometric methods and want to stop re-implementing the same pipeline, take a look. 70 people cloned it in the first week; you might find it useful too.

*Thanks to the devs, collaborators, friends that were in on the process.*
