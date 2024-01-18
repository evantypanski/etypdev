---
layout: barebones
title: A Rails Mistake
date: 2017-08-29
permalink: /posts/a-rails-mistake/
---

My website was my first public facing application that I created, thus it was bound to have some naive errors.

The primary, glaring error that I made was to start using Rails.  I figured that learning how to use Ruby on Rails would provide job security alongside more applicable knowledge to be used in the future, such as "learning how to learn" as I always put it.  But, for a small scale personal website, not only was using Rails overkill, but it hindered my ability to learn what Rail's functions actually are and made me actively search how to cut down on the functions instead of learning how to use them.

Probably the most glaring issue is my decision to learn how to use a database, at least slightly.  I made models for posts and projects within my portfolio to be handled and served from the database, with a title, content, and more.  Thus, I needed a way to edit the database.  So, I used active admin to make an admin site, but for security reasons I wanted the admin site to be hidden unless I needed to make a post.  Code like this is the result:

> routes.rb
{:.filename}
```ruby
  get '/admin', to: 'application#page_not_found'
  get '/admin/login', to: 'application#page_not_found'
  get '/admin/dashboard', to: 'application#page_not_found'
  get '/admin/admin_users', to: 'application#page_not_found'
  get '/admin/admin_users/:id', to: 'application#page_not_found'
  get '/admin/admin_users/:id/edit', to: 'application#page_not_found'
  get '/admin/admin_users/new', to: 'application#page_not_found'
  get '/admin/comments', to: 'application#page_not_found'
  get '/admin/posts', to: 'application#page_not_found'
  get '/admin/posts/:id', to: 'application#page_not_found'
  get '/admin/posts/:id/edit', to: 'application#page_not_found'
  get '/admin/posts/new', to: 'application#page_not_found'
  get '/admin/posts/delete', to: 'application#page_not_found'
  get '/admin/projects', to: 'application#page_not_found'
  get '/admin/projects/:id', to: 'application#page_not_found'
  get '/admin/projects/:id/edit', to: 'application#page_not_found'
  get '/admin/projects/new', to: 'application#page_not_found'
  get '/admin/projects/delete', to: 'application#page_not_found'
```

This is some code that I am wildly ashamed of.  I tried to remedy the situation with many searches for "Regex in rails routing" and similar queries as well as going through the docs (maybe not as diligently as I should have), but I could not find anything to remedy the situation, so I stuck with 17 lines completely dedicated to rerouting from admin pages so no one could see my admin page and exploit some strange security bug I had not found.  I banked on the fact that my repository was private.

So yes, I made a mistake by learning Rails with a personal website.  Currently, my website is made with Jekyll, where the port from rails took less than an hour and writing posts and portfolio entries became infinitely easier (it took longer than one hour to understand the setup on AWS rather than Heroku, but that's a different story).  Furthermore, I no longer have to deal with a database, which I tried to use for images within posts, but it did not turn out too well.

I no longer worry about writing posts, as it's just simple markdown files.  I no longer wrestle with Rails to change layouts and format the website, as Jekyll is less bloated and simply consists of writing text files on my computer without using ckeditor to write posts, where I can hardly read what I write because the page is so bloated and badly made.  Using Vim for writing posts is simply far more efficient and fun.

So, what have I learned?  Don't go with an overkill solution to learn a skill.  Not only did it hinder my website's initial performance, but it also hindered my ability to learn Rails properly.  That should have went without saying, but a single-man, watered-down version of Rails is just an annoyingly bloated version of Jekyll.  Which sucks.
