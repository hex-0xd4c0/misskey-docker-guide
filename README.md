# Docker部署Misskey部署标准指南





## 1. Misskey是什么
它是一个由Misskey团队开发私人搭建的服务器软件，站与站之间的联合可以建立一个极大的私人社交网络。由于不流行，所以没有专门针对该平台的爬虫，所有数据也都掌握在站长手中。但是需要自行搭建。






## 2. 需要什么
1. **一个2C4G轻量级应用服务器** 用来跑Misskey

建议使用**Ubuntu版本22以上**来避免一系列**不必要的折腾**。

> Misskey服务器并不需要ECS那么大的算力，但是内存官方要求不小于3G（你也可以自行配制脚本在一个2G的服务器上运行但是后果自负）

2. **一个域名（必须）** 

用来绑定你的服务器。推荐Cloudflare，老牌，且有**官方适配**。

**Misskey服务器的域名一旦在联邦里投入使用后不可更改**，一旦更改，首先是会对服务器本身造成未知影响，其次是会造成服务器联邦混乱，**请慎重决定自己的域名**。**请不打开更换过域名绑定的服务器的联合功能。**

> **不要尝试将多个域名指向Misskey服务器**，这没有用，对于可能会造成的未知问题，后果自负。

> 若后期想**同时使用`captraw.com`和`hub.captraw.com`** 并且希望`captraw.com`能**自动跳转**到`hub.captraw.com`可以在**Cloudflare Dashboard**或者你的ISP里那边自行折腾**重定向规则**。

**请顺手关闭所有与安全相关的配置并强制打开TLS (Full)和SSL**

3. **一个对象存储** 
非必须，但是没有它，misskey将无法处理诸如头像之类的图片（头像全是默认头）

> 由于个人NAS通常不包含CDN服务，访问速度将及其缓慢。
另外频繁读写将导致NAS硬盘寿命锐减，所以不建议将NAS作为存储桶使用






## 3. Misskey服务器的架构（前置知识）
1. **Misskey服务器软件本身** ——通常使用Docker运行，极特殊情况下本地部署
2. **Redis缓存服务** ——用于缓存常用数据
3. **PostgreSQL数据库** ——存储服务器的文本数据
4. **Nginx反向代理服务** ——用于配置反向代理，缓冲并加速http(s)访问
5. **对象存储服务** 一般与Misskey不在同一服务器上，如果在同一服务器上，可能需要额外配置

> **不要尝试在发帖的时候加入💩山代码或者几百屏的报错！！！** 因为帖子的文本存储是交由PostgreSQL完成的，所以**过多的文本**可能会导致一些**轻量级服务器上的PostgreSQL崩溃**从而导致**炸服**。










## 4. 配置域名的DNS记录

如果是Cloudflare或者使用Cloudflare DNS，请跟着走。

1. 打开**Cloudflare Dashboard**
2. 鼠标点开你的域名（如`captraw.com`），左边找**DNS**，下拉打开点击**记录**
3. 点击**添加记录**，类型选择**A**，**名称**填你想要使用的子域，**地址**填你的服务器公网IP。
如你想使用`hub.captraw.com`左边就填`hub`，如果想要直接使用根域名那就填`@`，这样使用的就是`captraw.com`。

**不要在地址中填入ISP提供的内网IP**

> 若后期想同时使用`captraw.com`和`hub.captraw.com`并且希望`captraw.com`能自动跳转到`hub.captraw.com`可以在左边栏找**规则**自行折腾。

如果你有**IPv6**，可以在这里一并配置，请在添加完**A**类记录后再添加一个**AAAA**记录

4. **打开Cloudflare代理**，让**灰云**变成**橙云**

> 方便使用`Let's Encrypt`自动配置**SSL**和**https**，如果你**不是Cloudflare**或希望**手动配置https**，忽视这一步。如果你的ISP支持`Let's Encrypt`，请查找相似选项并将其关闭（请注意，有些ISP并不会提供代理服务）。**等服务器正常运行后请将其打开**以使用**CDN**和其他**安全服务**并**加速访问**。

> Cloudflare新建DNS记录时**默认是橙云**，如果你已经处于代理模式，请勿修改。

5. **保存**该记录










## 5. 获得你的Cloudflare API Key

