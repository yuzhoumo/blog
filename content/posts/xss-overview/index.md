---
title: Stored vs Reflected XSS
description: Brief overview of cross site scripting
date: 2021-04-29
tags: [security, code, programming]
---

In cross site scripting (XSS) vulnerabilities, the user of a website executes malicious JavaScript code injected by an attacker. This article explains two major categories of XSS vulnerabilities: "Reflected XSS" and "Stored XSS".

## How XSS Works

HTML uses tags to denote denote page layout. For instance, `<p></p>` tags indicate plain text. XSS takes advantage of inline scripting, where any HTML content surrounded by `<script></script>` tags are interpreted as Javascript code and executed. This means that if attackers are able to insert raw HTML into a page that a user is visiting, they can use that power to insert script tags and get a user to run their malicious code.

## Reflected XSS

Many sites use URL query parameters to pass information from the user to a remote server. Consider the following URL:

`https://www.google.com/search?q=helloworld`

This link takes you to a Google search for `helloworld`. Everything after the `?` denotes parameters that we want to send to Google, namely `q=helloworld`. This conveys that our query parameter `q` will be sent to Google with the value `helloworld`.

In reflected XSS, an attacker might take advantage of URL query paramameters to pass in malicious data to the webserver. This link could contain malicious Javascript code that might be executed by a vulnerable site. If the attacker sends this link to an unsuspecting user and gets that person to click on it, then the attacker's malicious code will execute when the user visits the URL.

While Google search shouldn't (hopefully) be vulnerable to this sort of attack, consider the following example of badly developed website. Say a website has a vulnerable search feature:

`https://vulnerable-site.com/search?q=helloworld`

Unlike `google.com`, this vulnerable website returns a page like this:

```html
<h1>Search</h1>
<h3>Search results for helloworld</h3>
<table>
  <tr>
    <td>No results!</td>
  </tr>
</table>
```

The server simply takes whatever value `q` is set to and inserts it directly into the page's HTML after "Search results for".

Notice how the vulnerable site just injects `helloworld` directly into the page. Now imagine if the attacker had created a link that looked like this:

`https://vulnerable-site.com/search?q=<script>EVIL_CODE</script>`

If the attacker gets a user to click on this link, the result page for that user would look something like this:

```html
<h1>Search</h1>
<h3>Search results for <script>EVIL_CODE</script></h3>
<table>
  <tr>
    <td>No results!</td>
  </tr>
</table>
```

In this example `EVIL_CODE` is just a placeholder, since this vulnerability allows the attacker to inject any Javascript code into the website.

## Stored XSS

Stored XSS is similar to reflected XSS with the exception that the malicious Javascript is stored on the server permanently. Say in this new example, we have a vulnerable social media site that creates posts using the following endpoint:

`https://vulnerable-site.com/post?content=helloworld`

Let's say that this endpoint takes whatever value is stored in `content` and, makes a post on the site with that content. Instead of sending a link to the victim, say the the attacker logs in to his own account and makes his malicious post using the following url:

`https://vulnerable-site.com/post?content=<script>EVIL_CODE</script>`

If the site inserts the content as raw HTML, then the malicious Javascript is loaded when any user views that post. This
is similar to the Reflected XSS attack from our first example, except now anyone who views the malicious post can be
exposed to the malicious code. The malicious posts gets stored on the server's database and sticks around even for other
users.

## Terminology

In practice, the distinction may not be as clear. In many cases, we can actually have XSS attacks that are really a mix of reflected and stored XSS (and even a third category called "DOM Based XSS"). If you are interested in learning more about this, OWASP (Open Web Application Security Project) has a really nice article on it linked [here](https://owasp.org/www-community/Types_of_Cross-Site_Scripting).
