# מייל — דרך Composio

**Composio בלבד.** לא gws, לא gcloud, לא IMAP. אם המשתמש חיבר את המייל בדרך אחרת,
הוא עדיין יעבוד — אבל ברירת המחדל וההסבר בהתקנה הם Composio.

## בדיקת חיבור

```bash
composio connections list
```

מחפשים `gmail` בסטטוס `ACTIVE`. יש כמה חיבורים ואחד `EXPIRED`? זה תקין —
מספיק שאחד פעיל. אין אף אחד פעיל:

```bash
composio link gmail
```

## המשיכה — שני שלבים, לא אחד

### שלב א' — מטא־דאטה בלבד, רחב

```bash
composio execute GMAIL_FETCH_EMAILS --params '{
  "query": "after:2025/09/01 -in:sent -category:promotions -category:social -category:updates",
  "verbose": false,
  "max_results": 100
}'
```

- `verbose:false` — פי 4 מהר, ומחזיר בדיוק את מה שצריך לסינון ראשוני:
  נושא, שולח, תאריך, תוויות.
- `-category:promotions -category:social -category:updates` — מוריד את רוב הרעש
  לפני שהוא עולה לך בטוקנים.
- `-in:sent` — לא סורקים את מה שאתה שלחת. אלה לא פניות.

**עימוד חובה.** התשובה מחזירה `nextPageToken`. ממשיכים עם `page_token` עד שהוא
ריק או חוזר על עצמו. בלי זה תסרוק 100 מיילים ותחשוב שסיימת.

### שלב ב' — הידרציה של מועמדים בלבד

רק אחרי שסיננת לפי נושא/שולח למי שנראה כמו פנייה עסקית:

```bash
composio execute GMAIL_FETCH_MESSAGE_BY_THREAD_ID --params '{"thread_id":"<threadId>"}'
```

**השרשור, לא ההודעה.** זה מה שעונה על השאלה האמיתית — האם ענית ומתי.
הודעה בודדת לא תגיד לך אם הלולאה נסגרה.

## יצירת הטיוטה

```bash
composio execute GMAIL_CREATE_EMAIL_DRAFT --params '{
  "recipient_email": "...",
  "subject": "Re: ...",
  "thread_id": "<threadId>",
  "is_html": true,
  "body": "<div dir=\"rtl\" style=\"text-align:right\">...</div>"
}'
```

- **`thread_id` חובה** — בלעדיו הטיוטה נוחתת כמייל חדש מנותק, והליד מקבל משהו
  שנראה כמו פנייה קרה במקום המשך שיחה.
- **`is_html:true` + `<div dir="rtl" style="text-align:right">` חובה לעברית.**
  `text/plain` מרונדר ב־Gmail משמאל לימין בלי קשר לתוכן העברי, והטיוטה
  תיראה שבורה. זו לא העדפה — זו התנהגות של Gmail.

## מלכודות שנתקלנו בהן

| מה קורה | למה | מה עושים |
|---|---|---|
| `max_results` מתעלם ומחזיר מעט | יש תקרה בצד השרת | לעמד, לא להעלות את המספר |
| HTTP 413 / פלט קטוע | משכת גוף מלא בסריקה רחבה | `verbose:false` בשלב א'. תמיד |
| `nextPageToken` חוזר זהה | סוף התוצאות | לעצור. אחרת לולאה אינסופית |
| 403 בהידרציה | חיבור לתיבה אחרת מזו שציפית | לוודא איזה חשבון פעיל ב־`connections list` |
| `before:` מפספס יום | נחתך לפי UTC ולא לפי שעון מקומי | להרחיב יום אחד לכל צד |