> 如果你不是Cloudflare或希望手动配置**SSL和https**，请跳过这一步。
如果你的ISP支持`Let's Encrypt`配置**SSL和https**，请**寻求其他有关CertBot的指导**。

1. 打开**Cloudflare Dashboard**
2. 右上角点击你的头像，点开，点**配置文件**
3. 左侧找**API令牌**，点开后注意下方的**API Key**，然后下面找**Global API Key**，点击它右侧的**查看**
4. 可能需要身份验证，验证完成后会显示你的API Key。请将其保存，部署Misskey时会用到。

**Global API Key权限极大，请妥善保管你的API Key，以免落入他人手中**










## 6. 如何跑动一个Misskey服务器


### 6.0 万事万物从**更新服务器组件**开始
```bash
# 注意，此命令会直接重启服务器，如希望手动重启，请删去sudo reboot
sudo apt update; sudo apt full-upgrade -y; sudo reboot
```






### 6.1官方一键安装脚本部署

一般我们使用Docker运行Misskey，因为Docker几乎不会损失性能，也不需要额外的配置，更不需要在本地编译。

推荐Misskey官方提供的一个Rootless Docker启动的脚本（本质也是辅助写Docker Compose文件并且部署依赖的一个脚本），我们可以尝试。
```bash
#注意！此命令将直接启动安装！
wget https://raw.githubusercontent.com/joinmisskey/bash-install/main/ubuntu.sh -O ubuntu.sh; sudo bash ubuntu.sh
```






#### **这个脚本做了哪些事？**

1. 询问你是**本地安装（To use systemed）**或者**Docker部署（To use docker）**

**请选择`n(To use docker)`！！！** 如果你想本地部署或本地编译，欢迎**没事找事**，本指南**不提供指导**。

接下来填入你的**服务器的IP地址**，如果你的服务器拥有ISP提供的**内网IP地址**，请**尝试使用内网IP地址**以避免意外的中间人攻击。如没有内网IP地址直接填公网IP即可。

> 实际上，脚本在这一点上写的**很烂**，因为Rootless Docker不支持使用localhost或其他方式直接访问宿主机的Redis和PostgreSQL所以脚本直接用服务器的对外IP来访问这两个东西，这是极易出错的。如果你败在这一步（Misskey的容器无法与Redis或PostgreSQL通信），请尝试后面的Docker Compose部署方案）

接下来指定**Docker Host名称**，如无冲突保持`docker_host`不变

接下来选择`Use docker hub image`并回车使用默认的`latest`版本。





2. **设置服务器初始化密码**

在部署完成后，**第一次**进入Misskey网站时，要求输入的密码，这个密码**仅会被使用一次**。
脚本会自动生成一行强密码，如果不希望使用自动生成的密码，可以删除后重新输入自己想设定的密码。

请**记下这个密码**，如丢失，将**无法完成服务器部署完成后的初始设置**。







3. **新建一个账户**并设置密码来以Rootless方式启动Docker并运行Misskey以避免root权限过大而导致的安全问题。

为了易于维护，建议将这个账户名**就设置为`misskey`**并选择一个你**记得住的密码**或设置一个密码并将其**保存在安全处**以备用。





4. **设置Misskey服务器绑定域名**

请填写你**实际使用时访问的域名**，比如`hub.captraw.com`，如果不希望使用根域名，请**不要填入你的根域名**（如`captraw.com`）。

> **再次提醒：在服务器投入联邦使用后，不要更换域名！！！**







5. **本地安装Nginx**

安装脚本会询问你是否安装Nginx，请选择`Y`。

> 如果你想以不配置Nginx反代的方式**折腾**一下，那你大可**没事找事**。






6. **开放防火墙端口**

有`ufw`和`iptables`两种方式，**建议选择iptables。**

> **注意你的Ubuntu版本** Ubuntu 24 **主动引入**了`iptables`防火墙配置，也就是此时**系统将预装`iptables`**，如果**此时选择了`ufw`**，那么 **`ufw`将与`iptables`** 冲突，此时安装程序将崩溃。

> 如果你有能力，或ISP的Dashboard支持，你可以优雅地**手动配置防火墙规则**，而不是使用这个黑箱程序。*不知道是什么的别瞎折腾容易翻车。*






