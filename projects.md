---
layout: page
title: Projects
link: true
permalink: /projects/
---

### List of Projects

{% for repository in site.github.public_repositories %}
  * [{{ repository.name }}]({{ repository.html_url }})
{% endfor %}
