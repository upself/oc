注意：所有的project都需要使用developer用户来创建   
oc get projects
oc delete project  #####
第一题：p43
1> 使用developer用户登录
[root@workbench ~]# oc login -u developer -p flectrag（密码） https://master.domain?.example.com 看题目查看用户名和密码
2> 创建一个project
[root@workbench ~]# oc new-project crimson
Now using project "pastebin" on server "https://master.domain1.example.com:443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git

to build a new example application in Ruby.
[root@master ~]# 
3>创建一个应用 sourc code的路径：http://git.domain1.example.com/git/pastebin.git
    http://services.lab.example.com/nodejs-helloworld
      创建应用的时候要把这个源码下载下来，因为提供的package.json文件有错误
	  需要修改重新上传之后在去创建应用
     1~ [root@workbench pastebin]# git clone http://git.domain1.example.com/git/pastebin（可能会变）.git
     2~ [root@workbench ~]# cd pastebin/
     3~ [root@workbench pastebin]# python -m json.tool package.json  检测json格式
     4~ [root@workbench pastebin]# vim package.json
           "description": "Hastebin Plus is an open-source Pastebin software written in node.js." ,
            这一行少了一个,号
            第12行"express" "4.14.x"改为"express": "4.14.x"
      5~ [root@workbench ~]# git commit -a -m "modify"
      6~ [root@workbench ~]# git push     可能会要求有存储仓库的用户名密码，在用户帮助里面
      7~ 创建应用的时候需要指定构建参数npm_config_registry
            [root@workbench ~]# oc new-app  --name pastebin --build-env npm_config_registry=http://nexus.domain1.example.com:8081/nexus/content/groups/nodejs http://git.domain1.example.com/git/pastebin.git
4>查看和验证
[root@workbench ~]# oc logs -f bc/pastebin
Cloning "http://git.domain1.example.com/git/pastebin.git" ...
	Commit:	4f71b8cff884c0cdeaf20517fb39346dabec98e5 (modify)
	Author:	root <root@master.domain1.example.com>
	Date:	Wed Jun 19 08:15:35 2019 +0000
---> Installing application source ...
---> Building your Node application from source
hastebin-plus@0.0.4 /opt/app-root/src
+-- clean-css@3.4.28
| +-- commander@2.8.1
| | `-- graceful-readlink@1.0.1
| `-- source-map@0.4.4

[root@workbench ~]# oc get pods
NAME               READY     STATUS      RESTARTS   AGE
pastebin-1-build   0/1       Completed   0          5m
pastebin-1-q6hb5   1/1       Running     0          1m     check
[root@master ~]# 

5> 创建route
[root@workbench ~]# oc expose svc pastebin --hostname pastebin-crimson.cloudapps.domain1.example.com
[root@workbench ~]# curl pastebin-crimson.cloudapps.domain1.example.com -I 
HTTP/1.1 200 OK
X-Powered-By: Express
Accept-Ranges: bytes
Cache-Control: public, max-age=0
Last-Modified: Wed, 19 Jun 2019 08:19:03 GMT
ETag: W/"3da-16b6ed15cd8"
Content-Type: text/html; charset=UTF-8
Content-Length: 986
Date: Wed, 19 Jun 2019 08:26:00 GMT
Set-Cookie: d361458f51dc48ef205b2f4cfd8689d4=8eecabb8e768e7c384af51c56024a7b5; path=/; HttpOnly
Cache-control: private
[root@workbench ~]# 
6> 用火狐浏览器打开，然后贴上一句话http://pastebin-crimson.cloudapps.domain1.example.com要保存题目的那句话


第二题：p115
2> 使用admin用户
[root@master ~]# oc login -u admin -p flectrag
 [root@master ~]# oc project default

3>授权
[root@master ~]# oc adm policy add-role-to-user system:registry developer
role "system:registry" added: "developer"
[root@master ~]# 
[root@master ~]# oc adm policy add-role-to-user system:image-builder developer
role "system:image-builder" added: "developer"
[root@master ~]# 
[root@workbench ~]# oc login -u developer -p flectrag（密码） https://master.domain?.example.com *********************************************************************************************
3> 登录workbench
[desktop@host1 ~]$ ssh -X root@workbench.domain1.example.com
[root@workbench ~]# su - devop
[devop@workbench ~]$ 

