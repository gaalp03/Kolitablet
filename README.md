# Kolitablet – Jarvis faliképernyő

Fekvő módú, falra szerelt Android tabletre (XGODY Q17, 11", 1280×800, Android 15, Fully Kiosk Browser) készült, egyetlen fájlból álló kijelző. Mutatja a nagy órát, az időjárást, a botok állapotát, a napi összefoglalót és a mai teendőket.

| Fájl | Mire való |
|---|---|
| `index.html` | Maga az oldal. A CSS, a JS és az SVG ikonok mind benne vannak, külső függőség nincs, offline is ugyanígy néz ki. |
| `data.json` | Minden változó tartalom. Az oldal 5 percenként újratölti. |
| `README.md` | Ez a leírás. |

## Kipróbálás

A `data.json` betöltéséhez HTTP szerver kell, mert a böngésző `file://` alatt nem engedi a `fetch`-et.

```bash
cd Kolitablet
python3 -m http.server 8000
```

Utána nyisd meg: <http://localhost:8000>. Tablet-méret: DevTools → eszköz mód, 1280×800.

Az elsődleges méret 1280×800. Mivel az Android eszközönként más device pixel ratiót használhat, a böngésző a kijelzőt kisebb CSS-viewportnak is láthatja. Az oldal ezeken is ellenőrizve van, görgetés és levágott szöveg nélkül:

- **1280×800** (DPR 1)
- **1067×667** (DPR 1.2)
- **960×600** (DPR 1.33)
- **1280×752** (látható navigációs sávval)

Minden méret egy közös alapegységből (`rem`) skálázódik, így az arányok mindenhol ugyanazok.

Ha az `index.html`-t simán duplakattal nyitod meg, a beépített mintaadatot látod, a köszönés mellett sárga **PÉLDA ADATOK** jelvénnyel. Ugyanez történik, ha a `data.json` hiányzik vagy hibás.

### Ubuntu szerver + Tailscale

```bash
# a mappában, ahol az index.html és a data.json van
python3 -m http.server 8080 --bind 0.0.0.0
```

A tabletről: `http://<szerver-tailscale-neve>:8080/`. Hosszú távra egy systemd service vagy nginx jobb; `tailscale serve` is működik.

### Fully Kiosk tippek

- **Start URL:** a fenti cím.
- **Képernyő:** *Device Management → Keep Screen On*.
- **Teljes képernyő:** *Web Content Settings → Hide Status/Navigation Bar*.
- **Szövegnagyítás:** a *Web Zoom / Text Zoom* maradjon 100%-on, különben a szövegek nem férnek ki.

### Éjszakai háttérvilágítás (a tableten kell beállítani)

Az oldal 22:00 és 05:55 között teljesen fekete, de egy IPS kijelző háttérvilágítása ettől még ég. Ha a tablet képernyőjét is ki akarod kapcsolni ugyanebben az időszakban, azt a **Fully Kiosk beállításaiban** kell megadni, nem a weboldalon:

- **Device Management → Screen Off Timer / Schedule Screen Off:** kikapcsolás 22:00-kor.
- **Device Management → Schedule Wakeup (Screen On):** bekapcsolás 05:55-kor.
- A pontos menünevek Fully-verziónként kicsit eltérhetnek. Az ütemezett ki/bekapcsoláshoz általában a Fully PLUS licenc és a *Device Admin* jogosultság kell.
- A tablet órája legyen Europe/Budapest időzónán, mert a Fully ütemezése a készülék idejét használja. Az oldal viszont mindig budapesti időt számol.

## Elrendezés

- **Bal felül:** napszakos köszönés, nagy színátmenetes óra (a másodperc kisebben mellette), dátum. Példa adatnál mellette a sárga „PÉLDA ADATOK” jelvény.
- **Középen felül:** időjárás. Látszik a hőmérséklet, a hőérzet, a min/max, az eső esélye, a szél és a 3 napos előrejelzés.
- **Jobb oldalt:** mai teendők. Kör jelölő, emlékeztetőnél csengő ikon, a következő esedékes tétel „KÖVETKEZŐ” kiemelést kap, fent a nyitott tételek száma.
- **Alul:** „Jarvis – mai összefoglaló” kártya (napi szöveg + élő/offline jelzés), mellette a bot-kártyák. Mindegyiken ikon, kategória, 3 soros állapotszöveg, állapot (Rendben / Infó / Figyelem / Hiba / Fut / Szünetel), „x órája” és egy nagy állapotszám.

## Működés röviden

- **Idő:** mindig Europe/Budapest szerint, a tablet időzónájától függetlenül.
- **Köszönés:** 5–10 óra „Jó reggelt”, 10–18 „Szép napot”, 18–22 „Jó estét”, egyébként „Jó éjszakát”.
- **Frissítés:** a `data.json` újratöltése 5 percenként (`settings.refreshMinutes`), valamint hálózat-visszatéréskor.
- **Ha a betöltés sikertelen:**
  - ha még soha nem jött be valódi adat, a beépített mintaadat látszik „PÉLDA ADATOK” jelvénnyel;
  - ha korábban már bejött, az utolsó valódi adat marad kint, a Jarvis kártyán „Offline · utolsó: hh:mm” felirattal;
  - ha az `updated` 12 óránál régebbi, a kártyán „Régi adat” jelenik meg.
- **Beégés elleni védelem:** az egész elrendezés 3 percenként néhány pixelt elmozdul (`settings.shiftMinutes`).
- **Éjszaka:** 22:00 és 05:55 között (budapesti idő, `settings.night`) a kijelző teljesen fekete. Ilyenkor semmi nem látszik, és semmi sem animálódik (a háttér és a tartalom ki is kerül a renderelésből). 05:55-kor magától visszajön a normál nézet, friss adatokkal. Koppintásra nem ébred fel, csak 10 másodpercre megjelenik középen egy halvány óra.
- **Levágás elleni védelem:** ha egy szöveg valamiért nem fér ki (pl. más betűtípus miatt), az oldal kevesebb, de egész sort mutat „…”-tal.

## A `data.json` felépítése

Minden mező opcionális; ami hiányzik, arra az oldal alapértéket használ.

```jsonc
{
  "updated": "2026-10-11T07:05:00+02:00",   // mikor készült az adat (ISO 8601)
  "example": false,                          // true → „PÉLDA ADATOK” jelvény akkor is, ha a fájl betöltődött

  "settings": {
    "name": "Péter",                         // a köszönésben
    "refreshMinutes": 5,                     // data.json újratöltési gyakoriság
    "night": { "start": "22:00", "end": "05:55" },  // teljesen fekete kijelző ebben a sávban
    "shiftMinutes": 3                        // beégés elleni eltolás gyakorisága
  },

  "briefing": {                              // a „Jarvis – mai összefoglaló” kártya szövege
    "morning": "…",                          // 14:00 előtt ez látszik
    "evening": "…"                           // 14:00 után ez
  },                                         // lehet egyszerű string is; ha hiányzik, az oldal maga rak össze egyet

  "weather": {
    "location": "Budapest",
    "temp": 14, "feels_like": 12,
    "condition": "Részben felhős",           // szöveges leírás
    "icon": "partly",                        // sun | partly | cloud | rain | snow | storm | fog | moon
    "code": 2,                               // helyette megadható WMO kód (pl. Open-Meteo) – ebből ikon + szöveg lesz
    "high": 18, "low": 9,
    "precip_chance": 20,                     // eső esélye, %
    "wind_kmh": 12,
    "forecast": [                            // max. 3 nap jelenik meg
      { "day": "Holnap", "icon": "rain", "high": 15, "low": 8 }
    ]
  },

  "bots": [                                  // 7 bot + a Jarvis kártya pont kitölti a 3×3-as rácsot
    {
      "id": "tippmix",
      "name": "Tippmix",
      "subtitle": "Kétnapos elemzés",        // kategória címke
      "icon": "tippmix",                     // grok | tippmix | games | promo | learn | sell | buy | bot – vagy egy emoji, pl. "🎮"
      "status": "ok",                        // ok | info | warn | error | running | idle
      "status_text": "Rendben",              // opcionális, felülírja az alap státuszfeliratot
      "value": "6 tipp",                     // nagy állapotszám a kártya jobb alsó sarkában (rövid!)
      "line": "Hétfő–kedd elemzés kész, legjobb szorzó 2,35; a részletek a chatben.",  // kb. 75 karakterig fér ki 3 sorban
      "updated": "2026-10-11T06:00:00+02:00"
    }
  ],

  "todos": [
    { "time": "10:00", "text": "Statisztika előadás", "priority": "high" },  // high = sárga jelölés
    { "time": "14:00", "text": "Csomag a portán", "kind": "reminder" },     // reminder = csengő ikon
    { "text": "Mosás" },                                                     // időpont nélkül
    { "time": "08:30", "text": "Reggeli edzés", "done": true }              // kész = áthúzva, a lista végén
  ]
}
```

A teendők sorrendje: előbb a nyitottak időrendben, aztán a kész tételek. Ami nem fér ki, az helyett „+N további” sor jelenik meg. 7-nél több bot esetén a kártyák kisebbek lesznek, és kevesebb sor fér ki belőlük.

A botjaid egyszerűen felülírhatják a `data.json` megfelelő részét (pl. `jq`-val vagy egy kis Python scripttel). Az oldal a következő frissítéskor magától átveszi a változást.
