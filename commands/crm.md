# סקיל: בניית CRM מאפס

בנה CRM עסקי בעברית (RTL) מאפס לפי המפרט הבא. המפרט הזה הוא שיקוף מלא של אפליקציה שנבנתה ועובדת בפועל — עקוב אחריו במדויק ואל תדלג על שלבים. **לא כולל** דאטה אמיתית של לקוחות ולא לוגו/מיתוג — אלה מוזנים בסוף על ידי המשתמש (ראה "לאחר הבנייה" בסוף הקובץ).

---

## טק סטאק

- **Framework:** Next.js 16.2.10 עם Turbopack (**לא** Next.js רגיל — יש breaking changes. קרא `node_modules/next/dist/docs/` לפני קוד)
- **UI:** Tailwind CSS v4 + shadcn/ui, style `base-nova` (variant של base-ui, **לא** radix)
- **ORM:** Prisma 7 עם PostgreSQL (Neon cloud database), דרך `@prisma/adapter-pg` (לא Prisma Client הישן) + `prisma.config.ts`
- **Deploy:** Vercel עם GitHub auto-deploy
- **וואטסאפ:** Green API — גם שליחה יזומה (פולו-אפ) וגם webhook נכנס
- **שפה:** TypeScript strict
- **פונט:** `next/font/google` — Heebo (`subsets: ["hebrew", "latin"]`)

### ⚠️ breaking change קריטי: `proxy.ts` במקום `middleware.ts`

בגרסה הזו של Next.js קובץ ה-middleware הראשי נקרא **`proxy.ts`** (בשורש הפרויקט, לא בתוך `app/`), עם פונקציה בשם `proxy` (לא `middleware`) ואותו export בשם `config` עם `matcher`. שימוש ב-`middleware.ts` הרגיל **לא יעבוד**.

---

## התקנה ראשונית

```bash
npx create-next-app@latest . --typescript --tailwind --app --src-dir=false --import-alias "@/*"
npx shadcn@latest init   # style: base-nova, baseColor: neutral, rtl: true, iconLibrary: lucide
npx shadcn@latest add button input label textarea checkbox badge card dialog dropdown-menu select table tabs
npm install prisma @prisma/client @prisma/adapter-pg pg --save
npm install -D @types/pg dotenv
npx prisma init
```

`components.json` הצפוי:
```json
{
  "$schema": "https://ui.shadcn.com/schema.json",
  "style": "base-nova",
  "rsc": true,
  "tsx": true,
  "tailwind": { "config": "", "css": "app/globals.css", "baseColor": "neutral", "cssVariables": true, "prefix": "" },
  "iconLibrary": "lucide",
  "rtl": true,
  "aliases": { "components": "@/components", "utils": "@/lib/utils", "ui": "@/components/ui", "lib": "@/lib", "hooks": "@/hooks" }
}
```

---

## סכמת Prisma (`prisma/schema.prisma`)

```prisma
generator client {
  provider = "prisma-client"
  output   = "../lib/generated/prisma"
}

datasource db {
  provider = "postgresql"
}

model Contact {
  id                 Int        @id @default(autoincrement())
  name               String
  phone              String?
  email              String?
  company            String?
  notes              String?
  status             String     @default("חדש")
  isExistingCustomer Boolean    @default(false)
  bookCount          String?
  whatsappSummary    String?
  createdAt          DateTime   @default(now())
  deals              Deal[]
  tasks              Task[]
  activities         Activity[]
}

model Deal {
  id        Int      @id @default(autoincrement())
  title     String
  value     Float?
  stage     String   @default("ליד")
  notes     String?
  contactId Int?
  contact   Contact? @relation(fields: [contactId], references: [id], onDelete: Cascade)
  tasks     Task[]
  createdAt DateTime @default(now())
}

model Task {
  id        Int       @id @default(autoincrement())
  title     String
  dueDate   DateTime?
  completed Boolean   @default(false)
  contactId Int?
  contact   Contact?  @relation(fields: [contactId], references: [id], onDelete: Cascade)
  dealId    Int?
  deal      Deal?     @relation(fields: [dealId], references: [id], onDelete: SetNull)
  createdAt DateTime  @default(now())
  updatedAt DateTime  @default(now()) @updatedAt
}

model Activity {
  id        Int      @id @default(autoincrement())
  type      String
  note      String
  contactId Int
  contact   Contact  @relation(fields: [contactId], references: [id], onDelete: Cascade)
  createdAt DateTime @default(now())
}
```

