.. title: "用 Fabric 跟 Cuisine 做自动化发布"
.. date: 2012-11-27 11:33
.. comments: true
.. categories: [技术, DevOps, fabric, cuisine]
.. keywords: devops, fabric, cuisine, 自动化发布
.. description: 用 Fabric 跟 Cuisine 做自动化发布


在招工广告上经常见到提到 DevOps。  
我是土人，不晓得是神马玩意儿，狗了一番，度了一把，貌似有点懂了：大概就是自动化部署的意思。  
Ops 就是 Operation, Dev 如无意外就是开发。  
两个单词加在一起，依据鄙人理解，大概就是把 Operation 这个动作以代码的方式反映出来。  

最简单的部署，就是一行行敲命令行敲上去。  
本人这类菜鸟，都是东抄一行，西抄一行命令那么搞出来的。  
人工介入太多的结果是，发布出来的东西可能会不稳定，因为不排除中间漏了东西。而且发布过程可能会有许多错误。  
解决办法就是让电脑替我们做事，最初步的方法就是使用部署脚本。比较高级一些，或者说比较专业的做法就是使用 Chef 或者 Puppet 这类的工具去做部署。  

之前也好奇试了一下 Chef，有感于一个 Chef recipie 实在需要太多文件了。搞不懂用shell都是几行的东西，他能搞 n 个文件夹，n 个文件出来。然后这玩意儿还很火。囧rz...  
鉴于本人智商有限，刚刚接触 DevOps 这种东西，还是玩一些 low tech 的东西吧。  
DevOps 中，Python 界的[Fabric](http://fabfile.org) 也经常被提起。而据说 Instagram 就是用 [Fabric](http://fabfile.org) 去做的。而有人在这 [Fabric](http://fabfile.org) 之上做了 [Cuisine](https://github.com/sebastien/cuisine)，多做一层封装，提供多一些的抽象，简化了某些 API。

废话少说，奉上代码：  
### 一、配置系统基本环境的代码
主要入口是 prepare_system，其中必须的参数是你必须提供 admin_user 用户名。  
这段代码会创建一个新的用户，而其属于 admin 组。这个用户会使用你账号下的 DSA public key，以供后来免密码登陆。

这样你以后就可以把 root 的 ssh 登陆给禁止掉，可以防止别人暴力破解 root 密码。
上面也有配置 iptables 的防火墙代码，从网上抄来的，很小白，请大家不要攻击。。。

其中也有不少都可以删掉的东西，譬如 prepare_rbenvs, prepare_devenv 。用不到尽管删除掉。


```
fab -H yourhost -u root prepare_system:new_admin
```
{% gist 4151899 %}

### 二、自动生成 Octopress 的博客代码。
使用方法很简单。先写好博客，commit&push 后就用调用 fabric 进行部署。


```
fab -H keyonly.com -u your_user deploy
```

```python
from fabric.api import sudo, cd
from fabric.context_managers import prefix, settings
from cuisine import dir_exists, file_write, file_exists, upstart_ensure, run


site_cfg = """
server {
    server_name    www.keyonly.com    keyonly.com  keyonly.test;
    access_log /var/log/nginx/keyonly.access.log;
    index index.html index.htm;

    location / {
        root /srv/keyonly.com/;
        try_files $uri $uri/ /index.html;
    }
}
"""


def test_exists():
    with cd('~'):
        if not dir_exists('blogging'):
            print 'not exist'
        else:
            print 'exist'


def deploy():
    with cd('~'):
        if not dir_exists('blogging'):
            run('mkdir blogging')
            with cd('blogging'):
                run('git clone git://github.com/imathis/octopress.git')
                run('git clone git://github.com/tly1980/my_blog.git')

    with cd('~/blogging/octopress'):
        with prefix('source ~/.bash_profile'):
            # install the desire ruby version
            run('bundle install')

    with cd('~/blogging/my_blog'):
        run('git pull')

    with cd('~/blogging/octopress'):
        with settings(warn_only=True):
            run('rm Rakefile _config.yml config.rb source')

        run('ln -s ../my_blog/Rakefile .')
        run('ln -s ../my_blog/_config.yml .')
        run('ln -s ../my_blog/config.rb .')
        run('ln -s ../my_blog/source .')
        run('rake generate')

    with cd('~'):
        with settings(warn_only=True):
            sudo('rm -rvf /srv/keyonly.com')

        sudo('cp -r blogging/octopress/public /srv/keyonly.com')
        sudo('chmod -R 0755 /srv/keyonly.com')

    file_write('/etc/nginx/sites-available/keyonly.com', site_cfg, sudo=True)
    if not file_exists('/etc/nginx/sites-enabled/keyonly.com'):
        sudo('ln -s /etc/nginx/sites-available/keyonly.com /etc/nginx/sites-enabled/keyonly.com')

    upstart_ensure('nginx')
```
