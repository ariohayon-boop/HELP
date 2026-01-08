# HELP - מערכת ניהול לידים לעסקים

> המזכירה שעובדת בזמן שאתה חי 🚀

## 📋 תוכן עניינים

- [סקירה כללית](#סקירה-כללית)
- [מבנה הפרויקט](#מבנה-הפרויקט)
- [התקנה והגדרה](#התקנה-והגדרה)
- [הגדרת Supabase](#הגדרת-supabase)
- [העלאה לאוויר](#העלאה-לאוויר)
- [שימוש במערכת](#שימוש-במערכת)
- [מבנה הנתונים](#מבנה-הנתונים)
- [פתרון בעיות](#פתרון-בעיות)

---

## 🎯 סקירה כללית

HELP היא מערכת SaaS לניהול לידים ואוטומציה של WhatsApp לעסקים קטנים ו-solopreneurs.

### רכיבי המערכת:

| רכיב | תיאור | קובץ |
|------|-------|------|
| **דף נחיתה** | דף שיווקי לגיוס לידים | `index.html` |
| **מערכת CRM** | ניהול לידים, סטטוסים, הערות | `crm.html` |
| **Supabase** | בסיס נתונים + Realtime | ענן |

---

## 📁 מבנה הפרויקט

```
help/
├── index.html          # דף הנחיתה (Landing Page)
├── crm.html            # מערכת ה-CRM
└── README.md           # מסמך זה
```

---

## ⚙️ התקנה והגדרה

### שלב 1: הורדת הקבצים

הורד את שני הקבצים (`index.html` ו-`crm.html`) לתיקייה בממחשב שלך.

### שלב 2: קבלת Supabase Anon Key

1. היכנס ל-[Supabase Dashboard](https://supabase.com/dashboard)
2. בחר את הפרויקט שלך (או צור חדש)
3. לך ל-**Settings** → **API**
4. העתק את ה-**anon public** key

### שלב 3: עדכון הקבצים

בשני הקבצים (`index.html` ו-`crm.html`), מצא את השורה:

```javascript
const SUPABASE_ANON_KEY = 'YOUR_ANON_KEY_HERE';
```

והחלף את `YOUR_ANON_KEY_HERE` ב-key שהעתקת.

---

## 🗄️ הגדרת Supabase

### יצירת טבלת הלידים

1. היכנס ל-Supabase Dashboard
2. לך ל-**SQL Editor**
3. הדבק את הקוד הבא והרץ אותו:

```sql
-- =============================================
-- HELP CRM - Database Schema
-- =============================================

-- טבלת לידים ראשית
CREATE TABLE help_leads (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    
    -- פרטי הליד
    name TEXT NOT NULL,
    phone TEXT NOT NULL,
    business_type TEXT NOT NULL,
    clients_per_day TEXT,
    
    -- סטטוס
    status TEXT DEFAULT 'new' CHECK (status IN (
        'new',                  -- חדש
        'followup_initial',     -- פולואפ ראשוני
        'followup_qualified',   -- פולואפ איכותי
        'interested',           -- מעוניין
        'demo',                 -- הדגמה
        'closed_won',           -- נסגר
        'not_interested'        -- לא מעוניין
    )),
    
    -- סיבת דחייה (אם לא מעוניין)
    rejection_reason TEXT CHECK (rejection_reason IN (
        'expensive',       -- יקר
        'bad_timing',      -- טיימינג לא טוב
        'not_connected',   -- לא התחבר למוצר
        NULL
    )),
    
    -- תזכורת
    reminder_date DATE,
    
    -- הערות ופעילות (JSON)
    notes JSONB DEFAULT '[]',
    activities JSONB DEFAULT '[]',
    
    -- חותמות זמן
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- =============================================
-- אינדקסים לביצועים
-- =============================================

CREATE INDEX idx_help_leads_status ON help_leads(status);
CREATE INDEX idx_help_leads_created_at ON help_leads(created_at DESC);
CREATE INDEX idx_help_leads_reminder_date ON help_leads(reminder_date);
CREATE INDEX idx_help_leads_phone ON help_leads(phone);

-- =============================================
-- טריגר לעדכון אוטומטי של updated_at
-- =============================================

CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_update_help_leads_updated_at
    BEFORE UPDATE ON help_leads
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- =============================================
-- הפעלת Realtime לעדכונים בזמן אמת
-- =============================================

ALTER TABLE help_leads REPLICA IDENTITY FULL;

-- =============================================
-- Row Level Security (RLS)
-- =============================================

ALTER TABLE help_leads ENABLE ROW LEVEL SECURITY;

-- מדיניות גישה - מאפשר הכל (שנה לפי הצורך)
CREATE POLICY "Allow all operations on help_leads" 
    ON help_leads 
    FOR ALL 
    USING (true) 
    WITH CHECK (true);

-- =============================================
-- הוספת הטבלה ל-Realtime publications
-- =============================================

-- וודא שיש publication
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM pg_publication WHERE pubname = 'supabase_realtime'
    ) THEN
        CREATE PUBLICATION supabase_realtime;
    END IF;
END $$;

-- הוסף את הטבלה ל-publication
ALTER PUBLICATION supabase_realtime ADD TABLE help_leads;
```

4. לחץ **Run** להרצת הקוד

### הפעלת Realtime (אם לא עובד אוטומטית)

1. לך ל-**Database** → **Replication**
2. מצא את `help_leads`
3. וודא ש-**Insert**, **Update**, **Delete** מסומנים

---

## 🚀 העלאה לאוויר

### אופציה 1: Vercel (מומלץ)

#### דף נחיתה:
1. צור repo חדש ב-GitHub
2. העלה את `index.html`
3. חבר ל-[Vercel](https://vercel.com)
4. Deploy!

#### CRM (בנפרד או באותו repo):
- אם באותו repo: גש ל-`yoursite.vercel.app/crm.html`
- אם נפרד: צור repo נוסף ו-deploy

### אופציה 2: Netlify

1. גרור את התיקייה ל-[Netlify Drop](https://app.netlify.com/drop)
2. זהו! 🎉

### אופציה 3: GitHub Pages

1. העלה ל-GitHub
2. Settings → Pages → Deploy from branch
3. בחר `main` ו-`/ (root)`

---

## 📖 שימוש במערכת

### דף הנחיתה

1. לקוח ממלא טופס
2. הנתונים נשלחים ל-Supabase
3. ליד חדש נוצר עם סטטוס `new`

### מערכת ה-CRM

#### דשבורד
- צפייה בסטטיסטיקות כלליות
- לידים אחרונים
- פירוט לפי סטטוס

#### ניהול לידים
- **חיפוש**: לפי שם, טלפון, או סוג עסק
- **סינון**: לפי סטטוס או תאריך
- **לחיצה על ליד**: פתיחת פאנל פרטים

#### פאנל ליד
- **Quick Actions**: WhatsApp, התקשר, תזכורת
- **עדכון סטטוס**: לחיצה על הסטטוס הרצוי
- **הוספת הערות**: תיעוד שיחות ומידע
- **היסטוריה**: צפייה בכל הפעילות

#### סטטוסים זמינים

| סטטוס | משמעות | צבע |
|-------|--------|-----|
| חדש | ליד חדש שנכנס | סגול |
| פולואפ ראשוני | יצרנו קשר, לא זמין כרגע | כתום |
| פולואפ איכותי | יודע מוצר ומחיר | כחול |
| מעוניין | הביע עניין | ירוק |
| הדגמה | קבע הדגמה | כחול |
| נסגר | הפך ללקוח! | ירוק |
| לא מעוניין | סירב | אדום |

#### סיבות לדחייה
- **יקר לי**: המחיר גבוה
- **טיימינג לא טוב**: לחזור בתאריך מסוים
- **לא התחבר למוצר**: לא מתאים לו

---

## 📊 מבנה הנתונים

### טבלת help_leads

| שדה | סוג | תיאור |
|-----|-----|-------|
| `id` | UUID | מזהה ייחודי |
| `name` | TEXT | שם הליד |
| `phone` | TEXT | טלפון |
| `business_type` | TEXT | סוג העסק |
| `clients_per_day` | TEXT | כמות לקוחות ביום |
| `status` | TEXT | סטטוס נוכחי |
| `rejection_reason` | TEXT | סיבת דחייה (אם רלוונטי) |
| `reminder_date` | DATE | תאריך תזכורת |
| `notes` | JSONB | מערך הערות |
| `activities` | JSONB | מערך פעילויות |
| `created_at` | TIMESTAMP | תאריך יצירה |
| `updated_at` | TIMESTAMP | תאריך עדכון אחרון |

### מבנה הערה (notes)

```json
{
    "id": "unique-id",
    "text": "תוכן ההערה",
    "author": "שם הכותב",
    "created_at": "2025-01-08T12:00:00Z"
}
```

### מבנה פעילות (activities)

```json
{
    "type": "status_change",
    "text": "סטטוס שונה לפולואפ",
    "created_at": "2025-01-08T12:00:00Z"
}
```

---

## 🔧 פתרון בעיות

### ליד לא נשמר

1. **בדוק את ה-Anon Key**: וודא שהעתקת נכון
2. **בדוק RLS**: וודא שהמדיניות מאפשרת INSERT
3. **בדוק Console**: פתח F12 וחפש שגיאות

### Realtime לא עובד

1. לך ל-Supabase → Database → Replication
2. וודא ש-`help_leads` מסומן
3. בדוק שה-publication קיים

### עיצוב שבור

1. וודא שכל הקוד הועתק נכון
2. בדוק שאין תווים מיוחדים שנוספו

### שגיאת CORS

אם מקבל שגיאת CORS:
1. וודא שה-URL של Supabase נכון
2. בדוק ש-anon key תקין

---

## 🎨 התאמה אישית

### שינוי צבעים

בראש הקובץ, מצא את `:root` ושנה את הצבעים:

```css
:root {
    --primary: #06D6A0;      /* צבע ראשי */
    --secondary: #118AB2;    /* צבע משני */
    --accent: #06FFA5;       /* צבע הדגשה */
    --navy: #073B4C;         /* צבע כהה */
}
```

### הוספת שדות

1. הוסף את השדה לטבלה ב-Supabase
2. הוסף input בטופס
3. עדכן את ה-handleSubmit

---

## 📞 תמיכה

נתקלת בבעיה? 
- פתח Issue ב-GitHub
- צור קשר ישירות

---

## 📜 רישיון

MIT License - השתמש בחופשיות! 🎉

---

**נבנה עם ❤️ בישראל**
