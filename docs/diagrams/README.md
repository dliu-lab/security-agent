# HLD diagrams

The [Security Agent HLD](../high-level-design.md) embeds PNG images so readers do not need Mermaid rendering support. Each image has an editable Mermaid `.mmd` source beside it.

| Diagram | Source | Rendered image |
| --- | --- | --- |
| System architecture | [system-architecture.mmd](system-architecture.mmd) | [system-architecture.png](system-architecture.png) |
| Assessment flow | [assessment-flow.mmd](assessment-flow.mmd) | [assessment-flow.png](assessment-flow.png) |
| Assessment lifecycle | [assessment-lifecycle.mmd](assessment-lifecycle.mmd) | [assessment-lifecycle.png](assessment-lifecycle.png) |
| Marketplace and deployment distribution | [deployment-distribution.mmd](deployment-distribution.mmd) | [deployment-distribution.png](deployment-distribution.png) |

When changing a diagram, update its source, render a replacement PNG with an approved local Mermaid renderer, visually review labels and connectors, and commit both files together. Open the image at full size to inspect small labels.

These images were rendered locally with Mermaid 11.17.2, the default theme, a white background, and a pixel scale of 2. No external rendering service is required.