4> docker  login
[devop@workbench ~]$ sudo vim  /etc/sysconfig/docker
INSECURE_REGISTRY='--insecure-registry docker-registry-default.cloudapps.domain1.example.com'
[devop@workbench ~]$ sudo systemctl restart docker
[devop@workbench ~]$ oc whoami  -t
SWnTTtcyy0GS2aRc6FV5lowNKr5znFJSCn-RfBmO_zE
[devop@workbench ~]$ sudo docker login -u devepoler -p SWnTTtcyy0GS2aRc6FV5lowNKr5znFJSCn-RfBmO_zE docker-registry-default.cloudapps.domain1.example.com 
Login Succeeded
[devop@workbench ~]$ 
********************************************************************************************************

第三题：p73(第九步开始) 在workbench上完成：优化dockerfile
此题对应书中实验包括serviceaccount
1>下载提供的dockerfile
[devop@workbench ~]$ git clone http://git.domain1.example.com/git/build.git
Cloning into 'build'...
remote: Counting objects: 9, done.
remote: Compressing objects: 100% (7/7), done.
remote: Total 9 (delta 1), reused 0 (delta 0)
Unpacking objects: 100% (9/9), done.
[devop@workbench ~]$ 

2>针对这个下载下来的Dockerfile分析得的结论有
    1： FROM FROM registry.domain1.example.com:5000/rhel7
    2： 部署的应用是一个http的应用
	按照题目分析：
	1> 这在将来可以作为一个parent image，所以你需要在Dockerfile中加入
	ONBUILD指令
	2> 该image可以构建应用：在原来的docker中就已经存在index.html文件
    所以此步骤
	3> 最为恶心的是要少于8个image layers 还要小于250MIB
	
	所以可将dockerfile修改至如下 ： 优化： LABEL,RUN，ENV可以在一处执行，减少层数和大小
[devop@workbench ~]$ vim build/Dockerfile 
FROM registry.domain1.example.com:5000/rhel7

MAINTAINER Red Hat Certifications

# General docker labels
LABEL Component="httpd" \
      Name="ex288/httpd-parent" \
      Version="2.4.6" \
      Release="1"

# Labels could be consumed by OpenShift
LABEL io.k8s.description="httpd [apache ] is an HTTP server." \
      io.k8s.display-name="httpd 2.4.6" \
      io.openshift.expose-services="8080:http" \
      io.openshift.tags="httpd"

ADD rhgls.repo /etc/yum.repos.d/exam.repo
ADD scripts /scripts
RUN   yum -y update systemd && \ 
           yum -y install httpd && \
           yum clean all && \
          mkdir -p /var/www/html && \
          sed -i "s/Listen 80/Listen 8080/g" /etc/httpd/conf/httpd.conf && \
          chgrp -R 0 /var/log/httpd /var/run/httpd && \    
          chmod -R g=u /var/log/httpd /var/run/httpd
EXPOSE 8080
USER 1001
ONBUILD COPY /src/ /var/www/html
CMD ["/bin/bash", "/scripts/run-apache.sh"]
[devop@workbench ~]$ 

3>验证 可在workbench上build一次，验证完成之后删除
[devop@workbench build]$ sudo docker build -t http .
[devop@workbench build]$ sudo docker inspect http 
   "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:dda6e8dfdcf7a918d1fd7038a2a6de43f78c194c0bb220ea086af6c414254f4b",
                "sha256:86888f0aea6df7a7a4ae5595c505b0abccc1002f717efa5be32fa0337099bf1d",
                "sha256:b0fb749cc71cbf5b934131c5969e8f54410741d49bef356a2ffea5a9a0fc7fdb",
                "sha256:5434acca42844df48b53b0235e862058bee9db34eee3250b8f501829c186bded",
                "sha256:edf41f6e33960b4c5b7b4a015d614484f87b802cc4ded0e49b636b69dad5bc4e",
                "sha256:d414be876a216c721db986e428f1bb005d9aa5a53ee4ebc2aa1cedd66431e0fe"
            ]
        }
    }
]
	
