---
layout: bughunt
icon: fas fa-bug
order: 4
---

{% include lang.html %}

{% assign posts = site.categories['Bughunt'] %}

<div id="post-list" class="flex-grow-1 px-xl-1">
  {% for post in posts %}
    <article class="card-wrapper card">
      <a href="{{ post.url | relative_url }}" class="post-preview row g-0 flex-md-row-reverse">
        {% assign card_body_col = '12' %}

        {% if post.image %}
          {% assign src = post.image.path | default: post.image %}

          {% if post.media_subpath %}
            {% unless src contains '://' %}
              {% assign src = post.media_subpath
                | append: '/'
                | append: src
                | replace: '///', '/'
                | replace: '//', '/'
              %}
            {% endunless %}
          {% endif %}

          {% assign alt = post.image.alt | xml_escape | default: 'Preview Image' %}

          {% comment %}
            A plain <img> here gets caught by refactor-content.html's image
            post-processing, which (since page.layout != 'home') wraps it in
            its own <a class="popup img-link">, nesting an <a> inside this
            card's outer <a class="post-preview">. Nested anchors are invalid
            HTML and broke the card's size/click hit-area. A background-image
            div sidesteps that transform entirely.
          {% endcomment %}
          <div class="col-md-5">
            <div
              class="preview-img"
              role="img"
              aria-label="{{ alt }}"
              style="background-image: url('{{ src }}'); background-size: cover; background-position: center;"
            ></div>
          </div>

          {% assign card_body_col = '7' %}
        {% endif %}

        <div class="col-md-{{ card_body_col }}">
          <div class="card-body d-flex flex-column">
            <h1 class="card-title my-2 mt-md-0">{{ post.title }}</h1>

            <div class="card-text content mt-0 mb-3">
              <p>{% include post-summary.html %}</p>
            </div>

            <div class="post-meta flex-grow-1 d-flex align-items-end">
              <div class="me-auto">
                <!-- posted date -->
                <i class="far fa-calendar fa-fw me-1"></i>
                {% include datetime.html date=post.date lang=lang %}
              </div>
            </div>
            <!-- .post-meta -->
          </div>
          <!-- .card-body -->
        </div>
      </a>
    </article>
  {% endfor %}
</div>
<!-- #post-list -->
