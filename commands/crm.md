# סקיל: בניית CRM מאפס

בנה CRM עסקי בעברית (RTL) מאפס לפי המפרט הבא. עקוב אחרי הסדר המפורט ואל תדלג על שלבים.

---

## טק סטאק

- **Framework:** Next.js 16.2.9 עם Turbopack (לא Next.js רגיל — יש breaking changes)
- **UI:** Tailwind CSS + shadcn/ui (base-ui variant, לא radix)
- **ORM:** Prisma עם PostgreSQL (Neon cloud database)
- **Deploy:** Vercel עם GitHub auto-deploy
- **וואטסאפ:** Green API (שליחת הודעות בפועל, לא רק קישורי `wa.me`)
- **שפה:** TypeScript strict

---

## סכמת Prisma

```prisma
model Contact {
  id                 Int        @id @default(autoincrement())
  name               String
  phone              String?
  email              String?
  company            String?
  notes              String?
  status             String     @default("חדש")
  isExistingCustomer Boolean    @default(false)
  whatsappSummary    String?
  createdAt          DateTime   @default(now())
  deals              Deal[]
  tasks              Task[]
  activities         Activity[]
}

model Deal {
  id          Int       @id @default(autoincrement())
  title       String
  value       Float?
  stage       String    @default("ליד")
  notes       String?
  contactId   Int?
  contact     Contact?  @relation(fields: [contactId], references: [id], onDelete: Cascade)
  tasks       Task[]
  createdAt   DateTime  @default(now())
}

model Task {
  id          Int       @id @default(autoincrement())
  title       String
  dueDate     DateTime?
  completed   Boolean   @default(false)
  contactId   Int?
  contact     Contact?  @relation(fields: [contactId], references: [id], onDelete: Cascade)
  dealId      Int?
  deal        Deal?     @relation(fields: [dealId], references: [id], onDelete: SetNull)
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @default(now()) @updatedAt
}

model Activity {
  id          Int       @id @default(autoincrement())
  type        String
  note        String
  contactId   Int
  contact     Contact   @relation(fields: [contactId], references: [id], onDelete: Cascade)
  createdAt   DateTime  @default(now())
}
```

`Task.updatedAt` הוא שדה קריטי — הטבלה ב-`/contacts` ממיינת וכל שליחת פולו-אפ דוחה את המשימה הקרובה, אז חייבים `@updatedAt` כדי שהתאריך יתעדכן אוטומטית.

---

## עמודים ופיצ'רים

### `/contacts` — רשימת אנשי קשר

טבלה עם העמודות הבאות (מימין לשמאל, לפי RTL):

1. **שם** — לינק לפרופיל, ומתחתיו תאריך ההצטרפות (`formatJoinDate` — יום + חודש בעברית, שעה)
2. **חדש/קיים** — Badge לחיץ שמחליף `isExistingCustomer` בלחיצה בודדת: 🆕 חדש (כחול) / 🔁 קיים (אדום)
3. **טלפון** — מספר הטלפון + אמוג'י 💬 שפותח קישור `https://wa.me/{toWhatsAppNumber(phone)}` בטאב חדש (שיחה ידנית, לא דרך ה-API)
4. **משימה קרובה** — כותרת המשימה הראשונה שלא הושלמה + כפתור ✓ לסימון כהושלם + כפתור ✏️ לעריכה + כפתור 📅 להוספת משימה חדשה. מתחת לזה, אם קיימת, שורת הפעילות האחרונה (`activities[0].note`) בירוק
5. **תאריך שינוי משימה** — תאריך ושעה של `task.updatedAt` בשתי שורות ("תאריך - ..." / "שעה - ...") מודגשות
6. **שם העסק** — Badge לחיץ עם 🏢, פותח `Input` inline לעריכה (שמירה ב-blur או Enter, ביטול ב-Escape). אם ריק: "הוסף עסק"
7. **סטטוס** — `Select` עם Badge צבעוני בערך הנבחר, פותח את כל האפשרויות (ראה סטטוסים למטה)
8. **פעולות** — תפריט נפתח 🗨️ פולו אפ (ראה וואטסאפ למטה) + כפתור ✏️ עריכת איש קשר + כפתור 🗑️ מחיקה

