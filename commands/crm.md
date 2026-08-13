# סקיל: בניית CRM מאפס

בנה CRM עסקי בעברית (RTL) מאפס לפי המפרט הבא. עקוב אחרי הסדר המפורט ואל תדלג על שלבים.

---

## טק סטאק

- **Framework:** Next.js 16.2.9 עם Turbopack (לא Next.js רגיל — יש breaking changes)
- **UI:** Tailwind CSS + shadcn/ui (base-ui variant, לא radix)
- **ORM:** Prisma עם PostgreSQL (Neon cloud database)
- **Deploy:** Vercel עם GitHub auto-deploy
- **שפה:** TypeScript strict

---

## סכמת Prisma

```prisma
model Contact {
  id             Int          @id @default(autoincrement())
  name           String
  phone          String?
  email          String?
  company        String?
  notes          String?
  status         String       @default("חדש")
  bookCount      String?
  whatsappSummary String?
  createdAt      DateTime     @default(now())
  deals          Deal[]
  tasks          Task[]
  activities     Activity[]
}

model Deal {
  id          Int       @id @default(autoincrement())
  title       String
  value       Float?
  stage       String    @default("ליד")
  notes       String?
  contactId   Int?
  contact     Contact?  @relation(fields: [contactId], references: [id])
  tasks       Task[]
  createdAt   DateTime  @default(now())
}

model Task {
  id          Int       @id @default(autoincrement())
  title       String
  dueDate     DateTime?
  completed   Boolean   @default(false)
  contactId   Int?
  contact     Contact?  @relation(fields: [contactId], references: [id])
  dealId      Int?
  deal        Deal?     @relation(fields: [dealId], references: [id])
  createdAt   DateTime  @default(now())
}

model Activity {
  id          Int       @id @default(autoincrement())
  type        String
  note        String
  contactId   Int
  contact     Contact   @relation(fields: [contactId], references: [id])
  createdAt   DateTime  @default(now())
}
```

---

## עמודים ופיצ'רים

### `/contacts` — רשימת אנשי קשר
- טבלה עם: שם (לינק לפרופיל), טלפון + כפתור וואטסאפ 💬, משימה ראשונה ממתינה + כפתור ✓ לסימון כהושלם, סטטוס (dropdown inline), כמה ספרים (1/2/3+), כפתור 💬 פולו אפ עם 4 תבניות הודעה, עריכה, מחיקה
- חיפוש + פילטר לפי סטטוס
- ניתן להוסיף איש קשר חדש (Dialog)
- סטטוסים: חדש / 🔥 חם / 🌡️ פושר / ❄️ קר / ⚙️ בטיפול / ✅ סגור

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
- סטטיסטיקות מהירות

---

## API Routes

```
GET/POST  /api/contacts
GET/PUT/DELETE /api/contacts/[id]
GET/POST  /api/tasks
GET/PUT/DELETE /api/tasks/[id]
GET/POST  /api/deals
GET/PUT/DELETE /api/deals/[id]
GET/POST  /api/activities
```

כל route שמבצע קריאה ל-DB חייב להכיל:
```typescript
export const dynamic = "force-dynamic";
```

---

## אוטנטיקציה

קוקי פשוט `crm_auth=1`. עמוד `/login` עם סיסמה קבועה. Middleware מגן על כל הנתיבים חוץ מ-`/login`, `/api/auth`, `/api/whatsapp`.

---

## וואטסאפ

פונקציית עזר:
```typescript
export function toWhatsAppNumber(phone: string): string {
  const digits = phone.replace(/\D/g, "");
  if (digits.startsWith("972")) return digits;
  if (digits.startsWith("0")) return "972" + digits.slice(1);
  return "972" + digits;
}
```

קישור: `https://wa.me/{number}?text={encodeURIComponent(message)}`

תבניות פולו אפ (מותאמות אישית עם שם הלקוח):
1. חסרים פרטים
2. סגירה — "האם החלטת להתקדם?"
3. ניסינו לתפוס ללא מענה ❤️
4. צ'ק אין — "איך מתקדם?"

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

---

## בעיות ידועות והימנעות

| בעיה | פתרון |
|------|--------|
| Vercel ESLint errors | `ignoreDuringBuilds: true` בנext.config.ts |
| Turbopack build cache מגיש גרסה ישנה | לשנות קצת בקובץ אחר כדי לאלץ recompile |
| `react-hooks/set-state-in-effect` | ESLint disabled בVercel |
| `DialogTrigger` בbase-ui | להשתמש ב-`render={<Button />}` ולא `asChild` |
| `onValueChange` ב-Select מחזיר `string | null` | לטפל ב-null: `!v \|\| v === "none" ? "" : v` |
| ContactLink לא לחיץ | להשתמש ב-`onClick={() => window.location.href = ...}` במקום `<Link>` |
| API route לא מחזיר `id` של contact | להוסיף `id: true` ב-`select` של `include` |

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
5. API routes (contacts, tasks, deals, activities)
6. Layout + ניווט RTL
7. עמוד `/contacts`
8. עמוד `/contacts/[id]`
9. עמוד `/tasks`
10. עמוד `/deals`
11. דשבורד `/`
12. Push ל-GitHub → Vercel
