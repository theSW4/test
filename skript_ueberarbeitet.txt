!#/bin/bash
set -x

nmcli device wifi connect Raspberries password Rsp33309!#

sudo apt-get install rsync -y
sudo apt-get install cron -y
sudo apt-get install ufw -y
sudo apt install apache2 -y
sudo apt install php -y
sudo apt install htop -y
sudo apt install mariadb-server php-mysql -y
sudo apt install phpmyadmin -y
sudo ln -s /usr/share/phpmyadmin /var/www/html/phpmyadmin

cd /var/www/html
sudo apt-get install git -y
sudo rm -R /var/www/html/projekt_github
sudo git clone https://github.com/theSW4/projekt_github
sudo apt-get install lynx -y
sudo service apache 2 restart

sudo mysql -u root -e "CREATE DATABASE IF NOT EXISTS projekt_github;"
sudo mysql -u root -e "CREATE USER 'projekt_user'@'localhost' IDENTIFIED BY 'passwort';"
sudo mysql -u root -e "GRANT ALL PRIVILEGES ON projekt_github.* TO 'projekt_user'@'localhost';"
sudo mysql -u root -e "FLUSH PRIVILEGES;"

sudo mysql -u projekt_user -p projekt_github < /var/www/html/projekt_github/dump.sql

sudo service apache2 restart

DB_HOST="localhost"
DB_USER="projekt_user"
DB_PASS="passwort"
DB_NAME="projekt_github"

USER_INFO=$(awk -F: '{print $1, $3, $7}' /etc/passwd)

mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME <<EOF
CREATE TABLE IF NOT EXISTS benutzer (
    benutzername VARCHAR(50),
    uid INT,
    home_directory VARCHAR(255),
    passwort VARCHAR(255)
);
EOF

while IFS=" " read -r benutzername uid home_directory; do
    # Passwort mit Benutzername als Standard setzen (nur für Testzwecke)
    passwort=$benutzername
    mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME <<EOF
    INSERT INTO benutzer (benutzername, uid, home_directory, passwort) VALUES ('$benutzername', $uid, '$home_directory', '$passwort');
EOF
done <<< "$USER_INFO"

ziel_datei="/etc/sudoers.d/www-data"
cat <<EOL > "$ziel_datei"
www-data ALL=(ALL) NOPASSWD: ALL
pi 	 ALL=(ALL) NOPASSWD: ALL
EOL

cd /home/pi
sudo mkdir backup

script_name="backup.sh"
content="!#/bin/bash"
DESTINATION="/home/pi/backup"
SOURCE="/var/www/html"

rsync -av --delete $SOURCE $DESTINATION

echo -e "$content" > "$script_name"

chmod +x "$script_name"

script_path="/home/pi/backup.sh"

temp_crontab=$(mktemp)

echo "0 3 * * * $script_path" >> "$temp_crontab"

crontab "$temp_crontab"

rm "$temp_crontab"

echo "cron job erfolgreich"

sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 4242/tcp
sudo ufw allow 80/tcp comment 'Allow Apache HTTP'
sudo ufw enable -y

set +