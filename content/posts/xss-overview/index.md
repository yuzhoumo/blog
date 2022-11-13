---
title: Stored vs Reflected XSS
description: Brief overview of cross site scripting
date: 2021-04-29
tags: [security, code, programming]
---

In cross site scripting vulnerabilities, the user of a website executes malicious JavaScript code injected by an attacker. This article is a beginner friendly explanation of two major categories of XSS (Cross Site Scripting) vulnerabilities: "Reflected XSS" and "Stored XSS".

## How XSS Works

HTML uses tags to denote denote page layout. For instance, `<p></p>` tags indicate plain text. XSS takes advantage of inline scripting, where any HTML content surrounded by `<script></script>` tags are interpreted as Javascript code and executed. This means that if attackers are able to insert raw HTML into a page that a user is visiting, they can use that power to insert script tags and get a user to run their malicious code.

## Reflected XSS

Many sites use URL query parameters to pass information from the user to a remote server. Consider the following URL:

`https://www.google.com/search?q=helloworld`

This link takes you to a Google search for `helloworld`. Everything after the `?` is data that we want to send to Google, namely `q=helloworld`. This conveys that our query `q` will be sent to Google with the value `helloworld`.

In reflected XSS, an attacker might take advantage of this fact by sending the server a malicious query. This link could contain malicious Javascript code that could be executed by a vulnerable site. If the attacker sends this link to an unsuspecting user and gets that person to click on it, then the attacker's malicious code will execute when the user visits the URL.

Consider the following example. Say a website has a vulnerable search endpoint:

`https://vulnerable-site.com/search?q=helloworld`

Unlike `google.com`, our vulnerable website returns a page that looks like this:

```html
<h1>Search</h1>
<h3>Search results for helloworld</h3>
<table>
  <tr>
    <td>No results!</td>
  </tr>
</table>
```

It simply takes whatever value `q` is set to and puts it directly into the webpage after "Search results for".

Notice how the vulnerable site just injects `helloworld` directly into the page. Now imagine if the attacker had created a link that looked like this:

`https://vulnerable-site.com/search?q=<script>EVIL_CODE</script>`

If the attacker gets a user to click on this link, the result page would look something like this:

```html
<h1>Search</h1>
<h3>Search results for <script>EVIL_CODE</script></h3>
<table>
  <tr>
    <td>No results!</td>
  </tr>
</table>
```

In this example `EVIL_CODE` is just a placeholder, but this vulnerability allows the attacker to inject any Javascript code into the website.

## Stored XSS

Stored XSS is similar to reflected XSS with the exception that the malicious Javascript is stored on the server permanently. Say we have a vulnerable API enpoint:

`https://vulnerable-site.com/post?content=helloworld`

Let's say that this enpoint takes whatever value is stored in `content` and makes a Twitter style post on the site with that content. Instead of sending a link to the victim, say the the attacker logs in and makes his malicious post using the following url (which is stored in the website's database):

`https://vulnerable-site.com/post?content=<script>EVIL_CODE</script>`

If the site inserts the content as raw HTML, then the malicious Javascript is loaded when any user views that post.

## Terminology

In many cases, we can actually have XSS attacks that are really a mix of reflected and stored XSS (and even a third category called "DOM Based XSS"). If you are interested in learning more about this, OWASP (Open Web Application Security Project) has a really nice article on it linked [here](https://owasp.org/www-community/Types_of_Cross-Site_Scripting).
