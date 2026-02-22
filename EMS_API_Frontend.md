# EMS API — Frontend Dasturchi Uchun To'liq Qo'llanma

> **Oxirgi yangilanish:** Fevral 2026
> **Til:** Uzbek
> **Uchun:** React / TypeScript frontend dasturchilari

---

## Mundarija

1. [Tizim haqida umumiy ma'lumot](#1-tizim-haqida-umumiy-malumot)
2. [Autentifikatsiya — JWT token](#2-autentifikatsiya--jwt-token)
3. [Asosiy sozlamalar — apiRequest funksiyasi](#3-asosiy-sozlamalar--apirequest-funksiyasi)
4. [TypeScript turlari — barcha interface'lar](#4-typescript-turlari--barcha-interfacelar)
5. [Hisoblar (Accounts) API](#5-hisoblar-accounts-api)
6. [Xabarlar (Messages) API](#6-xabarlar-messages-api)
7. [Kiruvchi quti (Inbox) API](#7-kiruvchi-quti-inbox-api)
8. [Mavzular (Threads) API](#8-mavzular-threads-api)
9. [Qo'shimcha fayllar (Attachments) API](#9-qoshimcha-fayllar-attachments-api)
10. [Webhooks — Nylas hodisalari](#10-webhooks--nylas-hodisalari)
11. [SSE — Real-time hodisalar](#11-sse--real-time-hodisalar)
12. [Xatoliklarni boshqarish](#12-xatoliklarni-boshqarish)
13. [Sahifalash (Pagination)](#13-sahifalash-pagination)
14. [Ruxsat va kirish nazorati](#14-ruxsat-va-kirish-nazorati)
15. [Null maydonlarni xavfsiz boshqarish](#15-null-maydonlarni-xavfsiz-boshqarish)
16. [Barcha endpoint'lar jadvali](#16-barcha-endpointlar-jadvali)

---

## 1. Tizim haqida umumiy ma'lumot

**EMS (Email Management System)** — bu kompaniyaning email hisoblarini boshqarish tizimi. Frontend dasturchi ushbu API orqali quyidagilarni amalga oshiradi:

| Imkoniyat | Tavsif |
|-----------|--------|
| Email hisoblarni ulash | Gmail, GoDaddy va boshqa provayderlarni ulash |
| Xabarlar bilan ishlash | Xabarlarni o'qish, yuborish, javob berish |
| Kiruvchi qutini boshqarish | O'qish, yulduz qo'yish, arxivlash |
| Mavzular (threads) | Xabarlar zanjirini ko'rish |
| Fayllarni yuklab olish | Biriktirilgan fayllarni S3 orqali olish |
| Real-time yangilanishlar | SSE (Server-Sent Events) orqali jonli bildirishnomalar |

### Muhim texnik tushunchalar

**Nylas** — bu tizim email provayderlar (Gmail, GoDaddy va h.k.) bilan bog'lanish uchun uchinchi tomon xizmati. `grant_id` — bu Nylas OAuth jarayoni orqali olinadigan unikal identifikator bo'lib, backend tomonidan boshqariladi. Frontend dasturchi odatda `grant_id` ni bevosida yaratmaydi.

**UUID** — barcha resurslar (hisoblar, xabarlar, mavzular, qo'shimchalar) UUID formatidagi `id` bilan identifikatsiya qilinadi.
Misol: `"550e8400-e29b-41d4-a716-446655440000"`

**Kompaniya darajasida filtrlash** — JWT token'da foydalanuvchining kompaniyalari ro'yxati mavjud. API faqat shu kompaniyalarga tegishli ma'lumotlarni qaytaradi.

---

## 2. Autentifikatsiya — JWT token

### Token qanday ishlaydi

Ko'p endpointlar **Authorization** sarlavhasida JWT Bearer token talab qiladi:

```
Authorization: Bearer <jwt_token>
```

Token ichida quyidagi ma'lumotlar saqlanadi (frontend uchun muhim):

```json
{
  "user_id": 123,
  "role": "admin",
  "companies": [1, 2, 5]
}
```

> **Eslatma:** Foydalanuvchi faqat `companies` ro'yxatidagi kompaniyalarga tegishli ma'lumotlarga kirish huquqiga ega.

### SSE uchun alohida autentifikatsiya

`GET /sse/events` endpoint'i boshqacha ishlaydi — token URL query param sifatida uzatiladi:

```
GET /sse/events?token=<jwt_token>
```

Buning sababi: `EventSource` brauzer API'si maxsus HTTP sarlavhalarni qo'shishga imkon bermaydi.

### Webhook uchun autentifikatsiya yo'q

`POST /webhooks/nylas` — Bu endpoint faqat backend tomonidan boshqariladi. Nylas `x-nylas-signature` sarlavhasi orqali autentifikatsiya qiladi. Frontend dasturchi bu endpoint bilan bevosita ishlamaydi.

### Token saqlash

```typescript
// token.ts
const TOKEN_KEY = "jwt_token";

export const TokenStorage = {
  save: (token: string): void => {
    localStorage.setItem(TOKEN_KEY, token);
  },
  get: (): string | null => {
    return localStorage.getItem(TOKEN_KEY);
  },
  remove: (): void => {
    localStorage.removeItem(TOKEN_KEY);
  },
};
```

---

## 3. Asosiy sozlamalar — apiRequest funksiyasi

Barcha API chaqiruvlari uchun yagona yordamchi funksiya. Bu funksiya:
- Token sarlavhasini avtomatik qo'shadi
- `204 No Content` javobini to'g'ri boshqaradi
- Xatoliklarni standart formatda chiqaradi

```typescript
// api.ts
import { TokenStorage } from "./token";

export const BASE_URL = "https://api.example.com"; // o'z API URL'ingizni yozing

export interface ApiError {
  status: number;
  detail: string;
}

export async function apiRequest<T>(
  endpoint: string,
  options: RequestInit = {}
): Promise<T> {
  const token = TokenStorage.get();

  const response = await fetch(`${BASE_URL}${endpoint}`, {
    ...options,
    headers: {
      "Content-Type": "application/json",
      // Token mavjud bo'lsa, sarlavhaga qo'shiladi
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      // options.headers ichidagilar ustidan yozadi (masalan, maxsus sarlavhalar)
      ...options.headers,
    },
  });

  // 204 No Content — DELETE operatsiyalarida qaytadi, body bo'lmaydi
  if (response.status === 204) {
    return null as T;
  }

  // JSON parse qilishga urinib ko'ramiz
  const data = await response.json().catch(() => ({}));

  // Xato bo'lsa, standart ApiError formatida throw qilamiz
  if (!response.ok) {
    throw {
      status: response.status,
      detail: data.detail ?? "Noma'lum xato yuz berdi",
    } as ApiError;
  }

  return data as T;
}
```

### Foydalanish misoli

```typescript
// Foydalanish
try {
  const accounts = await apiRequest<EmailAccount[]>("/accounts/");
  console.log(accounts);
} catch (error) {
  const err = error as ApiError;
  if (err.status === 401) {
    // Tokenni o'chirib, login sahifasiga yo'naltirish
    TokenStorage.remove();
    window.location.href = "/login";
  } else {
    alert(`Xato: ${err.detail}`);
  }
}
```

---

## 4. TypeScript turlari — barcha interface'lar

Quyida barcha API response va request uchun TypeScript turlari keltirilgan.

```typescript
// types.ts

// ─────────────────────────────────────────────
// UMUMIY TURLAR
// ─────────────────────────────────────────────

/** Sahifalangan javob — barcha list endpointlar shu formatda qaytaradi */
export interface PaginatedResponse<T> {
  items: T[];      // Hozirgi sahifadagi elementlar
  total: number;   // Jami elementlar soni (barcha sahifalar bo'yicha)
  page: number;    // Joriy sahifa raqami (1 dan boshlanadi)
  size: number;    // Har bir sahifadagi elementlar soni
}

/** Kompaniya ma'lumotlari */
export interface Company {
  id: number;
  name: string;
}

/** Email manzil ob'ekti — yuboruvchi/qabul qiluvchi ma'lumotlari */
export interface EmailAddress {
  name: string | null;  // Shaxs ismi (bo'sh bo'lishi mumkin)
  email: string;        // Email manzili (doim mavjud)
}

// ─────────────────────────────────────────────
// EMAIL HISOB TURLARI
// ─────────────────────────────────────────────

/** Email hisobini yaratish uchun so'rov */
export interface EmailAccountCreate {
  company_id: number;         // Kompaniya identifikatori (majburiy)
  email_address: string;      // Email manzili (majburiy, unikal)
  display_name?: string;      // Ko'rsatish nomi (ixtiyoriy)
  provider?: string;          // Provayder: "godaddy", "gmail" va h.k. (standart: "godaddy")
  grant_id?: string;          // Nylas grant ID (backend tomonidan boshqariladi, ixtiyoriy)
}

/** Email hisobini yangilash uchun so'rov (barcha maydonlar ixtiyoriy) */
export interface EmailAccountUpdate {
  display_name?: string;
  provider?: string;
  grant_id?: string;
}

/** Email hisobi API javobi */
export interface EmailAccount {
  id: string;                    // UUID
  company_id: number;
  email_address: string;
  display_name: string | null;   // Null bo'lishi mumkin
  provider: string;              // "godaddy", "gmail" va h.k.
  status: "active" | "syncing" | "disconnected";
  last_sync_at: string | null;   // ISO 8601 format, null bo'lishi mumkin
  created_at: string;            // ISO 8601 format
  company: Company | null;       // Null bo'lishi mumkin
}

// ─────────────────────────────────────────────
// XABAR TURLARI
// ─────────────────────────────────────────────

/** Email yuborish uchun so'rov (yangi xabar va javob uchun bir xil) */
export interface SendEmailRequest {
  email_account_id: string;      // Qaysi hisobdan yuborilsin (UUID)
  to: EmailAddress[];            // Qabul qiluvchilar ro'yxati (majburiy)
  cc?: EmailAddress[] | null;    // Nusxa oluvchilar (ixtiyoriy)
  bcc?: EmailAddress[] | null;   // Yashirin nusxa oluvchilar (ixtiyoriy)
  subject: string;               // Mavzu (majburiy)
  body_html?: string | null;     // HTML format tana (ixtiyoriy)
  body_text?: string | null;     // Oddiy matn tana (ixtiyoriy)
  reply_to_message_id?: string | null; // Javob beriladigan xabar ID (faqat reply uchun)
}

/** Xabar API javobi */
export interface Message {
  id: string;                       // UUID
  thread_id: string;                // Bu xabar qaysi mavzuga tegishli
  email_account_id: string;         // Qaysi hisobga tegishli
  internet_message_id: string;      // Email protokolining unikal ID'si (RFC 822)
  from_address: string;             // Yuboruvchi email manzili
  from_name: string | null;         // Yuboruvchi ismi (null bo'lishi mumkin)
  to_addresses: EmailAddress[];     // Qabul qiluvchilar
  cc_addresses: EmailAddress[] | null;  // CC manzillar (null bo'lishi mumkin)
  bcc_addresses: EmailAddress[] | null; // BCC manzillar (null bo'lishi mumkin)
  subject: string;                  // Mavzu
  body_html: string | null;         // HTML tana (null bo'lishi mumkin)
  body_text: string | null;         // Oddiy matn tana (null bo'lishi mumkin)
  snippet: string | null;           // Qisqa ko'rinish (null bo'lishi mumkin)
  direction: "inbound" | "outbound"; // "inbound" = kiruvchi, "outbound" = chiquvchi
  has_attachments: boolean;         // Qo'shimcha fayl bormi?
  sent_at: string;                  // Yuborilgan vaqt (ISO 8601)
  created_at: string;               // Tizimda yaratilgan vaqt (ISO 8601)
  attachments: Attachment[];        // Qo'shimcha fayllar ro'yxati
}

/** Sahifalangan xabarlar javobi */
export type PaginatedMessages = PaginatedResponse<Message>;

// ─────────────────────────────────────────────
// KIRUVCHI QUTI TURLARI
// ─────────────────────────────────────────────

/** Kiruvchi quti elementi (Inbox item) */
export interface InboxItem {
  id: string;                      // UUID (bu inbox entry ID, xabar ID emas!)
  is_read: boolean;                // O'qilganmi?
  is_starred: boolean;             // Yulduz qo'yilganmi?
  is_archived: boolean;            // Arxivlanganmi?
  read_at: string | null;          // O'qilgan vaqt (null = hali o'qilmagan)
  created_at: string;              // Inbox'ga qo'shilgan vaqt
  message: MessageSummary;         // Xabar qisqacha ma'lumoti
  thread_subject: string;          // Mavzu sarlavhasi
  thread_message_count: number;    // Mavzudagi jami xabarlar soni
  account_email: string;           // Qaysi email hisobiga tegishli
}

/** Xabar qisqacha ma'lumoti (InboxItem ichida keladi) */
export interface MessageSummary {
  id: string;
  from_address: string;
  from_name: string | null;
  subject: string;
  snippet: string | null;
  sent_at: string;
  direction: "inbound" | "outbound";
  has_attachments: boolean;
}

/** Sahifalangan Inbox javobi */
export type PaginatedInbox = PaginatedResponse<InboxItem>;

/** O'qilmagan xabarlar soni (hisoblar bo'yicha) */
export interface UnreadCount {
  account_id: string;        // Hisob UUID
  account_email: string;     // Hisob email manzili
  unread_count: number;      // O'qilmagan xabarlar soni
}

// ─────────────────────────────────────────────
// MAVZU TURLARI
// ─────────────────────────────────────────────

/** Mavzu ro'yxati elementi (qisqacha) */
export interface ThreadSummary {
  id: string;                    // UUID
  email_account_id: string;
  subject: string;               // Mavzu sarlavhasi
  last_message_at: string;       // Oxirgi xabar vaqti (ISO 8601)
  message_count: number;         // Mavzudagi xabarlar soni
  do_not_delete: boolean;        // Himoya belgisi — true bo'lsa o'chirib bo'lmaydi
  created_at: string;            // Yaratilgan vaqt
}

/** Mavzu tafsilotlari (xabarlar bilan birga) */
export interface ThreadDetail extends ThreadSummary {
  messages: Message[];           // Bu mavzudagi barcha xabarlar
}

/** Mavzuni yangilash uchun so'rov */
export interface ThreadUpdate {
  do_not_delete?: boolean;       // Himoya belgisini o'zgartirish
}

/** Sahifalangan mavzular javobi */
export type PaginatedThreads = PaginatedResponse<ThreadSummary>;

// ─────────────────────────────────────────────
// QO'SHIMCHA FAYL TURLARI
// ─────────────────────────────────────────────

/** Qo'shimcha fayl ma'lumotlari */
export interface Attachment {
  id: string;                    // UUID
  message_id: string;            // Qaysi xabarga tegishli
  filename: string;              // Fayl nomi (masalan, "report.pdf")
  content_type: string;          // MIME turi (masalan, "application/pdf")
  size_bytes: number;            // Fayl hajmi (baytlarda)
  is_inline: boolean;            // true = email tanasiga joylashtirilgan rasm/fayl
                                 // false = yuklab olinadigan qo'shimcha
  created_at: string;            // ISO 8601
}

/** Fayl yuklab olish URL javobi */
export interface AttachmentDownloadResponse {
  download_url: string;          // AWS S3 presigned URL (~1 soat amal qiladi)
}

// ─────────────────────────────────────────────
// SSE HODISA TURLARI
// ─────────────────────────────────────────────

/** SSE ulanish muvaffaqiyatli bo'lganda birinchi keluvchi hodisa */
export interface SSEConnectedEvent {
  message: string;    // "connected"
  user_id: number;    // Foydalanuvchi ID
}

/** Yangi xabar yaratildi hodisasi */
export interface SSEMessageCreatedEvent {
  type: "message.created";
  thread_id: string;    // Qaysi mavzuga qo'shildi
  message_id: string;   // Yangi xabar ID
  company_id: number;   // Qaysi kompaniyaga tegishli
}

/** Xabar yangilandi hodisasi */
export interface SSEMessageUpdatedEvent {
  type: "message.updated";
  thread_id: string;
  message_id: string;
  company_id: number;
}
```

---

## 5. Hisoblar (Accounts) API

Email hisoblar — Gmail, GoDaddy va boshqa provayderlardan ulangan pochta qutilari.

### 5.1 Email hisob yaratish

```
POST /accounts/
Authorization: Bearer <token>
Status: 201 Created
```

**So'rov:**
```json
{
  "company_id": 123,
  "email_address": "support@acme.com",
  "display_name": "Qo'llab-quvvatlash bo'limi",
  "provider": "godaddy",
  "grant_id": "grant_abc123xyz"
}
```

**Javob:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "company_id": 123,
  "email_address": "support@acme.com",
  "display_name": "Qo'llab-quvvatlash bo'limi",
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

**Muhim maydonlar:**
- `status` — yangi hisob `"syncing"` holatidan boshlanadi, keyinchalik `"active"` yoki `"disconnected"` bo'ladi
- `last_sync_at` — birinchi marta `null` bo'ladi (hali sinxronizatsiya bo'lmagan)
- `grant_id` — Nylas OAuth orqali olinadi, odatda backend boshqaradi

**Xato holatlari:**
- `400` — Bu email manzil yoki `grant_id` allaqachon mavjud
- `403` — Foydalanuvchida ushbu `company_id` ga kirish huquqi yo'q

**TypeScript kodi:**
```typescript
// accounts.api.ts
import { apiRequest } from "./api";
import type { EmailAccount, EmailAccountCreate } from "./types";

export async function createAccount(
  data: EmailAccountCreate
): Promise<EmailAccount> {
  return apiRequest<EmailAccount>("/accounts/", {
    method: "POST",
    body: JSON.stringify(data),
  });
}
```

---

### 5.2 Barcha hisoblarni olish

```
GET /accounts/
Authorization: Bearer <token>
Status: 200 OK
```

**Javob:** EmailAccount ob'ektlari massivi (sahifalash yo'q — barchasi bir sahifada)

```json
[
  {
    "id": "550e8400-...",
    "company_id": 123,
    "email_address": "support@acme.com",
    "display_name": "Qo'llab-quvvatlash",
    "provider": "godaddy",
    "status": "active",
    "last_sync_at": "2026-02-21T08:00:00Z",
    "created_at": "2026-01-15T09:00:00Z",
    "company": { "id": 123, "name": "Acme Inc." }
  }
]
```

**TypeScript kodi:**
```typescript
export async function listAccounts(): Promise<EmailAccount[]> {
  return apiRequest<EmailAccount[]>("/accounts/");
}
```

---

### 5.3 Bitta hisobni olish

```
GET /accounts/{account_id}
Authorization: Bearer <token>
Status: 200 OK
```

**Xato holatlari:**
- `404` — Hisob topilmadi yoki kirish huquqi yo'q

**TypeScript kodi:**
```typescript
export async function getAccount(accountId: string): Promise<EmailAccount> {
  return apiRequest<EmailAccount>(`/accounts/${accountId}`);
}
```

---

### 5.4 Hisobni yangilash

```
PATCH /accounts/{account_id}/
Authorization: Bearer <token>
Status: 200 OK
```

**So'rov (faqat o'zgartiriladigan maydonlarni yuboring):**
```json
{
  "display_name": "Yangi nom"
}
```

**TypeScript kodi:**
```typescript
import type { EmailAccountUpdate } from "./types";

export async function updateAccount(
  accountId: string,
  data: EmailAccountUpdate
): Promise<EmailAccount> {
  return apiRequest<EmailAccount>(`/accounts/${accountId}/`, {
    method: "PATCH",
    body: JSON.stringify(data),
  });
}
```

---

### 5.5 Hisobni o'chirish (uzish)

```
DELETE /accounts/{account_id}
Authorization: Bearer <token>
Status: 204 No Content
```

> Bu hisob API'dan uziladi. `204` javobida body bo'lmaydi — `apiRequest` funksiyasi `null` qaytaradi.

**TypeScript kodi:**
```typescript
export async function deleteAccount(accountId: string): Promise<void> {
  await apiRequest<null>(`/accounts/${accountId}`, {
    method: "DELETE",
  });
}
```

**React'da foydalanish:**
```typescript
// Tasdiqlash bilan o'chirish
async function handleDeleteAccount(accountId: string) {
  if (!confirm("Bu hisobni o'chirishni xohlaysizmi?")) return;

  try {
    await deleteAccount(accountId);
    // UI dan olib tashlaymiz
    setAccounts((prev) => prev.filter((acc) => acc.id !== accountId));
  } catch (error) {
    const err = error as ApiError;
    alert(`Xato: ${err.detail}`);
  }
}
```

---

## 6. Xabarlar (Messages) API

### 6.1 Xabar yuborish

```
POST /messages/send/
Authorization: Bearer <token>
Status: 200 OK
```

**So'rov:**
```json
{
  "email_account_id": "550e8400-e29b-41d4-a716-446655440000",
  "to": [
    { "name": "Mijoz", "email": "mijoz@example.com" }
  ],
  "cc": null,
  "bcc": null,
  "subject": "Murojaat bo'yicha javob",
  "body_html": "<p>Hurmatli mijoz, murojaatingiz qabul qilindi.</p>",
  "body_text": "Hurmatli mijoz, murojaatingiz qabul qilindi."
}
```

**Javob:** To'liq `Message` ob'ekti

**Muhim:**
- `to` massivi bo'sh bo'lmasligi kerak
- `body_html` va `body_text` ikkalasi ham ixtiyoriy, lekin kamida bittasi bo'lishi tavsiya etiladi
- `email_account_id` — xabar qaysi email hisobdan yuborilishini belgilaydi

**TypeScript kodi:**
```typescript
// messages.api.ts
import { apiRequest } from "./api";
import type { Message, SendEmailRequest } from "./types";

export async function sendEmail(data: SendEmailRequest): Promise<Message> {
  return apiRequest<Message>("/messages/send/", {
    method: "POST",
    body: JSON.stringify(data),
  });
}
```

**React'da foydalanish:**
```typescript
async function handleSend() {
  try {
    const sent = await sendEmail({
      email_account_id: selectedAccountId,
      to: [{ name: recipientName, email: recipientEmail }],
      subject: subject,
      body_html: `<p>${body}</p>`,
      body_text: body,
    });
    console.log("Xabar yuborildi, ID:", sent.id);
    // Muvaffaqiyat bildirishnomasi ko'rsatish
  } catch (error) {
    const err = error as ApiError;
    alert(`Yuborishda xato: ${err.detail}`);
  }
}
```

---

### 6.2 Xabarlar ro'yxatini olish (sahifalangan)

```
GET /messages/?page=1&size=25&account_id=<uuid>
Authorization: Bearer <token>
Status: 200 OK
```

**Query parametrlari:**

| Parametr | Turi | Majburiy | Tavsif |
|----------|------|----------|--------|
| `page` | number | Yo'q | Sahifa raqami (standart: 1) |
| `size` | number | Yo'q | Sahifadagi elementlar soni (standart: 25, max: 100) |
| `account_id` | UUID | Yo'q | Faqat shu hisob xabarlarini filtrlash |

**Javob:**
```json
{
  "items": [ /* Message ob'ektlari */ ],
  "total": 542,
  "page": 1,
  "size": 25
}
```

**TypeScript kodi:**
```typescript
export async function listMessages(params?: {
  page?: number;
  size?: number;
  account_id?: string;
}): Promise<PaginatedMessages> {
  const query = new URLSearchParams();
  if (params?.page) query.set("page", String(params.page));
  if (params?.size) query.set("size", String(params.size));
  if (params?.account_id) query.set("account_id", params.account_id);

  const queryStr = query.toString();
  return apiRequest<PaginatedMessages>(
    `/messages/${queryStr ? `?${queryStr}` : ""}`
  );
}
```

---

### 6.3 Bitta xabarni olish

```
GET /messages/{message_id}
Authorization: Bearer <token>
Status: 200 OK
```

**Javob:** To'liq `Message` ob'ekti (`attachments` massivi bilan birga)

**Xato holatlari:**
- `404` — Xabar topilmadi yoki kirish huquqi yo'q

**TypeScript kodi:**
```typescript
export async function getMessage(messageId: string): Promise<Message> {
  return apiRequest<Message>(`/messages/${messageId}`);
}
```

---

### 6.4 Xabarga javob berish

```
POST /messages/{message_id}/reply
Authorization: Bearer <token>
Status: 200 OK
```

**So'rov:** Yuborish bilan bir xil `SendEmailRequest` formati

```json
{
  "email_account_id": "550e8400-...",
  "to": [{ "name": "Mijoz", "email": "mijoz@example.com" }],
  "subject": "Re: Asl mavzu",
  "body_html": "<p>Javobim...</p>",
  "body_text": "Javobim...",
  "reply_to_message_id": "33333333-3333-3333-3333-333333333333"
}
```

> `reply_to_message_id` ni URL dagi `message_id` bilan bir xil qiymatga o'rnating.

**TypeScript kodi:**
```typescript
export async function replyToMessage(
  messageId: string,
  data: SendEmailRequest
): Promise<Message> {
  return apiRequest<Message>(`/messages/${messageId}/reply`, {
    method: "POST",
    body: JSON.stringify({
      ...data,
      reply_to_message_id: messageId,
    }),
  });
}
```

---

## 7. Kiruvchi quti (Inbox) API

Inbox — foydalanuvchining shaxsiy kiruvchi qutisi. Har bir inbox elementi (`InboxItem`) bitta xabar bilan bog'liq va o'qildi/yulduz/arxiv holatlarini saqlaydi.

> **Inbox vs Threads farqi:**
> - **Inbox** — foydalanuvchi uchun shaxsiy ko'rinish. O'qish/yulduz/arxiv holatlari foydalanuvchiga tegishli. Filtrlash (o'qilmagan, yulduzli) uchun ishlatiladi.
> - **Threads** — xabarlar zanjiri. Hamma foydalanuvchilar uchun bir xil. Suhbat tarixi ko'rish uchun ishlatiladi.

### 7.1 Inbox ro'yxatini olish

```
GET /inbox/?page=1&size=25&is_read=false&account_id=<uuid>
Authorization: Bearer <token>
Status: 200 OK
```

**Query parametrlari:**

| Parametr | Turi | Majburiy | Tavsif |
|----------|------|----------|--------|
| `page` | number | Yo'q | Sahifa raqami (standart: 1) |
| `size` | number | Yo'q | Sahifadagi elementlar soni (standart: 25, max: 100) |
| `is_read` | boolean | Yo'q | `false` — faqat o'qilmaganlar, `true` — faqat o'qilganlar |
| `account_id` | UUID | Yo'q | Faqat shu hisob inbox elementlarini ko'rsatish |

**Javob:**
```json
{
  "items": [
    {
      "id": "55555555-5555-5555-5555-555555555555",
      "is_read": false,
      "is_starred": false,
      "is_archived": false,
      "read_at": null,
      "created_at": "2026-02-21T12:00:00Z",
      "message": {
        "id": "33333333-...",
        "from_address": "mijoz@example.com",
        "from_name": "Ali Valiyev",
        "subject": "Yordam kerak",
        "snippet": "Salom, sizga murojaat qilmoqchiman...",
        "sent_at": "2026-02-21T11:58:00Z",
        "direction": "inbound",
        "has_attachments": false
      },
      "thread_subject": "Yordam kerak",
      "thread_message_count": 1,
      "account_email": "support@acme.com"
    }
  ],
  "total": 87,
  "page": 1,
  "size": 25
}
```

**TypeScript kodi:**
```typescript
// inbox.api.ts
import { apiRequest } from "./api";
import type { PaginatedInbox } from "./types";

export async function listInbox(params?: {
  page?: number;
  size?: number;
  is_read?: boolean;
  account_id?: string;
}): Promise<PaginatedInbox> {
  const query = new URLSearchParams();
  if (params?.page !== undefined) query.set("page", String(params.page));
  if (params?.size !== undefined) query.set("size", String(params.size));
  if (params?.is_read !== undefined) query.set("is_read", String(params.is_read));
  if (params?.account_id) query.set("account_id", params.account_id);

  const queryStr = query.toString();
  return apiRequest<PaginatedInbox>(
    `/inbox/${queryStr ? `?${queryStr}` : ""}`
  );
}
```

**Foydalanish misoli:**
```typescript
// Faqat o'qilmaganlarni olish
const unread = await listInbox({ is_read: false, page: 1, size: 25 });

// Ma'lum hisob uchun barcha inbox elementlarini olish
const accountInbox = await listInbox({ account_id: "550e8400-..." });
```

---

### 7.2 O'qilmagan xabarlar sonini olish

```
GET /inbox/unread-counts
Authorization: Bearer <token>
Status: 200 OK
```

**Javob:** Har bir hisob bo'yicha o'qilmagan xabarlar soni

```json
[
  {
    "account_id": "550e8400-e29b-41d4-a716-446655440000",
    "account_email": "support@acme.com",
    "unread_count": 12
  },
  {
    "account_id": "661e8400-...",
    "account_email": "sales@acme.com",
    "unread_count": 3
  }
]
```

**TypeScript kodi:**
```typescript
import type { UnreadCount } from "./types";

export async function getUnreadCounts(): Promise<UnreadCount[]> {
  return apiRequest<UnreadCount[]>("/inbox/unread-counts");
}
```

**React sidebar uchun foydalanish:**
```typescript
// Umumiy o'qilmagan sonni hisoblash
const counts = await getUnreadCounts();
const totalUnread = counts.reduce((sum, c) => sum + c.unread_count, 0);
// Natija: 15
```

---

### 7.3 Inbox elementini o'qilgan deb belgilash

```
PATCH /inbox/{inbox_id}/read
Authorization: Bearer <token>
Status: 200 OK
```

> **Diqqat:** URL'dagi `inbox_id` — bu `InboxItem.id`, xabar ID'si emas!

**Javob:** Yangilangan `InboxItem` ob'ekti

```json
{
  "id": "55555555-...",
  "is_read": true,
  "read_at": "2026-02-21T14:30:00Z",
  "..."
}
```

**TypeScript kodi:**
```typescript
export async function markAsRead(inboxId: string): Promise<InboxItem> {
  return apiRequest<InboxItem>(`/inbox/${inboxId}/read`, {
    method: "PATCH",
  });
}
```

**React'da optimistik yangilanish bilan foydalanish:**
```typescript
async function handleMarkRead(inboxId: string) {
  // Avval UI'ni darhol yangilaymiz (optimistic update)
  setItems((prev) =>
    prev.map((item) =>
      item.id === inboxId
        ? { ...item, is_read: true, read_at: new Date().toISOString() }
        : item
    )
  );

  try {
    // Keyin serverga yuboramiz
    await markAsRead(inboxId);
  } catch (error) {
    // Xato bo'lsa, orqaga qaytaramiz
    setItems((prev) =>
      prev.map((item) =>
        item.id === inboxId ? { ...item, is_read: false, read_at: null } : item
      )
    );
    alert("O'qilgan deb belgilab bo'lmadi");
  }
}
```

---

### 7.4 Inbox elementini o'qilmagan deb belgilash

```
PATCH /inbox/{inbox_id}/unread
Authorization: Bearer <token>
Status: 200 OK
```

**TypeScript kodi:**
```typescript
export async function markAsUnread(inboxId: string): Promise<InboxItem> {
  return apiRequest<InboxItem>(`/inbox/${inboxId}/unread`, {
    method: "PATCH",
  });
}
```

---

### 7.5 Yulduz qo'yish / olib tashlash (toggle)

```
PATCH /inbox/{inbox_id}/star
Authorization: Bearer <token>
Status: 200 OK
```

> Bu endpoint toggle vazifasini bajaradi: agar `is_starred: false` bo'lsa `true` ga, `true` bo'lsa `false` ga o'zgartiradi.

**TypeScript kodi:**
```typescript
export async function toggleStar(inboxId: string): Promise<InboxItem> {
  return apiRequest<InboxItem>(`/inbox/${inboxId}/star`, {
    method: "PATCH",
  });
}
```

---

### 7.6 Inbox elementini arxivlash

```
PATCH /inbox/{inbox_id}/archive
Authorization: Bearer <token>
Status: 200 OK
```

**TypeScript kodi:**
```typescript
export async function archiveInboxItem(inboxId: string): Promise<InboxItem> {
  return apiRequest<InboxItem>(`/inbox/${inboxId}/archive`, {
    method: "PATCH",
  });
}
```

---

### 7.7 Butun mavzuni o'qilgan deb belgilash

```
PATCH /inbox/threads/{thread_id}/read
Authorization: Bearer <token>
Status: 200 OK
```

> Bir mavzudagi **barcha** inbox elementlarini bir vaqtda o'qilgan deb belgilaydi.

**Javob:**
```json
{
  "updated": 5
}
```
`updated` — o'qilgan deb belgilangan elementlar soni.

**TypeScript kodi:**
```typescript
export async function markThreadAsRead(
  threadId: string
): Promise<{ updated: number }> {
  return apiRequest<{ updated: number }>(
    `/inbox/threads/${threadId}/read`,
    { method: "PATCH" }
  );
}
```

---

## 8. Mavzular (Threads) API

Mavzu (Thread) — bir mavzuga tegishli xabarlar zanjiri. Masalan, bir mijoz bilan bo'lgan barcha muloqot bitta mavzu ostida guruhlanadi.

### 8.1 Mavzular ro'yxatini olish

```
GET /threads/?page=1&size=25&account_id=<uuid>
Authorization: Bearer <token>
Status: 200 OK
```

**Query parametrlari:**

| Parametr | Turi | Majburiy | Tavsif |
|----------|------|----------|--------|
| `page` | number | Yo'q | Sahifa raqami |
| `size` | number | Yo'q | Sahifadagi elementlar soni (max: 100) |
| `account_id` | UUID | Yo'q | Faqat shu hisob mavzularini filtrlash |

**Javob:**
```json
{
  "items": [
    {
      "id": "44444444-4444-4444-4444-444444444444",
      "email_account_id": "22222222-...",
      "subject": "Qo'llab-quvvatlash so'rovi",
      "last_message_at": "2026-02-21T14:00:00Z",
      "message_count": 5,
      "do_not_delete": false,
      "created_at": "2026-02-20T09:00:00Z"
    }
  ],
  "total": 120,
  "page": 1,
  "size": 25
}
```

**TypeScript kodi:**
```typescript
// threads.api.ts
import { apiRequest } from "./api";
import type { PaginatedThreads } from "./types";

export async function listThreads(params?: {
  page?: number;
  size?: number;
  account_id?: string;
}): Promise<PaginatedThreads> {
  const query = new URLSearchParams();
  if (params?.page) query.set("page", String(params.page));
  if (params?.size) query.set("size", String(params.size));
  if (params?.account_id) query.set("account_id", params.account_id);

  const queryStr = query.toString();
  return apiRequest<PaginatedThreads>(
    `/threads/${queryStr ? `?${queryStr}` : ""}`
  );
}
```

---

### 8.2 Mavzu tafsilotlarini olish (xabarlar bilan)

```
GET /threads/{thread_id}
Authorization: Bearer <token>
Status: 200 OK
```

**Javob:** `ThreadDetail` — barcha xabarlar (`messages` massivi) bilan birga

```json
{
  "id": "44444444-4444-4444-4444-444444444444",
  "email_account_id": "22222222-...",
  "subject": "Qo'llab-quvvatlash so'rovi",
  "last_message_at": "2026-02-21T14:00:00Z",
  "message_count": 3,
  "do_not_delete": false,
  "created_at": "2026-02-20T09:00:00Z",
  "messages": [
    { "/* To'liq Message ob'ekti */": "..." },
    { "/* To'liq Message ob'ekti */": "..." },
    { "/* To'liq Message ob'ekti */": "..." }
  ]
}
```

**Xato holatlari:**
- `404` — Mavzu topilmadi yoki kirish huquqi yo'q

**TypeScript kodi:**
```typescript
import type { ThreadDetail } from "./types";

export async function getThread(threadId: string): Promise<ThreadDetail> {
  return apiRequest<ThreadDetail>(`/threads/${threadId}`);
}
```

---

### 8.3 Mavzuni yangilash

```
PATCH /threads/{thread_id}
Authorization: Bearer <token>
Status: 200 OK
```

**So'rov:** (faqat o'zgartiriladigan maydonlar)
```json
{
  "do_not_delete": true
}
```

`do_not_delete: true` — mavzuni himoya belgisi bilan belgilaydi (o'chirib bo'lmaydi).

**TypeScript kodi:**
```typescript
import type { ThreadDetail, ThreadUpdate } from "./types";

export async function updateThread(
  threadId: string,
  data: ThreadUpdate
): Promise<ThreadDetail> {
  return apiRequest<ThreadDetail>(`/threads/${threadId}`, {
    method: "PATCH",
    body: JSON.stringify(data),
  });
}

// Mavzuni muhim deb belgilash
async function handleProtectThread(threadId: string) {
  const updated = await updateThread(threadId, { do_not_delete: true });
  console.log("Mavzu himoyalandi:", updated.do_not_delete); // true
}
```

---

## 9. Qo'shimcha fayllar (Attachments) API

### 9.1 Fayl yuklab olish URL'ini olish

```
GET /attachments/{attachment_id}/download
Authorization: Bearer <token>
Status: 200 OK
```

**Javob:**
```json
{
  "download_url": "https://s3.amazonaws.com/bucket/file.pdf?X-Amz-Signature=...&X-Amz-Expires=3600"
}
```

> **Muhim:** `download_url` — bu AWS S3 **presigned URL**. Taxminan **1 soat** davomida amal qiladi. URL'ni saqlab qo'ymang — har safar yuklab olishdan oldin yangi so'rov yuboring.

**Xato holatlari:**
- `404` — Qo'shimcha fayl topilmadi

**TypeScript kodi:**
```typescript
// attachments.api.ts
import { apiRequest } from "./api";
import type { Attachment, AttachmentDownloadResponse } from "./types";

export async function getAttachmentDownloadUrl(
  attachmentId: string
): Promise<AttachmentDownloadResponse> {
  return apiRequest<AttachmentDownloadResponse>(
    `/attachments/${attachmentId}/download`
  );
}
```

**React'da foydalanish — faylni brauzerda ochish:**
```typescript
async function handleDownload(attachment: Attachment) {
  try {
    const { download_url } = await getAttachmentDownloadUrl(attachment.id);
    // Yangi oynada faylni ochamiz (brauzer yuklab olishni boshlaydi)
    window.open(download_url, "_blank");
  } catch (error) {
    const err = error as ApiError;
    alert(`Yuklab olishda xato: ${err.detail}`);
  }
}
```

**Faylni dasturiy tarzda yuklab olish:**
```typescript
async function downloadFile(attachment: Attachment) {
  const { download_url } = await getAttachmentDownloadUrl(attachment.id);

  // <a> elementi orqali yuklab olish
  const link = document.createElement("a");
  link.href = download_url;
  link.download = attachment.filename; // Fayl nomini saqlaymiz
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}
```

---

### 9.2 Xabarning barcha qo'shimchalarini olish

```
GET /attachments/message/{message_id}
Authorization: Bearer <token>
Status: 200 OK
```

**Javob:** `Attachment` ob'ektlari massivi (sahifalash yo'q)

```json
[
  {
    "id": "66666666-6666-6666-6666-666666666666",
    "message_id": "33333333-3333-3333-3333-333333333333",
    "filename": "hisobot.pdf",
    "content_type": "application/pdf",
    "size_bytes": 125440,
    "is_inline": false,
    "created_at": "2026-02-21T12:00:00Z"
  },
  {
    "id": "77777777-...",
    "message_id": "33333333-...",
    "filename": "logo.png",
    "content_type": "image/png",
    "size_bytes": 8192,
    "is_inline": true,
    "created_at": "2026-02-21T12:00:00Z"
  }
]
```

**`is_inline` maydonini tushunish:**
- `is_inline: true` — Rasm yoki fayl email tanasiga joylashtirilgan (ko'rinadigan rasmlar). Odatda alohida ko'rsatilmaydi.
- `is_inline: false` — Yuklab olinadigan qo'shimcha (paperclip ikonkasi bilan ko'rsatiladi).

**TypeScript kodi:**
```typescript
export async function listMessageAttachments(
  messageId: string
): Promise<Attachment[]> {
  return apiRequest<Attachment[]>(`/attachments/message/${messageId}`);
}
```

**Faqat yuklab olinadigan qo'shimchalarni filtrlash:**
```typescript
const allAttachments = await listMessageAttachments(messageId);
// Faqat yuklab olinadigan (inline bo'lmagan) qo'shimchalar
const downloadable = allAttachments.filter((att) => !att.is_inline);
```

---

## 10. Webhooks — Nylas hodisalari

> **Frontend dasturchi uchun eslatma:** Webhooks endpoint'lari to'liq **backend tomonidan** boshqariladi. Frontend dasturchi bu endpoint'lar bilan bevosita ishlamaydi. Lekin ushbu bo'lim tizimni to'liq tushunish uchun muhim.

### Webhook qanday ishlaydi

1. Nylas tizimi (email provider) email hodisasi sodir bo'lganda (yangi xabar keldi, o'qildi va h.k.) sizning serveringizga **POST /webhooks/nylas** ga HTTP so'rov yuboradi.
2. Server **x-nylas-signature** sarlavhasi orqali so'rov Nylasdan kelganini tekshiradi.
3. Tekshiruv o'tsa, backend ma'lumotni bazaga saqlaydi va SSE orqali frontend'ga xabar yuboradi.

### 10.1 Webhook tekshiruv (Challenge)

```
GET /webhooks/nylas?challenge=<tasodifiy_matn>
Auth: Yo'q
Status: 200 OK (oddiy matn)
```

Nylas webhook URL'ni ro'yxatdan o'tkazishda bir marta `challenge` parametrini yuboradi. Server uni xuddi shu ko'rinishda qaytarishi kerak. Bu backend boshqaradi.

### 10.2 Nylas webhook qabul qilish

```
POST /webhooks/nylas
Auth: x-nylas-signature sarlavhasi (Nylas tomonidan yuboriladi)
Status: 200 OK yoki 401 Unauthorized
```

**Nylas yuboradigan payload:**
```json
{
  "specversion": "1.0",
  "type": "message.created",
  "source": "nylas",
  "id": "delivery-id-abc123",
  "time": 1740000000,
  "data": {
    "object": {
      "id": "nylas-msg-id",
      "grant_id": "grant-id",
      "thread_id": "nylas-thread-id",
      "subject": "Yangi xabar",
      "from": [{ "email": "from@example.com", "name": "Yuboruvchi" }],
      "to": [{ "email": "to@example.com", "name": "Qabul qiluvchi" }],
      "date": 1740000000,
      "body": "...",
      "snippet": "...",
      "attachments": [],
      "folders": ["inbox"]
    }
  }
}
```

**Frontend uchun ahamiyatli:**
Backend bu webhookni qayta ishlagandan so'ng, SSE orqali frontend'ga bildirishnoma yuboradi. Frontend faqat SSE'ni tinglaydi.

---

## 11. SSE — Real-time hodisalar

SSE (Server-Sent Events) — server o'z xohishi bilan clientga ma'lumot yuboradigan texnologiya. EMS yangi xabar kelganda yoki xabar yangilanganda SSE orqali frontend'ni xabardor qiladi.

### SSE endpoint

```
GET /sse/events?token=<jwt_token>
Auth: URL query param orqali (Bearer sarlavhasi emas!)
Content-Type: text/event-stream
```

### SSE hodisalari

| Hodisa | Qachon keladi |
|--------|---------------|
| `connected` | SSE ulanishi muvaffaqiyatli o'rnatilganda |
| `message.created` | Yangi xabar tizimga qo'shilganda |
| `message.updated` | Mavjud xabar yangilanganda |

### SSE ma'lumot formati

```
event: connected
data: {"message":"connected","user_id":123}

event: message.created
data: {"type":"message.created","thread_id":"44444444-...","message_id":"33333333-...","company_id":123}

event: message.updated
data: {"type":"message.updated","thread_id":"44444444-...","message_id":"33333333-...","company_id":123}
```

### To'liq SSE implementatsiyasi (auto-reconnect bilan)

```typescript
// sse.ts
import { TokenStorage } from "./token";
import { BASE_URL } from "./api";
import type {
  SSEConnectedEvent,
  SSEMessageCreatedEvent,
  SSEMessageUpdatedEvent,
} from "./types";

interface SSEHandlers {
  onConnected?: (data: SSEConnectedEvent) => void;
  onMessageCreated?: (data: SSEMessageCreatedEvent) => void;
  onMessageUpdated?: (data: SSEMessageUpdatedEvent) => void;
}

/**
 * SSE ulanishini boshlaydi va avtomatik qayta ulanishni ta'minlaydi.
 * @returns cleanup funksiyasi (komponent unmount bo'lganda chaqiriladi)
 */
export function connectSSE(handlers: SSEHandlers): () => void {
  let eventSource: EventSource | null = null;
  let retryTimer: ReturnType<typeof setTimeout> | null = null;
  let isDestroyed = false; // Komponent unmount bo'lganini kuzatish uchun

  function connect() {
    if (isDestroyed) return;

    const token = TokenStorage.get();
    if (!token) {
      console.warn("SSE: Token topilmadi, ulanilmadi.");
      return;
    }

    eventSource = new EventSource(`${BASE_URL}/sse/events?token=${token}`);

    // Ulanish muvaffaqiyatli
    eventSource.addEventListener("connected", (event: MessageEvent) => {
      const data = JSON.parse(event.data) as SSEConnectedEvent;
      handlers.onConnected?.(data);
      console.log("SSE ulanish o'rnatildi:", data);
    });

    // Yangi xabar keldi
    eventSource.addEventListener("message.created", (event: MessageEvent) => {
      const data = JSON.parse(event.data) as SSEMessageCreatedEvent;
      handlers.onMessageCreated?.(data);
    });

    // Xabar yangilandi
    eventSource.addEventListener("message.updated", (event: MessageEvent) => {
      const data = JSON.parse(event.data) as SSEMessageUpdatedEvent;
      handlers.onMessageUpdated?.(data);
    });

    // Xato — 5 soniyadan so'ng qayta ulanamiz
    eventSource.onerror = () => {
      console.warn("SSE ulanish uzildi. 5 soniyadan so'ng qayta ulaniladi...");
      eventSource?.close();
      eventSource = null;
      if (!isDestroyed) {
        retryTimer = setTimeout(connect, 5000);
      }
    };
  }

  connect();

  // Cleanup funksiyasi — React useEffect'dan qaytariladi
  return () => {
    isDestroyed = true;
    if (retryTimer) clearTimeout(retryTimer);
    eventSource?.close();
    eventSource = null;
  };
}
```

### React Hook sifatida ishlatish

```typescript
// useSSE.ts
import { useEffect } from "react";
import { connectSSE } from "./sse";
import type { SSEMessageCreatedEvent, SSEMessageUpdatedEvent } from "./types";

interface UseSSEOptions {
  onMessageCreated?: (data: SSEMessageCreatedEvent) => void;
  onMessageUpdated?: (data: SSEMessageUpdatedEvent) => void;
}

export function useSSE(options: UseSSEOptions) {
  useEffect(() => {
    const cleanup = connectSSE({
      onConnected: (data) => {
        console.log("SSE ulandi, foydalanuvchi ID:", data.user_id);
      },
      onMessageCreated: options.onMessageCreated,
      onMessageUpdated: options.onMessageUpdated,
    });

    // Komponent unmount bo'lganda ulanishni yopamiz
    return cleanup;
  }, []); // Faqat bir marta ishga tushadi
}
```

### Komponentda foydalanish

```typescript
// InboxPage.tsx
import { useSSE } from "./useSSE";
import { listInbox, getUnreadCounts } from "./inbox.api";

function InboxPage() {
  const [items, setItems] = useState<InboxItem[]>([]);
  const [unreadCount, setUnreadCount] = useState(0);

  // SSE'ni yoqamiz — komponent umri davomida ishlaydi
  useSSE({
    onMessageCreated: async (event) => {
      console.log("Yangi xabar keldi:", event.message_id);
      // Inbox'ni yangilaymiz
      const updated = await listInbox({ page: 1, size: 25, is_read: false });
      setItems(updated.items);
      // O'qilmagan sonini yangilaymiz
      const counts = await getUnreadCounts();
      const total = counts.reduce((sum, c) => sum + c.unread_count, 0);
      setUnreadCount(total);
    },
    onMessageUpdated: async (event) => {
      console.log("Xabar yangilandi:", event.message_id);
      // O'qilmagan sonini yangilaymiz
      const counts = await getUnreadCounts();
      const total = counts.reduce((sum, c) => sum + c.unread_count, 0);
      setUnreadCount(total);
    },
  });

  return (
    <div>
      <h1>Kiruvchi quti ({unreadCount} ta o'qilmagan)</h1>
      {/* ... */}
    </div>
  );
}
```

---

## 12. Xatoliklarni boshqarish

### HTTP status kodlari

| Kod | Ma'nosi | Frontend qanday munosabat qilishi kerak |
|-----|---------|------------------------------------------|
| `200` | Muvaffaqiyat | Ma'lumotni ko'rsatish |
| `201` | Resurs yaratildi | Muvaffaqiyat xabari ko'rsatish, ro'yxatni yangilash |
| `204` | Tarkib yo'q | Muvaffaqiyat (DELETE uchun) — elementni UI'dan olib tashlash |
| `400` | Noto'g'ri so'rov | Foydalanuvchiga xato sababini ko'rsatish |
| `401` | Autentifikatsiya xatosi | Tokenni o'chirib, login sahifasiga yo'naltirish |
| `403` | Ruxsat yo'q | "Kirish taqiqlangan" xabari ko'rsatish |
| `404` | Topilmadi | "Topilmadi" xabari ko'rsatish |
| `500` | Server xatosi | "Server xatosi, keyinroq urinib ko'ring" xabari |

### Xato javobi formati

```json
{
  "detail": "Email account with this email address/grant already exists."
}
```

Barcha xato javoblari `detail` maydoni bilan keladi.

### Markazlashtirilgan xatoliklarni boshqarish

```typescript
// error-handler.ts
import type { ApiError } from "./api";
import { TokenStorage } from "./token";

export function handleApiError(error: unknown): string {
  const err = error as ApiError;

  switch (err.status) {
    case 400:
      return `Noto'g'ri ma'lumot: ${err.detail}`;

    case 401:
      // Token eskirgan — login sahifasiga yo'naltirish
      TokenStorage.remove();
      window.location.href = "/login";
      return "Sessiya muddati tugagan. Qayta kiring.";

    case 403:
      return "Bu amalni bajarish uchun ruxsat yo'q.";

    case 404:
      return "So'ralgan ma'lumot topilmadi.";

    case 500:
      return "Server xatosi yuz berdi. Iltimos, keyinroq urinib ko'ring.";

    default:
      return err.detail ?? "Noma'lum xato yuz berdi.";
  }
}
```

### React'da foydalanish

```typescript
// React komponentda
const [error, setError] = useState<string | null>(null);

async function fetchData() {
  try {
    setError(null);
    const data = await listInbox();
    setItems(data.items);
  } catch (err) {
    setError(handleApiError(err));
  }
}

// JSX'da
{error && (
  <div className="error-banner">
    <p>{error}</p>
    <button onClick={() => setError(null)}>Yopish</button>
  </div>
)}
```

---

## 13. Sahifalash (Pagination)

### Qanday ishlaydi

Barcha ro'yxat endpointlari (messages, inbox, threads) sahifalash qo'llab-quvvatlaydi:

```
GET /inbox/?page=1&size=25
```

- **Standart sahifa hajmi:** 25 ta element
- **Maksimal sahifa hajmi:** 100 ta element
- **Sahifa raqami:** 1 dan boshlanadi

**Javob tuzilishi:**
```json
{
  "items": [ /* hozirgi sahifadagi elementlar */ ],
  "total": 542,
  "page": 1,
  "size": 25
}
```

**Jami sahifalar soni:**
```typescript
const totalPages = Math.ceil(total / size);
// 542 ta element, 25 ta elementli sahifalar → Math.ceil(542/25) = 22 sahifa
```

**Offset hisobi (backend tomonidan bajariladi):**
```
offset = (page - 1) * size
// Sahifa 2, size 25 uchun: offset = (2-1) * 25 = 25 (birinchi 25 ta o'tkazib yuboriladi)
```

### Sahifalash komponenti

```typescript
// Pagination.tsx
interface PaginationProps {
  page: number;
  size: number;
  total: number;
  onPageChange: (page: number) => void;
}

function Pagination({ page, size, total, onPageChange }: PaginationProps) {
  const totalPages = Math.ceil(total / size);

  return (
    <div className="pagination">
      <button
        onClick={() => onPageChange(page - 1)}
        disabled={page <= 1}
      >
        Oldingi
      </button>

      <span>
        {page} / {totalPages} ({total} ta)
      </span>

      <button
        onClick={() => onPageChange(page + 1)}
        disabled={page >= totalPages}
      >
        Keyingi
      </button>
    </div>
  );
}
```

### Cheksiz aylantirish (Infinite Scroll) implementatsiyasi

```typescript
// useInfiniteInbox.ts
const [items, setItems] = useState<InboxItem[]>([]);
const [page, setPage] = useState(1);
const [hasMore, setHasMore] = useState(true);
const [loading, setLoading] = useState(false);

async function loadMore() {
  if (loading || !hasMore) return;
  setLoading(true);

  try {
    const result = await listInbox({ page, size: 25 });
    setItems((prev) => [...prev, ...result.items]);
    // Yana elementlar qolganmi?
    setHasMore(items.length + result.items.length < result.total);
    setPage((prev) => prev + 1);
  } finally {
    setLoading(false);
  }
}
```

---

## 14. Ruxsat va kirish nazorati

### Kompaniya darajasida nazorat

JWT token'da foydalanuvchining kompaniyalari ro'yxati saqlanadi:

```json
{
  "user_id": 123,
  "role": "admin",
  "companies": [1, 2, 5]
}
```

API faqat bu kompaniyalarga tegishli ma'lumotlarni qaytaradi:

| Resurs | Filtrlash qoidasi |
|--------|-------------------|
| Hisoblar | Faqat `account.company_id in [1, 2, 5]` |
| Xabarlar | Faqat `message.email_account.company_id in [1, 2, 5]` |
| Mavzular | Faqat `thread.email_account.company_id in [1, 2, 5]` |
| Inbox | Faqat foydalanuvchining o'z inbox elementlari |

### Ruxsat xatolarini boshqarish

- **403 Forbidden** — Resurs mavjud, lekin siz unga kira olmaysiz
- **404 Not Found** — Resurs mavjud emas **yoki** siz kira olmaysiz (ma'lumot sizish oldini olish uchun)

```typescript
// 403 va 404 ni bir xil tarzda boshqarish
async function getAccountSafe(accountId: string): Promise<EmailAccount | null> {
  try {
    return await getAccount(accountId);
  } catch (error) {
    const err = error as ApiError;
    if (err.status === 403 || err.status === 404) {
      // Yo'q yoki kirish taqiqlangan — ikkalasi ham null qaytaramiz
      return null;
    }
    throw error; // Boshqa xatolarni yuqoriga uzatamiz
  }
}
```

---

## 15. Null maydonlarni xavfsiz boshqarish

API'da ba'zi maydonlar `null` bo'lishi mumkin. Bularni to'g'ri boshqarmasangiz, runtime xatolari yuzaga keladi.

### Null bo'lishi mumkin bo'lgan maydonlar

| Maydon | Null bo'lishi sababi |
|--------|---------------------|
| `from_name` | Yuboruvchi ismi ko'rsatmagan bo'lishi mumkin |
| `body_html` | Xabar faqat text formatida bo'lishi mumkin |
| `body_text` | Xabar faqat HTML formatida bo'lishi mumkin |
| `snippet` | Qisqa ko'rinish generatsiya bo'lmagan bo'lishi mumkin |
| `cc_addresses` | CC manzillar bo'lmagan bo'lishi mumkin |
| `bcc_addresses` | BCC manzillar bo'lmagan bo'lishi mumkin |
| `read_at` | Hali o'qilmagan bo'lishi mumkin |
| `last_sync_at` | Hisob hali sinxronizatsiya bo'lmagan bo'lishi mumkin |
| `display_name` | Ko'rsatish nomi kiritilmagan bo'lishi mumkin |
| `company` | Kompaniya ma'lumotlari yuklanmagan bo'lishi mumkin |

### Xavfsiz yordamchi funksiyalar

```typescript
// helpers.ts
import DOMPurify from "dompurify"; // npm install dompurify @types/dompurify

/**
 * Yuboruvchi nomini xavfsiz olish.
 * Ism bo'lmasa, email manzilini qaytaradi.
 */
export function getSenderName(message: Message | MessageSummary): string {
  return message.from_name ?? message.from_address;
}

/**
 * Xabar tanasini xavfsiz olish.
 * HTML → text → standart xabar tartibida tekshiradi.
 */
export function getMessageBody(message: Message): string {
  return message.body_html ?? message.body_text ?? "(Xabar tarkibi yo'q)";
}

/**
 * HTML tanani XSS hujumlaridan tozalab qaytaradi.
 * MUHIM: body_html ni bevosita innerHTML sifatida ishlatmang!
 */
export function safeHtml(html: string | null | undefined): string {
  if (!html) return "<p>(Xabar tarkibi yo'q)</p>";
  return DOMPurify.sanitize(html);
}

/**
 * Qisqa ko'rinishni xavfsiz olish.
 */
export function getSnippet(
  message: Message | MessageSummary,
  fallback = "(Ko'rinish yo'q)"
): string {
  return message.snippet ?? fallback;
}

/**
 * ISO 8601 sanani foydalanuvchiga qulay formatga o'tkazish.
 */
export function formatDate(isoString: string | null | undefined): string {
  if (!isoString) return "—";
  const date = new Date(isoString);
  return new Intl.DateTimeFormat("uz-UZ", {
    year: "numeric",
    month: "short",
    day: "numeric",
    hour: "2-digit",
    minute: "2-digit",
  }).format(date);
}

/**
 * Email manzillar massivini ko'rsatish uchun satrga aylantirish.
 */
export function formatAddresses(
  addresses: EmailAddress[] | null | undefined
): string {
  if (!addresses || addresses.length === 0) return "—";
  return addresses
    .map((addr) => (addr.name ? `${addr.name} <${addr.email}>` : addr.email))
    .join(", ");
}

/**
 * Fayl hajmini o'qilishi mumkin bo'lgan formatga o'tkazish.
 */
export function formatFileSize(bytes: number): string {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / (1024 * 1024)).toFixed(1)} MB`;
}
```

### React'da foydalanish

```typescript
// MessageCard.tsx
import { getSenderName, getSnippet, formatDate } from "./helpers";

function MessageCard({ item }: { item: InboxItem }) {
  return (
    <div className={`message-card ${item.is_read ? "" : "unread"}`}>
      {/* Null xavfsiz yuboruvchi nomi */}
      <span className="sender">{getSenderName(item.message)}</span>

      {/* Mavzu */}
      <span className="subject">{item.thread_subject}</span>

      {/* Qisqa ko'rinish — null bo'lsa standart matn */}
      <span className="snippet">{getSnippet(item.message)}</span>

      {/* Sana — null bo'lsa "—" ko'rsatiladi */}
      <span className="date">{formatDate(item.message.sent_at)}</span>

      {/* Yulduz */}
      {item.is_starred && <span className="star">★</span>}
    </div>
  );
}

// MessageDetail.tsx
import { getSenderName, safeHtml, formatDate, formatAddresses } from "./helpers";

function MessageDetail({ message }: { message: Message }) {
  return (
    <div className="message-detail">
      <h2>{message.subject}</h2>
      <p>Kimdan: {getSenderName(message)}</p>
      <p>Kimga: {formatAddresses(message.to_addresses)}</p>
      {message.cc_addresses && (
        <p>Nusxa: {formatAddresses(message.cc_addresses)}</p>
      )}
      <p>Sana: {formatDate(message.sent_at)}</p>

      {/* XSS himoyasi bilan HTML ko'rsatish */}
      <div
        className="message-body"
        dangerouslySetInnerHTML={{ __html: safeHtml(message.body_html) }}
      />
    </div>
  );
}
```

---

## 16. Barcha endpoint'lar jadvali

| Metod | Endpoint | Maqsad | Auth | Javob kodi |
|-------|----------|--------|------|------------|
| POST | `/accounts/` | Email hisob yaratish | Bearer | 201 |
| GET | `/accounts/` | Barcha hisoblarni olish | Bearer | 200 |
| GET | `/accounts/{id}` | Bitta hisobni olish | Bearer | 200 / 404 |
| PATCH | `/accounts/{id}/` | Hisobni yangilash | Bearer | 200 |
| DELETE | `/accounts/{id}` | Hisobni o'chirish | Bearer | 204 |
| POST | `/messages/send/` | Email yuborish | Bearer | 200 |
| GET | `/messages/` | Xabarlar ro'yxati | Bearer | 200 |
| GET | `/messages/{id}` | Bitta xabarni olish | Bearer | 200 / 404 |
| POST | `/messages/{id}/reply` | Xabarga javob berish | Bearer | 200 |
| GET | `/inbox/` | Inbox ro'yxati (sahifalangan) | Bearer | 200 |
| GET | `/inbox/unread-counts` | O'qilmagan sonlar | Bearer | 200 |
| PATCH | `/inbox/{id}/read` | O'qilgan deb belgilash | Bearer | 200 |
| PATCH | `/inbox/{id}/unread` | O'qilmagan deb belgilash | Bearer | 200 |
| PATCH | `/inbox/{id}/star` | Yulduz qo'yish/olib tashlash | Bearer | 200 |
| PATCH | `/inbox/{id}/archive` | Arxivlash | Bearer | 200 |
| PATCH | `/inbox/threads/{id}/read` | Butun mavzuni o'qilgan belgilash | Bearer | 200 |
| GET | `/threads/` | Mavzular ro'yxati | Bearer | 200 |
| GET | `/threads/{id}` | Mavzu tafsilotlari | Bearer | 200 / 404 |
| PATCH | `/threads/{id}` | Mavzuni yangilash | Bearer | 200 |
| GET | `/attachments/{id}/download` | Yuklab olish URL'ini olish | Bearer | 200 / 404 |
| GET | `/attachments/message/{id}` | Xabar qo'shimchalari | Bearer | 200 |
| GET | `/webhooks/nylas?challenge=X` | Nylas tekshiruv | Yo'q | 200 (plain text) |
| POST | `/webhooks/nylas` | Nylas webhookni qabul qilish | Signature | 200 / 401 |
| GET | `/sse/events?token=X` | Real-time SSE stream | URL param | stream |

---

## Tez boshlash uchun fayl tuzilishi

```
src/
├── api.ts               ← apiRequest funksiyasi, BASE_URL, ApiError
├── token.ts             ← TokenStorage (localStorage wrapper)
├── types.ts             ← Barcha TypeScript interface'lar
├── helpers.ts           ← Null boshqarish, sana formatlash, HTML tozalash
├── sse.ts               ← SSE ulanish va auto-reconnect
├── error-handler.ts     ← Markazlashtirilgan xato boshqarish
│
├── api/
│   ├── accounts.api.ts     ← Hisob operatsiyalari
│   ├── messages.api.ts     ← Xabar operatsiyalari
│   ├── inbox.api.ts        ← Inbox operatsiyalari
│   ├── threads.api.ts      ← Mavzu operatsiyalari
│   └── attachments.api.ts  ← Qo'shimcha fayl operatsiyalari
│
└── hooks/
    └── useSSE.ts        ← React hook SSE uchun
```

---

*Ushbu qo'llanma EMS API ning barcha 24 ta endpoint'ini, to'liq TypeScript turlarini, React namuna kodlarini, xatoliklarni boshqarish, sahifalash va null-xavfsiz yordamchi funksiyalarni o'z ichiga oladi.*
