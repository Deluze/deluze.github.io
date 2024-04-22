---
layout: home
---

<h1>Bart Ramaekers</h1>
A place about the things I care to share. <br>
Most likely about reverse engineering, low-level stuff, optimizations and games.

<h2>About me</h2>
Born in 1999. <br>
Self-taught programmer, and full-time software engineer. <br>
In my free time I tend to reverse engineer games occasionaly. <br>
Dutch, residing in Düsseldorf (Germany) for work.

<h2>Links</h2>
Want to follow me, or get in contact? Don't be shy :)
<ul>
  <li><a href="https://github.com/deluze" target="_blank">GitHub (Deluze)</a></li>
  <li><a href="https://x.com/brtramaekers" target="_blank">Twitter (brtramaekers)</a></li>
  <li><a href="https://www.linkedin.com/in/bart-ramaekers/" target="_blank">LinkedIn</a></li>
</ul>

<h2>Projects</h2>
<ul>
  <li>
    <a href="https://github.com/deluze/qpang-essence-emulator">QPang Server Emulator</a>: Server emulator for a shooter game from 2013.
  </li>
  <li>More projects will be published soon... surely.</li>
</ul>

<h2>Posts</h2>
<ul class="post-list">
  {%- assign count = 0 -%}
  {%- for post in site.posts-%}
  <li>
    {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
      <h3>
        <span><a class="post-link" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a></span>
        <span class="post-meta">| {{ post.date | date: date_format }}</span>
      </h3>
  </li>
  {%- endfor -%}
</ul>
