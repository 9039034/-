# מדריך התקנה מלא: Ollama + Open WebUI
### AI מקומי, חינמי לגמרי, עם זיכרון מהצ'אטים · חיפוש באינטרנט · RAG על המסמכים שלך

---

## מה נבנה כאן?

בסוף המדריך יהיה לך ChatGPT פרטי שרץ 100% על המחשב שלך:

- **מנוע:** Ollama (מריץ את המודלים)
- **ממשק:** Open WebUI (צ'אט בדפדפן, נראה כמו ChatGPT)
- **זיכרון:** כל השיחות נשמרות מקומית + פיצ'ר Memory שזוכר עובדות עליך בין שיחות
- **חיפוש באינטרנט:** דרך DuckDuckGo (בלי מפתח API, בלי תשלום)
- **RAG:** טוענים מסמכים לצ'אט והמודל עונה על בסיסם

**עלות חודשית: ₪0. מפתחות API בתשלום: אין. שליחת דאטה החוצה: אין.**
הדבר היחיד שעולה זה החשמל והחומרה שכבר יש לך.

---

## דרישות חומרה (חשוב לקרוא לפני שמתחילים)

| RAM | מה מומלץ להריץ |
|-----|----------------|
| 8 GB | מודל קטן בלבד (למשל `qwen3.5:3b`) — יעבוד, אבל איטי |
| 16 GB | `qwen3.5:7b` — נקודת האיזון הטובה ביותר |
| 32 GB+ או GPU ייעודי | מודלים גדולים יותר (`deepseek-r1:14b`) וחלון קונטקסט ארוך |

**כלל אצבע:** תריץ קודם מודל קטן אחד בצורה חלקה. רק אחר כך תשנה משתנה אחד בכל פעם (גודל מודל, קונטקסט, GPU).

אם יש לך GPU של NVIDIA — מצוין, זה יאיץ הכל פי כמה. אם לא, זה עדיין יעבוד על CPU, פשוט יותר לאט.

---

# חלק א' — התקנת Docker

Docker הוא הדרך הכי נקייה להריץ את שני הרכיבים ביחד. חינם לשימוש אישי.

## Windows

1. הורד את **Docker Desktop** מ־ https://www.docker.com/products/docker-desktop
2. הרץ את קובץ ההתקנה. אם הוא מבקש להפעיל WSL 2 — אשר.
3. הפעל מחדש את המחשב אם התבקשת.
4. פתח את Docker Desktop וודא שהוא רץ (אייקון הלוויתן בשורת המשימות).
5. בדוק שהכל תקין — פתח PowerShell והרץ:

```bash
docker --version
```

אם קיבלת מספר גרסה — אתה מוכן.

## Linux (אובונטו / דביאן)

```bash
# התקנה מהירה של Docker
curl -fsSL https://get.docker.com | sh

# הוסף את המשתמש שלך לקבוצת docker (כדי לא לרוץ עם sudo כל פעם)
sudo usermod -aG docker $USER

# התנתק והתחבר מחדש, ואז בדוק:
docker --version
```

---

# חלק ב' — הקבצים

הקובץ **`docker-compose.yml`** כבר נמצא בתיקייה הזו. אם אתה מתחיל מאפס בתיקייה חדשה:

```bash
mkdir local-ai
cd local-ai
```

והעתק לתוכה את הקובץ `docker-compose.yml` מהמאגר הזה. התוכן שלו:

```yaml
services:
  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    volumes:
      - ./ollama:/root/.ollama
    restart: unless-stopped
    # === יש לך GPU של NVIDIA? הסר את סימני ה-# מ-6 השורות הבאות ===
    # deploy:
    #   resources:
    #     reservations:
    #       devices:
    #         - driver: nvidia
    #           count: all
    #           capabilities: [gpu]

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    volumes:
      - ./open-webui:/app/backend/data
    depends_on:
      - ollama
    ports:
      - "3000:8080"
    environment:
      - "OLLAMA_BASE_URL=http://ollama:11434"
      # הפעלת חיפוש אינטרנט עם DuckDuckGo (חינם, בלי מפתח):
      - "ENABLE_RAG_WEB_SEARCH=True"
      - "RAG_WEB_SEARCH_ENGINE=duckduckgo"
      - "RAG_WEB_SEARCH_RESULT_COUNT=3"
    restart: unless-stopped
```

> **הערה:** הגדרות ה-`environment` כאן כבר מפעילות את חיפוש האינטרנט אוטומטית. אם משהו לא נתפס בגרסה שלך, נגדיר את זה גם ידנית דרך הממשק בהמשך (חלק ה').

---

# חלק ג' — הרצה ומשיכת מודלים

## שלב 1 — הפעל את הכל

מתוך התיקייה עם ה-`docker-compose.yml`, הרץ:

```bash
docker compose up -d
```

הפעם הראשונה תיקח כמה דקות (מוריד את ה-images). בסיום, שני קונטיינרים ירוצו ברקע.

## שלב 2 — משוך מודל שפה (LLM)

```bash
# מודל ראשי — נקודת איזון מצוינת ל-16GB RAM:
docker exec -it ollama ollama pull qwen3.5:7b
```

אם יש לך פחות זיכרון, במקום זה:

```bash
docker exec -it ollama ollama pull qwen3.5:3b
```

## שלב 3 — משוך מודל embedding (הכרחי ל-RAG וחיפוש)

```bash
docker exec -it ollama ollama pull nomic-embed-text
```

זה מודל קטן שמתרגם טקסט לוקטורים — בלעדיו RAG וחיפוש האינטרנט לא יעבדו טוב.

---

# חלק ד' — כניסה ראשונה לממשק

1. פתח דפדפן ולך אל: **http://localhost:3000**
2. במסך הראשון — צור חשבון **Admin** מקומי (שם, אימייל, סיסמה).
   > החשבון הזה נשמר רק אצלך במחשב. זה לא נרשם לשום שירות ענן.
3. אתה בפנים! בחר את המודל `qwen3.5:7b` מהתפריט למעלה ותוכל להתחיל לדבר.

---

# חלק ה' — ההגדרות הקריטיות (אל תדלג!)

זה הפער בין setup שעובד מצוין לבין אחד ש"ממציא" תשובות.

## 1. הגדל את חלון הקונטקסט (הבאג הכי נפוץ)

ברירת המחדל של Ollama היא **2048 טוקנים בלבד** — קטן מדי ל-RAG ולחיפוש אינטרנט, כי המידע שנשלף גדול יותר מהחלון ופשוט נחתך.

**התיקון:**
1. לחץ על שם המשתמש שלך (למטה/למעלה) → **Admin Panel**
2. **Settings** → **Models**
3. בחר את המודל שלך (אייקון עיפרון)
4. מצא את **Context Length (num_ctx)** והעלה ל-**8192** (או יותר אם יש לך זיכרון)
5. שמור

## 2. הגדר את מנוע ה-Embedding

1. **Admin Panel** → **Settings** → **Documents**
2. **Embedding Model Engine** → בחר **Ollama**
3. **Embedding Model** → הקלד בדיוק: `nomic-embed-text`
4. שמור

## 3. ודא שחיפוש האינטרנט פעיל

1. **Admin Panel** → **Settings** → **Web Search**
2. הפעל את **Enable Web Search**
3. **Web Search Engine** → בחר **duckduckgo** (לא צריך מפתח API)
4. שמור

---

# חלק ו' — איך משתמשים בכל פיצ'ר

## חיפוש באינטרנט

בתוך חלון הצ'אט, לחץ על אייקון ה-**+** (או הגלובוס) ליד תיבת ההקלדה והפעל **Web Search**. עכשיו כשתשאל שאלה עדכנית, המודל יחפש ברשת ויכניס את התוצאות לתשובה.

## RAG — לשאול על המסמכים שלך

**אפשרות א' (חד-פעמי בצ'אט):**
בתיבת ההקלדה הקלד `#` ואז גרור/העלה קובץ (PDF, Word, טקסט). המודל יענה על בסיס תוכן הקובץ.

**אפשרות ב' (ספריית ידע קבועה):**
1. בתפריט הצד → **Workspace** → **Knowledge**
2. צור אוסף חדש והעלה אליו מסמכים
3. בצ'אט, הקלד `#` ובחר את האוסף

## זיכרון בין שיחות (Memory)

1. **Settings** (של המשתמש, לא Admin) → **Personalization** → **Memory**
2. הפעל את זה
3. עכשיו תוכל להגיד למודל "תזכור ש..." והוא ישמור את זה לשיחות עתידיות

היסטוריית השיחות עצמה נשמרת אוטומטית — הכל יושב מקומית בקובץ SQLite בתוך `./open-webui`.

---

# חלק ז' — תחזוקה שוטפת

## עצירה והפעלה

```bash
# עצירה:
docker compose down

# הפעלה מחדש:
docker compose up -d
```

הנתונים שלך (שיחות, מסמכים, הגדרות) נשמרים בתיקיות `./ollama` ו-`./open-webui` — הם לא נמחקים כשעוצרים.

## עדכון לגרסה חדשה

```bash
docker compose pull
docker compose up -d
```

## משיכת מודלים נוספים

```bash
# רשימת מודלים פופולריים לניסיון:
docker exec -it ollama ollama pull deepseek-r1:8b     # הסקה חזקה
docker exec -it ollama ollama pull llama3.1:8b        # כללי, מאוזן
docker exec -it ollama ollama pull mistral-small      # מהיר

# לראות מה כבר מותקן אצלך:
docker exec -it ollama ollama list
```

---

# פתרון תקלות נפוצות

| בעיה | פתרון |
|------|-------|
| הצ'אט "ממציא" תשובות ולא משתמש במסמכים | חלון הקונטקסט קטן מדי — העלה ל-8192+ (חלק ה', סעיף 1) |
| RAG/חיפוש לא עובדים | לא הגדרת מנוע embedding — ודא ש-`nomic-embed-text` מוגדר |
| הכל איטי מאוד | המודל גדול מדי לזיכרון שלך — עבור למודל קטן יותר (`3b`) |
| `localhost:3000` לא נטען | ודא ששני הקונטיינרים רצים: `docker ps` |
| חיפוש אינטרנט מחזיר כלום | ודא ש-DuckDuckGo נבחר וש-Web Search מופעל בצ'אט עצמו |

---

# סיכום — סדר הפעולות הקצר

```
1. התקן Docker
2. צור תיקייה + קובץ docker-compose.yml
3. docker compose up -d
4. משוך מודל:      docker exec -it ollama ollama pull qwen3.5:7b
5. משוך embedding: docker exec -it ollama ollama pull nomic-embed-text
6. פתח localhost:3000, צור חשבון admin
7. הגדל קונטקסט ל-8192, הגדר embedding, הפעל web search
8. תיהנה 🎉
```

**הכל חינם. הכל מקומי. הכל אצלך.**
