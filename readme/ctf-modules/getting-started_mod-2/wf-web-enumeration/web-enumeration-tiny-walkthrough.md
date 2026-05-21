# Web Enumeration tiny Walkthrough

Use this walkthrough to apply the web enumeration techniques and get the flag.

### Goal

Try the web enumeration techniques from this section against the target.

Then use what you find to reach the flag.

{% stepper %}
{% step %}
### Spawn the target and Pwnbox

I spawned the target system and Pwnbox.

<figure><img src="../../../../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>
{% endstep %}

{% step %}
### Open the target in the browser

<figure><img src="../../../../.gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../../../.gitbook/assets/image (16).png" alt=""><figcaption></figcaption></figure>

Check the IP in the browser.

Then review the page source.
{% endstep %}

{% step %}
### Run Gobuster against the target

I ran Gobuster on `IP:Port` using `SecLists/common.txt`.

<figure><img src="../../../../.gitbook/assets/image (17).png" alt=""><figcaption></figcaption></figure>

Gobuster returned `200` for `/index.php` and `/robots.txt`.
{% endstep %}

{% step %}
### Review `robots.txt`

Next, browse to `IP/robots.txt`.

`index.php` is also accessible.

<figure><img src="../../../../.gitbook/assets/image (18).png" alt=""><figcaption></figcaption></figure>
{% endstep %}

{% step %}
### Open the admin login page

Next, browse to `IP/admin-login-page.php`.

<figure><img src="../../../../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

This reaches the admin panel.

Now press `Ctrl+U` to review the page source.
{% endstep %}

{% step %}
### Find the test credentials

<figure><img src="../../../../.gitbook/assets/image (20).png" alt=""><figcaption></figcaption></figure>

The source shows test credentials.

Try those credentials on the login form.
{% endstep %}

{% step %}
### Log in and capture the flag

<figure><img src="../../../../.gitbook/assets/image (21).png" alt=""><figcaption></figcaption></figure>

The login works.

The flag is now visible.
{% endstep %}
{% endstepper %}

### Key takeaway

One short enumeration chain led to the flag:

* Gobuster found useful paths.
* `robots.txt` exposed the next step.
* Page source leaked credentials.
