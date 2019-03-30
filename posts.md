---
layout: article
title: Page - Article Header Overlay Background Image
article_header:
  type: overlay
  theme: dark
  background_color: '#203028'
  background_image:
    gradient: 'linear-gradient(135deg, rgba(34, 139, 87 , .4), rgba(139, 34, 139, .4))'
    src: /docs/assets/images/cover3.jpg
---
<div class="layout--home">
  {%- include paginator.html -%}
</div>
<script>
  {%- include scripts/home.js -%}
</script>


