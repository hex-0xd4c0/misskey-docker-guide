# Misskey部署完全指南





## 1. Misskey是什么
它是一个由Misskey团队开发私人搭建的服务器软件，站与站之间的联合可以建立一个极大的私人社交网络。由于不流行，所以没有专门针对该平台的爬虫，所有数据也都掌握在站长手中。但是需要自行搭建。






## 2. 需要什么
1. **一个2C4G轻量级应用服务器** 用来跑Misskey

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
> 另外频繁读写将导致NAS硬盘寿命锐减，所以不建议将NAS作为存储桶使用






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

4. **关闭Cloudflare代理**，让**橙云**变成**灰云**

（方便使用`Let's Encrypt`自动配置**SSL**和**https**，如果你**不是Cloudflare**或希望**手动配置https**，忽视这一步。如果你的ISP支持`Let's Encrypt`，请查找相似选项并将其关闭（请注意，有些ISP并不会提供代理服务）。**等服务器正常运行后请将其打开**以使用**CD**N和其他**安全服务**并**加速访问**。）

5. **保存**该记录










## 5. 获得你的Cloudflare API Key

> 如果你不是Cloudflare或希望手动配置**SSL和https**，请跳过这一步。
> 如果你的ISP支持`Let's Encrypt`配置**SSL和https**，请**寻求其他有关Cerbot的指导**。

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

接下来选择`Use docker hub image`并回车使用默认的`latest`版本。

2. 执行`adduser`**新建一个账户**并设置密码来以Rootless方式启动Docker并运行Misskey以避免root权限过大而导致的安全问题。

为了易于维护，建议将这个账户名**就设置为`misskey`**并选择一个你**记得住的密码**或设置一个密码并将其**保存在安全处**以备用。

3. **设置Misskey服务器绑定域名**

请填写你**实际使用时访问的域名**，比如`hub.captraw.com`，如果不希望使用根域名，请**不要填入你的根域名**（如`captraw.com`）。

4. **本地安装Nginx**

安装脚本会询问你是否安装Nginx，请选择`Y`。

> 如果你想以不配置Nginx反代的方式**折腾**一下，那你大可**没事找事**。

5. **开放防火墙端口**

有`ufw`和`iptables`两种方式，**建议选择iptables。**

> **注意你的Ubuntu版本** Ubuntu 24 **主动引入**了`iptables`防火墙配置，也就是此时**系统将预装`iptables`**，如果**此时选择了`ufw`**，那么 **`ufw`将与`iptables`** 冲突，此时安装程序将崩溃。

> 如果你有能力，或ISP的Dashboard支持，你可以优雅地**手动配置防火墙规则**，而不是使用这个黑箱程序。*不知道是什么的别瞎折腾容易翻车。*

6. **使用Cerbot配置SSL和https**

会询问你**是否需要配置Cerbot**，如果希望**自行配制SSL和https**，请选择`n`（否）；如果你之前按照流程获得了**Cloudflare Global API Key**，或者你的ISP支持`Let's Encrypt`并且你知道该如何配置，建议选择`Y`是。

会询问你**是否使用Cloudflare**，如果是Cloudflare，请选择`Y`是，如果不是，请**寻求其他关于Cerbot的指导**。

填入你的**Cloudflare账号**的**邮箱**。随后填入之前取得的**Global API Key**。

7. **本地安装PostgreSQL**

**如果你曾运行过这个安装脚本，并且之间执行的过程中发现PostgreSQL已经安装过了，请选择`n`（否）并按照之前配置时使用的用户名，密码和数据库名称进行配置**

安装过程中会要求设置**Misskey使用的PostgreSQL数据库实例的用户名**，为方便维护，建议直接使用`misskey`或者一个**不易丢失或遗忘**的用户名。接着会要求设置**数据库的密码**，同样，确保密码**安全且不会丢失或遗忘**。

接下来会设置**Misskey使用的数据库的名称**，如不冲突，直接使用使用**默认的`mk1`即可**。

**如果你正在尝试作死在一台服务器上部署第二个Misskey实例，请在第一步时就选择`n`（否）并使用其他数据库名称。实际上，这么做并不受支持，祝贵服安康。**

```tips
小知识：PostgreSQL的监听端口是5432，这可能对你的故障排除有帮助
```

4. **本地安装Redis**

**同样，如果你曾运行过这个安装脚本，并且之间执行的过程中发现Redis已经安装过了，请选择`n`（否）并按照之前配置时使用的密码进行配置**

同样会要求设置密码，请确保其安全且不会丢失或遗忘。

```tips
小知识：Redis的监听端口是6379，这可能对你的故障排除有帮助
```

5. **设置监听端口**

默认给了`3000`，如无冲突，请直接回车。

6. **本地安装Docker** 尝试docker run（运行）并顺手拉取Misskey的镜像（Image）
```
可能有点缓慢，如果失败了可以再来一次
```
**请确保服务器空间足够安装Misskey**

*技巧：如果你的服务器在**国内**，这意味着你可能需要**一天或几天的时间**来拉取Docker Hub的Image，而这时候是**不能断开ssh**的，更别提笔电合盖了。这时候，一个叫做`screen`的程序的**帮你把安装进度挂在后台**的功能就极其重要了，这意味着即使你断开了ssh，安装进度也会继续。详细使用办法**请自行安装（`sudo apt install screen`）查询（`screen --help`）***

