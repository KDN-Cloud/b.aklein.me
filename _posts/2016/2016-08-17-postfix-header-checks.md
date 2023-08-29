---
title: 'Postfix Header Checks'
date: 2016-08-17
description: Using head checks with Postfix.
tags:
- Postfix
- Mail
---

This is an example of some <a href="http://www.postfix.org/header_checks.5.html" target="_blank" >Postfix Header Checks</a> which I use to help with mail spammers. This has been working as expected really well. One thing to note is that I did need to comment out `#/^Precedence:.*bulk/ REJECT We do not accept your spam` because it would prevent auto-reply emails from being sent such as vacation or out-of-office responses. 

To load these header checks I am using the  <a href="http://www.postfix.org/pcre_table.5.html" target="_blank" >pcre</a> format defined inside `main.cf`.

`header_checks = pcre:/etc/postfix/header_checks`

### GitHub Gist
<script src="https://gist.github.com/variablenix/611107e3daa295a21e7ce34c46fb6d90.js"></script>