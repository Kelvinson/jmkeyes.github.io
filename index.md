---
layout: default
image:  openroad.jpg
---
<ul class="listing">
    {% for post in site.posts %}
        <li>
            <span class="post-meta">{{ post.date | date: "%B %-d, %Y" }}</span>
            <a class="post-link" href="{{ post.url | prepend: site.baseurl }}">{{ post.title }}</a>
        </li>
    {% endfor %}
</ul>