7. **使用CertBot配置SSL和https**

会询问你**是否需要配置CertBot**，如果希望**自行配制SSL和https**，请选择`n`（否）；如果你之前按照流程获得了**Cloudflare Global API Key**，或者你的ISP支持`Let's Encrypt`并且你知道该如何配置，建议选择`Y`是。

会询问你**是否使用Cloudflare**，如果是Cloudflare，请选择`Y`是，如果不是，请**寻求其他关于CertBot的指导**。

填入你的**Cloudflare账号**的**邮箱**。随后填入之前取得的**Global API Key**。






8. **本地安装PostgreSQL**

> **如果你曾运行过这个安装脚本，并且之间执行的过程中发现PostgreSQL已经安装过了，请选择`n`（否）并按照之前配置时使用的用户名，密码和数据库名称进行配置**

安装过程中会要求设置**Misskey使用的PostgreSQL数据库实例的用户名**，为方便维护，建议直接使用`misskey`或者一个**不易丢失或遗忘**的用户名。接着会要求设置**数据库的密码**，同样，确保密码**安全且不会丢失或遗忘**。

接下来会设置**Misskey使用的数据库的名称**，如不冲突，直接使用使用**默认的`mk1`即可**。

> **如果你正在尝试作死在一台服务器上部署第二个Misskey实例，请在第一步时就选择`n`（否）并使用其他数据库名称。实际上，这么做并不受支持，祝贵服安康。**

> 小知识：PostgreSQL的监听端口是5432，这可能对你的故障排除有帮助








9. **本地安装Redis**

> **同样，如果你曾运行过这个安装脚本，并且之间执行的过程中发现Redis已经安装过了，请选择`n`（否）并按照之前配置时使用的密码进行配置**

同样会要求设置密码，请确保其**安全且不会丢失或遗忘**。

> 小知识：Redis的监听端口是6379，这可能对你的故障排除有帮助







10. **设置监听端口**

默认给了`3000`，如无冲突，请直接回车。





11. **本地安装Docker** 尝试docker run（运行）并顺手拉取Misskey的镜像（Image）

可能有点缓慢，如果失败了可以再来一次

**请确保服务器空间足够安装Misskey**

> 小知识：如果你的服务器在**国内**，这意味着你可能需要**一天或几天的时间**来拉取Docker Hub的Image，而这时候是**不能断开ssh**的，更别提笔电合盖了。这时候，一个叫做`screen`的程序的**帮你把安装进度挂在后台**的功能就极其重要了，这意味着即使你断开了ssh，安装进度也会继续。详细使用办法**请自行安装（`sudo apt install screen`）查询（`screen --help`）**







#### **接下来脚本会自动进行的事情**
1. **本地安装Node.js**
2. **本地安装pnpm**
3. **按照之前的配置依次本地安装PostgreSQL、Redis**
4. **安装FFmpeg**
5. **按照先前配置安装Nginx**
6. **安装git和build-essential**
7. **使用ufw配置防火墙**
8. **配置https**
9. **安装Misskey (Docker Run)**




理论上，当你看到`Now listening on port 3000`时，Misskey已经**部署完成**，你可以使用你的域名访问你的服务器。






若果失败了，请使用下面的Docker Compose的方案：











### 6.2 使用Docker Compose部署

> 此方法并非官方教程中现场编译的办法，如果你希望**现场编译**，请参考**官方教程**：
https://misskey-hub.net/cn/docs/for-admin/install/guides/docker

> 此方法需要**手动配置Nginx**。

> 此方法需要使用`CertBot`手动配置https，但是你仍然可以使用Cloudflare方式。

> 如果你之前使用了官方脚本并且前面都没有问题，直到`docker run`拉取完镜像真正运行的时候出错了，可以直接跳至 **《创建配置文件》** 处。



#### 0. 此方法搭建的出的服务器的架构：
```
宿主机：
  Nginx
Docker容器：
  redis
  postgresql
  misskey
```






#### 1. 正文开始前，万变不离其宗：
```bash
# 警告，此命令会直接重启服务器，若希望手动重启，请勿执行sudo roboot
sudo apt update
sudo apt upgrade
sudo reboot
```







#### 2. 配置防火墙

有`iptables`和`ufw`两种方式。

