# EMS API — Frontend Dasturchi Uchun To'liq Qo'llanma

> **Ushbu hujjat kimlar uchun?**
> Backend bilan ishlaydigan frontend dasturchilar uchun. Har bir endpoint uchun tayyor
> JavaScript/TypeScript misollar, UI uchun izohlar va amaliy maslahatlar berilgan.
> Backend ichki ishlash mexanizmlari (Redis, Nylas, Pydantic va h.k.) tushuntirilmagan —
> faqat frontendga kerakli narsalar yozilgan.

---

## Mundarija

1. [Umumiy Ko'rinish](#1-umumiy-korinish)
2. [Sozlash — Base URL va Umumiy So'rov Funksiyasi](#2-sozlash)
3. [Autentifikatsiya va Ruxsatlar](#3-autentifikatsiya-va-ruxsatlar)
4. [HTTP Holat Kodlari](#4-http-holat-kodlari)
5. [Javob Formati](#5-javob-formati)
6. [Barcha Endpointlar Jadvali](#6-barcha-endpointlar-jadvali)
7. [Akkountlar](#7-akkountlar)
8. [Xabarlar](#8-xabarlar)
9. [Inbox](#9-inbox)
10. [Iplar (Threads)](#10-iplar-threads)
11. [Qo'shimchalar (Attachments)](#11-qoshimchalar-attachments)
12. [Webhooklar](#12-webhooklar)
13. [SSE — Real Vaqt Yangilanishlar](#13-sse--real-vaqt-yangilanishlar)
14. [To'liq Sxemalar](#14-toliq-sxemalar)
15. [Xatolarni Boshqarish](#15-xatolarni-boshqarish)
16. [Sahifalash (Pagination)](#16-sahifalash-pagination)
17. [Ruxsat va Kirish Nazorati](#17-ruxsat-va-kirish-nazorati)
18. [TypeScript Interfeyslari](#18-typescript-interfeyslari)

---

## 1. Umumiy Ko'rinish

EMS (Email Management System) — **Elektron Pochta Boshqaruv Tizimi** uchun HTTP API.

**Nima qila oladi:**
- Gmail, GoDaddy va boshqa pochta qutilari bilan ulash
- Email yuborish, qabul qilish, javob berish
- Inbox boshqaruvi (o'qilgan/o'qilmagan, yulduzcha, arxiv)
- Suhbat iplari (threads) boshqaruvi
- Fayl qo'shimchalari (attachments) yuklab olish
- Real vaqtda yangilanishlar (SSE orqali)
- Ko'p kompaniya va ko'p foydalanuvchi qo'llab-quvvatlash

**Frontend uchun asosiy tushuncha:**
Foydalanuvchi `companies: [1, 2, 5]` ro'yxati bo'lgan JWT token bilan tizimga kiradi.
Server **faqat shu kompaniyalarga tegishli** ma'lumotlarni qaytaradi — frontendda qo'shimcha filtr yozish shart emas.

---

## 2. Sozlash

### Base URL va umumiy so'rov funksiyasi

Barcha so'rovlar bir xil pattern bo'yicha ishlaydi:
`Authorization: Bearer <token>` sarlavhasi + JSON body.

```typescript
const BASE_URL = "https://api.example.com"; // o'zingizning API URL ingiz

// Tokenni saqlash/olish
const TokenStorage = {
  save: (token: string) => localStorage.setItem("jwt_token", token),
  get: () => localStorage.getItem("jwt_token"),
  remove: () => localStorage.removeItem("jwt_token"),
};

// Umumiy so'rov yordamchisi — barcha API chaqiruvlar shu orqali o'tadi
async function apiRequest<T>(
  endpoint: string,
  options: RequestInit = {}
): Promise<T> {
  const token = TokenStorage.get();

  const response = await fetch(`${BASE_URL}${endpoint}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
  });

  // 204 No Content (DELETE operatsiyalarida javob bo'lmaydi)
  if (response.status === 204) return null as T;

  const data = await response.json().catch(() => ({}));

  if (!response.ok) {
    // { status: 404, detail: "Message not found" } shaklida xato tashlaydi
    throw { status: response.status, detail: data.detail ?? "Xato yuz berdi" };
  }

  return data as T;
}
```

> **CORS haqida:** Agar development muhitida `localhost`dan API ga so'rov yuborsangiz
> va CORS xatosi chiqsa — bu backendda hal qilinishi kerak bo'lgan muammo.
> Frontend tomonidan `mode: "cors"` yoki `credentials: "include"` qo'shish kerak bo'lishi mumkin,
> buni backend dasturchi bilan birgalikda hal qiling.

### Bildirishnoma (Notification) yordamchisi

Quyidagi hujjatda `showNotification()` funksiyasi ishlatiladi.
Bu sizning UI kutubxonangizga bog'liq — o'zingiznikini yozing:

```typescript
// Misol: oddiy alert (ishlab chiqishda)
function showNotification(message: string, type: "error" | "warning" | "info" | "success") {
  console.log(`[${type.toUpperCase()}] ${message}`);
  // Haqiqiy ilovada: toast, snackbar, yoki modal ko'rsating
  // Masalan: toast.error(message) — react-hot-toast bilan
  // Yoki: notification.open({ message, type }) — Ant Design bilan
}
```

### Vaqtni Formatlash Yordamchilari

API barcha vaqtlarni **ISO 8601** formatida qaytaradi (`"2026-02-21T09:15:00Z"`).
UI da ko'rsatish uchun formatlashtiring:

```typescript
// Sana va vaqtni o'zbek/lotin formatida ko'rsatish
function formatDate(isoString: string): string {
  const date = new Date(isoString);
  return date.toLocaleDateString("uz-UZ", {
    year: "numeric",
    month: "long",
    day: "numeric",
  });
  // → "21-fevral, 2026"
}

function formatTime(isoString: string): string {
  const date = new Date(isoString);
  return date.toLocaleTimeString("uz-UZ", {
    hour: "2-digit",
    minute: "2-digit",
  });
  // → "09:15"
}

function formatDateTime(isoString: string): string {
  return `${formatDate(isoString)}, ${formatTime(isoString)}`;
  // → "21-fevral, 2026, 09:15"
}

// Nisbiy vaqt ("3 daqiqa oldin", "2 soat oldin")
function timeAgo(isoString: string): string {
  const diff = Date.now() - new Date(isoString).getTime();
  const minutes = Math.floor(diff / 60000);
  const hours = Math.floor(diff / 3600000);
  const days = Math.floor(diff / 86400000);

  if (minutes < 1)  return "Hozir";
  if (minutes < 60) return `${minutes} daqiqa oldin`;
  if (hours < 24)   return `${hours} soat oldin`;
  return `${days} kun oldin`;
}

// Ishlatish
// item.message.sent_at → "2026-02-21T09:15:00Z"
// formatDateTime(item.message.sent_at) → "21-fevral, 2026, 09:15"
// timeAgo(item.message.sent_at) → "3 soat oldin"
```

---

## 3. Autentifikatsiya va Ruxsatlar

### 3.1 Bearer Token (JWT) — Asosiy usul

Ko'pgina endpointlar `Authorization` sarlavhasida JWT token talab qiladi:

```
Authorization: Bearer <jwt_token>
```

Token ichida quyidagi ma'lumotlar bo'ladi:

```json
{
  "user_id": 123,
  "role": "admin",
  "companies": [1, 2, 5]
}
```

**Frontend uchun:** Token backend tomonidagi login endpointidan olinadi.
Frontendda uni `localStorage` da saqlang va har so'rovga sarlavha sifatida qo'shing
(yuqoridagi `apiRequest` funksiyasi buni avtomatik qiladi).

**Ruxsat qoidalari (server tomonida bajariladi, frontendda qo'shimcha ish kerak emas):**
- Foydalanuvchi faqat o'z `companies` ro'yxatidagi kompaniyalar ma'lumotlarini ko'ra oladi
- Akkount, xabar, ip va inbox yozuvlari kompaniya darajasida filtrlanadi

---

### 3.2 SSE Autentifikatsiyasi

`GET /sse/events` so'rovida token sarlavhada **emas**, URL parametrida yuboriladi.
Chunki brauzerdagi `EventSource` API sarlavha qo'shishni qo'llab-quvvatlamaydi.

```
GET /sse/events?token=<jwt_token>
```

```typescript
const token = TokenStorage.get();
const sse = new EventSource(`${BASE_URL}/sse/events?token=${token}`);
```

---

### 3.3 Webhook — Imzo tekshiruvi

`POST /webhooks/nylas` va `GET /webhooks/nylas` — bu endpointlar **to'liq backend tomonida** boshqariladi.

**Frontend dasturchisi uchun:** Bu endpointlarni o'zingiz chaqirmaysiz.
Nylas xizmati serverga to'g'ridan-to'g'ri murojaat qiladi.
Backend esa webhook orqali kelgan yangilanishlarni qayta ishlaydi va
SSE orqali frontendga xabar beradi.

---

## 4. HTTP Holat Kodlari

| Kod | Ma'nosi | Frontend amali |
|-----|---------|---------------|
| `200` | Muvaffaqiyatli | Ma'lumotni ko'rsating |
| `201` | Yangi resurs yaratildi | Ro'yxatni yangilang, muvaffaqiyat xabari ko'rsating |
| `204` | Javob tanasi yo'q (o'chirish) | Elementni UI dan olib tashlang |
| `400` | Noto'g'ri ma'lumot | `error.detail` xabarini formda ko'rsating |
| `401` | Token yo'q yoki muddati tugagan | Login sahifasiga yo'naltiring |
| `403` | Ruxsat yo'q | "Sizda bu amalni bajarishga ruxsat yo'q" |
| `404` | Topilmadi | "Ma'lumot topilmadi" xabarini ko'rsating |
| `500` | Server xatosi | "Server xatosi. Qaytadan urinib ko'ring" |

---

## 5. Javob Formati

API uchta turdagi javob qaytaradi:

### Sahifalangan (Paginated) — ro'yxat endpointlari uchun

```json
{
  "items": [ /* ma'lumotlar massivi */ ],
  "total": 100,
  "page": 1,
  "size": 25
}
```

Quyidagi hujjatda `PaginatedResponse<T>` TypeScript tipi ishlatiladi.
To'liq ta'rifi [18-bo'limda](#18-typescript-interfeyslari) bor, qisqacha:

```typescript
// Hujjat bo'ylab ishlatiladigan umumiy sahifalangan javob tipi
interface PaginatedResponse<T> {
  items: T[];    // ma'lumotlar massivi
  total: number; // jami elementlar soni (barcha sahifalarda)
  page: number;  // joriy sahifa raqami
  size: number;  // bir sahifadagi elementlar soni
}

// Jami sahifalar sonini hisoblash
const totalPages = Math.ceil(result.total / result.size);
const hasNextPage = result.page < totalPages;
```

### Sahifalanmagan ro'yxat

```json
[ /* ma'lumotlar massivi */ ]
```

### Bitta resurs

```json
{ /* resurs obyekti */ }
```

---

## 6. Barcha Endpointlar Jadvali

| Metod | Endpoint | Maqsad |
|-------|----------|--------|
| `POST` | `/accounts/` | Email akkount yaratish |
| `GET` | `/accounts/` | Akkountlar ro'yxati |
| `GET` | `/accounts/{account_id}` | Akkount tafsilotlari |
| `PATCH` | `/accounts/{account_id}/` | Akkountni yangilash |
| `DELETE` | `/accounts/{account_id}` | Akkountni uzish |
| `GET` | `/messages/` | Xabarlar ro'yxati (sahifalangan) |
| `GET` | `/messages/{message_id}` | Xabar tafsilotlari |
| `POST` | `/messages/send/` | Email yuborish |
| `POST` | `/messages/{message_id}/reply` | Xabarga javob berish |
| `GET` | `/inbox/` | Inbox elementlari (sahifalangan) |
| `GET` | `/inbox/unread-counts` | Har akkount bo'yicha o'qilmagan sonlar |
| `PATCH` | `/inbox/{inbox_id}/read` | O'qilgan deb belgilash |
| `PATCH` | `/inbox/{inbox_id}/unread` | O'qilmagan deb belgilash |
| `PATCH` | `/inbox/{inbox_id}/star` | Yulduzcha qo'yish/olib tashlash |
| `PATCH` | `/inbox/{inbox_id}/archive` | Arxivlash |
| `PATCH` | `/inbox/threads/{thread_id}/read` | Butun ipni o'qilgan belgilash |
| `GET` | `/threads/` | Iplar ro'yxati (sahifalangan) |
| `GET` | `/threads/{thread_id}` | Ip va uning xabarlari |
| `PATCH` | `/threads/{thread_id}` | Ipni yangilash |
| `GET` | `/attachments/{attachment_id}/download` | Yuklab olish URL-ini olish |
| `GET` | `/attachments/message/{message_id}` | Xabar qo'shimchalari |
| `GET` | `/webhooks/nylas?challenge=X` | Webhook tasdiqlash (backend) |
| `POST` | `/webhooks/nylas` | Nylas hodisalarini qabul qilish (backend) |
| `GET` | `/sse/events?token=X` | Real vaqt hodisa oqimi |

---

## 7. Akkountlar

Email akkountlar — tizimga ulangan Gmail, GoDaddy va boshqa pochta qutilari.

---

### 7.1 Yangi Email Akkount Yaratish

**Endpoint:** `POST /accounts/`
**Holat kodi:** `201 Created`

**Frontendda ishlatilishi:** Foydalanuvchi yangi pochta qutisini ulamoqchi bo'lganda.

```typescript
interface CreateAccountData {
  company_id: number;       // majburiy — qaysi kompaniyaga tegishli
  email_address: string;    // majburiy — noyob email manzil
  display_name?: string;    // ixtiyoriy — yuborilgan emaillarda ko'rinadigan ism
  provider?: string;        // ixtiyoriy — standart: "godaddy". "gmail" ham bo'lishi mumkin
  grant_id?: string;        // ixtiyoriy — Nylas grant identifikatori (quyida tushuntirish bor)
}

// grant_id nima va qayerdan olinadi?
//
// Nylas — tizim email provayderlar (Gmail, GoDaddy) bilan ulanish uchun
// ishlatadigan uchinchi tomon xizmat. Foydalanuvchi Nylas orqali
// o'z emailini ulashga ruxsat berganda, Nylas "grant_id" beradi.
//
// Odatdagi oqim:
//   1. Frontend: foydalanuvchini Nylas OAuth sahifasiga yo'naltiradi
//   2. Foydalanuvchi: o'z email hisobiga ruxsat beradi
//   3. Nylas: grant_id ni frontendga qaytaradi (redirect URL orqali)
//   4. Frontend: shu grant_id ni POST /accounts/ ga yuboradi
//
// grant_id ni qayerdan olish haqida backend dasturchi yoki Nylas
// integratsiyasi hujjatiga qarang.

async function createAccount(data: CreateAccountData): Promise<EmailAccount> {
  return apiRequest<EmailAccount>("/accounts/", {
    method: "POST",
    body: JSON.stringify(data),
  });
}

// Ishlatish
try {
  const account = await createAccount({
    company_id: 123,
    email_address: "support@acme.com",
    display_name: "Support Team",
    provider: "godaddy",
    grant_id: "grant_abc123xyz",
  });
  // account.status === "syncing" bo'ladi — sinxronizatsiya boshlangan
} catch (err) {
  // 400 → "Bu email allaqachon ro'yxatdan o'tgan"
  // 403 → "Siz bu kompaniyaga akkount qo'sha olmaysiz"
}
```

**Server qaytaradigan javob:**
```json
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
```

**Javob maydonlari:**
| Maydon | Tur | UI uchun izoh |
|--------|-----|---------------|
| `id` | UUID string | Keyingi so'rovlarda akkount identifikatori |
| `status` | string | `"active"` / `"syncing"` / `"disconnected"` — ko'rsatish uchun |
| `last_sync_at` | string \| null | Oxirgi sinxronizatsiya vaqti, `null` bo'lsa hali sinxronizatsiya bo'lmagan |
| `company` | object \| null | Kompaniya nomi va IDsi |

**`status` uchun UI:**
```typescript
const statusLabel = {
  active: "Faol",
  syncing: "Sinxronizatsiya...",
  disconnected: "Uzilgan",
};
const statusColor = {
  active: "green",
  syncing: "orange",
  disconnected: "red",
};
```

**Xatolar:**
- `400` → Email manzil yoki grant_id allaqachon mavjud
- `403` → Ko'rsatilgan `company_id` ga kirish huquqi yo'q

---

### 7.2 Email Akkountlar Ro'yxati

**Endpoint:** `GET /accounts/`
**Holat kodi:** `200 OK`
**Sahifalash:** Yo'q — barcha akkountlar bir yo'la qaytariladi.

```typescript
async function getAccounts(): Promise<EmailAccount[]> {
  return apiRequest<EmailAccount[]>("/accounts/");
}

// Akkount tanlash dropdownini to'ldirish uchun
const accounts = await getAccounts();
// Faqat foydalanuvchi kompaniyalariga tegishli akkountlar qaytariladi
```

---

### 7.3 Bitta Akkount Tafsilotlari

**Endpoint:** `GET /accounts/{account_id}`
**Holat kodi:** `200 OK`

```typescript
async function getAccount(accountId: string): Promise<EmailAccount> {
  return apiRequest<EmailAccount>(`/accounts/${accountId}`);
}
```

**Xato:**
- `404` → Akkount topilmadi yoki kirish huquqi yo'q

---

### 7.4 Akkountni Yangilash

**Endpoint:** `PATCH /accounts/{account_id}/`
**Holat kodi:** `200 OK`

Faqat o'zgartirmoqchi bo'lgan maydonlarni yuboring (partial update):

```typescript
interface UpdateAccountData {
  display_name?: string; // ko'rsatma nomini yangilash
  status?: string;       // holatni yangilash
}

async function updateAccount(
  accountId: string,
  data: UpdateAccountData
): Promise<EmailAccount> {
  return apiRequest<EmailAccount>(`/accounts/${accountId}/`, {
    method: "PATCH",
    body: JSON.stringify(data),
  });
}

// Faqat nomni o'zgartirish
await updateAccount("550e8400-...", { display_name: "Asosiy Support" });

// Faqat holatni o'zgartirish
await updateAccount("550e8400-...", { status: "active" });
```

---

### 7.5 Akkountni Uzish (O'chirish)

**Endpoint:** `DELETE /accounts/{account_id}`
**Holat kodi:** `204 No Content`

> **Muhim:** O'chiriladigan ma'lumot yo'q — xabarlar va iplar bazada saqlanib qoladi.
> Faqat akkount holati `"disconnected"` ga o'zgaradi.

```typescript
async function deleteAccount(accountId: string): Promise<void> {
  await apiRequest<null>(`/accounts/${accountId}`, { method: "DELETE" });
  // 204 qaytadi — javob tanasi yo'q
}

// UI da: o'chirishdan oldin tasdiqlash dialogi ko'rsating
const confirmed = window.confirm("Akkountni uzishni tasdiqlaysizmi?");
if (confirmed) {
  await deleteAccount("550e8400-...");
  // Akkountni ro'yxatdan olib tashlang
  setAccounts((prev) => prev.filter((a) => a.id !== accountId));
}
```

---

## 8. Xabarlar

Xabarlar — alohida emaillar (kiruvchi, chiquvchi, yuborilgan).

---

### 8.1 Xabarlar Ro'yxati (Sahifalangan)

**Endpoint:** `GET /messages/`
**Holat kodi:** `200 OK`

```typescript
interface GetMessagesParams {
  page?: number;        // standart: 1, min: 1
  size?: number;        // standart: 25
  account_id?: string;  // ixtiyoriy — muayyan akkount bo'yicha filtr
}

async function getMessages(
  params: GetMessagesParams = {}
): Promise<PaginatedResponse<Message>> {
  const query = new URLSearchParams();
  if (params.page)       query.set("page", String(params.page));
  if (params.size)       query.set("size", String(params.size));
  if (params.account_id) query.set("account_id", params.account_id);

  return apiRequest<PaginatedResponse<Message>>(`/messages/?${query}`);
}

// Barcha xabarlar (1-sahifa)
const result = await getMessages({ page: 1, size: 25 });
console.log(result.items);   // xabarlar massivi
console.log(result.total);   // jami xabarlar soni
console.log(result.page);    // joriy sahifa

// Muayyan akkount xabarlari
const accountMessages = await getMessages({
  account_id: "550e8400-...",
  page: 1,
});
```

**Server qaytaradigan javob:**
```json
{
  "items": [ /* Message obyektlari */ ],
  "total": 542,
  "page": 1,
  "size": 25
}
```

> Xabarlar `sent_at` bo'yicha kamayish tartibida (eng yangi birinchi).
> `account_id` ko'rsatilmasa — foydalanuvchining barcha akkountlaridan xabarlar qaytariladi.

---

### 8.2 Bitta Xabar Tafsilotlari

**Endpoint:** `GET /messages/{message_id}`
**Holat kodi:** `200 OK`

```typescript
async function getMessage(messageId: string): Promise<Message> {
  return apiRequest<Message>(`/messages/${messageId}`);
}

// Xabar ochilganda — tarkibini ko'rsatish uchun
const message = await getMessage("770e8400-...");
// message.body_html — HTML tarkibini iframe yoki sanitize qilib ko'rsating
// message.attachments — qo'shimchalar massivi (to'liq yuklangan)
```

> Bu endpointda `attachments` massivi ham to'liq qaytariladi.

**Xato:**
- `404` → Xabar topilmadi yoki kirish huquqi yo'q

---

### 8.3 Email Yuborish

**Endpoint:** `POST /messages/send/`
**Holat kodi:** `200 OK`

```typescript
interface SendEmailData {
  email_account_id: string;                    // majburiy — qaysi akkountdan yuborish
  to: { name?: string; email: string }[];      // majburiy — qabul qiluvchilar
  cc?: { name?: string; email: string }[];     // ixtiyoriy — CC
  bcc?: { name?: string; email: string }[];    // ixtiyoriy — BCC
  subject: string;                             // majburiy — mavzu
  body_html: string;                           // majburiy — HTML tarkib
  body_text?: string;                          // ixtiyoriy — oddiy matn (fallback)
  reply_to_message_id?: string;                // ixtiyoriy — javob beriladigan xabar ID
}

async function sendEmail(data: SendEmailData): Promise<Message> {
  return apiRequest<Message>("/messages/send/", {
    method: "POST",
    body: JSON.stringify(data),
  });
}

// Ishlatish
const sent = await sendEmail({
  email_account_id: "550e8400-e29b-41d4-a716-446655440000",
  to: [
    { name: "John Customer", email: "john@customer.com" }
  ],
  cc: [
    { name: "CC Shaxs", email: "cc@customer.com" }
  ],
  bcc: null,
  subject: "Re: Savolingiz",
  body_html: "<p>Murojaat uchun rahmat!</p>",
  body_text: "Murojaat uchun rahmat!",
  reply_to_message_id: null,
});
// sent — yangi yaratilgan Message obyekti
```

> **Nima bo'ladi yuborilgandan keyin:**
> - Ip (thread) yaratiladi yoki yangilanadi
> - Kompaniya foydalanuvchilarining inboxlariga taqsimlanadi
> - SSE orqali barcha ulangan foydalanuvchilarga xabar beriladi — frontendda inbox avtomatik yangilanadi

**Xato:**
- `403` → Email akkountning kompaniyasiga kirish huquqi yo'q

---

### 8.4 Xabarga Javob Berish

**Endpoint:** `POST /messages/{message_id}/reply`
**Holat kodi:** `200 OK`

```typescript
async function replyToMessage(
  messageId: string,
  data: SendEmailData
): Promise<Message> {
  return apiRequest<Message>(`/messages/${messageId}/reply`, {
    method: "POST",
    body: JSON.stringify(data),
  });
}

// Asl xabar ID si yo'l parametrida, qolgan ma'lumotlar body da
const reply = await replyToMessage("770e8400-...", {
  email_account_id: "550e8400-...",
  to: [{ email: "customer@external.com" }],
  subject: "Re: Narx haqida savol",
  body_html: "<p>Javob bu yerda...</p>",
});
// Yangi javob xuddi shu ipda (thread) yaratiladi
```

**Xato:**
- `404` → Asl xabar topilmadi yoki kirish huquqi yo'q

---

## 9. Inbox

Inbox — har bir foydalanuvchi uchun alohida. O'qilgan/o'qilmagan, yulduzcha va arxiv holatini saqlaydi.

### Inbox va Threads — qachon qaysinisini ishlatish kerak?

Bu ikki endpoint bir-biriga o'xshash ko'rinadi, lekin maqsadi farqli:

| | `GET /inbox/` | `GET /threads/` |
|---|---|---|
| **Maqsad** | Foydalanuvchining shaxsiy kiruvchi qutisi | Barcha email suhbatlar ro'yxati |
| **Nima qaytaradi** | InboxItem (o'qilganlik holati bilan) | ThreadSummary (faqat ip ma'lumoti) |
| **O'qilgan/O'qilmagan** | Ha — `is_read`, `is_starred`, `is_archived` | Yo'q — bu ma'lumotlar yo'q |
| **Filtr** | `is_read`, `account_id` | `account_id` |
| **Qachon ishlatiladi** | Asosiy inbox ekrani (Gmail'dagi kabi) | Barcha suhbatlar ro'yxati ko'rinishi |

**Amaliy qoida:**
- **Inbox ekrani** (o'qilmagan, yulduzcha, arxiv tugmalari bor) → `GET /inbox/` ishlatiladi
- **Threads ekrani** (faqat suhbat ro'yxati, holat tugmalari yo'q) → `GET /threads/` ishlatiladi
- **Ip ichini ochganda** (barcha xabarlarni ko'rish) → `GET /threads/{thread_id}` ishlatiladi
- **Ip ochilganda barcha xabarlarni o'qilgan qilish** → `PATCH /inbox/threads/{thread_id}/read` ishlatiladi

---

### 9.1 Inbox Elementlari Ro'yxati (Sahifalangan)

**Endpoint:** `GET /inbox/`
**Holat kodi:** `200 OK`

```typescript
interface GetInboxParams {
  page?: number;         // standart: 1, min: 1
  size?: number;         // standart: 25, maks: 100
  is_read?: boolean;     // false=faqat o'qilmagan, true=faqat o'qilgan, ko'rsatilmasa=ikkalasi
  account_id?: string;   // ixtiyoriy — muayyan akkount bo'yicha filtr
}

async function getInbox(
  params: GetInboxParams = {}
): Promise<PaginatedResponse<InboxItem>> {
  const query = new URLSearchParams();
  if (params.page)               query.set("page", String(params.page));
  if (params.size)               query.set("size", String(params.size));
  if (params.is_read !== undefined) query.set("is_read", String(params.is_read));
  if (params.account_id)         query.set("account_id", params.account_id);

  return apiRequest<PaginatedResponse<InboxItem>>(`/inbox/?${query}`);
}

// Barcha (arxivlanmagan) inboxni olish
const all = await getInbox({ page: 1, size: 25 });

// Faqat o'qilmagan xabarlar
const unread = await getInbox({ is_read: false });

// Muayyan akkountdan faqat o'qilmagan
const filtered = await getInbox({
  account_id: "550e8400-...",
  is_read: false,
});
```

> **`is_read` parametri haqida:**
> Bu parametr InboxItem'dagi `is_read` maydon qiymati bo'yicha filtrlaydi:
> - `is_read=false` → `is_read` maydoni `false` bo'lgan elementlar → **o'qilmagan** xabarlar
> - `is_read=true` → `is_read` maydoni `true` bo'lgan elementlar → **o'qilgan** xabarlar
> - Ko'rsatilmasa → ikkalasi ham qaytariladi
>
> Arxivlangan elementlar **har doim sukut bo'yicha** ro'yxatdan chiqarib tashlanadi.

**Server qaytaradigan javob (InboxItem):**
```json
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
    "subject": "Narx haqida savol",
    "snippet": "Salom, menda savol bor...",
    "direction": "inbound",
    "has_attachments": false,
    "sent_at": "2026-02-21T09:15:00Z"
  },
  "thread_subject": "Narx haqida savol",
  "thread_message_count": 2,
  "account_email": "support@acme.com"
}
```

**InboxItem maydonlari uchun UI:**

| Maydon | UI maqsadi |
|--------|-----------|
| `id` | PATCH so'rovlari uchun inbox yozuvi ID si |
| `is_read` | O'qilmagan → qalin shrift, o'qilgan → oddiy shrift |
| `is_starred` | Yulduzcha ikonka holati |
| `is_archived` | Bu false bo'lgan elementlarni ko'rsating (arxivlangan avtomatik filtrlanadi) |
| `read_at` | "O'qildi: soat 11:00" kabi ko'rsatish uchun |
| `message.snippet` | Ro'yxatda xabarning qisqa ko'rinishi |
| `message.direction` | `"inbound"` = kiruvchi, `"outbound"` = chiquvchi |
| `message.has_attachments` | Qo'shimcha ikonkasini ko'rsatish uchun |
| `thread_message_count` | "3 ta xabar" kabi ko'rsatish uchun |
| `account_email` | Qaysi akkountga kelganini ko'rsatish uchun |

---

### 9.2 O'qilmagan Sonlarni Olish (Badge uchun)

**Endpoint:** `GET /inbox/unread-counts`
**Holat kodi:** `200 OK`
**Parametrlar:** Yo'q

```typescript
interface UnreadCount {
  account_id: string;
  email_address: string;
  unread_count: number;
}

async function getUnreadCounts(): Promise<UnreadCount[]> {
  return apiRequest<UnreadCount[]>("/inbox/unread-counts");
}

// Sidebar yoki akkount tanlash menyusida badge ko'rsatish
const counts = await getUnreadCounts();
// [
//   { account_id: "...", email_address: "support@acme.com", unread_count: 5 },
//   { account_id: "...", email_address: "sales@acme.com",   unread_count: 12 }
// ]

// Badge ko'rsatish uchun
const totalUnread = counts.reduce((sum, c) => sum + c.unread_count, 0);
```

---

### 9.3 Inbox Yozuvini O'qilgan Deb Belgilash

**Endpoint:** `PATCH /inbox/{inbox_id}/read`
**Holat kodi:** `200 OK`
**So'rov tanasi:** Bo'sh `{}`

```typescript
async function markAsRead(inboxId: string): Promise<InboxEntry> {
  return apiRequest<InboxEntry>(`/inbox/${inboxId}/read`, {
    method: "PATCH",
    body: JSON.stringify({}),
  });
}

// Foydalanuvchi xabarni ochganda avtomatik chaqiring
const updated = await markAsRead("990e8400-...");
// updated.is_read === true
// updated.read_at === "2026-02-21T11:00:00Z"
```

---

### 9.4 Inbox Yozuvini O'qilmagan Deb Belgilash

**Endpoint:** `PATCH /inbox/{inbox_id}/unread`
**Holat kodi:** `200 OK`
**So'rov tanasi:** Bo'sh `{}`

```typescript
async function markAsUnread(inboxId: string): Promise<InboxEntry> {
  return apiRequest<InboxEntry>(`/inbox/${inboxId}/unread`, {
    method: "PATCH",
    body: JSON.stringify({}),
  });
}

// "O'qilmagan deb belgilash" tugmasi bosilganda
const updated = await markAsUnread("990e8400-...");
// updated.is_read === false
// updated.read_at === null
```

---

### 9.5 Yulduzcha Qo'yish / Olib Tashlash (Toggle)

**Endpoint:** `PATCH /inbox/{inbox_id}/star`
**Holat kodi:** `200 OK`
**So'rov tanasi:** Bo'sh `{}`

```typescript
async function toggleStar(inboxId: string): Promise<InboxEntry> {
  return apiRequest<InboxEntry>(`/inbox/${inboxId}/star`, {
    method: "PATCH",
    body: JSON.stringify({}),
  });
}

// Yulduzcha tugmasi bosilganda
const updated = await toggleStar("990e8400-...");
// updated.is_starred — oldingi qiymatning teskarisi (toggle)

// UI ni optimistik yangilash (server javobini kutmasdan)
setInboxItems((prev) =>
  prev.map((item) =>
    item.id === inboxId
      ? { ...item, is_starred: !item.is_starred }
      : item
  )
);
// Keyin server javobi bilan tasdiqlash
```

---

### 9.6 Inbox Yozuvini Arxivlash

**Endpoint:** `PATCH /inbox/{inbox_id}/archive`
**Holat kodi:** `200 OK`
**So'rov tanasi:** Bo'sh `{}`

```typescript
async function archiveItem(inboxId: string): Promise<InboxEntry> {
  return apiRequest<InboxEntry>(`/inbox/${inboxId}/archive`, {
    method: "PATCH",
    body: JSON.stringify({}),
  });
}

// Arxivlangandan keyin elementni ro'yxatdan olib tashlang
await archiveItem("990e8400-...");
setInboxItems((prev) => prev.filter((item) => item.id !== inboxId));
// updated.is_archived === true
// Arxivlangan elementlar standart /inbox/ so'rovida qaytmaydi
```

---

### 9.7 Butun Ipni O'qilgan Deb Belgilash

**Endpoint:** `PATCH /inbox/threads/{thread_id}/read`
**Holat kodi:** `200 OK`
**So'rov tanasi:** Bo'sh `{}`

```typescript
async function markThreadAsRead(
  threadId: string
): Promise<{ updated: number }> {
  return apiRequest<{ updated: number }>(
    `/inbox/threads/${threadId}/read`,
    { method: "PATCH", body: JSON.stringify({}) }
  );
}

// Ip sahifasini ochganda barcha xabarlarni o'qilgan deb belgilash
const result = await markThreadAsRead("880e8400-...");
console.log(`${result.updated} ta xabar o'qilgan deb belgilandi`);
// result.updated — o'qilgan deb belgilangan yozuvlar soni
```

---

## 10. Iplar (Threads)

Iplar — bir suhbatdagi bog'liq xabarlar guruhi.

---

### 10.1 Iplar Ro'yxati (Sahifalangan)

**Endpoint:** `GET /threads/`
**Holat kodi:** `200 OK`

```typescript
interface GetThreadsParams {
  account_id?: string; // ixtiyoriy — muayyan akkount bo'yicha filtr
  page?: number;       // standart: 1, min: 1
  size?: number;       // standart: 25, maks: 100
}

async function getThreads(
  params: GetThreadsParams = {}
): Promise<PaginatedResponse<ThreadSummary>> {
  const query = new URLSearchParams();
  if (params.account_id) query.set("account_id", params.account_id);
  if (params.page)       query.set("page", String(params.page));
  if (params.size)       query.set("size", String(params.size));

  return apiRequest<PaginatedResponse<ThreadSummary>>(`/threads/?${query}`);
}
```

**Server qaytaradigan javob:**
```json
{
  "items": [
    {
      "id": "880e8400-e29b-41d4-a716-446655440000",
      "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
      "subject": "Narx haqida savol",
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
```

> Iplar `last_message_at` bo'yicha kamayish tartibida (eng so'nggi faol ip birinchi).

**`do_not_delete` maydoni:** `true` bo'lsa — muhim/himoyalangan ip deb belgilangan, o'chirish tugmasini yashiring.

---

### 10.2 Ip Tafsilotlari va Xabarlari

**Endpoint:** `GET /threads/{thread_id}`
**Holat kodi:** `200 OK`

```typescript
async function getThread(threadId: string): Promise<ThreadDetail> {
  return apiRequest<ThreadDetail>(`/threads/${threadId}`);
}

// Ip sahifasini ochganda
const thread = await getThread("880e8400-...");

// thread.messages — barcha xabarlar, ENG ESKI BIRINCHI (vaqt o'sish tartibida)
// UI da suhbat ko'rinishi uchun: pastdan yuqoriga yoki yuqoridan pastga
// Har bir message ichida attachments massivi ham to'liq bor
```

**Server qaytaradigan javob:**
```json
{
  "id": "880e8400-e29b-41d4-a716-446655440000",
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "subject": "Narx haqida savol",
  "last_message_at": "2026-02-21T10:00:00Z",
  "message_count": 3,
  "do_not_delete": false,
  "created_at": "2026-02-21T08:00:00Z",
  "messages": [
    /* MessageResponse massivi — eng eski birinchi */
  ]
}
```

**Xato:**
- `404` → Ip topilmadi yoki kirish huquqi yo'q

---

### 10.3 Ipni Yangilash

**Endpoint:** `PATCH /threads/{thread_id}`
**Holat kodi:** `200 OK`

```typescript
async function updateThread(
  threadId: string,
  data: { do_not_delete?: boolean }
): Promise<ThreadDetail> {
  return apiRequest<ThreadDetail>(`/threads/${threadId}`, {
    method: "PATCH",
    body: JSON.stringify(data),
  });
}

// Ipni muhim deb belgilash (o'chishdan himoya)
await updateThread("880e8400-...", { do_not_delete: true });

// Himoyani olib tashlash
await updateThread("880e8400-...", { do_not_delete: false });
```

> Hozircha faqat `do_not_delete` yangilanishi mumkin.

---

## 11. Qo'shimchalar (Attachments)

---

### 11.1 Xabar Qo'shimchalari Ro'yxati

**Endpoint:** `GET /attachments/message/{message_id}`
**Holat kodi:** `200 OK`

```typescript
async function getAttachments(messageId: string): Promise<Attachment[]> {
  return apiRequest<Attachment[]>(`/attachments/message/${messageId}`);
}

const attachments = await getAttachments("770e8400-...");
```

**Server qaytaradigan javob:**
```json
[
  {
    "id": "aa0e8400-e29b-41d4-a716-446655440000",
    "message_id": "770e8400-e29b-41d4-a716-446655440000",
    "filename": "hisob-faktura.pdf",
    "content_type": "application/pdf",
    "size_bytes": 123456,
    "is_inline": false,
    "created_at": "2026-02-21T08:16:00Z"
  }
]
```

**Maydonlar uchun UI:**

| Maydon | UI maqsadi |
|--------|-----------|
| `filename` | Fayl nomini ko'rsating |
| `content_type` | Fayl turini ikonka bilan ko'rsating (`application/pdf` → PDF ikonka) |
| `size_bytes` | O'qilishi qulay formatda: KB, MB |
| `is_inline` | `true` → email tanasiga joylashtirilgan (odatda ko'rsatmang); `false` → yuklab olinadigan qo'shimcha |

```typescript
// size_bytes ni o'qilishi qulay formatga aylantirish
function formatFileSize(bytes: number): string {
  if (bytes < 1024)             return `${bytes} B`;
  if (bytes < 1024 * 1024)      return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}
// formatFileSize(123456) → "120.6 KB"

// Faqat inline bo'lmaganlarni ko'rsating
const visibleAttachments = attachments.filter((a) => !a.is_inline);
```

---

### 11.2 Qo'shimchani Yuklab Olish URL-ini Olish

**Endpoint:** `GET /attachments/{attachment_id}/download`
**Holat kodi:** `200 OK`

```typescript
async function getDownloadUrl(
  attachmentId: string
): Promise<{ download_url: string }> {
  return apiRequest<{ download_url: string }>(
    `/attachments/${attachmentId}/download`
  );
}
```

**Server qaytaradigan javob:**
```json
{
  "download_url": "https://s3.amazonaws.com/bucket/attachments/path/to/file.pdf?AWSAccessKeyId=...&Signature=...&Expires=..."
}
```

> `download_url` — oldindan imzolangan AWS S3 URL. Taxminan **1 soat** amal qiladi.
> Saqlab qo'ymang — har safar yangi URL oling.

**Brauzerda yuklab olishni boshlash:**

```typescript
async function downloadAttachment(
  attachmentId: string,
  filename: string
): Promise<void> {
  // 1-qadam: presigned URL olish
  const { download_url } = await getDownloadUrl(attachmentId);

  // 2-qadam: brauzerda yuklab olishni boshlash
  const link = document.createElement("a");
  link.href = download_url;
  link.download = filename; // fayl nomi
  link.target = "_blank";
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

// Yuklab olish tugmasi bosilganda
await downloadAttachment("aa0e8400-...", "hisob-faktura.pdf");
```

---

## 12. Webhooklar

> **Frontend uchun:** Bu bo'limdagi endpointlarni frontenddan **chaqirmaysiz**.
> Webhooklar — Nylas (email xizmati) va backend server o'rtasida to'g'ridan-to'g'ri ishlaydi.
> Backend webhook orqali kelgan yangilanishni qayta ishlab, SSE orqali frontendga xabar beradi.
> Bu bo'limni tushunish majburiy emas, lekin quyida qanday ishlashi tushuntirilgan.

---

### 12.1 Webhook Challenge (Tasdiqlash)

**Endpoint:** `GET /webhooks/nylas`
**Auth:** Yo'q

Nylas webhook ro'yxatdan o'tkazishda bu endpointni chaqirib, backend serverining egaliligini tekshiradi. Backend `challenge` parametrini aynan qaytaradi.

```
GET /webhooks/nylas?challenge=abc123
→ Javob: abc123  (text/plain)
```

---

### 12.2 Webhook Hodisalarini Qabul Qilish

**Endpoint:** `POST /webhooks/nylas`
**Auth:** `x-nylas-signature` sarlavhasi orqali imzo tekshiruvi

Nylas yangi email kelganda yoki o'zgarganda bu endpointga POST so'rov yuboradi.
Backend bu hodisani qayta ishlab, SSE orqali frontendga `message.created` hodisasi yuboradi.

Webhook payload shakli (`message.created` uchun):
```json
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
      "subject": "Yangi murojaat",
      "from": [{ "email": "customer@external.com", "name": "John Customer" }],
      "to": [{ "email": "support@acme.com", "name": "Support" }],
      "cc": null,
      "bcc": null,
      "date": 1708937400,
      "body": "Salom, menda savol bor...",
      "snippet": "Salom, menda savol bor...",
      "attachments": [],
      "folders": ["inbox"]
    }
  }
}
```

Backend muvaffaqiyatli qabul qilganda: `{ "status": "ok" }`
Imzo noto'g'ri bo'lsa: `401 Unauthorized`

**Hodisa turlari (backenddan SSE orqali frontendga keladi):**
- `message.created` — yangi xabar keldi
- `message.updated` — xabar o'zgartirildi
- `grant.expired` — akkount muddati tugadi (foydalanuvchiga ogohlantirish ko'rsating)
- va boshqalar

---

## 13. SSE — Real Vaqt Yangilanishlar

SSE (Server-Sent Events) — server yangi hodisa bo'lganda frontendga avtomatik push notification yuboradi.
Har N sekundda polling o'rniga — doimiy ulanish, server xabar beradi.

---

### 13.1 SSE Hodisa Oqimiga Ulanish

**Endpoint:** `GET /sse/events`
**Auth:** URL parametrida JWT (`?token=<jwt>`)
**Media turi:** `text/event-stream` (uzluksiz oqim)

```typescript
function connectSSE(
  onMessageCreated: (data: SSEEventData) => void,
  onMessageUpdated: (data: SSEEventData) => void,
  onThreadCreated?: (data: SSEEventData) => void,
  onThreadUpdated?: (data: SSEEventData) => void
): EventSource {
  const token = TokenStorage.get();
  const sse = new EventSource(`${BASE_URL}/sse/events?token=${token}`);

  // Ulanish o'rnatildi
  sse.addEventListener("connected", (e: MessageEvent) => {
    const data = JSON.parse(e.data);
    // data = { message: "connected", user_id: 123 }
    console.log("SSE ulandi:", data.user_id);
  });

  // Yangi xabar keldi
  sse.addEventListener("message.created", (e: MessageEvent) => {
    const data: SSEEventData = JSON.parse(e.data);
    onMessageCreated(data);
  });

  // Xabar yangilandi
  sse.addEventListener("message.updated", (e: MessageEvent) => {
    const data: SSEEventData = JSON.parse(e.data);
    onMessageUpdated(data);
  });

  // Yangi ip yaratildi
  sse.addEventListener("thread.created", (e: MessageEvent) => {
    const data: SSEEventData = JSON.parse(e.data);
    onThreadCreated?.(data);
  });

  // Ip yangilandi
  sse.addEventListener("thread.updated", (e: MessageEvent) => {
    const data: SSEEventData = JSON.parse(e.data);
    onThreadUpdated?.(data);
  });

  // Ulanish xatosi (token muddati tugagan, internet uzilgan va h.k.)
  sse.onerror = () => {
    console.warn("SSE ulanishi uzildi");
    sse.close();
  };

  return sse;
}

// Avtomatik qayta ulanish (token yangilangandan keyin)
function connectSSEWithRetry(
  onMessageCreated: (data: SSEEventData) => void,
  onMessageUpdated: (data: SSEEventData) => void
): () => void {
  let sseConn: EventSource | null = null;
  let retryTimer: ReturnType<typeof setTimeout> | null = null;

  function connect() {
    const token = TokenStorage.get();
    if (!token) return; // Token yo'q — login sahifasiga yo'naltirish kerak

    sseConn = new EventSource(`${BASE_URL}/sse/events?token=${token}`);

    sseConn.addEventListener("connected", () => {
      console.log("SSE ulandi");
    });

    sseConn.addEventListener("message.created", (e: MessageEvent) => {
      onMessageCreated(JSON.parse(e.data));
    });

    sseConn.addEventListener("message.updated", (e: MessageEvent) => {
      onMessageUpdated(JSON.parse(e.data));
    });

    sseConn.onerror = () => {
      sseConn?.close();
      // 5 soniyadan keyin qayta urinish
      retryTimer = setTimeout(connect, 5000);
    };
  }

  connect();

  // Tozalash funksiyasini qaytaradi
  return () => {
    if (retryTimer) clearTimeout(retryTimer);
    sseConn?.close();
  };
}
```

**SSE hodisasi ma'lumotlari tuzilmasi:**

```json
{
  "type": "message.created",
  "thread_id": "880e8400-e29b-41d4-a716-446655440000",
  "message_id": "770e8400-e29b-41d4-a716-446655440000",
  "company_id": 123
}
```

**SSE hodisa turlari:**

| Hodisa | Ma'lumot | Frontend amali |
|--------|---------|----------------|
| `connected` | `{ message, user_id }` | Ulanish tasdiqlandi |
| `message.created` | `{ type, thread_id, message_id, company_id }` | Inbox yangilang, badge sonini yangilang |
| `message.updated` | `{ type, thread_id, message_id, company_id }` | Shu xabarni qayta yuklang |
| `thread.created` | `{ type, thread_id, message_id, company_id }` | Iplar ro'yxatini yangilang |
| `thread.updated` | `{ type, thread_id, message_id, company_id }` | Shu ipni qayta yuklang |
| `:` (keepalive) | — | Hech narsa qilmang — ulanish tirik ekanini bildiradi, 15 soniyada bir keladi |

**React ilovada ishlatish:**

```typescript
useEffect(() => {
  // Avtomatik qayta ulanishli versiya
  const disconnect = connectSSEWithRetry(
    // message.created — yangi xabar keldi
    async (data) => {
      // Inboxni yangilash
      const freshInbox = await getInbox({ page: 1 });
      setInboxItems(freshInbox.items);
      // Badge sonlarini yangilash
      const counts = await getUnreadCounts();
      setUnreadCounts(counts);
    },
    // message.updated — xabar o'zgartirildi
    async (data) => {
      // Faqat shu xabarni qayta yuklash
      const msg = await getMessage(data.message_id);
      setMessages((prev) =>
        prev.map((m) => (m.id === data.message_id ? msg : m))
      );
    }
  );

  // Komponent unmount bo'lganda ulanishni yoping
  return () => disconnect();
}, []);
```

> **Server tomonida filtrlash:** Server faqat sizning `companies` ro'yxatingizga tegishli hodisalarni yuboradi. Frontendda qo'shimcha filtr kerak emas.
> **Token muddati tugaganda** yoki **mijoz uzilganda** — ulanish avtomatik yopiladi.

---

## 14. To'liq Sxemalar

Barcha API javoblarida uchraydigan ma'lumot tuzilmalari.

---

### CompanyInfo
```json
{
  "id": 123,
  "name": "Acme Corporation"
}
```

---

### EmailRecipient
```json
{
  "name": "John Doe",
  "email": "john@example.com"
}
```
> `name` — ixtiyoriy, `email` — majburiy va to'g'ri email formatida.

---

### EmailAccountResponse (to'liq)
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "company_id": 123,
  "email_address": "support@acme.com",
  "display_name": "Support Team",
  "provider": "godaddy",
  "status": "active",
  "last_sync_at": "2026-02-21T10:30:00Z",
  "created_at": "2026-02-21T10:30:00Z",
  "company": { "id": 123, "name": "Acme Corporation" }
}
```

---

### MessageResponse (to'liq)
```json
{
  "id": "770e8400-e29b-41d4-a716-446655440000",
  "thread_id": "880e8400-e29b-41d4-a716-446655440000",
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "internet_message_id": "<msg.1234567890@acme.com>",
  "from_address": "sender@example.com",
  "from_name": "Sender Name",
  "to_addresses": [{ "name": "Recipient", "email": "to@example.com" }],
  "cc_addresses": null,
  "bcc_addresses": null,
  "subject": "Mavzu",
  "body_html": "<p>Email tarkibi</p>",
  "body_text": "Email tarkibi",
  "snippet": "Email tarkibi...",
  "direction": "inbound",
  "has_attachments": false,
  "sent_at": "2026-02-21T09:15:00Z",
  "created_at": "2026-02-21T09:16:00Z",
  "attachments": []
}
```

**Frontendda e'tibor bering:**
- `body_html` — HTML tarkib. XSS dan himoya qilish uchun albatta sanitize qiling (`DOMPurify` tavsiya etiladi) yoki `iframe` ichida ko'rsating.
- `snippet` — Ro'yxat ko'rinishida qisqa matn uchun.
- `direction` — `"inbound"` (kelgan) yoki `"outbound"` (yuborilgan).
- `internet_message_id` — Email standartidagi texnik ID. UI da ko'rsatilmaydi, backend uchun.

### `null` maydonlarni boshqarish

`Message` obyektidagi bir qancha maydon `null` bo'lishi mumkin. UI da ko'rsatishdan oldin tekshiring:

```typescript
// from_name null bo'lishi mumkin — email manzilni fallback sifatida ishlating
function getSenderName(message: Message): string {
  return message.from_name ?? message.from_address;
  // "John Customer" yoki "john@customer.com"
}

// body_html null bo'lishi mumkin — body_text ga fallback
function getMessageBody(message: Message): string {
  return message.body_html ?? message.body_text ?? "(Xabar tarkibi yo'q)";
}

// body_html ko'rsatishda XSS himoyasi (DOMPurify kutubxonasi bilan)
// npm install dompurify @types/dompurify
import DOMPurify from "dompurify";

function safeHtml(html: string | null): string {
  if (!html) return "<p>(Xabar tarkibi yo'q)</p>";
  return DOMPurify.sanitize(html);
}

// React da ishlatish:
// <div dangerouslySetInnerHTML={{ __html: safeHtml(message.body_html) }} />

// snippet null bo'lishi mumkin
function getSnippet(message: Message): string {
  return message.snippet ?? message.body_text?.slice(0, 100) ?? "";
}

// cc_addresses va bcc_addresses null bo'lishi mumkin
function formatRecipients(
  recipients: { name?: string; email: string }[] | null
): string {
  if (!recipients || recipients.length === 0) return "";
  return recipients.map((r) => r.name ?? r.email).join(", ");
}

// last_sync_at null bo'lishi mumkin (hali sinxronizatsiya bo'lmagan)
function getSyncStatus(account: EmailAccount): string {
  if (!account.last_sync_at) return "Hali sinxronizatsiya bo'lmagan";
  return `Oxirgi sinxronizatsiya: ${timeAgo(account.last_sync_at)}`;
}

// read_at null bo'lishi mumkin (hali o'qilmagan)
function getReadTime(entry: InboxEntry): string {
  if (!entry.read_at) return "O'qilmagan";
  return `O'qildi: ${formatDateTime(entry.read_at)}`;
}
```

---

### AttachmentResponse (to'liq)
```json
{
  "id": "aa0e8400-e29b-41d4-a716-446655440000",
  "message_id": "770e8400-e29b-41d4-a716-446655440000",
  "filename": "hisob-faktura.pdf",
  "content_type": "application/pdf",
  "size_bytes": 123456,
  "is_inline": false,
  "created_at": "2026-02-21T08:16:00Z"
}
```

---

### InboxEntryResponse (to'liq)
```json
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
```

---

### ThreadDetailResponse (to'liq)
```json
{
  "id": "880e8400-e29b-41d4-a716-446655440000",
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "subject": "Narx haqida savol",
  "last_message_at": "2026-02-21T10:00:00Z",
  "message_count": 3,
  "do_not_delete": false,
  "created_at": "2026-02-21T08:00:00Z",
  "messages": [ /* MessageResponse massivi — eng eski birinchi */ ]
}
```

---

### SendEmailRequest (yuborish va javob berish uchun)
```json
{
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "to": [{ "name": "Recipient", "email": "to@example.com" }],
  "cc": [{ "name": "CC", "email": "cc@example.com" }],
  "bcc": null,
  "subject": "Salom",
  "body_html": "<p>Salom</p>",
  "body_text": "Salom",
  "reply_to_message_id": null
}
```

---

## 15. Xatolarni Boshqarish

### Xato javoblari formati

Barcha xatolar quyidagi formatda keladi:

```json
{ "detail": "Xato tavsifi" }
```

**Haqiqiy server xato xabarlari:**
```json
{ "detail": "Email account with this email address/grant already exists." }
{ "detail": "Invalid or expired token" }
{ "detail": "No access to this email account" }
{ "detail": "Message not found" }
```

---

### Global xato boshqaruvchi

```typescript
interface ApiError {
  status: number;
  detail: string;
}

function handleApiError(error: unknown): void {
  const err = error as ApiError;

  switch (err.status) {
    case 400:
      // Forma validatsiya xatosi — detail ni ko'rsating
      showNotification(err.detail, "error");
      break;

    case 401:
      // Token yo'q yoki muddati tugagan — login sahifasiga yo'naltiring
      TokenStorage.remove();
      window.location.href = "/login";
      break;

    case 403:
      showNotification("Sizda bu amalni bajarishga ruxsat yo'q", "warning");
      break;

    case 404:
      showNotification("Ma'lumot topilmadi", "info");
      break;

    case 500:
      showNotification("Server xatosi. Keyinroq urinib ko'ring", "error");
      break;

    default:
      showNotification("Kutilmagan xato yuz berdi", "error");
  }
}

// Barcha API chaqiruvlarini try/catch bilan o'rang
async function safeGetInbox() {
  try {
    return await getInbox();
  } catch (err) {
    handleApiError(err);
    return null;
  }
}
```

---

## 16. Sahifalash (Pagination)

Barcha ro'yxat endpointlari `page` va `size` parametrlarini qabul qiladi.

**Offset formulasi:**
```
offset = (sahifa - 1) * o'lcham
```

**2-sahifani olish (har sahifada 25 ta):**
```
/messages/?page=2&size=25
// offset = (2 - 1) * 25 = 25 → birinchi 25 tani o'tkazib, keyingi 25 tani ol
```

**Maksimal `size` qiymatlari:**

| Endpoint | Standart | Maksimal |
|----------|----------|---------|
| `GET /messages/` | 25 | cheksiz (amalda 25 tavsiya) |
| `GET /inbox/` | 25 | **100** |
| `GET /threads/` | 25 | **100** |
| `GET /accounts/` | — | sahifalash yo'q |

**Jami sahifalar soni:**
```json
{ "total": 542, "page": 1, "size": 25 }
// maks_sahifalar = ceil(542 / 25) = 22
```

### Sahifalash logikasi

```typescript
function getPaginationInfo(response: PaginatedResponse<unknown>) {
  const totalPages = Math.ceil(response.total / response.size);
  return {
    currentPage: response.page,
    totalPages,
    totalItems: response.total,
    pageSize: response.size,
    hasPrevPage: response.page > 1,
    hasNextPage: response.page < totalPages,
  };
}
```

### Cheksiz scroll (Infinite Scroll)

```typescript
let page = 1;
let loading = false;
let hasMore = true;

async function loadMoreMessages() {
  if (loading || !hasMore) return;
  loading = true;

  const result = await getMessages({ page, size: 25 });
  const { totalPages } = getPaginationInfo(result);

  appendToList(result.items);

  hasMore = page < totalPages;
  page++;
  loading = false;
}

// Scroll pastiga yetganda chaqiring
window.addEventListener("scroll", () => {
  const nearBottom =
    window.innerHeight + window.scrollY >= document.body.offsetHeight - 300;
  if (nearBottom) loadMoreMessages();
});
```

---

## 17. Ruxsat va Kirish Nazorati

**Frontend uchun asosiy tushuncha:**

Barcha endpointlar kompaniya darajasida kirish nazoratini amalga oshiradi.
Bu nazorat server tomonida bajariladi — frontendda qo'shimcha filtr yozish shart emas.

**Qanday ishlaydi:**
1. Foydalanuvchi JWT tokeni `companies: [1, 2, 5]` ro'yxatini o'z ichiga oladi
2. Server faqat shu kompaniyalarga tegishli resurslarni qaytaradi:
   - **Akkountlar** — `account.company_id` foydalanuvchi kompaniyalarida bo'lsa
   - **Xabarlar** — `message.email_account.company_id` foydalanuvchi kompaniyalarida bo'lsa
   - **Iplar** — `thread.email_account.company_id` foydalanuvchi kompaniyalarida bo'lsa
   - **Inbox** — faqat foydalanuvchining o'z inbox yozuvlari

**Xato javoblari:**
- `403 Forbidden` — Foydalanuvchi kompaniya/resursga kirish huquqiga ega emas
- `404 Not Found` — Resursga kirish imkoni yo'q (ma'lumot sizib chiqmasligi uchun 403 o'rniga 404 qaytariladi)

> **Muhim:** `404` xatosi resurs mavjud emas degani ham, ruxsat yo'q degani ham bo'lishi mumkin.
> Ikkalasi uchun ham "Topilmadi" xabarini ko'rsating.

---

## 18. TypeScript Interfeyslari

Barcha API javoblari uchun tayyor TypeScript tiplari:

```typescript
// Kompaniya
interface Company {
  id: number;
  name: string;
}

// Email qabul qiluvchi
interface EmailRecipient {
  name?: string;
  email: string;
}

// Email akkount
interface EmailAccount {
  id: string;
  company_id: number;
  email_address: string;
  display_name: string | null;
  provider: string;
  status: "active" | "syncing" | "disconnected";
  last_sync_at: string | null;  // ISO 8601 vaqt
  created_at: string;
  company: Company | null;
}

// Qo'shimcha
interface Attachment {
  id: string;
  message_id: string;
  filename: string;
  content_type: string;  // MIME turi, masalan "application/pdf"
  size_bytes: number;
  is_inline: boolean;    // true = email tanasiga joylashtirilgan
  created_at: string;
}

// Xabar (to'liq)
interface Message {
  id: string;
  thread_id: string;
  email_account_id: string;
  internet_message_id: string;
  from_address: string;
  from_name: string | null;
  to_addresses: EmailRecipient[];
  cc_addresses: EmailRecipient[] | null;
  bcc_addresses: EmailRecipient[] | null;
  subject: string;
  body_html: string | null;   // sanitize qiling!
  body_text: string | null;
  snippet: string | null;
  direction: "inbound" | "outbound";
  has_attachments: boolean;
  sent_at: string;
  created_at: string;
  attachments: Attachment[];
}

// Sahifalangan javob (umumiy generic tip)
interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  size: number;
}

// Inbox yozuvi (PATCH operatsiyalari javobida)
interface InboxEntry {
  id: string;
  user_id: number;
  message_id: string;
  thread_id: string;
  email_account_id: string;
  is_read: boolean;
  is_starred: boolean;
  is_archived: boolean;
  read_at: string | null;
  created_at: string;
}

// Inbox elementi (GET /inbox/ ro'yxatida)
interface InboxItem {
  id: string;
  is_read: boolean;
  is_starred: boolean;
  is_archived: boolean;
  read_at: string | null;
  created_at: string;
  message: {
    id: string;
    thread_id: string;
    from_address: string;
    from_name: string | null;
    subject: string;
    snippet: string | null;
    direction: "inbound" | "outbound";
    has_attachments: boolean;
    sent_at: string;
  };
  thread_subject: string;
  thread_message_count: number;
  account_email: string;
}

// O'qilmagan son
interface UnreadCount {
  account_id: string;
  email_address: string;
  unread_count: number;
}

// Ip xulasasi (GET /threads/ ro'yxatida)
interface ThreadSummary {
  id: string;
  email_account_id: string;
  subject: string;
  last_message_at: string;
  message_count: number;
  do_not_delete: boolean;
  created_at: string;
}

// Ip tafsiloti (GET /threads/{id} da)
interface ThreadDetail extends ThreadSummary {
  messages: Message[];  // eng eski birinchi
}

// SSE hodisa ma'lumotlari
interface SSEEventData {
  type: string;
  thread_id: string;
  message_id: string;
  company_id: number;
}

// SSE ulanish hodisasi
interface SSEConnectedData {
  message: string;  // "connected"
  user_id: number;
}

// API xatosi
interface ApiError {
  status: number;
  detail: string;
}

// Email yuborish so'rovi
interface SendEmailRequest {
  email_account_id: string;
  to: EmailRecipient[];
  cc?: EmailRecipient[] | null;
  bcc?: EmailRecipient[] | null;
  subject: string;
  body_html: string;
  body_text?: string | null;
  reply_to_message_id?: string | null;
}

// Akkount yaratish so'rovi
interface CreateAccountRequest {
  company_id: number;
  email_address: string;
  display_name?: string;
  provider?: string;
  grant_id?: string;
}
```

---

## Tezkor Nazorat Ro'yxati

Ilovani ishga tushirishdan oldin mana shu narsalarni tekshiring:

**Sozlash:**
- [ ] `BASE_URL` to'g'ri sozlangan
- [ ] JWT token `localStorage` da saqlanadi va `apiRequest` ga har so'rovda `Authorization` sarlavhasi sifatida qo'shiladi
- [ ] `showNotification()` funksiyasi UI kutubxonangizga ulangan (toast, snackbar va h.k.)
- [ ] CORS muammosi bo'lsa backend dasturchi bilan hal qilingan

**Autentifikatsiya:**
- [ ] `401` xatosida token o'chiriladi va login sahifasiga yo'naltiriladi
- [ ] `403` va `404` xatolarida foydalanuvchiga tushunarli xabar ko'rsatiladi

**Xabarlar va Inbox:**
- [ ] `body_html` ko'rsatishda XSS dan himoya qilingan (DOMPurify yoki `<iframe sandbox>`)
- [ ] `body_html` null bo'lganda `body_text` ga fallback qilingan
- [ ] `from_name` null bo'lganda `from_address` ko'rsatiladi
- [ ] `snippet` null bo'lganda `body_text`dan olinadi yoki bo'sh qoldiriladi
- [ ] ISO 8601 vaqt satrlari (`"2026-02-21T09:15:00Z"`) formatlanib ko'rsatiladi
- [ ] Inbox ekrani uchun `GET /inbox/` ishlatiladi (o'qilganlik holati kerak bo'lganda)
- [ ] Threads ekrani uchun `GET /threads/` ishlatiladi (faqat suhbat ro'yxati kerak bo'lganda)
- [ ] Inbox ro'yxatida arxivlangan elementlar ko'rsatilmaydi (server avtomatik filtrlab qaytaradi)
- [ ] O'qilmagan xabarlar uchun `getInbox({ is_read: false })` ishlatiladi
- [ ] Badge uchun `GET /inbox/unread-counts` ishlatiladi
- [ ] Xabar ochilganda `PATCH /inbox/{id}/read` chaqiriladi

**Qo'shimchalar:**
- [ ] Qo'shimcha yuklab olishda avval `GET /attachments/{id}/download` orqali URL olinadi (to'g'ridan-to'g'ri URL saqlanmaydi, 1 soatda muddati tugaydi)
- [ ] `is_inline: true` bo'lgan qo'shimchalar alohida "yuklab olish" tugmasi ko'rsatilmaydi

**Iplar:**
- [ ] `GET /threads/{id}` javobida xabarlar **eng eski birinchi** keladi — suhbat ko'rinishida to'g'ri tartibda ko'rsating

**Real vaqt (SSE):**
- [ ] SSE `EventSource` ulangan va `message.created` hodisasi inboxni yangilaydi
- [ ] SSE avtomatik qayta ulanishi (`connectSSEWithRetry`) ishlatilgan
- [ ] SSE komponent yo'q qilinganda (`unmount`) `disconnect()` chaqiriladi

**Sahifalash:**
- [ ] `/inbox/` va `/threads/` uchun maksimal `size: 100` hisobga olingan
- [ ] Sahifalar soni `Math.ceil(total / size)` bilan hisoblanadi
