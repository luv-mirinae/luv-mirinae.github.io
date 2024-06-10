---
title: AWS - Bitnami Ghost https 구성 후 preview 화면이 제대로 표시되지 않는 경우
layout: page
parent: Linux
---

# AWS - Bitnami Ghost https 구성 후 preview 화면이 제대로 표시되지 않는 경우

---

[Ghost - Amazon Lightsail](https://docs.aws.amazon.com/ko_kr/lightsail/latest/userguide/amazon-lightsail-quick-start-guide-ghost.html)

위 설명서대로 HTTPS를 구성했는데 Preview 페이지에서 화면이 제대로 표시되지 않는 문제가 발생합니다.

`/opt/bitnami/ghost/config.production.json` 파일에서 `"url": "http://my-url"` 부분을 `"url": "https://www.my-url"` 으로 수정합니다.

DNS 레코드의 CNAME 속성에서 www가 my-url을 가리키고 있어야 합니다.

```json
"url": "https://www.mirinae.blog"
  "server": {
    "port": 2368,
    "host": "0.0.0.0"
  }
```

`/opt/bitnami/apps/letsencrypt/conf/httpd-app.conf` 파일의 최상단에 `RequestHeader set X-Forwarded-Proto "https"`를 추가합니다.

```conf
# /opt/bitnami/apps/letsencrypt/conf/httpd-app.conf

RequestHeader set X-Forwarded-Proto "https"

<Directory "/opt/bitnami/apps/letsencrypt/.well-known">
    Options +MultiViews
    AllowOverride None
    <IfVersion < 2.3 >
        Order allow,deny
        Allow from all
    </IfVersion>
    <IfVersion >= 2.3>
        Require all granted
    </IfVersion>

</Directory>
```

파일 수정 후, 서비스를 재시작하면 Preview에서 컨텐츠가 정상적으로 표시됩니다.

```bash
sudo /opt/bitnami/ctlscript.sh restart
```

## 출처

- [Fixing SSL Certificate and HTTP to HTTPS Forwarding for Ghost](https://www.codymd.com/setting-up-ghost-blog-on-google-cloud-services)
