<div dir=rtl>

# نصب و کانفیگ nginx

---

## چرا nginx ؟

در حال حاظر بهترین راه مخفی ماندن ترافیک فیلترشکن، شبیه کردن ترافیک به `http` و `https` می‌باشد؛ همچنین پروتکل `websocket` و فریم ورک `grpc` اتصال سریعتر و پایدارتری در شرایط فعلی فیلترینگ دارند، بنابراین ما در این آموزش فقط این دو مورد را بررسی میکنیم و توصیه میکنیم که حتما از این دو مورد همراه با رمزنگاری `TLS` استفاده کنید.
استفاده از nginx مزایای زیر را برای ما دارد:

- میتوان با استفاده از قابلیت `reverse proxy` از روی یک پورت سرویسهای مختلفی راه اندازی کرد.

  - راه اندازی trojan و vless برای یوزرهای مختلف روی یک پورت
  - راه اندازی v2ray و shadowsocks روی یک پورت

- میتوان همزمان از چند دامنه برای یک سرویس استفاده کرد و دیگر نیاز نیست برای دامنه جدید سرویس جدیدی راه اندازی کنیم. مثلا همزمان میتوانیم هم از سرویس cdn دوسایت کلاودفلر و آروان استفاده کنیم.
- دریافت گواهی SSL با استفاده از `webroot` که دیگر نیاز نیست پورت 80 سرور باز باشد

---

## نصب nginx و دریافت گواهی SSL

برای نصب ابتدا با اجرای دستور زیر در ترمینال، وارد پوشه nginx میشویم:

```bash
cd /opt/freedom/nginx
```

قبل از راه اندازی لازم است تغییراتی را در فایل `nginx.conf` انجام دهیم بنابراین با دستور زیر فایل مورد نظر را باز میکنیم.

```bash
nano nginx.conf
```

در اینجا فرض میکنیم دو دامنه داریم. `sub.test.com` برای کلاودفلر و `sub.test1.com` برای آروان. دامنه های فوق را در خطوط زیر با دامنه های خود جایگزین کنید.

```nginx
server_name sub.test.com sub.test2.com;
```

```nginx
proxy_pass http://sub.test.com;
```

- توجه داشته باشید که `server_name`، در دو جا وجود دارد و باید هر دو را ویرایش کنید.
- درصورتی شما فقط از یک دامنه استفاده میکنید، دامنه `sub.test2.com` را حذف کنید.
- طبق آموزش [کلاودفلر](../cloudflare/README.md) شما دو رکورد dns برای هر دامنه دارید که یکی بدون پروکسی هست و دیگری با پروکسی، دقت کنید که در اینجا دامنه ای را که بدون پروکسی هست وارد میکنید.
- با انجام این تنظیمات، درصورتی که دامین شما در مرورگر بصورت مستقیم وارد شود، ارور 502 برگردانده خواهد شد. این ارور نشان میدهد روی دامین شما وبسایتی راه اندازی شده که مشکل سرور داخلی دارد و کمک میکند به دیرتر فیلتر شدن دامنه شما!!!

در این آموزش ما به طور پیشفرض برای پورت SSL از دو پورت 443 و 2083 استفاده میکنیم. این باعث میشود درصورتی که پورت 443 دارای اختلال بود، شما بدون انجام تغییرات در تنظیمات سرور و یا تنظیمات v2ray، تنها با تغییر پورت 443 به پورت 2083 در لینک یوزر خود، بتوانید به فیلترشکن متصل شوید. به عبارت دیگر همزمان دو پورت فعال خواهید داشت. اگر میخواهید پورت 2083 را تغییر داده و یا حذف کنید. خطوط زیر را ویرایش کنید.

```nginx
listen 2083 ssl http2 so_keepalive=on;
listen [::]:2083 ssl http2 ipv6only=on;
```

به نکات زیر توجه کنید:

- پورت 443 را تغییر ندهید
- اگر قصد دارید که از CDN استفاده کنید، باید پورتی که جایگزین پورت 2083 میکنید، یکی از پورت های موجود در لیست https آموزش [کلاودفلر](../cloudflare/README.md) باشد.

به ترتیب با زدن دکمه‌های `ctrl + x` و `y` و `enter` تغییرات را ذخیره کرده و فایل را ببندید.

### دریافت گواهی SSL

