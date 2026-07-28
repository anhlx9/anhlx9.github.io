---
title: "All Posts"
layout: archive
permalink: /all-posts/
author_profile: true
---

<p class="allposts-count">{{ site.posts | size }} bài viết</p>

<div id="allposts-list">
{% for post in site.posts %}
  {% include archive-single.html %}
{% endfor %}
</div>

<nav class="pagination" id="allposts-pagination" aria-label="Pagination"></nav>

<script>
(function () {
  // Số bài mỗi trang — đổi con số này nếu muốn.
  var PER_PAGE = 3;

  var list = document.getElementById('allposts-list');
  var nav  = document.getElementById('allposts-pagination');
  if (!list || !nav) return;

  var items = [].slice.call(list.querySelectorAll('.list__item'));
  var pages = Math.ceil(items.length / PER_PAGE);
  var current = 1;

  function render(scroll) {
    items.forEach(function (el, i) {
      var page = Math.floor(i / PER_PAGE) + 1;
      el.style.display = (page === current) ? '' : 'none';
    });
    buildNav();
    if (scroll) list.scrollIntoView({ behavior: 'smooth', block: 'start' });
  }

  function buildNav() {
    if (pages <= 1) { nav.innerHTML = ''; return; }
    var html = '<ul>';
    html += '<li><a href="#" data-page="' + (current - 1) + '" class="' +
            (current === 1 ? 'disabled' : '') + '">Previous</a></li>';
    for (var p = 1; p <= pages; p++) {
      html += '<li><a href="#" data-page="' + p + '" class="' +
              (p === current ? 'current' : '') + '">' + p + '</a></li>';
    }
    html += '<li><a href="#" data-page="' + (current + 1) + '" class="' +
            (current === pages ? 'disabled' : '') + '">Next</a></li>';
    html += '</ul>';
    nav.innerHTML = html;
  }

  nav.addEventListener('click', function (e) {
    var a = e.target.closest('a');
    if (!a) return;
    e.preventDefault();
    if (a.classList.contains('disabled') || a.classList.contains('current')) return;
    var p = parseInt(a.getAttribute('data-page'), 10);
    if (p >= 1 && p <= pages) { current = p; render(true); }
  });

  render(false);
})();
</script>