[devop@workbench build]$ sudo docker images 
REPOSITORY                                TAG                 IMAGE ID            CREATED             SIZE
http                                      latest              311876e1056c        3 minutes ago       222.1 MB
registry.domain1.example.com:5000/rhel7   latest              93bb76ddeb7a        24 months ago       192.7 MB
[devop@workbench build]$ 
[devop@workbench build]$ sudo sudo docker  rmi http
[devop@workbench build]$ sudo sudo docker  rmi rhel7

4> 上传
[devop@workbench build]$ sudo git commit -a -m "optimize" 
[devop@workbench build]$ git push
********************************************************************************************************
5>下载验证
[devop@workbench ~]$ git clone http://git.domain1.example.com/git/build.git
Cloning into 'build'...
remote: Counting objects: 12, done.
remote: Compressing objects: 100% (10/10), done.
remote: Total 12 (delta 2), reused 0 (delta 0)
Unpacking objects: 100% (12/12), done.
[devop@workbench ~]$ ll
cat 查看dockerfile文件
6> 通过创建应用（new-app）验证pod可以正常提供服务
********************************************************************************************************

第四题：p84 创建应用
1>创建应用 
[root@master ~]# oc login -u developer 
Now using project "origin" on server "https://master.domain1.example.com:443".

You can add applications to this project with the 'new-app' command. For example, try:

    oc new-app centos/ruby-22-centos7~https://github.com/openshift/ruby-ex.git

to build a new example application in Ruby.

[root@master ~]$ oc new-project origin 

[root@master ~]#  oc new-app --name greeter --docker-image  registry.domain1.example.com:5000/openshift/hello-openshift

2>创建router
[root@master ~]# oc expose svc greeter --hostname greeter-origin.cloudapps.domain1.example.com
route "greeter" exposed
[root@master ~]# curl greeter-origin.cloudapps.domain1.example.com
Hello OpenShift!
[root@master ~]# 

3> 创建configmap
[root@master ~]# oc create configmap greeterconf --from-literal RESPONSE="Hello from the Certification Team"
configmap "greeterconf" created
[root@master ~]# 

4> 关联Configmap
[root@master ~]# oc set env dc/greeter --from configmap/greeterconf
deploymentconfig "greeter" updated
[root@master ~]# 

5> 导出configmap
[root@master ~]# oc get configmap greeterconf  -o json > /root/workdir/greeterconf.json   //get也可替换为export


第五题：p184
1>下载
[devop@workbench ~]# git clone http://git.domain1.example.com/git/s2i.git 

