#### git push 和 git pull 总是卡住而且没有显示内容

可能是连接不上，git-remote-https 有问题；

需要添加代理；

根据 ClashX 的 socks5 端口进行代理的配置即可；

```shell
git config --global https.proxy socks5://127.0.0.1:9871

git config --global https.proxy socks5://127.0.0.1:9871

git config --global --unset http.proxy

git config --global --unset https.proxy
```

#### 需要输入用户名和密码

git 目前好像不支持输入用户名和密码了；

需要配置 ssh key 才行；

通过 `ssh-keygen -t rsa -C 'xus7@pku.edu.cn'` 来生成密钥对；

并在 github 的 settings -> ssh 里面增加密钥 id_rsa.pub；

测试链接 `ssh -T git@github.com` 成功就可以了；

#### 更换远端分支

因为之前用的是 https，设置 ssh 之后需要改成 ssh 的；

执行 `git remote remove origin` 移除关联的远端分支；

执行 `git remote add origin git@github.com:xus7Pku/levelDbNote.git` 关联到新的远端分支；