**请勿同时通过两种方式配置，这会导致`iptables`和`ufw`不兼容！**

由于**以`sudo`或`root`权限运行Docker**会**赋予Docker修改iptables**的权限，为避免出现*安装了`ufw`结果Docker配置了`iptables`* 的问题，**建议直接使用`iptables`方式**配置防火墙。

> 如果你是**Ubuntu 24**，请**不要使用ufw方式**，会**直接造成冲突！**
主笔没有测试其他系统有没有相似的情况！

> 另外，如果你的ISP提供了相应的安全功能（如阿里云），你需要**检查相应的安全组件**，使其也放行这两个端口，否则很容易被拦截。

> 如果你的ISP给VPS提供了**图形化**的防火墙配置界面，那么以下两种方式**都不用看了**，**优雅**地用GUI配置吧！

1. `iptables`方式
```bash
# 设置放行80和443
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 安装持久化工具
sudo apt install iptables-persistent -y

# 保存当前规则
sudo netfilter-persistent save

# 验证配置
sudo iptables -L -n -v
```

在输出中，你应该能看到类似下面两条关于端口 80 和 443 的 ACCEPT 规则。
```bash
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:80
    0     0 ACCEPT     tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:443
```

> 注意：如果你的服务器上运行着 Docker，Docker 会直接操作 iptables 规则。在配置时需要留意 Docker 的规则与自定义规则的顺序和兼容性，避免出现规则不生效的情况。*（这就体现出Rootless Docker的优越性了。）*


2. `ufw`方式

```bash
# 启用ufw
sudo ufw enable

# 设置ufw默认拒绝访问
sudo ufw default deny

# 设置有限制的ssh端口访问（22）
sudo ufw limit 22

# 设置开放80和443端口
sudo ufw allow 80
sudo ufw allow 443

# 确认ufw状态
sudo ufw status

# 永久化ufw设置
sudo systemctl enable ufw
```











#### 3. 安装Nginx

取得gpg key：
```bash
sudo apt install -y curl ca-certificates gnupg2 lsb-release ubuntu-keyring
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
```
**请确认终端是否输出`573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62`**

**若并未输出预期字符，将无法执行下面安装**

**该字符串有小概率更改，请自行前往Nginx官网确认：http://nginx.org/en/linux_packages.html#Ubuntu**

安装Nginx
```bash
echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] http://nginx.org/packages/ubuntu `lsb_release -cs` nginx" | sudo tee /etc/apt/sources.list.d/nginx.list
echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" | sudo tee /etc/apt/preferences.d/99nginx
sudo apt update
sudo apt install -y nginx

# 确认Nginx安装状态
systemctl status nginx

# 启动Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```
此时执行`curl http://localhost`将返回*Welcome to nginx!*

> 如遇到问题，请参考Nginx官方信息： http://nginx.org/en/linux_packages.html#Ubuntu









#### 4. 配置https
安装使用时Cloudflare的CertBot
> 如果你的ISP有适用的CertBot程序，请下载**适用于你的ISP的CertBot**，并**认真研究适用于你的ISP的CertBot认证方法**。如果没有，请**自行探索**。
```bash
sudo apt install -y certbot python3-certbot-dns-cloudflare

mkdir /etc/cloudflare
nano /etc/cloudflare/cloudflare.ini
```

编辑`cloudflare.ini`
```ini
dns_cloudflare_email = he0xd4c0@example.com
dns_cloudflare_api_key = xxxxxxxxxxxxxxxxxxxxxxxxxx
```
按`Ctrl+O`写入更改，按`Ctrl+X`退出`nano`。

```bash
# 使用chmod授予权限
sudo chmod 600 /etc/cloudflare/cloudflare.ini

# 将 example.ltd 更换为自己的域名！
sudo certbot certonly --dns-cloudflare --dns-cloudflare-credentials /etc/cloudflare/cloudflare.ini --dns-cloudflare-propagation-seconds 60 --server https://acme-v02.api.letsencrypt.org/directory -d example.tld -d *.example.tld
```
如果显示`*Congratulations!*`则配置完成。生成的.pem证书文件请**妥善**保存备用。（配置Nginx会用到）










#### 5. 配置Nginx
```bash
sudo nano /etc/nginx/conf.d/misskey.conf
```


