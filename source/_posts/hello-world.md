---
title: Hello World
date: 2019-04-20 16:30:00
tags: [hexo, netlify]
---

# Hello World

A blog can be a good personal knowledge management system. Countless bookmarks, notes and files that I stored over the years are not easily searchable and shareable.

I will be using this blog to document my projects, research and share some notes that may also be helpful for other developers. Writing and publishing what I learn will also help me to improve my communication and writing skills.

On this post I will cover how to setup a static website for free.

## Static site generators

An easy way to create a blog is to use a CMS, such as Wordpress but static site generators are preferable for this kind of content. Since the pages are not retrieved dynamically from a database, it provides better performance, higher security and cheaper/easier scaling. Some popular static generators can be found [here](https://www.staticgen.com).

Most static site generators are quite similar and even provide quick ways to migrate between them. I was inclined towards [Hexo](https://hexo.io) and [Hugo](https://gohugo.io) because they are fast and full-featured but ultimately went for Hexo as I am more familiar with the Node.js ecosystem and just wanted to get up and running asap.

### Hexo

To create this blog and run a development server I executed the following commands

```bash
npm install hexo-cli -g
hexo init briefbytes
cd briefbytes
npm install
hexo server
```

Posts are written in markdown. Additional [themes](https://hexo.io/themes/index.html) can be installed by cloning a theme from a repository to the **themes** folder:

```bash
git clone https://github.com/probberechts/hexo-theme-cactus.git themes/cactus
rm -rf themes/cactus/.git
```

A recommended alternative is to fork the theme and add it as a git submodule to the project, so that modifications to the theme can be made while allowing easy synchronization with the original repository.

After installing themes, they can be switched by editing the **theme** option in the file **_config.yml**. The [Hexo docs](https://hexo.io/docs/) are very good and provide additional information.

Some useful commands:

```bash
# Add post
hexo new "post name"
# Add page
hexo new page about
# Generate static files
hexo generate
```

Static web pages can be hosted for free in many places, within certain limits. I chose [GitHub](https://github.com) for version control and [Netlify](https://www.netlify.com) for hosting. The whole process is very simple, create a github repository, push the local changes and setup Netlify through the web interface to handle continuous integration and deployment.

## Conclusion

Static site generators are gaining popularity because of the advantages they provide, specially for developers that are used to a Git workflow. An example of a big website using Hexo is the [Vue.js docs](https://github.com/vuejs/vuejs.org). For a modern web development architecture based on client-side JavaScript, reusable APIs and prebuilt Markup, read about the [JAMstack](https://jamstack.org).