הערות:
- `Task.updatedAt` חייב `@updatedAt` — הטבלה ב-`/contacts` ממיינת לפיו וכל פולו-אפ דוחה משימה, אז חייב להתעדכן אוטומטית.
- `Contact.bookCount` קיים בסכמה וב-API (נשמר/נקרא) אבל **לא מוצג באף מסך UI כרגע** — שדה legacy, אפשר להשאיר או להסיר.
- `onDelete: Cascade` על `Task.contactId` ו-`Deal.contactId`, ו-`onDelete: SetNull` על `Task.dealId` — כדי שמחיקת איש קשר תמחוק גם משימות/עסקאות משויכות (כמו שההודעה למשתמש ב-`deleteContact` מזהירה).

### `prisma.config.ts` (Prisma 7 — נדרש, לא אופציונלי)

```typescript
import "dotenv/config";
import { defineConfig } from "prisma/config";

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: { path: "prisma/migrations" },
  datasource: { url: process.env["DATABASE_URL"] },
});
```

### `lib/prisma.ts`

```typescript
import { PrismaClient } from "@/lib/generated/prisma/client";
import { PrismaPg } from "@prisma/adapter-pg";

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient | undefined };
const adapter = new PrismaPg({ connectionString: process.env.DATABASE_URL });

export const prisma = globalForPrisma.prisma ?? new PrismaClient({ adapter });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;
```

---

## מבנה הפרויקט

```
proxy.ts
next.config.ts
prisma.config.ts
prisma/schema.prisma
app/
  layout.tsx
  globals.css
  page.tsx                          # דשבורד
  login/page.tsx
  contacts/page.tsx
  contacts/[id]/page.tsx
  tasks/page.tsx
  deals/page.tsx
  api/
    auth/route.ts
    whatsapp/route.ts                # webhook נכנס
    contacts/route.ts
    contacts/[id]/route.ts
    contacts/[id]/followup/route.ts
    tasks/route.ts
    tasks/[id]/route.ts
    deals/route.ts
    deals/[id]/route.ts
    activities/route.ts
components/
  nav.tsx
  contact-form-dialog.tsx
  task-form-dialog.tsx
  deal-form-dialog.tsx
  close-deal-dialog.tsx
  follow-up-menu.tsx
  ui/  (shadcn — button, input, label, textarea, checkbox, badge, card, dialog, dropdown-menu, select, table, tabs)
lib/
  prisma.ts
  statuses.ts
  deal-stages.ts
  whatsapp.ts
  green-api.ts
  date.ts
  utils.ts
```

---

## אוטנטיקציה

קוקי פשוט `crm_auth=1` (httpOnly, 30 יום). עמוד `/login` עם סיסמה קבועה (`CRM_PASSWORD`).

### `proxy.ts` (בשורש הפרויקט — **לא** `middleware.ts`)

```typescript
import { NextRequest, NextResponse } from "next/server";

const PUBLIC_PATHS = ["/login", "/api/auth", "/api/whatsapp"];

export function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const isPublic = PUBLIC_PATHS.some((path) => pathname === path || pathname.startsWith(path + "/"));
  if (isPublic) return NextResponse.next();

  const isAuthed = request.cookies.get("crm_auth")?.value === "1";
  if (!isAuthed) {
    if (pathname.startsWith("/api")) {
      return NextResponse.json({ error: "לא מורשה" }, { status: 401 });
    }
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("next", pathname);
    return NextResponse.redirect(loginUrl);
  }
  return NextResponse.next();
}

export const config = {
  matcher: ["/((?!_next/static|_next/image|favicon.ico|.*\\.(?:png|jpg|jpeg|svg|ico|gif|webp)$).*)"],
};
```

### `app/api/auth/route.ts`

```typescript
import { NextRequest, NextResponse } from "next/server";

export const dynamic = "force-dynamic";

export async function POST(request: NextRequest) {
  const { password } = await request.json();
  if (password !== process.env.CRM_PASSWORD) {
    return NextResponse.json({ error: "סיסמה שגויה" }, { status: 401 });
  }
  const response = NextResponse.json({ ok: true });
  response.cookies.set("crm_auth", "1", {
    httpOnly: true,
    sameSite: "lax",
    secure: process.env.NODE_ENV === "production",
    path: "/",
    maxAge: 60 * 60 * 24 * 30,
  });
  return response;
}

export async function DELETE() {
  const response = NextResponse.json({ ok: true });
  response.cookies.delete("crm_auth");
  return response;
}
```

### `app/login/page.tsx`

טופס בודד: `Input type="password"` + כפתור "כניסה". שולח `POST /api/auth` עם `{ password }`; בהצלחה מנתב ל-`?next=` (או `/`) עם `router.push` + `router.refresh()`. שגיאה מציגה "סיסמה שגויה". מציג לוגו (`/logo.png`) מעל הכרטיס. עטוף ב-`<Suspense>` כי משתמש ב-`useSearchParams`.

---

## Layout וניווט

