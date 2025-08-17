---
layout: post
title:  "Adding Jekyll SEO Tag to My Blog"
date:   2025-08-17 13:40:33 +0800
categories: jekyll update
---

Today I added the **Jekyll SEO Tag** plugin to *Jekyll & Bytes*. It’s a small step to make the blog more search-friendly — especially since I’m rebooting my digital presence and want people to find my content more easily.

### Why Use It?
Jekyll SEO Tag helps generate meta tags for your site — like title, description, author, and even Open Graph tags for social media. It’s a simple way to improve how your blog appears in search results and when shared online.

### How I Installed It
Here’s what I did:

1. **Added the plugin to my site `Gemfile`**:
   
   ```html
   gem 'jekyll-seo-tag'
   ```

2. **Added the plugin to my site `_config.yml`**:
   
   ```yaml
   plugins:
     - jekyll-seo-tag
   ```

3. **Included the tag in my head layout** (usually in `_includes/head.html`):
   
   ```html
   {% raw %}{% seo %}{% endraw %}
   ```

That’s it. No fuss. Now my blog has better metadata for search engines and social previews.

### Next Up
I’ll be tweaking the layout with Material Design and maybe adding a favicon. Still warming up, but it feels good to be writing again.

