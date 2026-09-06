# PrivEsc tiny Walkthrough

Use this walkthrough to move from the provided SSH access to `root` and recover the flag.

### Goal

Log in with the provided user credentials.

Use the allowed privilege path to move to another user.

Recover the `root` SSH key and use it to become `root`.

{% stepper %}
{% step %}
### Log in as `user1`

Start by connecting over SSH with the provided credentials.

This gives you the initial low-privilege shell.

<figure><img src="../../../../.gitbook/assets/image (47).png" alt=""><figcaption></figcaption></figure>
{% endstep %}

{% step %}
### Check the allowed `sudo` actions

Run `sudo -l` first.

This shows which commands or users you can access with elevated rights.

<figure><img src="../../../../.gitbook/assets/image (48).png" alt=""><figcaption></figcaption></figure>
{% endstep %}

{% step %}
### Switch to `user2`

Use the allowed `sudo -u` path to run a shell as `user2`.

That gives you access to the next set of files.

<figure><img src="../../../../.gitbook/assets/image (49).png" alt=""><figcaption></figcaption></figure>
{% endstep %}

{% step %}
### Read the user flag

Once you are `user2`, inspect the accessible files.

Reading `flag.txt` completes the user-level objective.

<figure><img src="../../../../.gitbook/assets/image (50).png" alt=""><figcaption></figcaption></figure>
{% endstep %}

{% step %}
### Recover the `root` private key

Keep checking for sensitive files after the first flag.

Here, the key finding is a readable `id_rsa` file that belongs to `root`.

<figure><img src="../../../../.gitbook/assets/image (51).png" alt=""><figcaption></figcaption></figure>
{% endstep %}

{% step %}
### Save the key locally

Copy the private key into a local file on your attacking box.

You need the key in a separate file before SSH will accept it.

<figure><img src="../../../../.gitbook/assets/image (52).png" alt=""><figcaption></figcaption></figure>
{% endstep %}

{% step %}
### Fix permissions and log in as `root`

Restrict the key permissions, then use it for SSH access:

```bash
chmod 600 local_key
ssh -i local_key root@<TARGET_IP> -p <PORT>
```

This opens a `root` shell on the target.

<figure><img src="../../../../.gitbook/assets/image (53).png" alt=""><figcaption></figcaption></figure>
{% endstep %}
{% endstepper %}

### Key takeaway

This escalation path stays simple:

* `sudo -l` reveals the next user context.
* `user2` access exposes sensitive files.
* A readable `root` private key leads straight to full access.
