---
layout: post
title: 'OpenAI alert enrichment'
date:   2023-03-21 
logo: 'fa fa-rocket'
---

Hi welcome to my blog.

After some LinkedIn post, I've decided to create a blog for myself.

I created the blog using `Jekyll`. 
[Jekyll](http://jekyllrb.com/) is a system to create blog-aware, [static](https://en.wikipedia.org/wiki/Static_web_page) web pages. Jekyll integrates nicely with [GitHub Pages](https://pages.github.com/). And Jekyll supports markdown.

Enjoy reading the posts!

jekyll code block markdown

{% highlight powershell %}
$apiKey = "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"
$url = "https://api.openai.com/v1/engines/davinci-codex/completions"

$body = @{
    prompt = "Write a PowerShell script to"
    max_tokens = 50
} | ConvertTo-Json

$headers = @{
    "Content-Type" = "application/json"
    "Authorization" = "Bearer $apiKey"
}

$response = Invoke-RestMethod -Uri $url -Method POST -Headers $headers -Body $body
$response.choices.text
{% endhighlight %}