**התנהגות נוספת:**
- שורות בסטטוס "✅ סגור" מודגשות ברקע ירוק (`bg-green-100`)
- מיון: לפי `task.updatedAt` של המשימה הקרובה (או `createdAt` אם אין משימה), מהחדש לישן
- חיפוש (debounce ~250ms) + פילטר סטטוס, שניהם מול ה-API
- הוספת איש קשר חדש דרך Dialog (`ContactFormDialog`)
- **חשוב לגבי רוחב הטבלה:** להשתמש ב-`table-fixed` עם רוחבים קבועים לעמודות הצדדיות (שם/פעולות) ו-`truncate`/`line-clamp` על טקסט ארוך (שם עסק, כותרת משימה) כדי שהטבלה לא תגלוש הצידה. כותרות עמודות (`TableHead`) צריכות לאפשר `whitespace-normal` כדי שלא יחפפו זו את זו.

**סגירת עסקה מהטבלה (`CloseDealDialog`):**
בחירת "✅ סגור" ב-`Select` הסטטוס **לא** שומרת סטטוס מיד — היא פותחת `CloseDealDialog` ששואל "כמה הלקוח קנה? (₪)". רק בשליחת הטופס:
1. הסטטוס של איש הקשר נשמר בפועל כ-"✅ סגור" (`updateStatus`)
2. נוצרת `Deal` חדשה: `POST /api/deals` עם `{ title: "מכירה - {שם}", value: הסכום, stage: "סגור-נוצח", contactId }`

אם המשתמש סוגר את הדיאלוג בלי לשלוח (Escape / קליק על הרקע / `onOpenChange(false)`), הסטטוס **לא** משתנה — כי ה-`Select` מוצג לפי `contact.status` שטרם עודכן, כך שהוא חוזר לערך הקודם אוטומטית. בחירה בכל סטטוס אחר (לא "✅ סגור") שומרת מיד כרגיל.

### `/contacts/[id]` — פרופיל לקוח
- כל הפרטים, היסטוריית פעילות, עסקאות, משימות
- עריכה inline

### `/tasks` — משימות
- רשימת משימות ממתינות וגמורות
- תאריך יעד + **שעה** (datetime-local)
- שם הלקוח = לינק לחיץ לפרופיל שלו
- כפתור ✏️ עריכה + ✕ מחיקה + ✓ checkbox
- מיון: קודם באיחור, אחר כך לפי תאריך, בסוף ללא תאריך
- תגיות: ⚠️ באיחור (אדום) / 🔥 היום (כתום) / 📅 מחר או מאוחר יותר

### `/deals` — עסקאות
- Kanban או טבלה לפי שלב

### `/` — דשבורד
- סטטיסטיקות מהירות: אנשי קשר, משימות ממתינות/באיחור, עסקאות פתוחות + שווין
- **💰 הכנסות מעסקאות סגורות** — סכום `value` של כל העסקאות בשלב `"סגור-נוצח"` (כולל אלו שנוצרו אוטומטית דרך `CloseDealDialog` ב-`/contacts`)
- כרטיס **💰 מכירות אחרונות** — 5 העסקאות האחרונות בשלב `"סגור-נוצח"`, ממוינות לפי `createdAt` (מהחדש לישן), עם שם הלקוח (לינק לפרופיל) והסכום

---

## סטטוסים

