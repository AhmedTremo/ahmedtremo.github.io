---
title: Starting A Blog Hosted On Github Pages
date: 2023-06-01 20:14 +0300
categories: [Blogging, Tutorial]
tags: [github-pages, blog, personal blog, jekyll]
author: tremo
---

## My first ever blog post

I've been thinking about starting a blog for a while now and I was procrastinating quite a bit :D. But, I finally did it and here I am writing my first ever blog post. Suprisingly enough, it will be about my experience setting up my blog and how you can do it too :D.

![Desktop View](/assets/img/posts/2023-06-01-starting-a-blog-on-gh-pages/Intro_meme.jpg){: width="500"}
_Scenario of a developer starting a blog_

### [YouTube Video Walkthrough](https://youtu.be/m1RYsmOMPLs)

## Why Github Pages?

I'm a progammer. I've always wanted to have a personal website to showcase my projects and share my thoughts. I've looked into various blogging platforms like [Wordpress](https://wordpress.com/), [Medium](https://medium.com/), [Substack](https://substack.com), and [Ghost](https://ghost.org/). But, I chose Github Pages with Jekyll because I wanted to have:
1. Full control over my blog and I wanted to customize it to my taste. 
2. A blog that is free and doesn't require me to pay for hosting. 
3. A blog that is simple, fast and easy to maintain doesn't require me to spend hours to configure it.

Did I convince you? OK, now let's break down the steps to setup your blogging site.

## Step 1: Decide Your Theme

This step is to quickly browse through the various Jekyll themes available on various websites and pick one that fits your taste

Few sites where you can grab these templates:

* <https://jekyllthemes.io/>
* <http://jekyllthemes.org/>
* <https://jekyll-themes.com/>
* <https://jamstackthemes.dev/ssg/jekyll/>

I personally picked the [Chirpy theme](https://github.com/cotes2020/chirpy-starter/) since it fits my expectations and it has a Dark theme :D.


## Step 2: Activate Github Pages

Once you pick the Jekyll theme, it's time to host it on Github Pages. The theme you picked usually comes with a set of instructions to configure and the instruction varies between different themes.

For Chirpy theme, the instructions are as follows:
1. Use the [template](https://github.com/cotes2020/chirpy-starter/generate) to create your own repository.
    - Make sure to name it as `<your-gh-username>.github.io`
    - After doing this step Github actions will build and deploy your blog automatically to `<your-gh-username>.github.io`
    - But you don't want just a template, you want to customized it to yourself. So, let's move on to the next step.
2. Clone the repository you just created.
3. Install Ruby and Jekyll on your machine through the [official guide](https://jekyllrb.com/docs/installation/).
4. Run `bundle install` to install the required gems.
5. Update the variables of `_config.yml` as needed. Some of them are typical options.
    - `url` is the address of your website
    - `avatar` is the profile picture in the sidebar
    - `timezone` is used to display the date and time of your posts
    - `lang` is the language of the site
6. Run `bundle exec jekyll s` to start the local server.

![Template Blog](/assets/img/posts/2023-06-01-starting-a-blog-on-gh-pages/template-blog.PNG){: width="700"}
_The empty template you should see_

If you face any issues, you can refer to the [Chripy theme's Getting started guide](https://chirpy.cotes.page/posts/getting-started/).

## Step 3: Setup Your Custom Root Domain

You need to visit one of the domain name registrar to buy a custom domain. There are multiple registrars to choose from: 

* [GoDaddy](https://www.godaddy.com/)
* [Google Domains](https://domains.google)
* [Name Cheap](https://www.namecheap.com/)
* ... and many more

*I personally chose GoDaddy since I had to pay ~10$ for 2-years plan*

### Configure Your Domain

After you purchase your domain, go into your domain management portal, click on manage DNS and add `A` type DNS records for github pages.

| Type | Data |
|------|------|
| A | 185.199.108.153 |
| A | 185.199.109.153 |
| A | 185.199.110.153 |
| A | 185.199.111.153 |
| CNAME | gh-username.github.io |

*(These `A` type DNS records map your domain name to the Github's IP address)*


So far, my DNS record looks like this:

![Desktop View](/assets/img/posts/2023-06-01-starting-a-blog-on-gh-pages/DNS-settings.PNG){: width="700"}
_My GoDaddy's DNS records_

### Configure Github Pages

Now that you have your domain's DNS setup, Let's head back to Github and configure your Github Pages to use your custom domain.

1. Go to your repository's settings page.
2. Scroll down to the **Pages** section.
3. Under **Custom domain** enter your domain name and click **Save**.

![Custom Domain](/assets/img/posts/2023-06-01-starting-a-blog-on-gh-pages/github-pages-custom-domain.PNG){: width="700"}
_My Github Pages Custom Domain page_

***Best Practice :*** Click on **Enforce HTTPS** to serve your blog via secure SSL connection. Your site will be configured with a free SSL certificate from [Let's Encrypt](https://letsencrypt.org/).

## Bonus Tip: Test Your Site

If you see a `404 error` or `Domain Not Found error` your DNS record might not be updated. Every time you update a DNS record, it takes few mins to several hours to propagate the WWW. So, give it sometime. To see if your domain is reachable, you could dig the DNS:

```bash
$ dig YOUR-DOMAIN.COM +noall +answer
```

Here is a reference from digging my website:

```bash
$ dig ahmedtremo.com +noall +answer
ahmedtremo.com.    0    IN    A    185.199.111.153
ahmedtremo.com.    0    IN    A    185.199.109.153
ahmedtremo.com.    0    IN    A    185.199.110.153
ahmedtremo.com.    0    IN    A    185.199.108.153
```

Hope you found this article useful. If you have any questions, you can check my [blog's repo](https://github.com/AhmedTremo/ahmedtremo.github.io) on Github or feel free to reach out to me on [Twitter](https://twitter.com/ahmedtremo) or [LinkedIn](https://www.linkedin.com/in/ahmedtremo/).


