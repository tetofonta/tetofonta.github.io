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

      {% capture cv_personal_data%} {% include {{site.cv.personal_data}} %} {%endcapture%}
      {{cv_personal_data | markdownify}}

    </div>
</div>

## Professional Experience
{% for experience in site.cv.professional_experience%}
  {% include {{experience}} %}
{%endfor%}

## Education
{% for edu in site.cv.education%}
  {% include {{edu}} %}
{%endfor%}

## Skills
{% include {{site.cv.skills}} %}

## Other Activities and positions
{% for oth in site.cv.other%}
  {% include {{oth}} %}
{%endfor%}