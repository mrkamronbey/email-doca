Bobur:
# EMS API Hujjati

Oxirgi yangilanish: Fevral 2026

## Umumiy Ko'rinish
Ushbu hujjat Email Boshqaruv Tizimi (EMS) uchun to'liq HTTP API ma'lumotnomasi bo'lib, barcha endpointlar `api_router` ostida joylashgan (qarang: `app/api/router.py`). API ko'p kompaniyali va ko'p foydalanuvchili qo'llab-quvvatlash bilan email akkountlarini, xabarlarni, iplarni, inboxni, qo'shimchalarni va real vaqtli SSE hodisalarini boshqaradi.

## Autentifikatsiya va Ruxsatlar

### Bearer Token Autentifikatsiyasi
Ko'pgina endpointlar Authorization sarlavhasida JWT Bearer token talab qiladi:

Authorization: Bearer <jwt_token>


Token CurrentUser ga dekodlanadi:

{
  "user_id": 123,
  "role": "admin",
  "companies": [1, 2, 5]
}


Ruxsat qoidalari:
- Foydalanuvchilar faqat o'zlarining companies ro'yxatidagi kompaniyalar ma'lumotlariga kira oladi.
- Akkount kirish huquqi account.company_id foydalanuvchi kompaniyalarida borligini tekshirish orqali tasdiqlanadi.
- Xuddi shunday xabarlar, iplar va inbox yozuvlari uchun ham (kompaniya darajasida filtrlash).

### SSE (Server tomonidan yuborilgan hodisalar) Autentifikatsiyasi
GET /sse/events so'rov parametrida autentifikatsiyadan foydalanadi:

GET /sse/events?token=<jwt_token>


### Webhook (Autentifikatsiyasiz)
- POST /webhooks/nylas haqiqiylikni x-nylas-signature sarlavhasi va verify_nylas_webhook() yordamchi dasturi yordamida tekshiradi.
- Challenge so'rovi GET /webhooks/nylas?challenge=<str> autentifikatsiyasiz.

---

## HTTP Holat Kodlari

| Kod | Ma'nosi |
|-----|---------|
| 200 | Muvaffaqiyat; so'rov bajarildi |
| 201 | Resurs yaratildi |
| 204 | Tarkib yo'q; o'chirish muvaffaqiyatli |
| 400 | Noto'g'ri so'rov; yaroqsiz payload yoki ma'lumotlar ziddiyati |
| 401 | Ruxsatsiz; token yoki webhook imzosi yo'q/yaroqsiz |
| 403 | Taqiqlangan; foydalanuvchida kompaniya/resursga kirish huquqi yo'q |
| 404 | Topilmadi; resurs mavjud emas |
| 500 | Server xatosi |

---

## Javob Formati

Sahifalangan javoblar quyidagi tuzilmaga amal qiladi:

{
  "items": [ /* resurs obyektlari massivi */ ],
  "total": 100,
  "page": 1,
  "size": 25
}


Sahifalanmagan ro'yxat javoblari massivlar:

[/* resurs obyektlari massivi */]


Bitta resurs javoblari to'g'ridan-to'g'ri obyektlar:

{ /* resurs obyekti */ }


---

## Endpointlar Xulasasi

| Metod | Endpoint | Maqsad |
|-------|----------|--------|
| POST | /accounts/ | Email akkount yaratish |
| GET | /accounts/ | Akkountlar ro'yxati |
| GET | /accounts/{account_id} | Akkount tafsilotlarini olish |
| PATCH | /accounts/{account_id}/ | Akkountni yangilash |
| DELETE | /accounts/{account_id} | Akkountni uzish |
| GET | /messages/ | Xabarlar ro'yxati (sahifalangan) |
| GET | /messages/{message_id} | Xabar tafsilotlarini olish |
| POST | /messages/send/ | Email yuborish |
| POST | /messages/{message_id}/reply | Xabarga javob berish |
| GET | /inbox/ | Inbox elementlari ro'yxati (sahifalangan) |
| GET | /inbox/unread-counts | Akkount bo'yicha o'qilmagan sonlarni olish |
| PATCH | /inbox/{inbox_id}/read | Yozuvni o'qilgan deb belgilash |
| PATCH | /inbox/{inbox_id}/unread | Yozuvni o'qilmagan deb belgilash |
| PATCH | /inbox/{inbox_id}/star | Yulduzchani almashtirish |
| PATCH | /inbox/{inbox_id}/archive | Yozuvni arxivlash |
| PATCH | /inbox/threads/{thread_id}/read | Ipni o'qilgan deb belgilash |
| GET | /threads/ | Iplar ro'yxati (sahifalangan) |
| GET | /threads/{thread_id} | Xabarlar bilan ipni olish |
| PATCH | /threads/{thread_id} | Ipni yangilash |
| GET | /attachments/{attachment_id}/download | Yuklab olish URL-ini olish |
| GET | /attachments/message/{message_id} | Xabar uchun qo'shimchalar ro'yxati |
| GET | /webhooks/nylas?challenge=X | Webhook challenge (tasdiqlash) |
| POST | /webhooks/nylas | Nylas hodisalarini qabul qilish |
| GET | /sse/events?token=X | Real vaqtli yangilanishlar uchun SSE oqimi |

---

## Endpointlar Batafsil Ma'lumotnomasi

### Akkountlar
Email akkountlar tizimga ulangan alohida Gmail/GoDaddy/va boshqa pochta qutilari.

