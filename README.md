#!/bin/bash
# Roundcube Webmail 一键安装脚本 | Ubuntu 20.04/22.04 | Nginx + PHP8.1 + MySQL

# ===================== 请手动修改这里的配置项 =====================
DOMAIN="mail.example.com"       # 你的访问域名
DB_NAME="roundcube"
DB_USER="roundcube"
DB_PWD="DB_Passwd@123"          # 数据库密码，务必改复杂
TIME_ZONE="Asia/Shanghai"
# =================================================================

# 1. 更新系统 & 安装全套依赖
apt update -y && apt upgrade -y
apt install -y nginx mysql-server php8.1-fpm php8.1-cli \
php8.1-mysql php8.1-mbstring php8.1-xml php8.1-curl php8.1-zip \
php8.1-gd php8.1-intl composer git unzip certbot python3-certbot-nginx

# 2. 配置 PHP8.1
PHP_INI="/etc/php/8.1/fpm/php.ini"
sed -i "s/^;date.timezone =.*/date.timezone = ${TIME_ZONE}/" ${PHP_INI}
sed -i "s/^upload_max_filesize =.*/upload_max_filesize = 25M/" ${PHP_INI}
sed -i "s/^post_max_size =.*/post_max_size = 25M/" ${PHP_INI}
sed -i "s/^memory_limit =.*/memory_limit = 128M/" ${PHP_INI}

# 重启PHP-FPM
systemctl restart php8.1-fpm
systemctl enable php8.1-fpm

# 3. 创建 MySQL 数据库 & 账号
mysql -u root << EOF
CREATE DATABASE IF NOT EXISTS ${DB_NAME} DEFAULT CHARSET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS '${DB_USER}'@'localhost' IDENTIFIED BY '${DB_PWD}';
GRANT ALL PRIVILEGES ON ${DB_NAME}.* TO '${DB_USER}'@'localhost';
FLUSH PRIVILEGES;
EXIT;
EOF

# 4. 下载 Roundcube 源码
cd /var/www
rm -rf roundcubemail
git clone https://github.com/roundcube/roundcubemail.git
chown -R www-data:www-data roundcubemail/
cd roundcubemail

# 安装项目依赖
composer install --no-dev
./bin/install-jsdeps.sh

# 目录权限（关键：临时/日志目录可写，配置目录严格权限）
chmod -R 755 temp/ logs/
chmod 600 config/
chown -R www-data:www-data .

# 5. 初始化完成提示
echo "============================================="
echo "Roundcube 环境安装完成！"
echo "域名：${DOMAIN}"
echo "数据库名：${DB_NAME}"
echo "数据库用户：${DB_USER}"
echo "数据库密码：${DB_PWD}"
echo "下一步：配置 Nginx 站点 + 访问网页安装向导"
echo "============================================="
