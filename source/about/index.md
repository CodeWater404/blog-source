---
title: 关于
date: 2024-02-16 22:07:08
type: "about"
comments: true
---

<style>
.about-wrap { max-width: 720px; margin: 0 auto; padding: 0 0 3rem; }

/* 头部 */
.about-hero { display: flex; align-items: center; gap: 2rem; margin-bottom: 2.5rem; padding-bottom: 2rem; border-bottom: 1px solid var(--card-border, #e4e6ea); }
.about-hero-text h1 { font-size: 1.8rem; font-weight: 700; margin: 0 0 0.4rem; color: var(--font-color, #333); }
.about-hero-text p { font-size: 1rem; color: var(--secondFont, #666); margin: 0; line-height: 1.7; }

/* 分区通用 */
.about-section { margin-bottom: 2.2rem; }
.about-section h2 { font-size: 1rem; font-weight: 700; color: var(--secondFont, #888); text-transform: uppercase; letter-spacing: .08em; margin: 0 0 1rem; }

/* 简介段落 */
.about-bio { font-size: 1rem; color: var(--font-color, #333); line-height: 1.85; margin: 0; }
.about-bio + .about-bio { margin-top: .8rem; }

/* 技术栈标签 */
.tag-group { display: flex; flex-wrap: wrap; gap: .5rem; }
.tag { display: inline-flex; align-items: center; gap: .35rem; background: var(--card-bg, #f5f6fa); border: 1px solid var(--card-border, #e4e6ea); border-radius: 6px; padding: .28rem .7rem; font-size: .82rem; color: var(--font-color, #444); font-family: ui-monospace, monospace; transition: border-color .2s, background .2s; }
.tag:hover { border-color: var(--btn-bg, #49b1f5); background: var(--btn-bg-hover, #e8f4fd); }
.tag .dot { width: 7px; height: 7px; border-radius: 50%; flex-shrink: 0; }
.dot-blue { background: #3b82f6; }
.dot-green { background: #22c55e; }
.dot-orange { background: #f97316; }
.dot-purple { background: #a855f7; }
.dot-pink { background: #ec4899; }
.dot-gray { background: #94a3b8; }

/* 博客写什么 */
.topic-list { list-style: none; padding: 0; margin: 0; display: flex; flex-direction: column; gap: .6rem; }
.topic-list li { display: flex; align-items: flex-start; gap: .8rem; font-size: .93rem; color: var(--font-color, #444); line-height: 1.6; }
.topic-list li::before { content: ''; width: 4px; height: 4px; border-radius: 50%; background: var(--btn-bg, #49b1f5); flex-shrink: 0; margin-top: .55rem; }

/* 当前在做 */
.now-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(200px, 1fr)); gap: .7rem; }
.now-card { background: var(--card-bg, #fff); border: 1px solid var(--card-border, #e4e6ea); border-radius: 8px; padding: .8rem 1rem; }
.now-card .label { font-size: .72rem; color: var(--secondFont, #999); text-transform: uppercase; letter-spacing: .06em; margin-bottom: .3rem; }
.now-card .value { font-size: .9rem; color: var(--font-color, #333); font-weight: 500; }

/* 联系 */
.contact-row { display: flex; gap: 1rem; flex-wrap: wrap; }
.contact-link { display: inline-flex; align-items: center; gap: .5rem; padding: .4rem .9rem; border-radius: 6px; border: 1px solid var(--card-border, #e4e6ea); font-size: .88rem; color: var(--font-color, #444); text-decoration: none; transition: border-color .2s, color .2s; }
.contact-link:hover { border-color: var(--btn-bg, #49b1f5); color: var(--btn-bg, #49b1f5); }
.contact-link i { font-size: .9rem; }
</style>

<div class="about-wrap">

<div class="about-hero">
  <div class="about-hero-text">
    <h1>嗨，我是阿水 👋</h1>
    <p>后端开发，Go 方向。平时在命令行里花的时间比在 IDE 里多。</p>
  </div>
</div>

<div class="about-section">
  <h2>关于我</h2>
  <p class="about-bio">写代码的人。喜欢把工具链打磨到顺手——不是为了折腾而折腾，而是减少每天的无效摩擦，让注意力留给真正要解决的问题。</p>
  <p class="about-bio">终端是我最常待的地方。用 kitty 开着几个分屏，ripgrep 搜代码，lazygit 管版本，偶尔用 Neovim 改点东西。这套工具链花了不少时间打磨，觉得值。</p>
  <p class="about-bio">GitHub 用来放代码，这里用来写字。</p>
</div>

<div class="about-section">
  <h2>技术栈</h2>
  <div class="tag-group">
    <span class="tag"><span class="dot dot-blue"></span>Go</span>
    <span class="tag"><span class="dot dot-blue"></span>gRPC</span>
    <span class="tag"><span class="dot dot-blue"></span>MySQL</span>
    <span class="tag"><span class="dot dot-blue"></span>Redis</span>
    <span class="tag"><span class="dot dot-green"></span>Linux</span>
    <span class="tag"><span class="dot dot-green"></span>Docker</span>
    <span class="tag"><span class="dot dot-green"></span>Git</span>
    <span class="tag"><span class="dot dot-orange"></span>kitty</span>
    <span class="tag"><span class="dot dot-orange"></span>Neovim</span>
    <span class="tag"><span class="dot dot-orange"></span>fzf</span>
    <span class="tag"><span class="dot dot-orange"></span>ripgrep</span>
    <span class="tag"><span class="dot dot-purple"></span>Hexo</span>
    <span class="tag"><span class="dot dot-pink"></span>macOS</span>
  </div>
</div>

<div class="about-section">
  <h2>这里写什么</h2>
  <ul class="topic-list">
    <li>Go 后端开发：语言细节、工程实践、踩过的坑</li>
    <li>终端工具链：配置调优、工作流整合、新工具上手记录</li>
    <li>开发效率：让重复的事情变快、让复杂的事情变清晰</li>
    <li>偶尔一些技术以外的碎片</li>
  </ul>
</div>

<div class="about-section">
  <h2>最近在做</h2>
  <div class="now-grid">
    <div class="now-card">
      <div class="label">📖 在读</div>
      <div class="value">Go 语言设计与实现</div>
    </div>
    <div class="now-card">
      <div class="label">🛠 在配</div>
      <div class="value">Neovim + LazyVim</div>
    </div>
    <div class="now-card">
      <div class="label">✍️ 在写</div>
      <div class="value">Mac 终端工具系列</div>
    </div>
    <div class="now-card">
      <div class="label">🎵 在听</div>
      <div class="value">民谣 / Lo-fi</div>
    </div>
  </div>
</div>

<div class="about-section">
  <h2>联系我</h2>
  <div class="contact-row">
    <a class="contact-link" href="https://github.com/CodeWater404" target="_blank"><i class="fab fa-github"></i> GitHub</a>
    <a class="contact-link" href="https://t.me/dylancody" target="_blank"><i class="fab fa-telegram"></i> Telegram</a>
    <a class="contact-link" href="mailto:dylanrieder@163.com"><i class="fas fa-envelope"></i> 邮件</a>
  </div>
</div>

</div>