#### 1. Email Akkount Yaratish
Endpoint: POST /accounts/
Auth: Bearer JWT (majburiy)
Holat kodi: 201 Created

So'rov tanasi:

{
  "company_id": 123,
  "email_address": "support@acme.com",
  "display_name": "Support Team",
  "provider": "godaddy",
  "grant_id": "grant_abc123xyz"
}


So'rov sxemasi tafsilotlari:
- company_id *(int, majburiy)*: Kirish nazorati uchun kompaniya ID si.
- email_address *(string, email formati, majburiy)*: Noyob email manzil. Agar email yoki grant_id allaqachon mavjud bo'lsa, rad etiladi.
- display_name *(string, ixtiyoriy)*: Yuborilgan emaillarda ko'rinadigan ism.
- provider *(string, standart: "godaddy")*: Email provayder turi (masalan, "godaddy", "gmail").
- grant_id *(string, ixtiyoriy)*: Akkount uchun Nylas grant ID si.

Javob tanasi:

{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "company_id": 123,
  "email_address": "support@acme.com",
  "display_name": "Support Team",
  "provider": "godaddy",
  "status": "syncing",
  "last_sync_at": null,
  "created_at": "2026-02-21T10:30:00Z",
  "company": {
    "id": 123,
    "name": "Acme Corporation"
  }
}


Javob sxemasi tafsilotlari:
- id *(UUID)*: Noyob akkount identifikatori.
- status *(string)*: "active" | "syncing" | "disconnected".
- last_sync_at *(datetime, nullable)*: Nylas bilan oxirgi sinxronizatsiya vaqti.
- created_at *(datetime)*: Yaratilish vaqti (UTC).
- company *(object, nullable)*: Kompaniya ma'lumotlari obyekti.

Xato holatlari:
- 400: Email manzil yoki grant_id allaqachon mavjud.
- 403: Foydalanuvchi ko'rsatilgan company_id ga kirish huquqiga ega emas.

---

#### 2. Email Akkountlar Ro'yxati
Endpoint: GET /accounts/
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Javob tanasi: EmailAccountResponse obyektlari massivi

Tafsilotlar:
- Foydalanuvchi kompaniyalaridagi barcha akkountlarni qaytaradi.
- Bu endpointda sahifalash yo'q (barcha akkountlar qaytariladi).

---

#### 3. Akkount Tafsilotlarini Olish
Endpoint: GET /accounts/{account_id}
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- account_id *(UUID)*: Akkount identifikatori.

Javob tanasi: EmailAccountResponse obyekti

Xato holatlari:
- 404: Akkount topilmadi yoki foydalanuvchining kirish huquqi yo'q.

---

#### 4. Akkountni Yangilash
Endpoint: PATCH /accounts/{account_id}/
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- account_id *(UUID)*: Akkount identifikatori.

So'rov tanasi (qisman yangilash):

{
  "display_name": "Main Support",
  "status": "active"
}


So'rov sxemasi tafsilotlari:
- display_name *(string, ixtiyoriy)*: Ko'rsatma nomini yangilash.
- status *(string, ixtiyoriy)*: Akkount holatini yangilash.

Javob tanasi: Yangilangan EmailAccountResponse

---

#### 5. Akkountni Uzish
Endpoint: DELETE /accounts/{account_id}
Auth: Bearer JWT (majburiy)
Holat kodi: 204 No Content

Yo'l parametrlari:
- account_id *(UUID)*: Akkount identifikatori.

Tafsilotlar:
- Akkountni Nylas dan uzadi.
- Akkount holatini "disconnected" ga o'rnatadi.
- Barcha bog'liq xabarlar/iplar ma'lumotlar bazasida saqlanib qoladi.

---

### Xabarlar
Xabarlar alohida emaillarni ifodalaydi (kiruvchi, chiquvchi, yuborilgan).

#### 1. Xabarlar Ro'yxati (Sahifalangan)
Endpoint: GET /messages/
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

So'rov parametrlari:
- page *(int, standart: 1, min: 1)*: Sahifa raqami.
- size *(int, standart: 25)*: Sahifadagi elementlar soni.
- account_id *(UUID, ixtiyoriy)*: Email akkount bo'yicha filtrlash.

Javob tanasi:

{
  "items": [ /* MessageResponse massivi */ ],
  "total": 542,
  "page": 1,
  "size": 25
}


Tafsilotlar:
- sent_at bo'yicha kamayish tartibida saralangan (eng yangi birinchi).
- Yashirin filtr: Faqat foydalanuvchi kompaniyalaridagi akkountlardan kelgan xabarlar.
- account_id ko'rsatilmasa, barcha foydalanuvchi akkountlaridan xabarlar ko'rsatiladi.

---

#### 2. Xabar Tafsilotlarini Olish
Endpoint: GET /messages/{message_id}
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- message_id *(UUID)*: Xabar identifikatori.

