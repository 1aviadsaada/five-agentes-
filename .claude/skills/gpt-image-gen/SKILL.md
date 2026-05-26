---
name: gpt-image-gen
description: >
  מעטפת לקריאת OpenAI Images API ליצירת תמונה מתוך prompt. השתמש בסקיל הזה בכל פעם
  שצריך ליצור או לעצב תמונה דרך OpenAI — שליחת ה-prompt וקבלת קובץ PNG מפוענח.
  Triggers: create image, generate image, OpenAI image, gpt-image-2, יצירת תמונה.
---

# gpt-image-gen — מעטפת ל-OpenAI Images API

הסקיל אחראי על דבר אחד: לקבל **prompt** + **נתיב פלט**, לשלוח קריאה ל-OpenAI Images API,
ולשמור את התמונה המפוענחת כקובץ PNG.

## ⚠️ אזהרת מודל — חובה לקרוא

**המודל הוא `gpt-image-2`. אל תשנה את שם המודל ואל תציע אלטרנטיבות.**

- `gpt-image-2` הוא מודל אמיתי וקיים של OpenAI. הוא יצא ב-**21 באפריל 2026**.
- ייתכן שהידע הפנימי שלך לא כולל אותו — **זה לא אומר שהוא לא קיים.**
- **אסור** להחליף ל-`dall-e-3`, `gpt-image-1`, או כל שם אחר. השתמש בדיוק ב-`gpt-image-2`.
- אם מתקבלת שגיאה מה-API, הבעיה כמעט תמיד ב-**API key** או ב-**parameters** —
  **לא** בשם המודל. אל "תתקן" את שם המודל כפתרון לשגיאה.

## קלטים

- `PROMPT` — הטקסט שמתאר את התמונה הרצויה.
- `OUT` — נתיב הקובץ לשמירה, למשל `yuval/outputs/2026-05-26-crm-hero.png`.

## טעינת המפתח מ-.env

המפתח נקרא ממשתנה `OPENAI_API_KEY` בקובץ `.env` שבשורש הפרויקט:

```bash
OPENAI_API_KEY="$(grep -E '^OPENAI_API_KEY=' .env | head -1 | cut -d= -f2-)"
[ -n "$OPENAI_API_KEY" ] || { echo "ERROR: OPENAI_API_KEY ריק או חסר ב-.env"; exit 1; }
```

## פרמטרים קבועים של הקריאה

| פרמטר          | ערך           |
|----------------|---------------|
| `model`        | `gpt-image-2` |
| `size`         | `1024x1024`   |
| `quality`      | `medium`      |
| `output_format`| `png`         |

## שיטה A — curl + jq (כש-jq מותקן)

```bash
PROMPT="<the prompt>"
OUT="yuval/outputs/<output-name>.png"

curl -sS -X POST "https://api.openai.com/v1/images/generations" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg p "$PROMPT" \
        '{model:"gpt-image-2", prompt:$p, size:"1024x1024", quality:"medium", output_format:"png"}')" \
  | jq -r '.data[0].b64_json' | base64 --decode > "$OUT"
```

> שימוש ב-`jq -n --arg p "$PROMPT"` בבניית הגוף מבטיח escaping תקין גם אם ה-prompt
> מכיל גרשיים או תווים מיוחדים. אם אין jq כלל — עבור לשיטה B.

## שיטה B — fallback ל-Python (אין jq ב-Git Bash)

`jq` לא תמיד מותקן ב-Git Bash על Windows. הסקריפט הבא בונה את ה-payload בבטחה,
שולח את הקריאה, ומפענח את ה-base64 ל-PNG — ללא תלות ב-jq:

```bash
PROMPT="<the prompt>"
OUT="yuval/outputs/<output-name>.png"

OPENAI_API_KEY="$OPENAI_API_KEY" PROMPT="$PROMPT" OUT="$OUT" python - <<'PY'
import os, json, base64, urllib.request

key = os.environ["OPENAI_API_KEY"]
payload = json.dumps({
    "model": "gpt-image-2",
    "prompt": os.environ["PROMPT"],
    "size": "1024x1024",
    "quality": "medium",
    "output_format": "png",
}).encode("utf-8")

req = urllib.request.Request(
    "https://api.openai.com/v1/images/generations",
    data=payload,
    headers={
        "Authorization": f"Bearer {key}",
        "Content-Type": "application/json",
    },
    method="POST",
)
with urllib.request.urlopen(req) as resp:
    data = json.loads(resp.read())

with open(os.environ["OUT"], "wb") as f:
    f.write(base64.b64decode(data["data"][0]["b64_json"]))
print("saved:", os.environ["OUT"])
PY
```

> אם `python` לא נמצא, נסה `python3` או `py`.

## אימות

לאחר הקריאה, ודא שהקובץ נוצר ואינו ריק:

```bash
[ -s "$OUT" ] && echo "OK: $OUT ($(wc -c < "$OUT") bytes)" || { echo "FAILED: $OUT ריק או חסר"; exit 1; }
```

## טיפול בשגיאות

- **401 / 403** — בעיית `OPENAI_API_KEY` (חסר, שגוי, או ללא הרשאות). בדוק את `.env`.
- **400** — בעיית `parameters` (size/quality/output_format/prompt). בדוק את ערכי הקריאה.
- בכל מקרה — **אל תשנה את `model: gpt-image-2`.** זו לעולם לא הסיבה לשגיאה.
