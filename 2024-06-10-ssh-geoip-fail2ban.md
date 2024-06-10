---
title: SSH에 GeoIP 및 Fail2ban 적용 (TCP Wrapper)
layout: page
parent: Linux
---

# SSH에 GeoIP 및 Fail2ban 적용 (TCP Wrappper)

---

## 주의 사항

이 문서는 TCP Wrapper(hosts.allow, hosts.deny)를 사용해 Fail2ban을 적용하는 방법에 대해 서술한 문서입니다.

Ubuntu 22.04 및 RHEL(CentOS) 8 이후 버전의 운영체제에서는 위 방법 대신, ufw(Ubuntu) 또는 firewalld(RedHat)와 연계하는 방식을 사용해야 합니다.

## GeoIP

### MaxMind GeoLite2 API 키 발급

MaxMind 사의 홈페이지에 가입 후 로그인합니다.

Account - Manage License Keys - Generate new license key 메뉴로 이동하여, License key description을 입력 후 Confirm 버튼을 클릭합니다.

[Sign In (MaxMind)](https://www.maxmind.com/en/account/login)

GeoIP.conf 파일을 다운로드할 수 있습니다. 파일의 내용은 아래와 비슷합니다.

```conf
# GeoIP.conf file for `geoipupdate` program, for versions >= 3.1.1.
# Used to update GeoIP databases from https://www.maxmind.com.
# For more information about this config file, visit the docs at
# https://dev.maxmind.com/geoip/updating-databases.

# `AccountID` is from your MaxMind account.
AccountID ######

# `LicenseKey` is from your MaxMind account
LicenseKey ######_#############################_###

# `EditionIDs` is from your MaxMind account.
EditionIDs GeoLite2-ASN GeoLite2-City GeoLite2-Country
```

### geoipupdate 패키지 설치

geoipupdate 패키지를 설치합니다.

[geoipupdate (GitHub)](https://github.com/maxmind/geoipupdate)

```bash
# Ubuntu
sudo add-apt-repository ppa:maxmind/ppa

sudo apt update
sudo apt install geoipupdate
```

```bash
# RedHat
sudo dnf install wget

wget https://github.com/maxmind/geoipupdate/releases/download/v7.0.1/geoipupdate_7.0.1_linux_amd64.rpm
sudo rpm -ivh ./geoipupdate_7.0.1_linux_amd64.rpm
```

### geoipupdate 구성

`/etc/GeoIP.conf` 파일을 다운로드한 GeoIP.conf 파일로 교체하거나,
`/etc/GeoIP.conf` 파일의 AccountID, LicenseKey, EditionIDs 항목을 다운로드한 파일의 내용으로 변경합니다.

```bash
...
# 계정 ID
AccountID YOUR_ACCOUNT_ID_HERE
# 라이선스 키
LicenseKey YOUR_LICENSE_KEY_HERE

# Enter the edition IDs of the databases you would like to update.
# Multiple edition IDs are separated by spaces.
# 사용할 제품 종류
EditionIDs GeoLite2-Country GeoLite2-City
...
```

### GeoIP Database 다운로드 및 업데이트

GeoIP 데이터베이스를 최신화합니다.

```bash
sudo geoipupdate -v
```

crontab에 등록하여 매일 데이터베이스를 최신화하도록 설정할 수 있습니다.

```bash
sudo vi /usr/local/bin/update-geoip-db.sh
```

```bash
#!/usr/bin/env bash

# /usr/local/bin/update-geoip-db.sh
geoipupdate -f /etc/GeoIP.conf -v
```

```bash
sudo crontab -e
```

```bash
# crontab (root)
# 매일 04시에 데이터베이스 업데이트 수행
0 4 * * * /usr/local/bin/update-geoip-db.sh >> /var/log/update-geoip-db.log 2>&1
```

### mmdbinspect 설치

IP 주소를 GeoIP 데이터베이스에서 검색해 위치 정보를 보여주는 명령어 패키지입니다.

사용중인 운영체제와 CPU 아키텍쳐에 맞는 패키지를 다운로드 후 설치합니다.

[mmdbinspect (GitHub)](https://github.com/maxmind/mmdbinspect)

```bash
# Ubuntu
wget https://github.com/maxmind/mmdbinspect/releases/download/v0.2.0/mmdbinspect_0.2.0_linux_amd64.deb
sudo dpkg -i ./mmdbinspect_0.2.0_linux_amd64.deb
```

```bash
# RedHat
wget https://github.com/maxmind/mmdbinspect/releases/download/v0.2.0/mmdbinspect_0.2.0_linux_amd64.rpm
sudo rpm -Uvhi mmdbinspect_0.2.0_linux_amd64.rpm
```

### mmdbinspect 명령어 확인

설치가 완료되었다면, 명령어를 실행하여 IP에 대한 위치 정보가 출력되는지 확인합니다.

```bash
# Ubuntu
sudo apt install jq
mmdbinspect -db /usr/share/GeoIP/GeoLite2-Country.mmdb 8.8.8.8 | jq
```

```bash
# RedHat
sudo dnf install jq
mmdbinspect -db /usr/share/GeoIP/GeoLite2-Country.mmdb 8.8.8.8 | jq
```

명령어 실행 결과는 대략적으로 아래와 같이 출력됩니다.

```json
[
  {
    "Database": "/usr/share/GeoIP/GeoLite2-Country.mmdb",
    "Records": [
      {
        "Network": "8.8.8.8/15",
        "Record": {
          "continent": {
            "code": "NA",
            "geoname_id": 6255149,
            "names": {
              "de": "Nordamerika",
              "en": "North America",
              "es": "Norteamérica",
              "fr": "Amérique du Nord",
              "ja": "北アメリカ",
              "pt-BR": "América do Norte",
              "ru": "Северная Америка",
              "zh-CN": "北美洲"
            }
          },
          "country": {
            "geoname_id": 6252001,
            "iso_code": "US",
            "names": {
              "de": "USA",
              "en": "United States",
              "es": "Estados Unidos",
              "fr": "États Unis",
              "ja": "アメリカ",
              "pt-BR": "EUA",
              "ru": "США",
              "zh-CN": "美国"
            }
          },
          "registered_country": {
            "geoname_id": 6252001,
            "iso_code": "US",
            "names": {
              "de": "USA",
              "en": "United States",
              "es": "Estados Unidos",
              "fr": "États Unis",
              "ja": "アメリカ",
              "pt-BR": "EUA",
              "ru": "США",
              "zh-CN": "美国"
            }
          }
        }
      }
    ],
    "Lookup": "8.8.8.8"
  }
]
```

## Log 설정

아래 스크립트는 mmdbinspect를 사용하여 접속하는 IP의 국가에 따라 ALLOW 또는 DENY를 로그 파일에 기록합니다.

```bash
#!/bin/bash

# /usr/local/bin/ssh-filter.sh

# UPPERCASE space-separated country codes to ACCEPT
ALLOW_COUNTRIES="KR JP"

if [ $# -ne 1 ]; then
        echo "Usage: `basename $0` <ip>" 1>&2
        exit 0
fi

IP=$1

# Get country ISO code
COUNTRY_CODE=`mmdbinspect -db /usr/share/GeoIP/GeoLite2-Country.mmdb $IP 2>/dev/null | jq -r '.[].Records[].Record.country.iso_code' 2>/dev/null`
COUNTRY_NAME=`mmdbinspect -db /usr/share/GeoIP/GeoLite2-Country.mmdb $IP 2>/dev/null | jq -r '.[].Records[].Record.country.names.en' 2>/dev/null`

if [ "$COUNTRY_CODE" = "null" ]; then
        COUNTRY_CODE=`mmdbinspect -db /usr/share/GeoIP/GeoLite2-Country.mmdb $IP 2>/dev/null | jq -r '.[].Records[].Record.registered_country.iso_code' 2>/dev/null`
        COUNTRY_NAME=`mmdbinspect -db /usr/share/GeoIP/GeoLite2-Country.mmdb $IP 2>/dev/null | jq -r '.[].Records[].Record.registered_country.names.en' 2>/dev/null`
fi

# Wrong input handling
if [ -z "$COUNTRY_CODE" ]; then
        echo "Error: Wrong IP address" 1>&2
        exit 0
fi

[[ "$ALLOW_COUNTRIES" =~ "$COUNTRY_CODE" ]] && RESPONSE="ALLOW" || RESPONSE="DENY"

if [ "$RESPONSE" = "ALLOW" ]; then
        exit 0
else
        #echo "$RESPONSE sshd connection from $IP ($COUNTRY_CODE, $COUNTRY_NAME)"
        logger "$RESPONSE sshd connection from $IP ($COUNTRY_CODE, $COUNTRY_NAME)"
        exit 1
fi
```

위 스크립트를 /usr/local/bin 디렉토리에 ssh-filter.sh 파일로 저장하고, 소유권과 권한을 부여합니다.

```bash
sudo vi /usr/local/bin/ssh-filter.sh
```

```bash
sudo chown root:root /usr/local/bin/ssh-filter.sh
sudo chmod 755 /usr/local/bin/ssh-filter.sh
```

/var/log/syslog 파일에 DENY로 시작하는 로그가 기록되었는지 확인합니다.

```bash
ssh-filter.sh 8.8.8.8
cat /var/log/syslog | tail -100 | grep DENY
Apr 18 09:58:40 home-server mirinae: DENY sshd connection from 8.8.8.8 (US, United States)
```

### TCP Wrapper 설정

/etc/hosts.deny 파일과 /etc/hosts.allow 파일에 sshd에 관한 항목을 추가해 sshd 서비스를 통한 접근이 반드시 ssh-filter.sh 파일을 거치도록 설정합니다.

```bash
echo "sshd: ALL" | sudo tee -a /etc/hosts.deny
echo "sshd: ALL: aclexec /usr/local/bin/ssh-filter.sh %a" | sudo tee -a /etc/hosts.allow
```

## Fail2ban

### Fail2ban 패키지 설치

Fail2ban 패키지를 설치합니다.

```bash
# Ubuntu
sudo apt install fail2ban
```

```bash
# RedHat
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo dnf --enablerepo epel install fail2ban
```

### Fail2ban jail 설정

접근 차단을 위한 jail 설정 파일을 구성합니다.

/etc/fail2ban/jail.d 디렉토리 하위에 \*.conf 파일이 있습니다. Ubuntu 환경에서는 defaults-debian.conf 파일을 수정합니다.

```conf
[DEFAULT]
# jail을 적용하지 않을 IP
ignoreip = 127.0.0.1/8 192.168.0.0/24
# findtime 시간동안 faillog를 감지
findtime = 600
# findtime 시간동안 허용할 최대 시도 횟수
maxretry = 3
# findtime 시간동안 maxretry회 시도 실패 시 접근을 차단할 시간
# -1은 영구 차단
bantime = -1

[sshd]
enabled = true
# sshd 포트 번호
#port = 22
# jail이 감지할 Log 파일
#logpath = /var/log/auth.log
```

## Fail2ban 서비스 활성화 및 재시작

Fail2ban 서비스를 재시작합니다.

```bash
sudo systemctl restart fail2ban
sudo systemctl enable fail2ban
```

## fail2ban-client 명령어

### 차단된 IP 주소 확인

```bash
sudo fail2ban-client status sshd
```

### 차단할 IP 수동 등록

```bash
sudo fail2ban-client set sshd banip xxx.xxx.xxx.xxx
```

### 차단한 특정 IP 차단 해제

```bash
sudo fail2ban-client set sshd unbanip xxx.xxx.xxx.xxx
```
