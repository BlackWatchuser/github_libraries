# 将本地代码 push 到 github

``` shell
> "+" => New repository => ...... => <github_libraries>

# ssh-keygen -t rsa -C "xuchao0630@126.com"

# mkdir github_libraries && cd github_libraries

# echo "# github_libraries" >> README.md

# cat id_rsa.pub

> github => Settings => SSH and GPG keys

# ssh -T git@github.com

# git config --global user.name "CHXU0088"
# git config --global user.email "xuchao0630@126.com"

# git init
Initialized empty Git repository in /Users/xuchao/github_libraries/.git/

# git add README.md

# git commit -m "first commit"
[master (root-commit) 9968647] first commit
 1 file changed, 1 insertion(+)
 create mode 100644 README.md

# git remote add origin https://github.com/CHXU0088/github_libraries.git

# git push -u origin master
Username for 'https://github.com': CHXU0088
Password for 'https://CHXU0088@github.com':
Enumerating objects: 3, done.
Counting objects: 100% (3/3), done.
Writing objects: 100% (3/3), 223 bytes | 223.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To https://github.com/CHXU0088/github_libraries.git
 * [new branch]      master -> master
Branch 'master' set up to track remote branch 'master' from 'origin'.
```

------

``` shell
# git status
# git add .
# git commit -m "tag"
# git push -u origin master
```