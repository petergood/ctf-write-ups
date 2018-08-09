# Military grade authentication
Problem: https://hack.cert.pl/challenge/military-grade-authentication

This problem is quite interesting and in my opinion should be a mandatory exercise for all aspiring web developers, as it shows that even a seemingly harmless `.git` directory on a production server can be an extremely dangerous issue.

Initial recon of `https://military-grade-authentication.ecsc18.hack.cert.pl/` shows the `.git` directory in `robots.txt`. Not many people are aware of this (more on that later), but it is actually possible to "download" a GIT repository from a production server and recover all of the source code that was under version control. The website in this challenge does not list directories, so we will recover the contents of `.git` by guessing where the contents of that directory is. If directory listing was enabled, recovering the directory would be as simple as running `wget` in recursive mode.

Most `.git` directories contain the following files:
+ .git/HEAD
+ .git/ORIG_HEAD
+ .git/index
+ .git/description
+ .git/config
+ .git/info/exclude
+ .git/logs/HEAD
+ .git/logs/refs/heads/master
+ .git/logs/refs/remotes/origin/master
+ .git/refs/heads/master/
+ .git/refs/remotes/origin/master
+ .git/objects/info/packs
+ .git/COMMIT_EDITMSG
+ .git/packed-refs
+ .git/refs/stash

Using these files, we can reconstruct the `objects` directory that contains the source code (divided up into hashed blocks). After downloading all of those files into a local `.git` directory (to create a valid repository), we can run `git log`

```
commit 249ddda51e7b92bbccdee32fa23ef62055163639
Author: Mr Robots <example@example.com>
Date:   Wed Jun 13 19:15:10 2018 +0200

    My grandpa gave me some tips on improving the security of my website

commit 278ebda69d92baa3dfa787d91ba77da517ceefaa
Author: Mr Robots <example@example.com>
Date:   Wed Jun 13 19:13:57 2018 +0200

    Init my awesome websitez
```

This reveals the commit hashes which we use to recover the objects from `.git/objects`. Objects are stored in subdirectories: `.git/objects/XX/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`, where there first block is the first two characters of the commit hash and the second block is the remaining thirty eight characters (eg. `.git/objects/24/9ddda51e7b92bbccdee32fa23ef62055163639`).

Knowing the commit hashes from `git log`, we can download the commit objects. Next, we run `git fsck` which reveals missing objects and trees. We then download the missing objects and repeat the process until `git fsck` reveals no errors.

Finally, to solve the problem, we run `git show` that reveals a weak password hash in a previous commit.
![img](https://i.imgur.com/PKqogec.png)

Technically we could brute-force the password using John The Ripper or hashcat, but simply using http://md5decrypt.net/en/ cracks the md5 hash in seconds. Logging into the website with the discovered credentials reveals the flag.

Despite the serious security risk, many people are completely unaware that accidentally uploading `.git` to a production server (or cloning a repository) can be such a critical issue. According to Internetwache, around ten thousand websites in the Alexa top 1M list contain a `.git` directory.
