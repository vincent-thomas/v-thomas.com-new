---
title: Do you really need next.js?
description: Web is soo overengineered
template: "pages/log.html"
date: 2025-08-31
draft: true
---

Did you know next.js has almost 600,000 lines of code (mostly ts and rust)? This is not even considering react (!!!!!). That is a ridiculous amount of code for a web framework.
It is so engineered that i wouldn't even consider it a web framework anymore. It's a high level build tool for building web servers and frontends.

> **Opinion:**
>
> Next.js is the wrong default choice for a SaaS. Just use something simpler.

And with that amount of code, it's not even complete! Within 10 years, next.js and react has **completely** changed (rewrite level changed) APIs 3 times!
That is insane. How can big apps be built ontop of this tech and it's devs still be efficient?

## Do you really need a 200kb frontend _"library"_?

Do you really need a complicated UI layer for your little SaaS with 3 users? Like 200kb complicated... You probably aren't even doing anything complicated.
UI shouldn't be complicated to develop, Its only responsibility should be to present the important data your service provide in a UX friendly way. UI development should strive for minimalism, not library functionality.


For example, this website doesn't need any advanced features, but it shares some html so i choose a [static site generator](https://www.getzola.org) (which really is overkill for me) without scss (don't need it).
Interactivity can then later get added with [htmx](https://htmx.org), [the hotwired family](https://hotwired.dev) or [alpinejs](https://alpinejs.dev).

Startups that only deal with CRUD data, and doesn't have complex UI (like alot of graphs or almost game like Interactivity)