Javob tanasi: MessageResponse obyekti (qo'shimchalar yuklangan holda)

Xato holatlari:
- 404: Xabar topilmadi yoki foydalanuvchining kirish huquqi yo'q.

---

#### 3. Email Yuborish
Endpoint: POST /messages/send/
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

So'rov tanasi:

{
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "to": [
    {
      "name": "John Customer",
      "email": "john@customer.com"
    }
  ],
  "cc": [
    {
      "name": "CC Person",
      "email": "cc@customer.com"
    }
  ],
  "bcc": null,
  "subject": "Re: Your Question",
  "body_html": "<p>Thanks for reaching out!</p>",
  "body_text": "Thanks for reaching out!",
  "reply_to_message_id": null
}


So'rov sxemasi tafsilotlari:
- email_account_id *(UUID, majburiy)*: Qaysi akkountdan yuborish.
- to *(massiv[EmailRecipient], majburiy)*: Qabul qiluvchilar (har biri ixtiyoriy name va majburiy `email` ga ega).
- cc *(massiv[EmailRecipient], ixtiyoriy)*: CC qabul qiluvchilar.
- bcc *(massiv[EmailRecipient], ixtiyoriy)*: BCC qabul qiluvchilar.
- subject *(string, majburiy)*: Email mavzusi.
- body_html *(string, majburiy)*: HTML email tanasi.
- body_text *(string, ixtiyoriy)*: Oddiy matn tanasi.
- reply_to_message_id *(UUID, ixtiyoriy)*: Javob beriladigan xabar ID si.

Javob tanasi: MessageResponse obyekti (yangi yuborilgan xabar)

Tafsilotlar:
- Akkountning grant ID si yordamida Nylas API orqali yuboriladi.
- Ip yaratiladi/yangilanadi.
- Xabar barcha kompaniya foydalanuvchilarining inboxlariga taqsimlanadi.
- Real vaqtli SSE yangilanishlari uchun Redis stream ga e'lon qilinadi.

Xato holatlari:
- 403: Foydalanuvchi email akkount kompaniyasiga kirish huquqiga ega emas.

---

#### 4. Xabarga Javob Berish
Endpoint: POST /messages/{message_id}/reply
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- message_id *(UUID)*: Asl xabar ID si.

So'rov tanasi: "Email Yuborish" bilan bir xil format

Javob tanasi: MessageResponse obyekti (yangi javob xabari)

Tafsilotlar:
- Asl xabar va bog'liq ipni topadi.
- Xuddi shu ipda yangi xabar yaratadi.

Xato holatlari:
- 404: Asl xabar topilmadi yoki foydalanuvchining kirish huquqi yo'q.

---

### Inbox
Inbox yozuvlari foydalanuvchi bo'yicha o'qilgan, yulduzcha qo'yilgan va arxivlangan holatni kuzatadi.

#### 1. Inbox Elementlari Ro'yxati (Sahifalangan)
Endpoint: GET /inbox/
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

So'rov parametrlari:
- page *(int, standart: 1, min: 1)*: Sahifa raqami.
- size *(int, standart: 25, maks: 100)*: Sahifadagi elementlar soni.
- is_read *(boolean, ixtiyoriy)*: O'qilganlik holati bo'yicha filtrlash. Ko'rsatilmasa, o'qilgan va o'qilmagan ikkalasi ham qaytariladi.
- account_id *(UUID, ixtiyoriy)*: Email akkount bo'yicha filtrlash.

Javob tanasi:

{
  "items": [ /* InboxItemResponse massivi */ ],
  "total": 18,
  "page": 1,
  "size": 25
}


Javob sxemasi tafsilotlari (InboxItemResponse):

{
  "id": "990e8400-e29b-41d4-a716-446655440000",
  "is_read": false,
  "is_starred": true,
  "is_archived": false,
  "read_at": null,
  "created_at": "2026-02-21T09:16:00Z",
  "message": {
    "id": "770e8400-e29b-41d4-a716-446655440000",
    "thread_id": "880e8400-e29b-41d4-a716-446655440000",
    "from_address": "customer@external.com",
    "from_name": "John Customer",
    "subject": "Question about pricing",
    "snippet": "Hi, I have a question...",
    "direction": "inbound",
    "has_attachments": false,
    "sent_at": "2026-02-21T09:15:00Z"
  },
  "thread_subject": "Question about pricing",
  "thread_message_count": 2,
  "account_email": "support@acme.com"
}


Tafsilotlar:
- Xabarning sent_at bo'yicha kamayish tartibida saralangan (eng yangi birinchi).
- Arxivlangan elementlar sukut bo'yicha chiqarib tashlangan (filtr: `is_archived == False`).
- is_read=true faqat o'qilmaganlarni ko'rsatadi; is_read=false faqat o'qilganlarni ko'rsatadi.

Filtrlar:
- is_read: O'qilmagan/o'qilgan holat.
- account_id: Muayyan akkount.
- Yashirin: Faqat foydalanuvchi kompaniyalaridan foydalanuvchining inbox yozuvlari.

---

#### 2. Akkount Bo'yicha O'qilmagan Sonlarni Olish
Endpoint: GET /inbox/unread-counts
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

So'rov parametrlari: Yo'q

Javob tanasi:

[
  {
    "account_id": "550e8400-e29b-41d4-a716-446655440000",
    "email_address": "support@acme.com",
    "unread_count": 5
  },
  {
    "account_id": "660e8400-e29b-41d4-a716-446655440001",
    "email_address": "sales@acme.com",
    "unread_count": 12
  }
]


Tafsilotlar:
- Akkount bo'yicha guruhlangan o'qilmagan xabarlar sonini qaytaradi.
- UI dagi badge bildirishnomalari uchun foydali.

---

#### 3. Inbox Yozuvini O'qilgan Deb Belgilash
Endpoint: PATCH /inbox/{inbox_id}/read
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- inbox_id *(UUID)*: Inbox yozuvi ID si.

So'rov tanasi: Bo'sh (`{}`)

Javob tanasi: Yangilangan InboxEntryResponse

Tafsilotlar:
- is_read ni true ga o'rnatadi.
- read_at ni joriy UTC vaqtiga o'rnatadi.

---

#### 4. Inbox Yozuvini O'qilmagan Deb Belgilash
Endpoint: PATCH /inbox/{inbox_id}/unread
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- inbox_id *(UUID)*: Inbox yozuvi ID si.

So'rov tanasi: Bo'sh (`{}`)

Javob tanasi: is_read = false, read_at = null bo'lgan yangilangan InboxEntryResponse

---

#### 5. Inbox Yozuvida Yulduzchani Almashtirish
Endpoint: PATCH /inbox/{inbox_id}/star
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- inbox_id *(UUID)*: Inbox yozuvi ID si.

So'rov tanasi: Bo'sh (`{}`)

Javob tanasi: is_starred almashtirilgan yangilangan InboxEntryResponse

---

#### 6. Inbox Yozuvini Arxivlash
Endpoint: PATCH /inbox/{inbox_id}/archive
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- inbox_id *(UUID)*: Inbox yozuvi ID si.

So'rov tanasi: Bo'sh (`{}`)

Javob tanasi: is_archived = true bo'lgan yangilangan InboxEntryResponse

Tafsilotlar:
- Arxivlangan yozuvlar standart inbox ro'yxati ko'rinishidan chiqarib tashlanadi.

---

#### 7. Butun Ipni O'qilgan Deb Belgilash
Endpoint: PATCH /inbox/threads/{thread_id}/read
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- thread_id *(UUID)*: Ip ID si.

So'rov tanasi: Bo'sh (`{}`)

Javob tanasi:

{
  "updated": 3
}


Tafsilotlar:
- Iptagi foydalanuvchining barcha inbox yozuvlarini o'qilgan deb belgilaydi.
- Yangilangan yozuvlar sonini qaytaradi.

---

### Iplar
Iplar bog'liq xabarlarni suhbat bo'yicha guruhlaydi.

#### 1. Iplar Ro'yxati (Sahifalangan)
Endpoint: GET /threads/
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

So'rov parametrlari:
- account_id *(UUID, ixtiyoriy)*: Email akkount bo'yicha filtrlash.
- page *(int, standart: 1, min: 1)*: Sahifa raqami.
- size *(int, standart: 25, maks: 100)*: Sahifadagi elementlar soni.

Javob tanasi:

{
  "items": [
    {
      "id": "880e8400-e29b-41d4-a716-446655440000",
      "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
      "subject": "Question about pricing",
      "last_message_at": "2026-02-21T10:00:00Z",
      "message_count": 3,
      "do_not_delete": false,
      "created_at": "2026-02-21T08:00:00Z"
    }
  ],
  "total": 42,
  "page": 1,
  "size": 25
}


Tafsilotlar:
- last_message_at bo'yicha kamayish tartibida saralangan (eng so'nggi birinchi).
- Yashirin filtr: Faqat foydalanuvchi kompaniyalaridagi akkountlardan iplar.

