---
layout: post
title: Understanding Turbolinks in Rails
date:   2015-02-13 12:54:32 -0400
categories: rails
---
So I have been having a fun time trying to figure out a way to get different Javascript features to work the way they are supposed to. Apparently, Turbolinks, which is enabled by default by Rails, is making your pages load faster. This is basically done by only loading the part of the page that needs to be updated. This is sending a asynchronous (AJAX) request via javascript.

> The request is fetching data for whatever is in the `<body>` and `<title>` DOM tags.

Turbolinks ends up loading the complete website from the server and replaces the `<body>` and `<title>` tags in the currently loaded DOM. You can disable certain links, so when clicked on them, the whole page reload will hit the server.

You can **DISABLE** Turbolinks competely by removing the turbolinks gem from your Gemfile and by removing the require in: `app/assets/javascript/application.js`

Also, don't forget to remove the data-turbolinks reference in your `app/views/layouts/application.html.erb`