برای دریافت گواهی SSL پیشفرض به ترتیب دستورات زیر را اجرا کنید

```bash
mkdir /opt/cert
```

```bash
openssl dhparam -out /opt/cert/dhparam.pem 2048
```

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /opt/cert/default.key -out /opt/cert/default.crt
```

از آنجایی که در این آموزش ما از دو دامنه استفاده میکنیم، میخواهیم به جای دریافت دو گواهی SSL برای هر دامنه، فقط یک گواهی از نوع `SAN (Subject Alternative Name) certificate` دریافت کنیم. در این صورت برای هر یوزر کافی است در لینک یوزر، با تغییر آدرس لینک، CDN مورد استفاده را تغییر دهیم.

برای دریافت گواهی SSL باید به طور موقت قسمت های مربوط به SSL را در فایل کانفیگ غیرفعال کنیم. دستور زیر را در ترمینال اجرا کنید.

```bash
sed -i -r 's/(listen .*\d+)/\1; #/g; s/(ssl_(certificate|certificate_key|trusted_certificate|dhparam) )/#;#\1/g; s/(server \{)/\1\n    ssl off;/g' /opt/freedom/nginx/nginx.conf
```

- توجه داشته باشید در صورتی که دستور فوق با موفقیت اجرا شود، شما هیچ پاسخ و خروجی دریافت نخواهید کرد !!!

با دستور زیر پورت های 80 و 443 و 2083 را در فایروال باز میکنیم.

- اگر پورت 2083 را در مرحله قبل تغییر داده اید، در اینجا 2083 را با پورت تغییر یافته جایگزین کنید

```bash
ufw allow 80 && ufw allow 443 && ufw allow 2083
```

- اگر پورت 2083 را در مرحله قبل حذف کرده اید، در اینجا هم نیاز به باز کردن آن نبوده و دستور مربوط به آن را اجرا نکنید.

```bash
ufw allow 80 && ufw allow 443
```

با دستور زیر سرور را اجرا میکنیم

```bash
docker-compose up -d
```

با دستورات زیر گواهی SSl را برای دامنه های خود دریافت میکنیم.

- ما در اینجا به جای دریافت گواهی `SSL` از نوع `RSA`، نوع ٍ`ECC` را دریافت میکنیم که امنیت و بازدهی بیشتری دارد. برای توضیحات بیشتر [بخش دریافت گواهی SSL](../get-ssl/README.md) را مطالعه کنید.
- دامنه های `sub.test.com` و `sub.test1.com` را با دامنه های خود جایگزین کنید.

```bash
acme.sh --issue --keylength ec-256 -d sub.test.com -d sub.test1.com -w /opt/freedom/nginx/webroot/
```

- در صورتی که از یک دامنه استفاده میکنید، به جای دستور بالا دستور زیر را اجرا کنید.

```bash
acme.sh --issue --keylength ec-256 -d sub.test.com -w /opt/freedom/nginx/webroot/
```

با دستور زیر گواهی های دریافت شده را در پوشه پیشفرض نصب میکنیم تا در صورت نیاز در سرویسهای دیگر هم از آن استفاده کنیم.

- در دستور زیر دامنه `sub.test.com` را با دامنه خود جایگزین کنید.
- اگر از دو دامنه استفاده میکنید توجه داشته باشید باید اولین دامنه از سمت چپ در دستور قبلی را جایگزین کنید که در اینجا `sub.test.com` هست.

```bash
acme.sh --installcert --ecc -d sub.test.com --key-file /opt/cert/private.key --cert-file /opt/cert/cert.crt --fullchain-file /opt/cert/fullchain.crt --ca-file /opt/cert/ca.crt
```

حال با دستور زیر به طور موقت سرور را غیرفعال میکنیم

```bash
docker-compose down
```

با دستور زیر قسمت های مربوط به SSL را در فایل `nginx.conf` که در مراحل قبل غیرفعال کرده بودیم، فعال میکنیم

```bash
sed -i -r -z 's/#?; ?#//g; s/(server \{)\n    ssl off;/\1/g' /opt/freedom/nginx/nginx.conf
```

با دستور زیر سرور را مجددا راه اندازی میکنیم

```bash
docker-compose up -d
```

سرور nginx ما با موفقیت راه اندازی شده و آماده استفاده هست.