```typescript
export const STATUSES = ["חדש", "🔥 חם", "🌡️ פושר", "❄️ קר", "📵 ללא מענה", "⚙️ בטיפול", "✅ סגור", "🚫 לא רלוונטי"];

export const STATUS_COLORS: Record<string, string> = {
  "חדש": "bg-blue-100 text-blue-800",
  "🔥 חם": "bg-red-100 text-red-800",
  "🌡️ פושר": "bg-orange-100 text-orange-800",
  "❄️ קר": "bg-sky-100 text-sky-800",
  "📵 ללא מענה": "bg-gray-100 text-gray-800",
  "⚙️ בטיפול": "bg-yellow-100 text-yellow-800",
  "✅ סגור": "bg-green-100 text-green-800",
  "🚫 לא רלוונטי": "bg-neutral-200 text-neutral-700",
};
```

---

## API Routes

```
GET/POST       /api/contacts
GET/PUT/DELETE /api/contacts/[id]
POST           /api/contacts/[id]/followup
GET/POST       /api/tasks
GET/PUT/DELETE /api/tasks/[id]
GET/POST       /api/deals
GET/PUT/DELETE /api/deals/[id]
GET/POST       /api/activities
```

כל route שמבצע קריאה ל-DB חייב להכיל:
```typescript
export const dynamic = "force-dynamic";
```

### `/api/contacts/[id]/followup`

מקבל `{ message }`, שולח בפועל דרך Green API, רושם `Activity` מסוג "פולו אפ", ודוחה את המשימה הפתוחה הקרובה (`orderBy dueDate asc`) ליום העסקים הבא:

```typescript
import { sendWhatsAppMessage } from "@/lib/green-api";
import { nextBusinessDay } from "@/lib/date";

// אחרי שליחה מוצלחת:
await prisma.activity.create({
  data: { type: "פולו אפ", note: `פולו אפ נשלח ${dateLabel} - "${message}"`, contactId },
});
const nextTask = await prisma.task.findFirst({ where: { contactId, completed: false }, orderBy: { dueDate: "asc" } });
if (nextTask) await prisma.task.update({ where: { id: nextTask.id }, data: { dueDate: nextBusinessDay() } });
```

---

## אוטנטיקציה

קוקי פשוט `crm_auth=1`. עמוד `/login` עם סיסמה קבועה. Middleware מגן על כל הנתיבים חוץ מ-`/login`, `/api/auth`, `/api/whatsapp`.

---

## וואטסאפ

### שליחה בפועל דרך Green API (`lib/green-api.ts`)

```typescript
import { toWhatsAppNumber } from "./whatsapp";

export async function sendWhatsAppMessage(phone: string, message: string) {
  const idInstance = process.env.GREEN_API_ID_INSTANCE;
  const apiTokenInstance = process.env.GREEN_API_API_TOKEN;
  if (!idInstance || !apiTokenInstance) {
    throw new Error("Green API אינו מוגדר (חסרים משתני סביבה)");
  }
  const chatId = `${toWhatsAppNumber(phone)}@c.us`;
  const res = await fetch(
    `https://api.green-api.com/waInstance${idInstance}/sendMessage/${apiTokenInstance}`,
    { method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ chatId, message }) }
  );
  if (!res.ok) throw new Error(`שליחת הודעת וואטסאפ נכשלה: ${await res.text()}`);
  return res.json();
}
```

צריך להגדיר משתני סביבה `GREEN_API_ID_INSTANCE` ו-`GREEN_API_API_TOKEN` (ב-`.env` מקומי וב-Vercel).

### פונקציית עזר להמרת מספר (`lib/whatsapp.ts`)

```typescript
export function toWhatsAppNumber(phone: string): string {
  const digits = phone.replace(/\D/g, "");
  if (digits.startsWith("972")) return digits;
  if (digits.startsWith("0")) return "972" + digits.slice(1);
  return "972" + digits;
}
```

קישור ידני (כפתור 💬 בעמודת הטלפון): `https://wa.me/{toWhatsAppNumber(phone)}` — נפתח בטאב חדש לשיחה ידנית, בלי לשלוח כלום אוטומטית.

### תבניות פולו אפ (`followUpTemplates`, בתפריט 🗨️ פולו אפ בעמודת הפעולות)