---

#### 2. Xabarlar Bilan Ipni Olish
Endpoint: GET /threads/{thread_id}
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- thread_id *(UUID)*: Ip ID si.

Javob tanasi: ThreadDetailResponse (xabarlar massivini o'z ichiga oladi)

Misol:

{
  "id": "880e8400-e29b-41d4-a716-446655440000",
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "subject": "Question about pricing",
  "last_message_at": "2026-02-21T10:00:00Z",
  "message_count": 3,
  "do_not_delete": false,
  "created_at": "2026-02-21T08:00:00Z",
  "messages": [
    /* vaqt tartibida saralangan MessageResponse obyektlari massivi */
  ]
}


Tafsilotlar:
- Xabarlar vaqt tartibida saralangan (eng eski birinchi).
- Har bir xabar o'zining qo'shimchalarini o'z ichiga oladi.

Xato holatlari:
- 404: Ip topilmadi yoki foydalanuvchining kirish huquqi yo'q.

---

#### 3. Ipni Yangilash
Endpoint: PATCH /threads/{thread_id}
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- thread_id *(UUID)*: Ip ID si.

So'rov tanasi (qisman yangilash):

{
  "do_not_delete": true
}


So'rov sxemasi tafsilotlari:
- do_not_delete *(boolean, ixtiyoriy)*: Ipni muhim/himoyalangan deb belgilash.

Javob tanasi: Yangilangan ThreadDetailResponse

---

### Qo'shimchalar
Qo'shimchalar xabarlarga kiritilgan fayllar.

#### 1. Qo'shimchani Yuklab Olish URL-ini Olish
Endpoint: GET /attachments/{attachment_id}/download
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- attachment_id *(UUID)*: Qo'shimcha ID si.

Javob tanasi:

{
  "download_url": "https://s3.amazonaws.com/bucket/attachments/path/to/file.pdf?AWSAccessKeyId=...&Signature=...&Expires=..."
}


Tafsilotlar:
- Oldindan imzolangan AWS S3 URL ni qaytaradi (taxminan 1 soat amal qiladi).
- URL autentifikatsiyani o'z ichiga oladi; to'g'ridan-to'g'ri yuklab olish uchun ulashish mumkin.

---

#### 2. Xabar Uchun Qo'shimchalar Ro'yxati
Endpoint: GET /attachments/message/{message_id}
Auth: Bearer JWT (majburiy)
Holat kodi: 200 OK

Yo'l parametrlari:
- message_id *(UUID)*: Xabar ID si.

Javob tanasi: AttachmentResponse obyektlari massivi

Misol:

[
  {
    "id": "aa0e8400-e29b-41d4-a716-446655440000",
    "message_id": "770e8400-e29b-41d4-a716-446655440000",
    "filename": "invoice.pdf",
    "content_type": "application/pdf",
    "size_bytes": 123456,
    "is_inline": false,
    "created_at": "2026-02-21T08:16:00Z"
  }
]


Javob sxemasi tafsilotlari:
- is_inline: true = email tanasiga joylashtirilgan, false = oddiy qo'shimcha.
- content_type: Faylning MIME turi.
- size_bytes: Fayl hajmi baytlarda.

---

### Webhooklar
Nylas dan (email provayder) real vaqtli hodisalarni qabul qilish.

#### 1. Webhook Challenge (Tasdiqlash)
Endpoint: GET /webhooks/nylas
Auth: Yo'q
Holat kodi: 200 OK

So'rov parametrlari:
- challenge *(string, majburiy)*: Nylas dan kelgan challenge qatori.

Javob tanasi: Oddiy matn (media turi: `text/plain`)

<challenge-qiymati>


Tafsilotlar:
- Nylas webhook ro'yxatdan o'tkazishda egalilikni tasdiqlash uchun buni chaqiradi.
- Challenge ni oddiy matn sifatida aynan qaytarish kerak.

---

#### 2. Webhook Hodisalarini Qabul Qilish
Endpoint: POST /webhooks/nylas
Auth: Imzo tekshiruvi (sarlavha: `x-nylas-signature`)
Holat kodi: 200 OK

So'rov sarlavhasi:

x-nylas-signature: <hisoblangan-imzo>


So'rov tanasi (message.created uchun misol):

{
  "specversion": "1.0",
  "type": "message.created",
  "source": "nylas",
  "id": "webhook-delivery-id-abc123",
  "time": 1708937400,
  "data": {
    "object": {
      "id": "nylas-message-id-xyz789",
      "grant_id": "grant_abc123xyz",
      "thread_id": "nylas-thread-id",
      "subject": "New inquiry",
      "from": [{"email": "customer@external.com", "name": "John Customer"}],
      "to": [{"email": "support@acme.com", "name": "Support"}],
      "cc": null,
      "bcc": null,
      "date": 1708937400,
      "body": "Hello, I have a question...",
      "snippet": "Hello, I have a question...",
      "attachments": [],
      "folders": ["inbox"]
    }
  }
}


Javob tanasi:

{
  "status": "ok"
}


Tafsilotlar:
- Imzo verify_nylas_webhook() yordamchi dasturi yordamida tekshiriladi.
- Payload process_webhook_event() tomonidan qayta ishlanadi.
- Hodisa turlari: "message.created", "message.updated", "grant.expired" va boshqalar.

Xato holatlari:
- 401: Imzo yaroqsiz yoki yo'q.

---

### SSE (Server tomonidan yuborilgan hodisalar)
Ulangan foydalanuvchi uchun real vaqtli hodisa oqimi.

#### 1. Hodisa Oqimi (Uzluksiz)
Endpoint: GET /sse/events
Auth: So'rov parametrida JWT (`?token=<jwt>`)
Holat kodi: 200 OK (uzluksiz)
Media turi: text/event-stream

So'rov parametrlari:
- token *(string, majburiy)*: JWT Bearer token.

Javob (Uzluksiz Format - Server-Sent Events):

event: connected
data: {"message":"connected","user_id":123}

event: message.created
data: {"type":"message.created","thread_id":"880e8400-e29b-41d4-a716-446655440000","message_id":"770e8400-e29b-41d4-a716-446655440000","company_id":123}

event: message.created
data: {"type":"message.created","thread_id":"880e8400-e29b-41d4-a716-446655440001","message_id":"770e8400-e29b-41d4-a716-446655440001","company_id":123}

:


**Hodisa Sxemasi Tafsilotlari:**

- **Ulanish Hodisasi:**

  event: connected
  data: {"message":"connected","user_id":<foydalanuvchi_id>}

  - SSE oqimi o'rnatilganda bir marta yuboriladi.

- **Ma'lumot Hodisalari:**

  event: <tur>
  data: {"type":"<tur>","thread_id":"<uuid>","message_id":"<uuid>","company_id":<int>}

  - Hodisa turlari: "message.created", "message.updated", "thread.created", "thread.updated" va boshqalar.
  - Foydalanuvchi kompaniyalariga tegishli har bir email/ip hodisasi uchun yuboriladi.

- **Keepalive (Yurak urishi):**

  :

  - Ulanishni tirik saqlash uchun davriy ravishda (15 soniya harakatsizlikdan keyin) yuboriladi.

**Tafsilotlar:**
- SSE doimiy; mijoz ulanadi va real vaqtda hodisalarni qabul qiladi.
- Hodisalar Redis stream dan keladi.
- Server tomonida filtrlash: Faqat foydalanuvchining `companies` ro'yxati uchun hodisalar.
- Token muddati tugaganda yoki mijoz uzilganda ulanish yopiladi.

---

## Sxemalar Ma'lumotnomasi

### CompanyInfo
```json
{
  "id": 123,
  "name": "Acme Corporation"
}
```

### EmailRecipient

{
  "name": "John Doe",
  "email": "john@example.com"
}

- name ixtiyoriy; email majburiy va to'g'ri email formatida bo'lishi kerak.

### MessageResponse

{
  "id": "770e8400-e29b-41d4-a716-446655440000",
  "thread_id": "880e8400-e29b-41d4-a716-446655440000",
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "internet_message_id": "<msg.1234567890@acme.com>",
  "from_address": "sender@example.com",
  "from_name": "Sender Name",
  "to_addresses": [{"name": "Recipient", "email": "to@example.com"}],
  "cc_addresses": null,
  "bcc_addresses": null,
  "subject": "Subject",
  "body_html": "<p>Email body</p>",
  "body_text": "Email body",
  "snippet": "Email body...",
  "direction": "inbound",
  "has_attachments": false,
  "sent_at": "2026-02-21T09:15:00Z",
  "created_at": "2026-02-21T09:16:00Z",
  "attachments": []
}


### AttachmentResponse

{
  "id": "aa0e8400-e29b-41d4-a716-446655440000",
  "message_id": "770e8400-e29b-41d4-a716-446655440000",
  "filename": "invoice.pdf",
  "content_type": "application/pdf",
  "size_bytes": 123456,
  "is_inline": false,
  "created_at": "2026-02-21T08:16:00Z"
}


### InboxEntryResponse

{
  "id": "990e8400-e29b-41d4-a716-446655440000",
  "user_id": 123,
  "message_id": "770e8400-e29b-41d4-a716-446655440000",
  "thread_id": "880e8400-e29b-41d4-a716-446655440000",
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "is_read": true,
  "is_starred": true,
  "is_archived": false,
  "read_at": "2026-02-21T11:00:00Z",
  "created_at": "2026-02-21T09:16:00Z"
}


### ThreadDetailResponse

{
  "id": "880e8400-e29b-41d4-a716-446655440000",
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "subject": "Question about pricing",
  "last_message_at": "2026-02-21T10:00:00Z",
  "message_count": 3,
  "do_not_delete": false,
  "created_at": "2026-02-21T08:00:00Z",
  "messages": [ /* MessageResponse massivi */ ]
}


---

## Xato Boshqaruvi

Barcha xato javoblari standart xato kodlari bilan detail xabarlarini qaytaradi:

400 Bad Request:

{
  "detail": "Email account with this email address/grant already exists."
}


401 Unauthorized:

{
  "detail": "Invalid or expired token"
}


403 Forbidden:

{
  "detail": "No access to this email account"
}


404 Not Found:

{
  "detail": "Message not found"
}


---

## Sahifalash Qo'llanmasi

Barcha sahifalangan endpointlar ushbu qoidaga amal qiladi:

Ofset hisoblash:

offset = (sahifa - 1) * o'lcham


Misol: 25 element bilan 2-sahifani olish:

/messages/?page=2&size=25
# offset = (2 - 1) * 25 = 25 (birinchi 25 tani o'tkazib yubor, keyingi 25 tani ol)


Maksimal Sahifa O'lchami: Ko'pgina endpointlar standart: size: 25, maksimum 100.

Jami Son: Har doim javobga kiritilgan:

{
  "total": 542,
  "page": 1,
  "size": 25
  // maks_sahifalar = ceil(542 / 25) = 22
}


---

## Tezkor Misollar

### Email Yuborish

curl -X POST https://api.example.com/messages/send/ \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
    "to": [{"name": "Customer", "email": "customer@example.com"}],
    "subject": "Support Response",
    "body_html": "<p>Thank you!</p>",
    "body_text": "Thank you!"
  }'


