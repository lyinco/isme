
<div class="grid cards" markdown>

-   :simple-zenn:{ .lg .middle } __三维重建__

    ---
    
    <>

-   :material-book-open:{ .lg .middle } __缺陷检测__

    ---
    <!-- - [博客系统完全指南](tutorials/blog-tutorial.md) -->

    - [zensical.toml 配置详解](tutorials/configuration.md)
    - [主题配置指南](tutorials/theme-customization.md)
    - [Markdown 扩展](tutorials/markdown-extensions.md)
    - [Markdown 高效写作](tutorials/zensical-markdown-tip.md)

-   :material-puzzle:{ .lg .middle } __PDF乱码字形字体识别__

    ---
    
    - [插件概览](blog/plugins/overview.md)
    - [博客](blog/plugins/blog.md)
    - [搜索](blog/plugins/search.md)
    - [标签](blog/plugins/tags.md)
    - [RSS](blog/plugins/rss.md)

-   :material-rocket:{ .lg .middle } __部署指南__

    ---
    
    - [GitHub Pages（推荐）](blog/deployment/github-pages.md)
    - [Netlify](blog/deployment/netlify.md)
    - [EdgeOne Pages](blog/deployment/edgeone-pages.md)
    - [GitLab Pages](blog/deployment/gitlab-pages.md)
    - [自托管](blog/deployment/self-hosted.md)

</div>


## 案例

- [Wcowin 的博客](https://wcowin.work)
- [Suffine Hub](https://sufine.top/)
- [Beyond欣's Notes](https://notes.beyondxin.top/index.html)
- [更多案例](showcase.md)
- 基于 Mkdocs、Zensical 的主题与站点汇总在：<a href="https://gist.github.com/Wcowin/d36a31b86aec04c203fec1562bb2391b" target="_blank">GitHub Gist</a>(支持评论提交)


---

<!--
  将所有页面级脚本和元数据统一放置在这里
-->
<!-- 访问统计区域 -->
<!-- <div style="text-align: center; margin: 2rem 0; font-size: 0.9rem;">
  本站访问量：<script async src="//finicounter.eu.org/finicounter.js"></script><span id="finicount_views" style="font-weight: bold; color: #518FC1;"></span>
</div> -->

<!-- Umami Analytics -->
<script defer src="https://cloud.umami.is/script.js" data-website-id="061b4dea-9b7b-4ffa-9071-74cde70f3dfb"></script>

<style>
/* body::before {
  --size: 35px;
  --line: color-mix(in hsl, canvasText, transparent 80%);
  content: '';
  height: 100vh;
  width: 100%;
  position: absolute;
  background: linear-gradient(
        90deg,
        var(--line) 1px,
        transparent 1px var(--size)
      )
      50% 50% / var(--size) var(--size),
    linear-gradient(var(--line) 1px, transparent 1px var(--size)) 50% 50% /
      var(--size) var(--size);
  -webkit-mask: linear-gradient(-20deg, transparent 50%, white);
          mask: linear-gradient(-20deg, transparent 50%, white);
  top: 80px;
  transform-style: flat;
  pointer-events: none;
  z-index: -1;
} */

@media (max-width: 768px) {
  body::before {
    display: none;
  }
}
</style>
