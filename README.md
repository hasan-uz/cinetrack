# 🎬 CineTrack

Film ve dizi takip platformu — Letterboxd tarzı bir sosyal film uygulaması.

Kullanıcılar TMDB üzerinden film/dizi arar, izleme listelerine ekler, puanlar, yorum yazar, başkalarının yorumlarını beğenir ve birbirini takip eder.

🌐 **Canlı Demo:** [https://cinetrack-7gqr.onrender.com](https://cinetrack-7gqr.onrender.com)


 `/admin/` paneline superuser ile girebilirsiniz.

---

##  Özellikler

- **Kullanıcı sistemi:** kayıt, giriş, çıkış, profil düzenleme
-  **TMDB arama:** film + dizi araması (pagination dahil)
-  **Watchlist:** 3 durum (izledim / izliyorum / izlemek istiyorum)
-  **Puanlama:** 1-10 arası
-  **Yorum sistemi:** yazma, düzenleme, silme
-  **AJAX beğeni:** sayfa yenilemeden yorum beğen
-  **AJAX takip:** kullanıcı takip et / takipten çık
-  **Profil istatistikleri:** izlenen sayısı, ortalama puan, tab'lı liste
-  **REST API:** Django REST Framework ile JSON endpoint'ler
- 📱 **Responsive UI:** Bootstrap 5 dark cinema teması

##  Teknoloji Stack'i

| Katman | Teknoloji |
|---|---|
| Backend | Django 5.x, Django REST Framework |
| Frontend | Bootstrap 5, Bootstrap Icons, Vanilla JS |
| Veritabanı | SQLite (dev), PostgreSQL (prod) |
| Third-party API | [TMDB](https://www.themoviedb.org/) |
| Config | python-decouple (env vars) |
| Python | 3.13+ |

##  Kurulum

### Gereksinimler

- Python 3.13+
- pip + venv
- Ücretsiz [TMDB API token'ı](https://www.themoviedb.org/settings/api) (v4 Bearer)

### Adımlar

```bash
# 1. Repo'yu klonla
git clone https://github.com/hasan-uz/cinetrack.git
cd cinetrack

# 2. Virtual environment
python3 -m venv venv
source venv/bin/activate    # Mac/Linux
# venv\Scripts\activate     # Windows

# 3. Bağımlılıklar
pip install -r requirements.txt

# 4. .env dosyasını oluştur (örnek için .env.example'a bak)
cp .env.example .env
# Sonra .env'i editör'de açıp gerçek değerleri gir

# 5. Veritabanı migration
python manage.py migrate

# 6. TMDB türlerini yükle
python manage.py seed_genres

# 7. Süperkullanıcı oluştur
python manage.py createsuperuser

# 8. Sunucuyu başlat
python manage.py runserver
```

Tarayıcıda `http://127.0.0.1:8000/` aç.

##  Yapılandırma

`.env` dosyası proje root'unda olmalı:

```env
SECRET_KEY=django-secret-key-buraya
DEBUG=True
TMDB_API_TOKEN=tmdb-bearer-token-buraya
```

- **SECRET_KEY**: `python -c "import secrets; print(secrets.token_urlsafe(50))"` ile üret
- **DEBUG**: dev'de True, prod'da False
- **TMDB_API_TOKEN**: TMDB → Settings → API → "API Read Access Token" (uzun JWT)

##  Test

```bash
python manage.py test
```

27 unit + integration test. Model constraint'leri, view'lar, AJAX endpoint'leri, DRF API ve authorization kontrollerini kapsıyor.

## API

DRF Browsable API: `http://127.0.0.1:8000/api/`

Hızlı referans:

| Endpoint | Method | Açıklama |
|---|---|---|
| `/api/movies/` | GET | Film/dizi listesi (search, ordering, pagination) |
| `/api/movies/{id}/` | GET | Tek film detayı |
| `/api/watch-entries/` | GET/POST | Watch listesi CRUD (auth) |
| `/api/reviews/` | GET/POST | Yorumlar (read public, write auth) |

Detaylı dökümantasyon için **[TECHNICAL.md](TECHNICAL.md)** dosyasına bak.

## 📁 Proje Yapısı

```
cinetrack/
├── cinetrack/          # Proje ayarları
│   ├── settings.py · urls.py
│   ├── wsgi.py · asgi.py
│   └── templates/
├── accounts/           # Kullanıcı sistemi: kayıt, giriş, profil, takip
│   ├── models.py · forms.py · views.py · urls.py
│   └── signals.py
├── movies/             # TMDB entegrasyonu, film ve dizi arama
│   ├── tmdb.py · services.py
│   ├── models.py · views.py · urls.py
│   └── management/     # Özel management komutları
├── tracking/           # Watchlist, puanlama, yorum, beğeni
│   └── models.py · forms.py · views.py · urls.py
├── api/                # Django REST Framework katmanı
│   ├── serializers.py · permissions.py
│   └── views.py · urls.py
├── templates/          # Ortak şablonlar (accounts, movies, registration)
├── manage.py
├── requirements.txt
├── build.sh
└── .env.example
```