### `app/layout.tsx`

`<html lang="he" dir="rtl">`, `<body>` עם `<Nav />` ואז `<main>{children}</main>`. `metadata.title` — שם העסק (ברירת מחדל גנרית, יוחלף בפועל).

### `components/nav.tsx`

Header עם לוגו במרכז (`Image src="/logo.png"`, לינק ל-`/`), ומתחתיו שורת ניווט: דשבורד (`/`) / אנשי קשר (`/contacts`) / משימות (`/tasks`) / עסקאות (`/deals`), עם הדגשת הקישור הפעיל לפי `usePathname`. כפתור "התנתקות" בקצה שקורא ל-`DELETE /api/auth` ומנתב ל-`/login`. לא מוצג בכלל בעמוד `/login` עצמו (`if (pathname === "/login") return null;`).

---

## עמודים ופיצ'רים

### `/` — דשבורד

טוען `contacts` + `tasks` + `deals` במקביל (`Promise.all`). מציג שורת כרטיסי סטטיסטיקה (`grid-cols-2 md:grid-cols-3 lg:grid-cols-6`):

1. אנשי קשר — `contacts.length`
2. משימות ממתינות — `tasks.filter(!completed).length`
3. משימות באיחור — `!completed && dueDate < now`, באדום אם > 0
4. עסקאות פתוחות — `stage` שאינו `"סגור-נוצח"`/`"סגור-הפסד"`
5. שווי עסקאות פתוחות — סכום `value` של הפתוחות
6. **💰 הכנסות מעסקאות סגורות** — סכום `value` של כל העסקאות בשלב `"סגור-נוצח"` (כולל אלו שנוצרו אוטומטית דרך `CloseDealDialog` ב-`/contacts`)

מתחת, שלושה כרטיסים (`grid-cols-1 md:grid-cols-3`):
- **💰 מכירות אחרונות** — 5 עסקאות `"סגור-נוצח"` אחרונות (`createdAt` יורד), שם לקוח (לינק לפרופיל) + סכום
- **אנשי קשר אחרונים** — 5 אחרונים לפי `createdAt`, עם Badge סטטוס
- **משימות באיחור** — רשימת המשימות מסעיף 3, לינק ל-`/tasks`

### `/contacts` — רשימת אנשי קשר

טבלה עם העמודות הבאות (מימין לשמאל, לפי RTL):

1. **שם** — לינק לפרופיל, ומתחתיו תאריך ההצטרפות (`formatJoinDate` — יום + חודש בעברית, שעה)
2. **חדש/קיים** — Badge לחיץ שמחליף `isExistingCustomer` בלחיצה בודדת: 🆕 חדש (כחול) / 🔁 קיים (אדום)
3. **טלפון** — מספר הטלפון + אמוג'י 💬 שפותח קישור `https://wa.me/{toWhatsAppNumber(phone)}` בטאב חדש (שיחה ידנית, לא דרך ה-API)
4. **משימה קרובה** — כותרת המשימה הראשונה שלא הושלמה + כפתור ✓ לסימון כהושלם + כפתור ✏️ לעריכה + כפתור 📅 להוספת משימה חדשה. מתחת לזה, אם קיימת, שורת הפעילות האחרונה מסוג "פולו אפ" (`activities[0].note`) בירוק
5. **תאריך שינוי משימה** — תאריך ושעה של `task.updatedAt` בשתי שורות ("תאריך - ..." / "שעה - ...") מודגשות
6. **שם העסק** — Badge לחיץ עם 🏢, פותח `Input` inline לעריכה (שמירה ב-blur או Enter, ביטול ב-Escape). אם ריק: "הוסף עסק"
7. **סטטוס** — `Select` עם Badge צבעוני בערך הנבחר, פותח את כל האפשרויות (ראה סטטוסים למטה)
8. **פעולות** — תפריט נפתח 🗨️ פולו אפ (ראה וואטסאפ למטה) + כפתור ✏️ עריכת איש קשר + כפתור 🗑️ מחיקה

**התנהגות נוספת:**
- שורות בסטטוס "✅ סגור" מודגשות ברקע ירוק (`bg-green-100 hover:bg-green-100`)
- מיון: לפי `task.updatedAt` של המשימה הקרובה (או `createdAt` אם אין משימה), מהחדש לישן
- חיפוש (debounce ~250ms, מול `search`/`status` ב-query string) + פילטר סטטוס, שניהם מול ה-API
- הוספת איש קשר חדש דרך Dialog (`ContactFormDialog`)
- מחיקה מציגה `confirm()`: "למחוק את איש הקשר? הפעולה תמחק גם משימות ועסקאות משויכות."
- **חשוב לגבי רוחב הטבלה:** להשתמש ב-`table-fixed` עם רוחבים קבועים לעמודות הצדדיות (שם/פעולות) ו-`truncate`/`line-clamp` על טקסט ארוך (שם עסק, כותרת משימה) כדי שהטבלה לא תגלוש הצידה. כותרות עמודות (`TableHead`) צריכות לאפשר `whitespace-normal` כדי שלא יחפפו זו את זו.

