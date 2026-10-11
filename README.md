# Kolitablet – Jarvis faliképernyő

Fekvő módú, falra szerelt Android tabletre (Fully Kiosk Browser) készült, egyetlen fájlból álló kijelző: óra, időjárás, botok állapota, mai teendők és magyar nyelvű felolvasás.

| Fájl | Mire való |
|---|---|
| `index.html` | Maga az oldal. A CSS, a JS és az ikonok mind benne vannak, külső függőség nincs. |
| `data.json` | Minden változó tartalom. Az oldal 5 percenként újratölti. |
| `README.md` | Ez a leírás. |

## Kipróbálás

A `data.json` betöltéséhez HTTP szerver kell, mert a böngésző `file://` alatt nem engedi a `fetch`-et.

```bash
cd Kolitablet
python3 -m http.server 8000
```

Utána nyisd meg: <http://localhost:8000>. Teljes képernyő: F11. Tablet-méret: DevTools → eszköz mód, 1280×800.

Ha az `index.html`-t simán duplakattal nyitod meg, a beépített **PÉLDA** adatokat látod. Ugyanez történik, ha a `data.json` hiányzik vagy hibás.

### Ubuntu szerver + Tailscale

```bash
# a mappában, ahol az index.html és a data.json van
python3 -m http.server 8080 --bind 0.0.0.0
```

A tabletről: `http://<szerver-tailscale-neve>:8080/`. Hosszú távra egy systemd service vagy nginx jobb; `tailscale serve` is működik.

### Fully Kiosk tippek

- **Start URL:** a fenti cím.
- **Felolvasás:** a Chrome (és a WebView) csak akkor enged hangot, ha a lapot oldalbetöltés után legalább egyszer megérintették. Ezért egy koppintás kell az automatikus felolvasás előtt; az oldal ki is írja, amíg ez nem történt meg.
- Ha a WebView-ban nincs Web Speech hang, az oldal automatikusan a Fully saját Android TTS-ét használja (`fully.textToSpeech`). Ehhez a Fully-ben be kell kapcsolni: *Advanced Web Settings → Enable JavaScript Interface*. Így koppintás nélkül is működik.
- Magyar hang: Android beállítások → Szövegfelolvasó → Google TTS → Magyar hangcsomag letöltése.
- A képernyő ne kapcsoljon ki: *Device Management → Keep Screen On*.

## Működés röviden

- **Idő:** mindig Europe/Budapest szerint, a tablet időzónájától függetlenül.
- **Köszönés:** 4–10 óra „Jó reggelt”, 10–18 „Jó napot”, 18–22 „Jó estét”, egyébként „Jó éjszakát”.
- **Frissítés:** a `data.json` újratöltése 5 percenként (`settings.refreshMinutes`), valamint hálózat-visszatéréskor.
- **Ha a betöltés sikertelen:**
  - ha még soha nem jött be valódi adat, a beépített mintaadat látszik, sárga „PÉLDA” jelöléssel;
  - ha korábban már bejött, az utolsó valódi adat marad kint, a Jarvis kártyán pedig „Offline” felirat jelenik meg.
- **Beégés elleni védelem:** az egész elrendezés 3 percenként néhány pixelt elmozdul (`settings.shiftMinutes`).
- **Éjszaka:** 23:00 és 6:00 között a kijelző elhalványul (`settings.night`). Koppintásra 1 percre felébred.
- **Felolvasás:** a Jarvis kártya gombjával indítható; menet közben ugyanez a gomb állítja le. Az automatikus reggeli és esti időpontot a `data.json` adja, de a csúszkás ikonnal tabletenként felülírható (a tablet `localStorage`-ában tárolódik).

## A `data.json` felépítése

Minden mező opcionális; ami hiányzik, arra az oldal alapértéket használ.

```jsonc
{
  "updated": "2026-10-11T07:05:00+02:00",   // mikor készült az adat (ISO 8601) – „x perce” kijelzéshez

  "settings": {
    "name": "Péter",                         // köszönésben és felolvasásban
    "refreshMinutes": 5,                     // data.json újratöltési gyakoriság
    "autoRead": { "enabled": true, "morning": "07:30", "evening": "21:30" },
    "speechRate": 1.0,                       // 0.7–1.4
    "voice": "",                             // pontos hangnév; üres = első magyar hang
    "night": { "start": "23:00", "end": "06:00" },
    "shiftMinutes": 3                        // beégés elleni eltolás gyakorisága
  },

  "briefing": {                              // rövid szöveg az óra alatt és a felolvasásban
    "morning": "…",                          // 14:00 előtt ez látszik
    "evening": "…"                           // 14:00 után ez
  },                                         // (lehet egyszerű string is)

  "weather": {
    "location": "Budapest",
    "temp": 14, "feels_like": 12,
    "condition": "Részben felhős",           // szöveges leírás
    "icon": "partly",                        // sun | partly | cloud | rain | snow | storm | fog | moon
    "code": 2,                               // helyette megadható WMO kód (pl. Open-Meteo) – ebből ikon + szöveg lesz
    "high": 18, "low": 9,
    "precip_chance": 20,                     // %
    "wind_kmh": 12,
    "forecast": [                            // max. 3 nap jelenik meg
      { "day": "Holnap", "icon": "rain", "condition": "esős", "high": 15, "low": 8 }
    ]
  },

  "bots": [                                  // tetszőleges számú; 7 + a Jarvis kártya fér ki szépen
    {
      "id": "tippmix",
      "name": "Tippmix",
      "subtitle": "Kétnapos elemzés",
      "icon": "tippmix",                     // grok | tippmix | games | promo | learn | sell | buy | bot – vagy egy emoji, pl. "🎮"
      "status": "ok",                        // ok | warn | error | running | idle
      "status_text": "Rendben",              // opcionális, felülírja az alap státuszfeliratot
      "value": "6 tipp",                     // nagy kiemelt érték (rövid!)
      "line": "Hétfő–kedd elemzés kész, legjobb szorzó 2,35.",  // állapotsor, kb. 50 karakterig fér ki
      "updated": "2026-10-11T06:00:00+02:00"
    }
  ],

  "todos": [
    { "time": "10:00", "text": "Statisztika előadás", "priority": "high" },  // high = sárga jelölés
    { "time": "14:00", "text": "Csomag a portán", "kind": "reminder" },     // reminder = harang ikon
    { "text": "Mosás" },                                                     // időpont nélkül
    { "time": "08:30", "text": "Reggeli edzés", "done": true }              // kész = áthúzva, a lista végén
  ]
}
```

A teendők sorrendje: előbb a nyitottak időrendben, aztán a kész tételek. A következő esedékes tétel „KÖVETKEZŐ” címkét kap. Ami nem fér ki, az helyett „+N további” sor jelenik meg.

A botjaid egyszerűen felülírhatják a `data.json` megfelelő részét (pl. `jq`-val vagy egy kis Python scripttel). Az oldal a következő frissítéskor magától átveszi a változást.
