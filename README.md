# System Design - Cоциальная сеть для путешественников

## Функциональные требования:
- web/mob ui
- загрузка фотографий
- создание/обновление постов (фотографии, текст, геометка)
- поставить оценку поста (лайк)
- оставить коментарий под постом
- подписка/отписка на пользователей
- получение списка мест по популярности
- получение списка постов по месту
- получение списка постов других пользователей в обратном хронологическом порядке, основанной на подписке (фид)
- получение списка постов одного пользователя в обратном хронологическом порядке

## Нефункциональные требования:
- DAU ~ 10М
- Активность пользователей: 
    1. В среднем пользователь просматривает просматривает ленту (постов + мест + комментарии - batch по 50) ~ 20 запросов в день
    2. В среднем пользователь просматривает просматривает ленту (фото) 20 * 50 * ~5 = 500 запросов в день
    3. В среднем пользователь оценивает посты ~ 10 запросов в день
    4. В среднем пользователь пишет комментарий 1 раз в день ~ 0.5 запрос в день 
    5. В среднем пользователь создает/обновляет посты несколько раз в неделю ~ 0.1 запрос в день
    6. В среднем пользователь лайкает 3 раз в день ~ 3 запроса в день
    7. остальную активность не учитываю (редкие) 
- СНГ аудитория
- Сезонность:
    1. годовая - сезоны отпусков и праздников (пики лето/нг, минимум осень/весна)
    2. недельная - пик выходные
    3. суточная - пик вечер
- Данные храним всегда
- Лимит: 
    1. 1000000 подписчиков
    2. 10 фото в посте
    3. 10000 символов описание поста
- SLO: 
    1. доступность 99.9% - 43 минуты простоя в месяц 
    2. Фиды/списки: P95 ≤ 1 с, P99 ≤ 3 с
    3. write-операции (лайк/коммент): P95 ≤ 300 мс, P99 ≤ 700 мс

## API

```
GET /places
GET /places/popular
GET /posts
GET /posts/following
POST /likes
POST /comments
POST /posts
POST /uploads/image
```

## Entities

```
User
  id (uuid)
  username (uniq), name, avatar_url
  created_at, status
  Follow (подписка, ориентируемся на Directed Graph)
  PK: (follower_id, followee_id) без surrogate id
  created_at

Post
  id (uuid)
  author_id (fk → users)
  description (text, до 10k; часто короткая)
  created_at, updated_at


PostImage
  id (uuid)
  post_id (fk)
  storage_key
  position (smallint)
  meta: width, height, content_type, size_bytes, хэш


Place
  id (uuid)
  name (varchar)
  geom (Point, PostGIS), country_code, admin_area

Like
  PK: (post_id, user_id)
  created_at

Comment
  id (uuid)
  post_id, user_id
  body (text; обычно короткая)
  created_at, updated_at
  parent_id (nullable) — если нужны треды 1 уровня

```

## Оценка нагрузки:

```
Post (одна строка)
  uuid (id): 16 B
  author_id: 16 B
  description: среднее 200 символов = ~200 B
  likes_count, comments_count: 4+4 = 8 B
  created_at, updated_at: 8+8 = 16 B
  
  Итого: 256B

PostImage
  id: 16 B
  post_id: 16 B
  storage_key: средний 80–120 B (возьмём 100 B)
  position: 2 B (выровняется до 4–8)
  width,height: 4+4=8 B
  size_bytes: 8 B
  
  Итого: 150B

Place
  id: 16 B
  name: 40 B
  geom (geography(Point)): ~32 B
  country_code: 2 B
  admin_area: среднее 20–40 B (30 B)
  счётчики + popularity: 4+4+8 = 16 B
  
  Итого: 136B

PostPlace
  post_id + place_id: 16+16 = 32 B
  weight: 2 B
  
  Итого: 34B

Like (без surrogate id)
  post_id + user_id: 16+16 = 32 B
  created_at: 8 B
  
  Итого: 40B
    
Comment
  id: 16 B
  post_id, user_id: 16+16=32 B
  parent_id (nullable): 16 B
  body: среднее 140 символов = 140
  created_at,updated_at: 16 B
  
  Итого: 220B
```

- Write-трафик:
 
  ```
  Пост: 
    ~1 KB payload (JSON + headers) «с запасом».
    RPS: 10000000 * 0.1 / 86400 = 12
    12 rps × 1 KB ≈ ~12 KB/s
  
  Картинки:
    среднее 1.5 MB/фото (~5)
    RPS: 10000000 * 0.1 * 5 / 86400 = 50 
    50 rps × 1.5 MB ≈ 75 MB/s в объектное хранилище
  
  Комментарии:
    среднee 100 B
    RPS: 10000000 * 0.5 / 86400 = ~50
    50 rps × 200B ≈ 1 KB/s
  
  Лайки:
    среднee 30 B
    RPS: 10000000 * 3 / 86400 = ~350
    350 rps × 200B ≈ 70 KB/s```

Read-трафик (клиентам)

  ```
  Пост + комментарии + локация:
    ~500B payload (JSON + headers) «с запасом».
    RPS: 10000000 * 50 / 86400 = ~5800
    5800 rps × 0.5 KB * 50(batch) ≈ ~145 MB/s
  
  Картинки: 
    возьмём 500 KB (не оригинал)
    RPS: 10000000 * 500 / 86400 = ~ 57900
    57900 rps × 500 KB ≈ ~29 GB/s```