粘贴：
```conf
# nginx configuration for Misskey
# 专为Misskey配置的nginx配置文件
# Created by joinmisskey/bash-install v3.3.1
# 由joinmisskey/bash-install v3.3.1创建
# 由hex-0xd4c0在2026年2月25日翻译注释

# For WebSocket
# WebSocket支持
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

proxy_cache_path /tmp/nginx_cache levels=1:2 keys_zone=cache1:16m max_size=1g inactive=720m use_temp_path=off;

server {
    listen 80;
    listen [::]:80;
    server_name example.tld;

    # For SSL domain validation
    # SSL域验证
    root /var/www/html;
    location /.well-known/acme-challenge/ { allow all; }
    location /.well-known/pki-validation/ { allow all; }

        # with https
        # 带HTTPS
    location / { return 301 https://$server_name$request_uri; }
}
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.tld;

    ssl_session_timeout 1d;
    ssl_session_cache shared:ssl_session_cache:10m;
    ssl_session_tickets off;

    # To use Let's Encrypt certificate
    # 使用Let's Encrypt证书
    ssl_certificate     /etc/letsencrypt/live/example.tld/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.tld/privkey.pem;

    # SSL protocol settings
    # SSL协议设置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_stapling on;
    ssl_stapling_verify on;
    # Change to your upload limit
    # 上传限制请修改
    client_max_body_size 250m;

    # Proxy to Node
    # 代理到Node.js
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_http_version 1.1;
        proxy_redirect off;

        # If it's behind another reverse proxy or CDN, remove the following.
        # 如果在另一个反向代理或CDN后面，请删除以下内容，包括但不限于Cloudflare小橙云代理。
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;

        # For WebSocket
        # WebSocket支持
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # Cache settings
        # 缓存设置
        proxy_cache cache1;
        proxy_cache_lock on;
        proxy_cache_use_stale updating;
                proxy_force_ranges on;
        add_header X-Cache $upstream_cache_status;
    }
}
```

请将所有的`example.tld`改成自己**实际使用的域名**。
将上文中的`ssl_certificate`和`ssl_certificate_key`后面的路径**替换为之前CertBot获得的pem证书**的路径。
如果你**打开了Cloudflare代理** *（有谁不开的？）*，请**删去上面文件中`location`下的这几行，以免影响网站访问：**
```conf
        # If it's behind another reverse proxy or CDN, remove the following.
        # 如果在另一个反向代理或CDN后面，请删除以下内容，包括但不限于Cloudflare小橙云代理。
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
```
按`Ctrl+O`保存更改，`Ctrl+X`退出`nano`。

确认配置文件是否正常。
```bash
sudo systemctl restart nginx
```
如果配置文件正常，重启nginx守护程序。
```bash
sudo systemctl restart nginx
```
确认Nginx运行状态。
```bash
sudo systemctl status nginx
```










#### 6. 安装`docker`
```bash
# 此命令适用于Debian系

# 添加Docker官方GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 添加仓库到Apt源:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
```





#### 7. 安装`Docker Compose`
```bash
sudo apt install docker-compose-plugin
```




#### 8. 新建Misskey专用账户
并非必须，但是处于**安全考虑**，建议这么做。
> 如果你想**作死**使用`root`运行然后**方便不怀好意之人**获取`misskey`权限就能直接**提权至`root`**那你大可一试。

使用`useradd`添加账户
```bash
useradd -m -U -s /bin/bash "misskey";
```
> 如果你想使用自定义的用户名，请替换`misskey`

登陆到新账户：
```bash
sudo -iu misskey
```
然后修改密码：
```bash
passwd
# 系统会要求你输入原密码，因为你没有设置过，直接留空回车

# 接着会要求你输入新密码，输入的密码不会被显示
# （不要怀疑自己没打出来，是真的不会有任何显示，连*都没有）
# 输入完成后直接回车

# 接着是确认密码，请确保两遍输入的密码相同
```

> 请确保密码**安全且不易丢失或遗忘**，你也不想在维护的时候**对着密码输入框发呆**吧。

> 如果你想让你的用户裸奔，你可以 ***逝 世*** 不设密码，**后果自负**。








#### 9. 创建配置文件

创建Docker Compose配置文件
```bash
nano docker-compose.yml
```