**סגירת עסקה מהטבלה (`CloseDealDialog`):**
בחירת "✅ סגור" ב-`Select` הסטטוס **לא** שומרת סטטוס מיד — היא פותחת `CloseDealDialog` ששואל "כמה הלקוח קנה? (₪)". רק בשליחת הטופס:
1. הסטטוס של איש הקשר נשמר בפועל כ-"✅ סגור" (`updateStatus`)
2. נוצרת `Deal` חדשה: `POST /api/deals` עם `{ title: "מכירה - {שם}", value: הסכום, stage: "סגור-נוצח", contactId }`

אם המשתמש סוגר את הדיאלוג בלי לשלוח (Escape / קליק על הרקע / `onOpenChange(false)`), הסטטוס **לא** משתנה — כי ה-`Select` מוצג לפי `contact.status` שטרם עודכן, כך שהוא חוזר לערך הקודם אוטומטית. בחירה בכל סטטוס אחר (לא "✅ סגור") שומרת מיד כרגיל. **הערה:** בחירה חוזרת ב-"✅ סגור" כשהסטטוס כבר "✅ סגור" לא מפעילה שוב את הדיאלוג (אין עדיין מנגנון למכירה חוזרת ללקוח שכבר סגור).

`components/close-deal-dialog.tsx`:
```typescript
"use client";

import { useState, useEffect } from "react";
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogFooter } from "@/components/ui/dialog";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";

export function CloseDealDialog({
  open, onOpenChange, contactName, onConfirm,
}: {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  contactName: string;
  onConfirm: (amount: number) => Promise<void> | void;
}) {
  const [amount, setAmount] = useState("");
  const [saving, setSaving] = useState(false);

  useEffect(() => { if (open) setAmount(""); }, [open]);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setSaving(true);
    await onConfirm(Number(amount));
    setSaving(false);
    onOpenChange(false);
  }

  return (
    <Dialog open={open} onOpenChange={onOpenChange}>
      <DialogContent dir="rtl" className="max-w-sm">
        <DialogHeader>
          <DialogTitle>✅ סגירת עסקה עם {contactName}</DialogTitle>
        </DialogHeader>
        <form onSubmit={handleSubmit} className="flex flex-col gap-4">
          <div>
            <Label htmlFor="closeAmount">כמה הלקוח קנה? (₪)</Label>
            <Input id="closeAmount" type="number" min="0" step="0.01" required autoFocus
              value={amount} onChange={(e) => setAmount(e.target.value)} />
          </div>
          <DialogFooter>
            <Button type="submit" disabled={saving}>{saving ? "שומר..." : "סגירה ורישום עסקה"}</Button>
          </DialogFooter>
        </form>
      </DialogContent>
    </Dialog>
  );
}
```

בעמוד `/contacts`, קריאת החיבור בין ה-`Select` ל-`CloseDealDialog`:
```typescript
const CLOSED_STATUS = "✅ סגור";
const [closeDealContact, setCloseDealContact] = useState<{ id: number; name: string } | null>(null);

async function confirmCloseDeal(amount: number) {
  if (!closeDealContact) return;
  await updateStatus(closeDealContact.id, CLOSED_STATUS);
  await fetch("/api/deals", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      title: `מכירה - ${closeDealContact.name}`,
      value: amount,
      stage: "סגור-נוצח",
      contactId: closeDealContact.id,
    }),
  });
  setCloseDealContact(null);
}

// ב-Select של הסטטוס:
onValueChange={(v) => {
  if (!v) return;
  if (v === CLOSED_STATUS) setCloseDealContact({ id: contact.id, name: contact.name });
  else updateStatus(contact.id, v);
}}
```

### `/contacts/[id]` — פרופיל לקוח

עמוד יחיד (`max-w-4xl`) עם:
- כותרת: שם + חברה (אם יש) + Badge סטטוס + Badge חדש/קיים, וכפתורי 🗨️ פולו אפ (`FollowUpMenu`) + ✏️ עריכה (פותח את אותו `ContactFormDialog`)
- כרטיס **פרטים**: טלפון (לינק `wa.me`), אימייל, הערות, סיכום וואטסאפ (אם קיים)
- כרטיס **עסקאות (N)**: כל `deal` — כותרת, שווי, Badge שלב
- כרטיס **משימות (N)**: checkbox להשלמה, קו חוצה על משימה שהושלמה, תאריך יעד, עריכה (`TaskFormDialog`) ומחיקה. כפתור "+ הוסף משימה". אם קיימת פעילות מסוג "פולו אפ" — מוצגת כרצועה ירוקה מתחת לרשימה
- כרטיס **היסטוריית פעילות (N)**: טופס להוספת הערה חופשית (`type: "תהליך"`, `POST /api/activities`) ואז רשימת כל הפעילויות (Badge סוג + תאריך + טקסט), מהחדש לישן

