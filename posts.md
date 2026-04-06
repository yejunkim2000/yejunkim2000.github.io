---
layout: default
title: Posts
permalink: /posts/
---

<section class="section" style="padding-top: 3rem;">

  <div class="section-header">
    <span class="section-label">all posts</span>
    <div class="section-line"></div>
  </div>

  <!-- Category filter buttons -->
  <div style="display: flex; flex-wrap: wrap; gap: 0.5rem; margin-bottom: 2rem;">
    <button onclick="filterPosts('all')" class="filter-btn active" data-filter="all"
      style="font-family: var(--mono); font-size: 11px; padding: 5px 12px; border: 1px solid var(--border-bright); border-radius: 4px; background: var(--bg-card); color: var(--text-secondary); cursor: pointer;">
      all
    </button>
    <button onclick="filterPosts('개발')" class="filter-btn" data-filter="개발"
      style="font-family: var(--mono); font-size: 11px; padding: 5px 12px; border: 1px solid; border-radius: 4px; background: rgba(79,142,247,0.08); border-color: var(--accent-blue); color: var(--accent-blue); cursor: pointer;">
      개발
    </button>
    <button onclick="filterPosts('CTF/Wargame')" class="filter-btn" data-filter="CTF/Wargame"
      style="font-family: var(--mono); font-size: 11px; padding: 5px 12px; border: 1px solid; border-radius: 4px; background: rgba(247,95,79,0.08); border-color: var(--accent-red); color: var(--accent-red); cursor: pointer;">
      CTF/Wargame
    </button>
    <button onclick="filterPosts('BugBounty')" class="filter-btn" data-filter="BugBounty"
      style="font-family: var(--mono); font-size: 11px; padding: 5px 12px; border: 1px solid; border-radius: 4px; background: rgba(247,167,79,0.08); border-color: var(--accent-amber); color: var(--accent-amber); cursor: pointer;">
      Bug Bounty
    </button>
    <button onclick="filterPosts('블로그/기술문서')" class="filter-btn" data-filter="블로그/기술문서"
      style="font-family: var(--mono); font-size: 11px; padding: 5px 12px; border: 1px solid; border-radius: 4px; background: rgba(79,247,200,0.08); border-color: var(--accent-teal); color: var(--accent-teal); cursor: pointer;">
      블로그/기술문서
    </button>
    <button onclick="filterPosts('논문/컨퍼런스')" class="filter-btn" data-filter="논문/컨퍼런스"
      style="font-family: var(--mono); font-size: 11px; padding: 5px 12px; border: 1px solid; border-radius: 4px; background: rgba(160,79,247,0.08); border-color: var(--accent-purple); color: var(--accent-purple); cursor: pointer;">
      논문/컨퍼런스
    </button>
    <button onclick="filterPosts('공모전/자격증')" class="filter-btn" data-filter="공모전/자격증"
      style="font-family: var(--mono); font-size: 11px; padding: 5px 12px; border: 1px solid; border-radius: 4px; background: rgba(106,247,79,0.08); border-color: var(--accent-green); color: var(--accent-green); cursor: pointer;">
      공모전/자격증
    </button>
  </div>

  <div class="posts-list" id="posts-container">
    {% for post in site.posts %}
      {% assign color = "teal" %}
      {% if post.category == "개발" %}{% assign color = "blue" %}
      {% elsif post.category == "CTF/Wargame" %}{% assign color = "red" %}
      {% elsif post.category == "BugBounty" %}{% assign color = "amber" %}
      {% elsif post.category == "블로그/기술문서" %}{% assign color = "teal" %}
      {% elsif post.category == "논문/컨퍼런스" %}{% assign color = "purple" %}
      {% elsif post.category == "공모전/자격증" %}{% assign color = "green" %}
      {% endif %}
      <a href="{{ post.url | relative_url }}" class="post-item" data-category="{{ post.category }}">
        <span class="post-date">{{ post.date | date: "%y.%m.%d" }}</span>
        <span class="post-title">{{ post.title }}</span>
        <span class="post-tag tag-{{ color }}">{{ post.category }}</span>
      </a>
    {% endfor %}

    {% if site.posts.size == 0 %}
    <div style="padding: 4rem 0; text-align:center; font-family: var(--mono); font-size: 13px; color: var(--text-muted);">
      // 아직 작성된 글이 없습니다.
    </div>
    {% endif %}
  </div>

</section>

<script>
function filterPosts(category) {
  const items = document.querySelectorAll('.post-item');
  items.forEach(item => {
    if (category === 'all' || item.dataset.category === category) {
      item.style.display = 'grid';
    } else {
      item.style.display = 'none';
    }
  });

  document.querySelectorAll('.filter-btn').forEach(btn => {
    btn.style.opacity = btn.dataset.filter === category ? '1' : '0.4';
  });
}
</script>
