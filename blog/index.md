## A blog from the CraneStation

I'm experimenting with this site as a way to blog about the
[Crane Compiler Organization](https://github.com/CraneStation/), the
[Cranelift code generator](https://github.com/CraneStation/cranelift),
and sometimes just about compilers in general.

<ul class="posts">
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      {{ post.excerpt }}
    </li>
  {% endfor %}
</ul>