### `/tasks` — משימות

שתי טבלאות נפרדות: **ממתינות (N)** ו-**הושלמו (N)** (השנייה מוצגת רק אם יש בה פריטים). עמודות: שם (לינק צבעוני — כל לקוח מקבל צבע פסאודו-רנדומלי קבוע מתוך 5 אפשרויות לפי `id % 5`), טלפון + 💬, משימה (עם הפעילות האחרונה של הלקוח אם יש), סטטוס (תגית תאריך), פעולות (🗨️ פולו אפ בגרסת pill + checkbox השלמה + ✏️ עריכה + ✕ מחיקה).

מיון: לפי `dueDate` עולה, ללא תאריך בסוף.

תגיות סטטוס (`dueTag`, על משימות לא-מושלמות עם `dueDate`):
- `due < now` → 🔴 `⚠️ באיחור {תאריך} {שעה}`
- `due < startOfTomorrow` (כלומר היום) → 🟠 `🔥 היום {שעה}`
- אחרת → 🔵 `📅 {תאריך} {שעה}`

### `/deals` — עסקאות

לוח Kanban לפי `DEAL_STAGES` (`grid-cols-1 md:grid-cols-3 lg:grid-cols-5`), עמודה לכל שלב עם ספירה ושווי כולל (`{count} · ₪{total}`). כל עסקה היא כרטיס לחיץ (פותח עריכה) עם כותרת, שווי, לינק ללקוח, וכפתור מחיקה. כפתור "+ עסקה חדשה" פותח `DealFormDialog` (כותרת, שווי, שלב, לקוח, הערות).

---

## סטטוסים (`lib/statuses.ts`)

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

## שלבי עסקה (`lib/deal-stages.ts`)

```typescript
export const DEAL_STAGES = ["ליד", "מו\"מ", "הצעת מחיר", "סגור-נוצח", "סגור-הפסד"];

export const DEAL_STAGE_COLORS: Record<string, string> = {
  "ליד": "bg-blue-100 text-blue-800",
  "מו\"מ": "bg-yellow-100 text-yellow-800",
  "הצעת מחיר": "bg-orange-100 text-orange-800",
  "סגור-נוצח": "bg-green-100 text-green-800",
  "סגור-הפסד": "bg-red-100 text-red-800",
};
```

---

## API Routes

```
GET/POST       /api/contacts                          # ?search=&status=
GET/PUT/DELETE /api/contacts/[id]
POST           /api/contacts/[id]/followup
POST           /api/whatsapp                            # webhook נכנס (ציבורי, מוגן ב-secret header)
GET/POST       /api/tasks
GET/PUT/DELETE /api/tasks/[id]
GET/POST       /api/deals
GET/PUT/DELETE /api/deals/[id]
GET/POST       /api/activities                          # ?contactId=
POST/DELETE    /api/auth
```

כל route שמבצע קריאה ל-DB חייב להכיל:
```typescript
export const dynamic = "force-dynamic";
```

### `GET /api/contacts`
`include`: `tasks` (רק לא-מושלמות, `orderBy dueDate asc`, `take 1`) + `activities` (רק `type: "פולו אפ"`, `orderBy createdAt desc`, `take 1`). חיפוש (`?search=`) עם `OR` על `name`/`phone`/`email`/`company` (`mode: "insensitive"`), פילטר (`?status=`) מדויק. `orderBy createdAt desc`.

### `GET /api/contacts/[id]`
`include`: `deals` (desc), `tasks` (`dueDate asc`), `activities` (desc) — הכל, לא רק אחד, לשימוש בפרופיל.

### `/api/contacts/[id]/followup`
מקבל `{ message }`, שולח בפועל דרך Green API, רושם `Activity` מסוג "פולו אפ", ודוחה את המשימה הפתוחה הקרובה (`orderBy dueDate asc`) ליום העסקים הבא:

