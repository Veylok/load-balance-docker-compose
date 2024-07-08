# load-balance-docker-compose
#!/bin/bash

# Yeni kullanıcı ekleme
adduser ekxlocal
usermod -aG sudo ekxlocal

# Hostname ayarlama
export USERNAME="deneme"
hostnamectl set-hostname $USERNAME

# Gerekli paketlerin kurulumu
apt update
apt install -y sssd-ad sssd-tools realmd adcli
apt install -y clamav

# AD keşfi ve katılımı
realm -v discover veysel.com
realm join veysel.com -U Administrator

# SSSD yapılandırması
cat > /etc/sssd/sssd.conf << EOF
[sssd]
domains = veysel.com
config_file_version = 2
services = nss, pam

[domain/veysel.com]
default_shell = /bin/bash
krb5_store_password_if_offline = True
cache_credentials = True
krb5_realm = VEYSEL.COM
realmd_tags = manages-system joined-with-adcli 
id_provider = ad
fallback_homedir = /home/%u
ad_domain = veysel.com
use_fully_qualified_names = False
ldap_id_mapping = True
access_provider = ad
ad_hostname = $USERNAME.veysel.com
EOF

# Sudoers yapılandırması
cat > /etc/sudoers.d/ad << EOF
%sudoers ALL=(ALL) ALL
EOF

# Polkit yapılandırması
sed -i -e '/^AdminIdentities/ s/\(.*\)/\1;unix-group:sudoers/' /etc/polkit-1/localauthority.conf.d/51-ubuntu-admin.conf

# Kaspersky Antivirüs kaldırma
apt remove -y kesl kesl-gui

# SSSD servisini yeniden başlatma
systemctl restart sssd.service

# Ev dizini taşıma ve izin ayarları
chown -Rh "deneme:domain users" /home/deneme

# PAM modüllerini güncelleme
sudo pam-auth-update --enable mkhomedir
