# CineTrack — Teknik Dökümantasyon

Bu dosya projenin **mimari kararlarını**, **model yapısını** ve **önemli implementasyon detaylarını** açıklar. README.md kurulum/kullanım odaklıdır; bu dosya jüri ve geliştiriciler içindir.

---

##  Mimari

CineTrack 4 Django app'i + 1 proje config'inden oluşur:
cinetrack (proje config)
     settings.py · urls.py · wsgi.py
                   │
    ┌──────────────┼──────────────┐
    ▼              ▼              ▼
accounts        movies        tracking         api
(auth +         (TMDB +       (CRUD +          (DRF wrapper)
 follow)         search)       like + comment)
    │              │              │              │
    └──────────────┴──────────────┴──────────────┘
                    models.py
              (cross-app relations)
### App sorumlulukları

| App | İçerik | Sorumluluk |
|---|---|---|
| `accounts` | Profile, Follow | Kullanıcı kimliği ve sosyal ilişkiler |
| `movies` | Movie, Genre | Film/dizi verisi + TMDB cache |
| `tracking` | WatchEntry, Review, ReviewLike, Comment | Kullanıcı–film etkileşimleri |
| `api` | Serializer'lar, ViewSet'ler | REST API (DRF) |

Bu ayrıştırma **separation of concerns** ilkesini uygular: her app tek bir domain'e odaklanır.

---

##  Veritabanı Modelleri

### Kullanıcı tarafı
**Follow constraint'leri:**
- `UniqueConstraint(follower, following)` 
- `CheckConstraint(follower != following)` 

### Film tarafı
`Movie` TMDB'den çekilen verinin **lokal cache'i**. `fetched_at` son güncelleme zamanını tutar; 7 günden eski kayıtlar yenilenir.

### Etkileşim tarafı
---

##  Önemli Tasarım Kararları

### 1. TMDB cache stratejisi




```python
# movies/services.py
def get_or_fetch_movie(tmdb_id, media_type):
    try:
        movie = Movie.objects.get(tmdb_id=tmdb_id, media_type=media_type)
        if movie.fetched_at > timezone.now() - timedelta(days=7):
            return movie  # cache fresh
    except Movie.DoesNotExist:
        pass
    # cache miss/stale → TMDB'den çek + DB'ye yaz
    data = tmdb.get_movie_details(tmdb_id)  # ya da get_tv_details
    return save_movie_from_tmdb(data, media_type)
```



### 2. Profile signal ile otomatik yaratma


```python
# accounts/signals.py
@receiver(post_save, sender=settings.AUTH_USER_MODEL)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)
```

Bu sayede her kullanıcının Profile'ı garantilenmiş olur, view'larda `try/except` gerekmez.

### 3. Authorization stratejisi (iki katmanlı)

```python
# tracking/views.py — başkasının yorumunu silmeye çalışsan 404 yer
review = get_object_or_404(Review, id=review_id, user=request.user)
```


**API seviyesi:** custom DRF permission
```python
# api/permissions.py
class IsOwnerOrReadOnly(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
        if request.method in permissions.SAFE_METHODS:
            return True
        return obj.user == request.user
```
Herkes GET, sadece sahibi PUT/DELETE.

### 4. Query efficiency: select_related & prefetch_related



```python
# accounts/views.py — profil sayfası
watch_entries = (
    WatchEntry.objects.filter(user=user_obj)
    .select_related('movie')                # FK için JOIN
    .prefetch_related('movie__genres')      # M2M için ayrı sorgu
)
```

50 film olsa bile sadece **2-3 SQL sorgusu** atılır (N+1 yaklaşımında 50+ olurdu).

### 5. İstatistikler: DB aggregation

Ortalama puan gibi metrikler Python'da değil DB'de hesaplanır:

```python
# accounts/views.py
stats = WatchEntry.objects.filter(
    user=user_obj, status='watched',
).aggregate(
    avg_rating=Avg('rating'),
    rated_count=Count('rating'),
)
```



### 6. AJAX endpoint tasarımı

AJAX endpoint'ler standart Django view + `JsonResponse`:

```python
# tracking/views.py
@login_required
@require_POST
def toggle_review_like_view(request, review_id):
    review = get_object_or_404(Review, id=review_id)
    like, created = ReviewLike.objects.get_or_create(user=request.user, review=review)
    if not created:
        like.delete()
        liked = False
    else:
        liked = True
    return JsonResponse({'liked': liked, 'count': review.likes.count()})
```

Frontend `fetch()` ile çağırır, `X-CSRFToken` header'ı cookie'den okunur.

### 7. Template reuse

`templates/accounts/_watch_grid.html` partial template'i 3 farklı tab'da (`İzledi`, `İzliyor`, `İstek Listesi`) yeniden kullanılır:

```django
{% include 'accounts/_watch_grid.html' with entries=watched empty_msg="..." %}
```

DRY ilkesi, kod tekrarı yok.

### 8. Env-based config

Hassas bilgiler `.env`'de tutulur, `python-decouple` ile okunur:

```python
SECRET_KEY = config('SECRET_KEY')
TMDB_API_TOKEN = config('TMDB_API_TOKEN')
```

`.gitignore` `.env`'i hariç tutar — API key'ler repo'ya sızmaz.

---

##  API Referansı

Browsable API: `/api/`

### Movies
### Watch Entries (auth gerekli)
### Reviews
### Auth

Session-based. Browsable API üzerinden:
- `/api/auth/login/`
- `/api/auth/logout/`

---

##  Test Stratejisi

27 test, 4 kategori:

| Kategori | Sayı | Ne test eder? |
|---|---|---|
| Model | 6 | UniqueConstraint, CheckConstraint, signal'ler |
| View | 8 | CRUD akışları, login gerekliliği, redirect'ler |
| AJAX | 3 | Like/follow toggle, JSON response yapısı |
| API | 5 | Public list, auth, owner-only edit |
| Authorization | 5 | Başkasının kaynağını silmeye çalışınca 404 |

Çalıştırma: `python manage.py test`

---

##  Gelecek Geliştirmeler

- [ ] Activity feed 
- [ ] Custom user lists 
- [ ] Tür-tabanlı film önerme algoritması
- [ ] PostgreSQL'e geçiş
- [ ] Docker + docker-compose desteği
- [ ] GitHub Actions ile CI/CD pipeline