```typescript
export const followUpTemplates = [
  { id: "missing-details", label: "חסרים פרטים", build: () => `היי, יצא לך לעבור על הקטלוג צריך הזמנה?` },
  { id: "closing", label: "סגירה", build: (name: string) => `היי ${name}, רציתי לדעת אם החלטת להתקדם - אנחנו כאן לכל שאלה :)` },
  { id: "no-answer", label: "ניסינו לתפוס ללא מענה", build: (name: string) => `היי ${name}, ניסינו לתפוס אותך ללא מענה :)` },
  { id: "check-in", label: "צ'ק אין", build: (name: string) => `היי ${name}, איך מתקדם? רציתי לשמוע ממך :)` },
];
```

לחיצה על תבנית שולחת אותה ישירות (לא פותחת `wa.me`), דרך `/api/contacts/[id]/followup`.

---

## הגדרות Vercel / Next.js

### `next.config.ts`
```typescript
// @ts-nocheck
const nextConfig = {
  eslint: {
    ignoreDuringBuilds: true,
  },
};
export default nextConfig;
```
**חשוב:** בלי זה הבנייה נכשלת על ESLint errors. `// @ts-nocheck` נדרש כי `NextConfig` לא כולל את `eslint` בטיפוסים של גרסה זו.

### משתני סביבה נדרשים
```
DATABASE_URL=...
GREEN_API_ID_INSTANCE=...
GREEN_API_API_TOKEN=...
```

---

## בעיות ידועות והימנעות

| בעיה | פתרון |
|------|--------|
| Vercel ESLint errors | `ignoreDuringBuilds: true` בnext.config.ts |
| Turbopack build cache מגיש גרסה ישנה | לשנות קצת בקובץ אחר כדי לאלץ recompile |
| `react-hooks/set-state-in-effect` | ESLint disabled בVercel |
| `DialogTrigger` בbase-ui | להשתמש ב-`render={<Button />}` ולא `asChild` |
| `onValueChange` ב-Select מחזיר `string \| null` | לטפל ב-null: `!v \|\| v === "none" ? "" : v` |
| ContactLink לא לחיץ | להשתמש ב-`onClick={() => window.location.href = ...}` במקום `<Link>` |
| API route לא מחזיר `id` של contact | להוסיף `id: true` ב-`select` של `include` |
| טבלת `/contacts` גולשת הצידה עם הרבה עמודות | `table-fixed` + רוחב קבוע לעמודות הצדדיות + `truncate` על טקסט ארוך (שם עסק, כותרת משימה) |
| כותרות עמודות חופפות זו את זו | לאפשר `whitespace-normal` על `TableHead` כדי שהתווית תגלוש לשורה שנייה |
| כפתור פולו-אפ תופס יותר מדי מקום בעמודת הפעולות | להשתמש בגרסה מכווצת (pill, `h-6 rounded-full`) כשצריך לחסוך רוחב |

---

## RTL

כל Dialog/חלון: `dir="rtl"`.
Layout ראשי: `dir="rtl"` ב-`layout.tsx`.
כל הטקסטים, כפתורים, ותוויות — בעברית.

---

## סדר בנייה מומלץ

1. `npx create-next-app` + התקנת תלויות (prisma, shadcn, tailwind)
2. סכמת Prisma + `prisma generate` + `prisma db push`
3. `next.config.ts` עם ESLint disabled
4. Middleware אוטנטיקציה + עמוד login
5. `lib/whatsapp.ts` (המרת מספר + תבניות) + `lib/green-api.ts` (שליחה בפועל) + `lib/date.ts` (`nextBusinessDay`)
6. API routes (contacts, tasks, deals, activities, כולל `contacts/[id]/followup`)
7. Layout + ניווט RTL
8. עמוד `/contacts`
9. עמוד `/contacts/[id]`
10. עמוד `/tasks`
11. עמוד `/deals`
12. דשבורד `/`
13. Push ל-GitHub → Vercel (עם משתני הסביבה של Green API)
