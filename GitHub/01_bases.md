# echo "# learngit" >> README.md

# git init
Initialized empty Git repository in /Users/xuchao/learngit/.git/

# git add README.md

# git commit -m "first commit"
[master (root-commit) 9968647] first commit
 1 file changed, 1 insertion(+)
 create mode 100644 README.md

# git remote add origin https://github.com/CHXU0088/learngit.git

# git push -u origin master
Username for 'https://github.com': CHXU0088
Password for 'https://CHXU0088@github.com':
Enumerating objects: 3, done.
Counting objects: 100% (3/3), done.
Writing objects: 100% (3/3), 223 bytes | 223.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To https://github.com/CHXU0088/learngit.git
 * [new branch]      master -> master
Branch 'master' set up to track remote branch 'master' from 'origin'.



# git init

# ssh-keygen -t rsa -C "xuchao0630@126.com"

# cat id_rsa.pub

github => Settings => SSH and GPG keys

# ssh -T git@github.com

# git config --global user.name "CHXU0088"
# git config --global user.email "xuchao0630@126.com"

# git status

# git add .
# git commit -m "tag"
# git remote add origin https://github.com/CHXU0088/<Projects>.git
# git push -f origin master | git push -u origin master