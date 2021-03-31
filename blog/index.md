## Sunfish's blog

I'm experimenting with this site as a way to blog.

<ul class="posts">
  {% for post in site.posts %}
    {% if post.tags contains 'draft' %}
      <!-- Draft. Don't publish in the index. -->
    {% else %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
    </li>
    {% endif %}
  {% endfor %}
</ul>