```typescript
import { sendWhatsAppMessage } from "@/lib/green-api";
import { nextBusinessDay } from "@/lib/date";

const contact = await prisma.contact.findUnique({ where: { id: Number(id) } });
if (!contact || !contact.phone) return NextResponse.json({ error: "לא נמצא איש קשר עם טלפון" }, { status: 400 });

await sendWhatsAppMessage(contact.phone, message); // זורק אם נכשל -> 502

await prisma.activity.create({
  data: { type: "פולו אפ", note: `פולו אפ נשלח ${dateLabel} - "${message}"`, contactId: contact.id },
});
const nextTask = await prisma.task.findFirst({ where: { contactId: contact.id, completed: false }, orderBy: { dueDate: "asc" } });
if (nextTask) await prisma.task.update({ where: { id: nextTask.id }, data: { dueDate: nextBusinessDay(now) } });
```

### `POST /api/whatsapp` — webhook נכנס (ליד חדש/הודעה מוואטסאפ)

ציבורי (רשום ב-`PUBLIC_PATHS` ב-`proxy.ts`) אבל מוגן בכותרת סוד:

```typescript
const secret = request.headers.get("x-webhook-secret");
if (!secret || secret !== process.env.WHATSAPP_WEBHOOK_SECRET) {
  return NextResponse.json({ error: "לא מורשה" }, { status: 401 });
}

const { phone, name, message } = await request.json();
const normalized = toWhatsAppNumber(phone);

// מחפש התאמה ידנית (לא query ב-DB) כי מספרי טלפון מאוחסנים בפורמטים שונים
const allContacts = await prisma.contact.findMany({ where: { phone: { not: null } }, select: { id: true, phone: true } });
const match = allContacts.find((c) => c.phone && toWhatsAppNumber(c.phone) === normalized);

const contact = match
  ? await prisma.contact.update({ where: { id: match.id }, data: { name: name?.trim() ? name : undefined } })
  : await prisma.contact.create({ data: { name: name?.trim() ? name : `ליד וואטסאפ ${phone}`, phone, status: "חדש" } });

await prisma.activity.create({ data: { type: "וואטסאפ", note: message || "פנייה חדשה בוואטסאפ", contactId: contact.id } });

return NextResponse.json({ ok: true, contactId: contact.id, isNew: !match });
```

### שאר ה-routes
CRUD סטנדרטי (`prisma.<model>.findMany/create/update/delete`) עם `Number(id)` להמרת פרמטרים, `?? null` על שדות אופציונליים. `tasks` כולל `contact.activities` (פולו-אפ אחרון) ו-`deal`; `deals` כולל `contact`.

---

## אוטנטיקציה (סיכום)

קוקי פשוט `crm_auth=1`. עמוד `/login` עם סיסמה קבועה (`CRM_PASSWORD`). `proxy.ts` (ראה למעלה) מגן על כל הנתיבים חוץ מ-`/login`, `/api/auth`, `/api/whatsapp`.

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

### `lib/date.ts`

```typescript
export function nextBusinessDay(from: Date = new Date()): Date {
  const d = new Date(from);
  d.setDate(d.getDate() + 1);
  while (d.getDay() === 5 || d.getDay() === 6) {
    d.setDate(d.getDate() + 1);
  }
  return d;
}
```

### `lib/whatsapp.ts`

```typescript
export function toWhatsAppNumber(phone: string): string {
  const digits = phone.replace(/\D/g, "");
  if (digits.startsWith("972")) return digits;
  if (digits.startsWith("0")) return "972" + digits.slice(1);
  return "972" + digits;
}

export const followUpTemplates = [
  { id: "missing-details", label: "חסרים פרטים", build: () => `היי, יצא לך לעבור על הקטלוג צריך הזמנה?` },
  { id: "closing", label: "סגירה", build: (name: string) => `היי ${name}, רציתי לדעת אם החלטת להתקדם - אנחנו כאן לכל שאלה :)` },
  { id: "no-answer", label: "ניסינו לתפוס ללא מענה", build: (name: string) => `היי ${name}, ניסינו לתפוס אותך ללא מענה :)` },
  { id: "check-in", label: "צ'ק אין", build: (name: string) => `היי ${name}, איך מתקדם? רציתי לשמוע ממך :)` },
];
```

קישור ידני (כפתור 💬 בעמודת הטלפון, ובפרופיל): `https://wa.me/{toWhatsAppNumber(phone)}` — נפתח בטאב חדש לשיחה ידנית, בלי לשלוח כלום אוטומטית.

### `components/follow-up-menu.tsx` — תפריט 🗨️ פולו אפ

`DropdownMenu` (לא Dialog!) שמוצג בשתי צורות — `pill` (עגול, קומפקטי, לעמודות צפופות כמו ב-`/tasks`) או רגיל (`ghost` button, כמו ב-`/contacts`). לא מוצג כלל אם אין `phone` (`if (!phone) return null;`). כותרת התפריט: "⚡ פולו אפ ל{name}". כל פריט הוא תבנית מ-`followUpTemplates` שמוצגת מלאה (הטקסט הסופי, לא רק התווית) ובלחיצה שולחת ישירות דרך `POST /api/contacts/[id]/followup` (**לא** פותחת `wa.me`). מציג "שולח..." ומנטרל את עצמו בזמן השליחה. בשגיאה — `alert(data.error)`.

