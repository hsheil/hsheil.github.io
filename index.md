---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults
# title: hsheil blog
layout: home
---


# Welcome to My Blog

Hello! Welcome to my blog where I share my thoughts, experiences, and projects. Stay tuned for updates on a variety of topics, including:

- AI and Machine Learning
- Programming
- Running and Cycling
- Investing

## Recent Posts

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <small> - {{ post.date | date: "%B %d, %Y" }}</small>
    </li>
  {% endfor %}
</ul>

## About Me

I like loads of things and in particular coding, cycling and running.

Feel free to check out my [Strava profile](https://www.strava.com/athletes/1871302) for the latest tours and [GitHub profile](https://github.com/hsheil) to see my latest projects.