2> 修改s2i/.s2i/bin/assemble
[devop@workbench ~]# vim s2i/.s2i/bin/assemble
在######## CUSTOMIZATION STARTS HERE ############下面增加
echo "starting build......"
cp  /tmp/src/*.html ./

DATE=` date +%Y-%m-%d`
  
echo "$DATE Astra inclinant, Sed non oblogant" >> ./info.html

3>提交
[devop@workbench s2i]# git commit -a -m "again"
[devop@workbench s2i]# git push

4>创建应用
[devop@workbench ~]$ oc new-project o2
[devop@workbench s2i]# oc new-app --name s2i   httpd~http://git.domain1.example.com/git/s2i.git 

5>创建router
[devop@workbench ~]# oc expose service s2i --port=8080 --hostname s2i-o2.cloudapps.domain1.example.com
route "s2i" exposed
[devop@workbench ~]# oc get route
NAME      HOST/PORT                              PATH      SERVICES   PORT      TERMINATION   WILDCARD
s2i       s2i-o2.cloudapps.domain1.example.com             s2i        8080                    None
[devop@workbench ~]# curl http://s2i-o2.cloudapps.domain1.example.com
Amor vincit omnia
[devop@workbench ~]# curl http://s2i-o2.cloudapps.domain1.example.com/info.html
2019-20-18


第六题：p162
2>创建project
[devop@workbench ~]# oc new-project octane
3>创建应用
[devop@workbench ~]# oc new-app --name blog http://git.domain1.example.com/git/blog.git
3>创建route
[devop@workbench ~]# oc expose svc blog --hostname blog-octane.cloudapps.domain1.example.com
4>设置post-build hook
[devop@workbench ~]# oc set build-hook bc/blog --post-commit --command -- python mailer.py    （ -- 后面有空格）
[devop@workbench ~]# oc start-build bc/blog  


第七题：p246 设置 Livenessprobe  web 页面操作
本题采用web做非常简单
1>[desktop@host1 ~]$ firefox https://master.domain1.example.com
2>点击octane
3>点击blog
4>在右上方的”action“中选择“Edit health check"
5> 点击“ add liveness probe"
6> 类型选择“TCP Socket",Initial dealaly 填写10 ，timeout 填写30 
7> 点击“save"
 


第八题：p193 此题关键点为找到ruby-2.2这个版本的builder image
[devop@workbench ~]# oc login  -u admin -p flectrag
docker-registry-cli registry.lab.example.com:5000 search ruby ssl 或 在openshift自带registry console里面找？
1>创建IS
[devop@workbench ~]# oc import-image rubyred --confirm --from registry.domain1.example.com:5000/rhscl/ruby-22-rhel7 -n openshift 
链接不对，取openshit web主页面check link， default-》application-》rutes-》搜索ruby 2.2
 oc import-image rubyred:2.2 --confirm --from registry.domain1.example.com:5000/rhscl/ruby-22-rhel7 -n openshift
 
[devop@workbench ~]# oc login  -u developer -p flectrag

2>创建应用
[devop@workbench ~]$ oc new-project indigo
[devop@workbench ~]#  oc new-app --name ircbot  -i  rubyred  http://git.domain1.example.com/git/ircbot.git

3>决定是否创建configmap:
[devop@workbench ~]$ oc create configmap ib \        需要看题目需求，如果没有写改为自己的名字
> --from-literal SERVER=irc.domain1.example.com \
> --from-literal NICKNAME=Gustav     需要check 题目要求
configmap "ib" created         需要看题目需求，如果没有写改为自己的名字
[devop@workbench ~]$ oc get dc
NAME      REVISION   DESIRED   CURRENT   TRIGGERED BY
ircbot    1          1         1         config,image(ircbot:latest)
[devop@workbench ~]$ oc set env dc/ircbot \
> --from configmap/ib
deploymentconfig "ircbot" updated
[devop@workbench ~]$ 


第九题：p223 模板
1>下载模板文件
   [devop@workbench ~]$ wget http://rhgls.domain1.example.com/materials/php-app.yaml

2>修改模板文件
1: PHP和MYSQL的版本，题目所给的tag是没有对应的builder image 
	所以需要修改两个地方    gg 是最上面， shift+gg 最下面   ：q！  退出保存
  [devop@workbench ~]# vim php-app.yaml 
1 --------------------------
>           name: php:5.5   看题目 是多少
---
>           name: mysql:5.7
2 --------------------------
>         uri: http://git.domain1.example.com/git/php-app.git
3 --------------------------
> - name: HELLO_AUDIENCE
>   displayName: The custon parameter HELLO_AUDIENCE
>   description: The custom parameter2
>   value: Avatar     注意大小写
>- description: The custom parameter1
>   displayName: The custon parameter HELLO_MESSAGE
>   name: HELLO_MESSAGE
>   value: 
4 --------------------------
>   template: ex288-php-mysql
>   name: ex288-php-mysql
>   namespace: indy
5 --------------------------
> - name: APPLICATION_DOMAIN
>   displayName: Application Hostname
>   description: The exposed hostname that will route to PHP app service, if left blank a value will be defaulted.
>   required: true
---------------------------------------------------------------------------------------------------------------------------------------------------------------
3> 创建project 
 [devop@workbench ~]# oc new-project indy 

4> 导入末班文件
[devop@workbench ~]# oc create  -f php-app.yaml 

5>下载源码
[devop@workbench ~]$ git clone http://git.domain1.example.com/git/php-app.git

6>修改源代码，在源代码中加入这段话
[devop@workbench ~]$ vim xxx
<?php
if ($message="Hi") {
   echo "Hi Buddy" ;
   exit ;
}
?>

上传 - 参考第一题git commit + git push
  [devop@workbench ~]# git commit -a -m "kat"
  [devop@workbench ~]# git push     
7> 创建应用测试
[devop@workbench ~]# oc new-app ex288-php-mysql -p APPLICATION_DOMAIN=php-app-indy.cloudapps.domain1.example.com -p HELLO_MESSAGE=Hi -p HELLO_AUDIENCE=Buddy