#### **接下来脚本会自动进行的事情**

7. **本地安装Node.js**
```bash
sudo rm /usr/share/keyrings/nodesource.gpg;
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/nodesource.gpg;
NODE_MAJOR=22; echo "deb [signed-by=/usr/share/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list;
sudo apt update;
sudo apt install -y nodejs;

node -v
```

8. **本地安装pnpm**
```bash
npm i -g pnpm
```

9. **按照之前的配置依次本地安装PostgreSQL、Redis**

安装PostgreSQL

```bash
sudo apt install -y postgresql-common
sudo sh /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh -i -v 15;

# 使用systemctl确认psql状态
systemctl status postgresql

# 运行psql并进行配置
sudo -u postgres psql
```

*设置用户名（ROLE）和密码（PASSWORD），**如手动部署请将`misskey`替换为希望设置的用户名，`hoge`替换为希望设置的密码**（建议保持`misskey`不动并设置一个安全且不易丢失、遗忘的密码）*

```SQL
CREATE ROLE misskey LOGIN PASSWORD 'hoge';
CREATE DATABASE mk1 OWNER misskey;
\q
```

安装Redis
*Redis相关链接：https://redis.io/docs/getting-started/installation/install-redis-on-linux/*

```bash
sudo apt-get install lsb-release curl gpg
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
sudo chmod 644 /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
sudo apt-get update
sudo apt-get install redis

# 启动Redis
sudo systemctl enable redis-server
sudo systemctl start redis-server

# 确认Redis状态
systemctl status redis-server
```

10. **安装FFmpeg**

```bash
sudo apt install ffmpeg
```

11. **按照先前配置安装Nginx**

*Nginx官方信息： http://nginx.org/en/linux_packages.html#Ubuntu*

取得gpg key：
```bash
sudo apt install -y curl ca-certificates gnupg2 lsb-release ubuntu-keyring
curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
gpg --dry-run --quiet --no-keyring --import --import-options import-show /usr/share/keyrings/nginx-archive-keyring.gpg
```
***手动部署时，请确认终端是否输出573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62***
**若并未输出预期字符，将无法执行下面安装**

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

12. **安装git和build-essential**

```bash
sudo apt update
sudo apt install -y git build-essential
```

13. **使用ufw配置防火墙** (如使用`iptables`请自行查询设置方法)

```bash
sudo ufw enable
sudo ufw default deny
sudo ufw limit 22
sudo ufw allow 80
sudo ufw allow 443

# 确认ufw配置
sudo ufw status

# 使用systemctl永久化设置
sudo systemctl enable ufw
```

14. **配置https**

```bash
sudo apt install -y certbot python3-certbot-dns-cloudflare

mkdir /etc/cloudflare
nano /etc/cloudflare/cloudflare.ini
```

在nano中编辑cloudflare.ini
```ini
dns_cloudflare_email = he0xd4c0@example.com
dns_cloudflare_api_key = xxxxxxxxxxxxxxxxxxxxxxxxxx
```
*按Ctrl+O写入更改，按Ctrl+X退出nano*

```bash
# 使用chmod授予权限
sudo chmod 600 /etc/cloudflare/cloudflare.ini

# 将 example.ltd 更换为自己的域名！
sudo certbot certonly --dns-cloudflare --dns-cloudflare-credentials /etc/cloudflare/cloudflare.ini --dns-cloudflare-propagation-seconds 60 --server https://acme-v02.api.letsencrypt.org/directory -d example.tld -d *.example.tld
```

15. **安装Misskey**

```bash
sudo su - misskey
```
*我自己其实常用`sudo -iu misskey`*

```bash
# 克隆misskey源代码
git clone -b master https://github.com/misskey-dev/misskey.git --recurse-submodules
cd misskey
git checkout master

# 安装依赖
NODE_ENV=production pnpm install --frozen-lockfile
```

```bash
# 设置misskey
nano .config/default.yml
```

```yml
# Misskey服务器公开时使用的域名，切记投入联邦使用后不要更改
url: https://example.tld/
# 监听端口默认3000，建议不要修改
port: 3000

# PostgreSQL设定，需要注意
db:
  host: localhost
  port: 5432
  db  : mk1 # Misskey实例使用的PostgreSQL数据库名
  user: misskey # PostgreSQL用户名，你之前没改过就是Misskey
  pass: hoge # PostgreSQL密码，此处修改为自己的密码！

# Redis设定，需要注意
redis:
  host: localhost
  port: 6379
  pass: "hoge" # Redis的密码，如果你之前有设置，此处请填入，否则删去这一行！！！

# ID类型设定
id: 'aidx'

# syslog
syslog:
  host: localhost
  port: 514
```
*请注意修改上文中的域名`example.com`，PostgreSQL设定和Redis的密码*

> Tip: 开发环境时，url指定为url: http://localhost:3000

若果失败了，请使用下面的Docker Compose的方案：












### Cloudflare 防火墙规则写法

Security → WAF → Custom Rules → Create rule

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
