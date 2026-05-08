---
title:  "Untangling the Fee Calculator: A Refactor Story"
date: 2026-05-05
mathjax: false
layout: post
categories: media
tags: [python, refactoring, pipeline-pattern, tect-debt, fintech, backend]
---

I've been working with a UAE-based fintech platform for a while now, and if
there's one thing that quietly becomes a monster over time, it's fee calculation
logic. It started as a simple function, a few conditions, maybe a discount here
and there – and before you know it you've got a spaghetti function with deeply
mingled logic, impossible to test or reuse without running the whole thing end
to end.

That was us. That was our `calculate_payment_methods_fee`.

## The Problem

The function *worked*. It calculated markups, discounts, service fees, taxes,
priorities – all correctly. But it was completely opaque and difficult to dissect.

The breaking point was when product started asking for variants. Payment links,
invoices, collect campaigns, in-person payments, foreign currency flows – all
subtly different, all sharing 80% of the same logic. Every new variant meant
another fork of the monolith. It wasn't sustainable.

## The Idea: Make It a Pipeline

The core insight was simple. Every step in that function had the same shape:
take a `payload` and a `context`, do something to the payload, return it. That's
just a pipeline. Each step is a transformer.

The entire thing rests on two small classes. That's it.

```python
class PipelineStep:
    def __init__(self, rounding=None):
        self.round = rounding() if rounding else RoundHalfUp()

    def __call__(self, payload: dict, context: dict) -> dict:
        raise NotImplementedError


class FeatureFeePipeline:
    def __init__(self, context: dict):
        self.context = context
        self._steps: list[PipelineStep] = []

    def pipe(self, *steps: PipelineStep) -> "FeatureFeePipeline":
        self._steps.extend(steps)
        return self

    def run(self, payload: dict) -> dict:
        for step in self._steps:
            payload = step(payload, self.context)
        return payload
```

So we pulled every piece of tangled logic out into its own callable class
inheriting from a `PipelineStep` base, and built a thin `FeatureFeePipeline`
runner that threads the payload through them in order:

```python
FeatureFeePipeline(context)
    .pipe(
        GetLinkPayload(link),
        ApplyMarkup(allow_mu, mu_config),
        ApplyDiscount(allow_discount, discount_config),
        ApplyServiceFee(allow_sf, sf_config),
        ApplyTax(allow_tax, tax_config),
        DecorateTotal(),
        MakeBackwardCompatible(),
        ...
    )
    .run({})
```

That's the full fee calculation for a payment link. Readable top to bottom, no
archaeology required.

_You could implement this with plain functions – each step is just
`(payload, context) -> payload`, nothing inherently object-oriented about it.
Classes just made configuration cleaner and the pipeline more readable._

_You'll also notice `MakeBackwardCompatible()` in there. The new pipeline
produces a richer payload shape – but we had existing consumers in production
already reading the old field names. Rather than a hard cutover, this step
aliases the new fields back to what the old code expected. Incremental adoption.
When we're ready to retire it, it's one line deleted from the declaration. Backward
compatibility as just another step in the list._

## The Part That Actually Paid Off

Each pipeline step is a class. Composing a different pipeline means listing different steps.

Take for example the charge pipeline. It stops before the cleanup phase to keep intermediate fields. Or the collect pipeline swaps `GetLinkPayload` for `GetCollectPayload` – same interface, different source, everything downstream unchanged. New feature request where we want to always show the highest markup? Just one extra step: `OverwriteHighestMarkup(allow_mu)`.

Adding, skipping, or swapping a step is a one-line change to the pipeline declaration. No conditionals buried in a monolith, no duplicated functions.

## Was It Worth It?

Business logic didn't change. From the outside, nothing is different.

Business logic stayed the same, but new pipeline variants after the refactor took under an hour with tests. Before, that meant a day of copy-paste-and-tweak and production anxiety.

The pipeline pattern wasn't novel – it just fit the problem better.

Sometimes the best refactor is the one that makes the next ten changes boring.

*Written while the deploy pipeline was running.*