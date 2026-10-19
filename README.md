# mitori.mn

Mitori LLC (Митори ХХК) компанийн вэбсайт. Нэг хуудастай, build шаардлагагүй static HTML.

## Бүтэц

| Файл | Юу вэ |
| --- | --- |
| `index.html` | Сайт өөрөө. Монгол, англи хэлтэй (`MN / EN` товч, эсвэл `https://mitori.mn/?lang=en`) |
| `404.html` | Олдоогүй хуудас |
| `favicon.svg`, `apple-touch-icon.png` | Хөтөч, утасны дүрс |
| `og-image.png` | Facebook, Messenger дээр линк хуваалцахад гарах зураг (1200×630) |
| `robots.txt`, `sitemap.xml` | Хайлтын системд |
| `assets/mitori-logo-horizontal.svg` | Толгой хэсгийн лого |
| `assets/fonts/` | Rubik 600 (гарчиг), Golos Text 400–600 (текст). Латин, кирилл, Ө/Ү үсэгтэй. SIL Open Font License |

## Компьютер дээрээ үзэх

`index.html`-ийг шууд давхар дарж нээж болно. Гэхдээ фонтыг бүрэн зөв харахын тулд жижиг сервер ажиллуулаарай. Доорх хоёрын аль нэг нь хангалттай.

Node.js-ээр:

```
cd C:\Code\mitori
npx serve -l 8080
```

Анх удаа "Ok to proceed?" гэж асуувал `y` гэж бичээд Enter дарна.

Python-оор:

```
cd C:\Code\mitori
python -m http.server 8080
```

Дараа нь хөтөч дээр `http://localhost:8080` нээнэ. Англи хувилбар: `http://localhost:8080/?lang=en`.

Серверийг зогсоохдоо terminal дээр `Ctrl + C` дарна.

## Серверт тавих (nginx, ozzeel.mn-тэй нэг сервер)

Сервер дээр ажиллах Claude сессэд зориулсан бүрэн заавар: [`DEPLOY-HANDOFF.md`](DEPLOY-HANDOFF.md). Доорх нь товч хувилбар.

1. **DNS:** домэйн бүртгэгч дээр `mitori.mn` болон `www.mitori.mn`-ийн A record-ыг серверийн IP хаяг руу заана.
2. **Файлуудыг серверт татах:**
   ```
   sudo git clone https://github.com/fastroc/mitori.git /var/www/mitori.mn
   ```
3. **nginx тохиргоо** `/etc/nginx/sites-available/mitori.mn`:
   ```
   server {
       listen 80;
       server_name mitori.mn www.mitori.mn;
       root /var/www/mitori.mn;
       index index.html;
       error_page 404 /404.html;

       location / { try_files $uri $uri/ =404; }
       location /assets/fonts/ { add_header Cache-Control "public, max-age=31536000, immutable"; }
       location ~ /\.            { deny all; }
       location ~* \.md$          { deny all; }
   }
   ```
4. **Идэвхжүүлэх:**
   ```
   sudo ln -s /etc/nginx/sites-available/mitori.mn /etc/nginx/sites-enabled/
   sudo nginx -t && sudo systemctl reload nginx
   ```
5. **HTTPS:** ozzeel.mn дээр ашигласан аргаараа сертификат авна. Жишээ нь certbot:
   ```
   sudo certbot --nginx -d mitori.mn -d www.mitori.mn
   ```
6. **Дараа нь шинэчлэх:**
   ```
   cd /var/www/mitori.mn && sudo git pull
   ```

## Компанийн мэдээлэл (D&B, Apple, Google-тэй яг ижил байх ёстой)

- Legal name: Mitori LLC (Митори ХХК)
- Registration number: 6601049
- Address: Apartment 95, Building 4, 13th Microdistrict, Khoroo 15, Bayanzurkh District, Ulaanbaatar, Mongolia
- Phone: +976 9909 1810
- Email: info@mitori.mn
