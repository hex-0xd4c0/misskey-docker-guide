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