### Filtrlar Bilan Inboxni Ro'yxatlash

# Muayyan akkountdan o'qilmagan xabarlarni olish, 1-sahifa
curl "https://api.example.com/inbox/?page=1&size=25&is_read=false&account_id=550e8400-e29b-41d4-a716-446655440000" \
  -H "Authorization: Bearer <jwt_token>"


### O'qilmagan Sonlarni Olish

curl https://api.example.com/inbox/unread-counts \
  -H "Authorization: Bearer <jwt_token>"


### Qo'shimchani Yuklab Olish

# 1. Qo'shimchani yuklab olish URL-ini olish
curl https://api.example.com/attachments/aa0e8400-e29b-41d4-a716-446655440000/download \
  -H "Authorization: Bearer <jwt_token>"

# Javob: {"download_url": "https://s3.amazonaws.com/...presigned..."}

# 2. Yuklab olish uchun URL dan foydalanish
curl https://s3.amazonaws.com/...presigned-url... -o file.pdf


### Real Vaqtli Hodisalarni Uzatish

# Token ni so'rov parametri sifatida SSE oqimiga ulanish
curl "https://api.example.com/sse/events?token=<jwt_token>"

# Qabul qilinadi:
# event: connected
# data: {"message":"connected","user_id":123}
#
# event: message.created
# data: {"type":"message.created","thread_id":"...","message_id":"...","company_id":123}


