------



# Docker使用教程

## 1 第一个SSH Key设置(如GitLab账号)

   使用此命令时三次Enter回车即可，将会在~/.ssh目录下生成id_rsa和id_rsa.pub(默认)。将id_rsa.pub中的内容复制粘贴到GitLab的SSH Key中。

    $ ssh-keygen -t rsa -C “gitlab注册邮箱”
    
## 2 第二个SSH Key设置(如GitHub账号)

   仍然使用此命令后，不要马上回车，设置生成文件名称，如id_rsa_github，然后再连续回车三次，将会在当前目录下生成如id_rsa_github和如id_rsa_github.pub文件，将两个文件复制到~/.ssh目录下，并将id_rsa_github.pub的内容复制到GitHub的SSH Key中。
   
    $ ssh-keygen -t rsa -C “githab注册邮箱”
    
## 3 添加私钥

   git将生成的私钥添加到known_hosts中
   
    $ ssh-add ~/.ssh/id_rsa
    $ ssh-add ~/.ssh/id_rsa_github
    
   如果执行ssh-add时提示**Could not open a connection to your authentication agent**，可执行以下命令：
   
    $ ssh-agent bash
    
   然后再执行ssh-add命令
   
    # 通过ssh-add -l来确认私钥列表
    $ ssh-add -l
    # 通过ssh-add -D来清空私钥列表
    $ ssh-add -D
    
## 4 修改配置文件

   在~/.ssh目录下新建一个config文件（文件格式文件，和rsa文件格式相同）
   
    touch config

   并添加以下内容
   
    # gitlab
    Host gitlab.com
        HostName gitlab.com
        PreferredAuthentications publickey
        IdentityFile ~/.ssh/id_rsa
    # github
    Host github.com
        HostName github.com
        PreferredAuthentications publickey
        IdentityFile ~/.ssh/id_rsa_github

## 5 测试

   如果输出**Hi xuyuan923! You've successfully authenticated, but GitHub does not provide shell access**，说明成功的连上github了。

    ssh-T git@"你的gitlab服务器地址"
    $ ssh -T git@github.com
