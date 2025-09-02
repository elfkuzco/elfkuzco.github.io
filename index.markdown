---
layout: default
show_author: false
show_date: false
show_site_description: true
---

Hello! I'm Uchechukwu Orji, and this is my technical blog where I share my experiences, learnings, and contributions in the world of software development..

## Recent Posts

{% for post in site.posts limit:5 %}
* [{{ post.title }}]({{ post.url | relative_url }}) - {{ post.date | date: "%B %d, %Y" }}
{% endfor %}

## Recent Projects

I regularly work on various software projects and contribute to open source. Here are some highlights:

- **Open Source Contributions**: Active contributor to various open source projects
- **Technical Tutorials**: Sharing knowledge and experiences through detailed guides
- **Project Development**: Building and documenting interesting software solutions

## Get in Touch

Feel free to reach out if you'd like to discuss technology, collaborate on projects, or just connect:

- GitHub: [@elfkuzco](https://github.com/elfkuzco)
- Email: [orjiuchechukwu52@yahoo.com](mailto:orjiuchechukwu52@yahoo.com)

Check out my [About](/about/) page to learn more about my background and interests.