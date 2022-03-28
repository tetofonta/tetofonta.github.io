---
layout: article
title: CV - Stefano Fontana
aside:
  toc: true
---

## Informazioni personali

<div style="display: flex; flex-direction: row; justify-content: flex-start;">

    <div style="background-image: url('/assets/images/profile.jpeg'); display: block; width: 40%; padding-bottom: 40%; background-position: center;background-size: contain; background-repeat: no-repeat"></div>

    <div style="padding-left: 2rem">

{% capture personal_data %}{% include cv/personal_data.md %}{% endcapture %}
{{ personal_data | markdownify }}

    </div>
</div>

## Professional Experience

{% include /cv/professional_experience/2020-now.md %}
{% include /cv/professional_experience/2019.md %}