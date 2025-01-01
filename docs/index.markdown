---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
---

<ul>
  {% for post in site.posts %}
    <li style="color: #0008; border: thin solid; border-left-width: 1rem; padding: 1rem; margin: 1.5rem 0rem;">
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
