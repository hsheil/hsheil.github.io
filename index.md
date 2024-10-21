---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: home
---


```python
import random; 
random.choice(['Running', 'Cycling', 'Coding', 'ML/AI', 'Investing'])
```

_Writing is good for the soul_

## Recent Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <small> - {{ post.date | date: "%B %d, %Y" }}</small>
    </li>
  {% endfor %}
</ul>

Check out my [Strava profile](https://www.strava.com/athletes/1871302) for the latest tours and [GitHub profile](https://github.com/hsheil) to see my latest projects.