---

## Ruxsat va Kirish Nazorati

Barcha endpointlar kompaniya darajasida kirish nazoratini amalga oshiradi:

1. Foydalanuvchida kompaniyalar ro'yxati mavjud (JWT dan): ["companies": [1, 2, 5]]
2. Resurslar kompaniya bo'yicha filtrlanadi:
   - Akkountlar: Faqat account.company_id foydalanuvchi kompaniyalarida bo'lsa
   - Xabarlar: Faqat message.email_account.company_id foydalanuvchi kompaniyalarida bo'lsa
   - Iplar: Faqat thread.email_account.company_id foydalanuvchi kompaniyalarida bo'lsa
   - Inbox: Faqat foydalanuvchining o'z inbox yozuvlari kirish mumkin bo'lgan kompaniyalardan

3. Xato javoblari:
   - 403 Forbidden: Foydalanuvchi kompaniya/resursga kirish huquqiga ega emas
   - 404 Not Found: Resursga kirish imkoni yo'q (ma'lumot sizib chiqmasligi uchun)

---

## Sxemalar (Xulosa)**
- EmailAccountCreate (so'rov)


{
  "company_id": 123,
  "email_address": "user@example.com",
  "display_name": "Support",
  "provider": "godaddy",
  "grant_id": "optional-grant-id"
}


- EmailAccountResponse (misol)


{
  "id": "11111111-1111-1111-1111-111111111111",
  "company_id": 123,
  "email_address": "user@example.com",
  "display_name": "Support",
  "provider": "godaddy",
  "status": "active",
  "last_sync_at": "2025-12-01T12:00:00Z",
  "created_at": "2025-11-01T09:00:00Z",
  "company": { "id": 123, "name": "Acme Inc." }
}


- SendEmailRequest (yuborish va javob berish uchun ishlatiladi)


{
  "email_account_id": "22222222-2222-2222-2222-222222222222",
  "to": [{ "name": "Recipient", "email": "to@example.com" }],
  "cc": [{ "name": "CC", "email": "cc@example.com" }],
  "bcc": null,
  "subject": "Hello",
  "body_html": "<p>Hi</p>",
  "body_text": "Hi",
  "reply_to_message_id": null
}


- MessageResponse (misol)


{
  "id": "33333333-3333-3333-3333-333333333333",
  "thread_id": "44444444-4444-4444-4444-444444444444",
  "email_account_id": "22222222-2222-2222-2222-222222222222",
  "internet_message_id": "<msg@example.com>",
  "from_address": "sender@example.com",
  "from_name": "Sender",
  "to_addresses": [{"name":"Recipient","email":"to@example.com"}],
  "cc_addresses": null,
  "bcc_addresses": null,
  "subject": "Hello",
  "body_html": "<p>Hi</p>",
  "body_text": "Hi",
  "snippet": "Hi...",
  "direction": "inbound",
  "has_attachments": false,
  "sent_at": "2025-12-01T12:01:00Z",
  "created_at": "2025-12-01T12:02:00Z",
  "attachments": []
}


- PaginatedMessageResponse shakli

{
  "items": [ /* MessageResponse obyektlari */ ],
  "total": 120,
  "page": 1,
  "size": 25
}


- ThreadSummary / ThreadDetailResponse (misollar)

Ip xulasasi elementi:

{
  "id": "44444444-4444-4444-4444-444444444444",
  "email_account_id": "22222222-2222-2222-2222-222222222222",
  "subject": "Support request",
  "last_message_at": "2025-12-01T12:00:00Z",
  "message_count": 3,
  "do_not_delete": false,
  "created_at": "2025-11-30T08:00:00Z"
}


Ip tafsiloti xuddi shu maydonlarni va qo'shimcha xabarlarni (`MessageResponse` massivi) qaytaradi.

- InboxItemResponse (misol)


{
  "id": "55555555-5555-5555-5555-555555555555",
  "is_read": false,
  "is_starred": false,
  "is_archived": false,
  "read_at": null,
  "created_at": "2025-12-01T12:00:00Z",
  "message": {/* MessageSummary */},
  "thread_subject": "Subject",
  "thread_message_count": 2,
  "account_email": "support@example.com"
}


- AttachmentResponse (misol)


{
  "id": "66666666-6666-6666-6666-666666666666",
  "message_id": "33333333-3333-3333-3333-333333333333",
  "filename": "file.pdf",
  "content_type": "application/pdf",
  "size_bytes": 12345,
  "is_inline": false,
  "created_at": "2025-12-01T12:00:00Z"
}


Endpoint ma'lumotnomasi

- Akkountlar (`/accounts`)
  - POST /accounts/ : Email akkountni yaratish/ulash.
    - Auth: Bearer JWT
    - So'rov tanasi: EmailAccountCreate (yuqoridagi misolga qarang)
    - Javob: EmailAccountResponse (201)
  - GET /accounts/ : Foydalanuvchi kompaniyalaridagi akkountlar ro'yxati.
    - Auth: Bearer JWT
    - Javob: list[EmailAccountResponse] (200)
  - GET /accounts/{account_id} : Bitta akkountni olish.
    - Auth: Bearer JWT
    - Javob: EmailAccountResponse (200) yoki 404
  - PATCH /accounts/{account_id}/ : Akkount maydonlarini yangilash.
    - Auth: Bearer JWT
    - So'rov: EmailAccountUpdate (qisman)
    - Javob: EmailAccountResponse (200)
  - DELETE /accounts/{account_id} : Akkountni uzish.
    - Auth: Bearer JWT
    - Javob: 204 No Content

- Xabarlar (`/messages`)
  - POST /messages/send/ : Email yuborish.
    - Auth: Bearer JWT
    - So'rov: SendEmailRequest (misolga qarang)
    - Javob: MessageResponse (200)
  - GET /messages/ : Sahifalangan xabarlar ro'yxati.
    - Auth: Bearer JWT
    - So'rov parametrlari: page, size, account_id (ixtiyoriy)
    - Javob: PaginatedMessageResponse (200)
  - GET /messages/{message_id} : Bitta xabarni olish.
    - Auth: Bearer JWT
    - Javob: MessageResponse (200) yoki 404
  - POST /messages/{message_id}/reply : Xabarga javob berish.
    - Auth: Bearer JWT
    - So'rov: SendEmailRequest
    - Javob: MessageResponse (200)

- Inbox (`/inbox`)
  - GET /inbox/ : Inbox elementlari ro'yxati (sahifalangan).
    - Auth: Bearer JWT
    - So'rov parametrlari: page, size, is_read, account_id
    - Javob: InboxListResponse (elementlar: `InboxItemResponse`)
  - GET /inbox/unread-counts : Akkount bo'yicha o'qilmagan sonlarni olish
    - Auth: Bearer JWT
    - Javob: UnreadCountResponse obyektlari ro'yxati
  - PATCH /inbox/{inbox_id}/read : Yozuvni o'qilgan deb belgilash
    - Auth: Bearer JWT
    - Javob: InboxEntryResponse (200)
  - PATCH /inbox/{inbox_id}/unread : O'qilmagan deb belgilash
    - Auth: Bearer JWT
    - Javob: InboxEntryResponse (200)
  - PATCH /inbox/{inbox_id}/star : Yulduzchani almashtirish
    - Auth: Bearer JWT
    - Javob: InboxEntryResponse (200)
  - PATCH /inbox/{inbox_id}/archive : Yozuvni arxivlash
    - Auth: Bearer JWT
    - Javob: InboxEntryResponse (200)
  - PATCH /inbox/threads/{thread_id}/read : Butun ipni o'qilgan deb belgilash
    - Auth: Bearer JWT
    - Javob: { "updated": <son> }

- Iplar (`/threads`)
  - GET /threads/ : Iplar ro'yxati (sahifalangan)
    - Auth: Bearer JWT
    - So'rov parametrlari: account_id, page, size
    - Javob: ThreadListResponse
  - GET /threads/{thread_id} : Xabarlar bilan ip tafsilotlari
    - Auth: Bearer JWT
    - Javob: ThreadDetailResponse yoki 404
  - PATCH /threads/{thread_id} : Ipni yangilash (masalan, `do_not_delete`)
    - Auth: Bearer JWT
    - So'rov: ThreadUpdate (qisman)
    - Javob: ThreadDetailResponse

- Qo'shimchalar (`/attachments`)
  - GET /attachments/{attachment_id}/download : Oldindan imzolangan yuklab olish URL-ini olish
    - Auth: Bearer JWT
    - Javob: { "download_url": "https://..." } yoki 404
  - GET /attachments/message/{message_id} : Xabar uchun qo'shimchalar ro'yxati
    - Auth: Bearer JWT
    - Javob: list[AttachmentResponse]

- Webhooklar (`/webhooks/nylas`)
  - GET /webhooks/nylas?challenge=<str> : Challenge ni oddiy matn sifatida qaytaradi (tasdiqlash uchun ishlatiladi)
    - Auth yo'q
    - Javob: oddiy matn challenge
  - POST /webhooks/nylas : Nylas webhooklarini qabul qilish
    - Bearer auth yo'q; buning o'rniga so'rov x-nylas-signature sarlavhasi va verify_nylas_webhook yordamchi dasturi yordamida tekshiriladi.
    - So'rov tanasi: Nylas tomonidan yuborilgan ixtiyoriy payload (qarang: `NylasWebhookPayload`)
    - Javob: muvaffaqiyatda { "status": "ok" } yoki yaroqsiz imzoda 401

- SSE (`/sse/events`)
  - GET /sse/events?token=<jwt> : Server-Sent Events oqimi
    - Foydalanuvchi tokenini token so'rov parametrida ko'rsating
    - Quyidagi ko'rinishdagi hodisalar bilan text/event-stream qaytaradi:

SSE hodisasi misoli (server tomonidan yuborilgan format):

event: message.created
data: {"type":"message.created","thread_id":"...","message_id":"...","company_id":123}



Webhook payload shakllari
- NylasWebhookPayload (yuqori daraja)


{
  "specversion": "1.0",
  "type": "message.created",
  "source": "nylas",
  "id": "delivery-id",
  "time": 1670000000,
  "data": { /* obyekt hodisa turiga qarab o'zgaradi */ }
}


- NylasWebhookMessageData (xabar hodisalari uchun data.object ichida)


{
  "id": "nylas-msg-id",
  "grant_id": "grant-id",
  "thread_id": "nylas-thread-id",
  "subject": "Hello",
  "from": [{"email":"from@example.com","name":"Sender"}],
  "to": [{"email":"to@example.com","name":"Recipient"}],
  "date": 1670000000,
  "body": "...",
  "snippet": "...",
  "attachments": [{ /* ... */ }],
  "folders": ["inbox"]
}


Eslatmalar va keyingi qadamlar
- API yuzasini app/api/* dan va JSON shakllarini app/schemas/*.dan ajratib oldim.
- Agar xohlasangiz, quyidagilarni amalga oshirishim mumkin:
  - Har bir endpoint uchun jonli curl misollari qo'shish (autentifikatsiya sarlavhasi misollari bilan).
  - Pydantic modellaridan olingan OpenAPI/Swagger ga mos JSON/YAML faylini yaratish.
  - Bir nechta endpoint uchun namunaviy birlik/integratsiya testlarini qo'shish.
