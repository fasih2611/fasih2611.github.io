---
title: "Fasih Tariq"
permalink: /
---

{% include diffusion-banner.html %}

I work on computer vision, mostly the parts that have to survive contact with
real data: detection and recognition, matching objects and people across
cameras, and the systems engineering that keeps a video pipeline running at
scale. I am also finishing a master's thesis on generative models.

## Interests

- **Detection, recognition, and re-identification.** Finding things, then
  recognising them again later, often without any unique identifier to match on.
- **Video pipelines at scale.** Throughput, GPU utilisation, memory, and the
  unglamorous engineering that decides whether a model is usable in production.
- **Representation learning and vector search.** Embeddings treated as the
  index rather than as the output.
- **Generative models.** Flow matching and mean flows for few-step generation,
  and why one-step samplers lose sample diversity even when their distributional
  metrics look excellent.

## Currently

Building and operating production vision systems, and writing a thesis on
trajectory overlap and one-step failure modes in mean-flow models. Notes and
experiments live in
[Thesis-Notes](https://github.com/fasih2611/Thesis-Notes) and
[mean-flow](https://github.com/fasih2611/mean-flow).

## Elsewhere

- [GitHub](https://github.com/{{ site.github_username }})
- [Email](mailto:{{ site.email }})
{%- if site.cv_url %}
- [CV]({{ site.cv_url | relative_url }})
{%- endif %}