粘贴：
```yml
services:
  postgres:
    image: postgres:15
    restart: unless-stopped
    environment:
      POSTGRES_DB: mk1
      POSTGRES_USER: misskey
      POSTGRES_PASSWORD: <your_db_password>
    volumes:
      - ./postgres:/var/lib/postgresql/data

  redis:
    image: redis:7
    restart: unless-stopped
    command: redis-server --requirepass <your_redis_password>
    volumes:
      - ./redis:/data

  misskey:
    image: misskey/misskey:latest
    restart: unless-stopped
    depends_on:
      - postgres
      - redis
    ports:
      - "3000:3000"
    volumes:
      - ./files:/misskey/files
      - ./default.yml:/misskey/.config/default.yml:ro
```
> 请将`<your_db_password>`和`<your_redis_password>`分别替换为**安全且不易丢失或遗忘**的PostgreSQL数据库密码和Redis密码。**这很重要。**







创建Misskey默认配置文件
```bash
nano default.yml
```

粘贴：
```yml
url: https://<你的域名>
port: 3000

setupPassword: '<初始化密码>'

db:
  host: postgres
  port: 5432
  db: mk1
  user: misskey
  pass: <your_db_password>

redis:
  host: redis
  port: 6379
  pass: <your_redis_password>

id: 'aid'
proxyRemoteFiles: true
signToActivityPubGet: true
```
> 请将`<你的域名>`**替换**为你**实际使用的域名**（如`hub.captraw.com` <—这是**我的域名，别填我的！！！**）。
请将`<初始化密码>`替换为你希望设定的初始化密码，**这个密码只会使用一次，但是没有它将无法完成部署之后的初始设置**

> 请将`<your_db_password>`和`<your_redis_password>`分别改为**你刚刚设定**的PostgreSQL密码和Redis密码







#### 10. 启动Misskey
```bash
docker compose up -d
```
> 你也可以使用`sudo`运行此命令，但其实普通权限就够了。







## 7. 联合问题

Misskey服务器支持相互联合组成巨大联邦，但是在联合过程中，出现了诸多障碍，主要是各种云服务平台的安全配置导致的。

以下是**常见**问题解决方案：

### 1. 区域限制

部分国内ISP的国内节点设置**默认所有VPS拒绝所有境外访问**，实际使用时，可以开放访问权限给部分地区或指定IP。

### 2. 证书问题

如果你的证书配置是崩的，那么发给他站的GET是不带签名的，不仅ISP可能会直接把请求挡了，Misskey也不会回应。（在Misskey中有相应设置，注意：关闭强制要求签名会使**安全性下降**）

### 3. Cloudflare Bot Fight Mode 自动程序攻击模式

> 💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩💩

> 如果你的ISP也有**相似功能**，那主责大概八九不离十了。

如果你的服务器的IP段在到你手里之前**不知被何人用过**，**而且还用得很脏**，在广大老牌厂商那里留了**不知道多少案底**，那么诸如Cloudflare之类的**老牌厂商**肯定不会允许这种IP段用爬虫访问他们**保护**的网站。（Misskey从他站获得数据的行为本质上也是一种爬虫行为）**特别是当他站的安全规则开得非常严格的时候。**

经过确认，笔者确定是Cloudflare的**Bot Fight Mode 自动程序攻击模式** **非常容易** **把Misskey的请求给403了**。

为了绕开这个安全组件，在**尽量保证安全的情况下避免误伤**，你应该在`Cloudflare Dashboard`上进行以下配置：
点开你自己的域名，进`Security → Security Rules → Create rule → Customize Rule`（中文是`安全性 → 安全规则 → 创建规则 → 自定义规则`）。
直接点**编辑表达式**，输入以下内容：
```cf
(
  http.request.headers["user-agent"][contains "Misskey"]
  and http.request.headers["signature"][exists]
  and (
        http.request.uri.path contains "/users/"
        or http.request.uri.path contains "/inbox"
        or http.request.uri.path contains "/.well-known/"
        or http.request.uri.path contains "/notes/"
      )
  and http.request.headers["user-agent"][contains "example.com"]
)
```
请将`example.com`替换为**被挡**的域名。

最后由He0xD4C0更新于2026年2月25日