```typescript
"use client";
import { useState } from "react";
import { DropdownMenu, DropdownMenuContent, DropdownMenuItem, DropdownMenuTrigger } from "@/components/ui/dropdown-menu";
import { Button } from "@/components/ui/button";
import { followUpTemplates } from "@/lib/whatsapp";

export function FollowUpMenu({ contactId, name, phone, onSent, pill = false }: {
  contactId: number; name: string; phone: string; onSent?: () => void; pill?: boolean;
}) {
  const [sending, setSending] = useState(false);
  if (!phone) return null;

  async function send(message: string) {
    setSending(true);
    try {
      const res = await fetch(`/api/contacts/${contactId}/followup`, {
        method: "POST", headers: { "Content-Type": "application/json" }, body: JSON.stringify({ message }),
      });
      if (!res.ok) {
        const data = await res.json().catch(() => ({}));
        alert(data.error || "שליחת ההודעה נכשלה");
        return;
      }
      onSent?.();
    } finally { setSending(false); }
  }

  return (
    <DropdownMenu>
      <DropdownMenuTrigger render={
        <Button variant={pill ? "outline" : "ghost"} size="sm" disabled={sending}
          className={pill ? "h-6 rounded-full border-green-300 bg-green-50 px-2 text-xs text-green-800 hover:bg-green-100" : undefined}>
          {sending ? "שולח..." : "🗨️ פולו אפ"}
        </Button>
      } />
      <DropdownMenuContent align="end" dir="rtl" className="flex w-80 flex-col gap-2 rounded-2xl p-3">
        <div className="border-b px-1 pb-2 text-right font-semibold">⚡ פולו אפ ל{name}</div>
        {followUpTemplates.map((template) => (
          <DropdownMenuItem key={template.id} onClick={() => send(template.build(name))}
            className="whitespace-normal rounded-lg border p-3 text-center text-sm leading-relaxed data-highlighted:bg-accent">
            {template.build(name)}
          </DropdownMenuItem>
        ))}
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

---

## Dialogs (טפסים)

כולם `"use client"`, `Dialog` מ-`components/ui/dialog` עם `dir="rtl"`, מחזיקים `values` state, טוענים `initial` ב-`useEffect` כש-`open` הופך `true`, ושולחים `POST` (חדש) או `PUT` (יש `id`) ואז `onOpenChange(false)` + `onSaved()`.

- **`ContactFormDialog`** — שם*, טלפון, אימייל, חברה, checkbox "לקוח קיים (לא חדש)", `Select` סטטוס, הערות (Textarea), סיכום וואטסאפ (Textarea).
- **`TaskFormDialog`** — כותרת*, תאריך+שעה (`type="datetime-local"`).
- **`DealFormDialog`** — כותרת*, שווי (₪, `type="number"`), `Select` שלב (`DEAL_STAGES`), `Select` איש קשר (טוען `GET /api/contacts` כש-`open`, כולל "ללא איש קשר"), הערות.
- **`CloseDealDialog`** — ראה קוד מלא למעלה בסעיף `/contacts`.

`onValueChange` של `Select` תמיד מטופל כך (base-ui מחזיר `string | null`):
```typescript
onValueChange={(v) => setValues({ ...values, field: !v || v === "none" ? "" : v })}
```

---

## הגדרות Vercel / Next.js

`next.config.ts` **ריק** (ללא `eslint.ignoreDuringBuilds` — הפרויקט עובר lint נקי עם `eslint-config-next` הרגיל):
```typescript
import type { NextConfig } from "next";
const nextConfig: NextConfig = {};
export default nextConfig;
```

`package.json` scripts:
```json
{
  "dev": "next dev --turbopack",
  "build": "prisma generate && next build --turbopack",
  "postinstall": "prisma generate",
  "start": "next start",
  "lint": "eslint"
}
```

### משתני סביבה נדרשים

```
DATABASE_URL=...              # Neon PostgreSQL
CRM_PASSWORD=...              # סיסמת הכניסה ל-/login
GREEN_API_ID_INSTANCE=...
GREEN_API_API_TOKEN=...
WHATSAPP_WEBHOOK_SECRET=...   # מגן על /api/whatsapp (header x-webhook-secret)
```

---

## בעיות ידועות והימנעות

| בעיה | פתרון |
|------|--------|
| `middleware.ts` לא נטען / לא חוסם דפים | הקובץ חייב להיקרא `proxy.ts` בשורש, עם `export function proxy` (לא `middleware`) — breaking change של הגרסה הזו |
| Prisma Client הישן (`new PrismaClient()` בלי adapter) לא מתחבר | Prisma 7 דורש `@prisma/adapter-pg` + `prisma.config.ts` — לא מספיק `DATABASE_URL` לבד |
| `DialogTrigger` בbase-ui | להשתמש ב-`render={<Button />}` ולא `asChild` |
| `onValueChange` ב-Select מחזיר `string \| null` | לטפל ב-null: `!v \|\| v === "none" ? "" : v` |
| ContactLink לא לחיץ | להשתמש ב-`onClick={() => window.location.href = ...}` במקום `<Link>` (או פשוט `<a href>` רגיל כמו בקוד בפועל) |
| API route לא מחזיר `id` של contact | להוסיף `id: true` ב-`select` של `include` |
| טבלת `/contacts` גולשת הצידה עם הרבה עמודות | `table-fixed` + רוחב קבוע לעמודות הצדדיות + `truncate` על טקסט ארוך (שם עסק, כותרת משימה) |
| כותרות עמודות חופפות זו את זו | לאפשר `whitespace-normal` על `TableHead` כדי שהתווית תגלוש לשורה שנייה |
| כפתור פולו-אפ תופס יותר מדי מקום בעמודת הפעולות | להשתמש בגרסה מכווצת (`pill`, `h-6 rounded-full`) כשצריך לחסוך רוחב |
| מחיקת איש קשר משאירה משימות/עסקאות יתומות | `onDelete: Cascade` על `Task.contactId` ו-`Deal.contactId` בסכמה |

---

## RTL

כל Dialog/חלון: `dir="rtl"`.
Layout ראשי: `dir="rtl"` ב-`html`, לא רק ב-`div` פנימי.
כל הטקסטים, כפתורים, ותוויות — בעברית.

---

## סדר בנייה מומלץ

1. `create-next-app` + `shadcn init` (base-nova, rtl: true) + הוספת קומפוננטות UI
2. `prisma.config.ts` + סכמת Prisma + `lib/prisma.ts` (עם `@prisma/adapter-pg`) + `prisma generate` + `prisma db push`
3. `next.config.ts` ריק, `package.json` scripts
4. `proxy.ts` (⚠️ לא `middleware.ts`) + `app/api/auth/route.ts` + `app/login/page.tsx`
5. `lib/whatsapp.ts` (המרת מספר + תבניות) + `lib/green-api.ts` (שליחה בפועל) + `lib/date.ts` (`nextBusinessDay`) + `lib/statuses.ts` + `lib/deal-stages.ts`
6. API routes: contacts, tasks, deals, activities, `contacts/[id]/followup`, `whatsapp` (webhook נכנס)
7. `app/layout.tsx` + `components/nav.tsx` (RTL, לוגו placeholder)
8. Dialogs: `contact-form-dialog`, `task-form-dialog`, `deal-form-dialog`, `close-deal-dialog`, `follow-up-menu`
9. `app/contacts/page.tsx` (כולל חיבור ל-`CloseDealDialog`)
10. `app/contacts/[id]/page.tsx`
11. `app/tasks/page.tsx`
12. `app/deals/page.tsx`
13. `app/page.tsx` (דשבורד, כולל כרטיס ההכנסות)
14. Push ל-GitHub → Vercel

---

## לאחר הבנייה — מה המשתמש צריך להשלים

הבנייה מסתיימת עם אפליקציה ריקה מדאטה וללא מיתוג. לפני שימוש בפועל, בקש מהמשתמש:

1. **Green API** — יצירת instance ב-[green-api.com](https://green-api.com), והזנת `GREEN_API_ID_INSTANCE` + `GREEN_API_API_TOKEN` (ב-`.env` מקומי וב-Vercel Environment Variables)
2. **לוגו ומיתוג** — החלפת `public/logo.png` בלוגו של העסק, ועדכון `alt`/`metadata.title` ב-`app/layout.tsx` ו-`components/nav.tsx` לשם העסק בפועל
3. **סיסמת כניסה** — קביעת `CRM_PASSWORD` משלו
4. **בסיס נתונים** — יצירת Neon Postgres (או ספק אחר) והזנת `DATABASE_URL`, ואז `prisma db push`
5. **(אופציונלי) Webhook נכנס** — אם רוצים ליצור אנשי קשר אוטומטית מהודעות וואטסאפ נכנסות, להגדיר `WHATSAPP_WEBHOOK_SECRET` ולחבר אותו ב-Green API כ-webhook ל-`/api/whatsapp`

**אין** להזין דאטה אמיתית של לקוחות בקוד או בסקיל הזה — כל לקוח מוזן בפועל דרך ה-UI (`+ איש קשר חדש`) אחרי שהמערכת רצה.
