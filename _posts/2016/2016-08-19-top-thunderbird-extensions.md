---
title: 'My Top Thunderbird mail extensions'
description: Thunderbird mail extensions
date: 2016-08-19
tags:
- Mail
- Thunderbird
- Extensions
- Top-Thunderbird-Mail-Extensions
---

# Intro
>> Mozilla Thunderbird is a free, open source, cross-platform email, news, and chat client developed by the Mozilla Foundation. 

Mozilla Thunderbird is arguably the best [Mail User Agent](https://en.wikipedia.org/wiki/Email_client) for the desktop. Being an avid user of email I thought I would list some of the extensions I find makes Thunderbird even better in no specific order.

### [uBlock Origin](https://addons.thunderbird.net/en-US/thunderbird/addon/ublock-origin/?src=search)
First on the list is uBlock Origin. I really think it's ridiculous to serve ads in 
emails, so this works really well for anyone looking to block all those annoying ads. This extension is probably not needed if emails are read in plain text.

### [Display Quota](https://addons.thunderbird.net/en-US/thunderbird/addon/display-quota/)
This is a nice extension to display your mail quota. I use quotas on my mail servers and like how this extension will tell you how many messages are in each folder. You can also have it give you a warning when you reach a certain percentage and modify it's appearance.

### [Enigmail](https://addons.thunderbird.net/en-US/thunderbird/addon/enigmail/)
This is a must have extension all Thunderbird users should have. It does a great job at what it was intended to do - sign & encrypt email messages. From my experience it has been quite stable.

### [ImportExportTools](https://addons.thunderbird.net/en-US/thunderbird/addon/importexporttools/)
This extension is great for those looking to import or export folders and messages. There are plenty of available options.

### [Manually sort folders](https://addons.thunderbird.net/en-US/thunderbird/addon/manually-sort-folders/)
I'm not sure why Thunderbird does not have native support for manually sorting folders, but this extension really does deliver. You can sort manually or automatically and re-order accounts in the folder pane. Definitely worth having. 

### [Markdown Here](https://addons.thunderbird.net/en-us/thunderbird/addon/markdown-here-revival/)
I really like using markdown and just so happen to write my blog using markdown, so thought why not extend support to other apps like Thunderbird. This extension works really well for writing email messages using markdown syntax. 

### [Send Later](https://addons.thunderbird.net/en-US/thunderbird/addon/send-later-3/)
I needed to send an email at a specific time and found Send Later to exist. I'm glad I came across this extension because it definitely excels at what it does. The caveat is that Thunderbird must be open for it to work, but the [support page](http://blog.kamens.us/send-later/#running) suggests some solutions.

I originally thought about writing a small script to do this, so decided to write something up that I could easily use on Linux and macOS systems.

```
#!/usr/bin/env bash

## use the 'at' command to send an outgoing email at a specific time
MAILTO=''
MAILFROM=''
SUBJECT=''
Cc=''
Bcc=''
AT="at 9:00 AM Today" # 'at' expressions: http://www.computerhope.com/unix/uat.htm

MESSAGE=''

# Begin script
$AT <<EMAIL
mail -s "$SUBJECT" -c "$Cc" -b "$Bcc" -r "$MAILFROM" "$MAILTO"
$MESSAGE
EMAIL

# EOF
```

### [Sieve](https://addons.mozilla.org/en-US/thunderbird/addon/sieve)
I use [pigeonhole](http://pigeonhole.dovecot.org) with Dovecot for Sieve support on my Linux server. I'm really glad this Thunderbird extension exists. It easily implements the ManageSieve protocol to securely manage Sieve Script on a remote IMAP server. For example, we can set a vacation notice.

```
require ["body","fileinto","vacation"];
# rule:[Vacation]
if true
{
	vacation :days 2 :addresses "hello@aklein.me" :subject "Out of Office" "Thanks for your message. I am on vacation and will respond to emails when I return.";
}
```

> I want to also point out you can grab the latest Thunderbird Sieve extension on [GitHub](https://github.com/thsmi/sieve/blob/master/README.md). I had to use a Development Build because the extension available from the official Mozilla page would hang and never make the initial connection.

So there you have all the extensions worth mentioning that I find make Thunderbird even better. Leave a comment if you have any other useful Thunderbird extensions!
