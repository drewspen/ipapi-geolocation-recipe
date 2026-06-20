# GTM IP Geolocation Recipe — ipapi.co

> **Capture city, postal code, timezone, ISP, and UTC offset on every page load — no server-side code required.**

A plug-and-play Google Tag Manager container that calls [ipapi.co](https://ipapi.co)'s free geolocation API on page initialization, pushes the results to the `dataLayer`, writes them to first-party session cookies, and forwards them to GA4 as a custom event — all in a single importable JSON file.

---

## ⬇ Download

**[GTM Container JSON — Google Drive](https://drive.google.com/file/d/1pv5qOaLi4WcBccOwui6AhUsrGzGXY_1u/view?usp=drivesdk)**

---

## Contents

```
ipapi-geolocation-recipe.json    ← GTM container export (import this)
README.md                        ← this file
```

---

## Architecture

```
Page Load
   │
   ▼
[Trigger] IPAPI Has Not Already Ran
   │  (Initialization trigger, fires only when _ipapi_ar cookie ≠ 1)
   │
   ▼
[Tag] IPAPI Get Location  (Custom HTML)
   │  1. Stamps guard cookie _ipapi_ar=1 synchronously
   │  2. fetch("https://ipapi.co/json/")
   │  3. On success → dataLayer.push({ event: 'geoIPAPI', ...geo fields })
   │
   ▼
[Trigger] geoIPAPI Custom Event
   │
   ├──▶ [Tag] IPAPI Cookie Writer  (Sandboxed JS Custom Template)
   │         Writes: _i_geoCity, _i_geoISP, _i_geoPostal,
   │                 _i_geoTimeZone, _i_geoUTCOffSet
   │
   └──▶ [Tag] IPAPI Send Event  (GA4 Event)
              Event name: geoIPAPI
              Parameters: GeoCity, GeoISP, GeoPostalCode,
                          GeoTimeZone, GeoUTCOffSet
```

---

## Container Assets

### Tags

| Name | Type | Trigger | Description |
|---|---|---|---|
| IPAPI Get Location | Custom HTML | IPAPI Has Not Already Ran | Calls ipapi.co, stamps guard cookie, pushes `geoIPAPI` event to dataLayer |
| IPAPI Cookie Writer | Custom Template (Cookie Writer) | geoIPAPI Custom Event | Writes five geo session cookies via sandboxed JS |
| IPAPI Send Event | GA4 Event | geoIPAPI Custom Event | Sends `geoIPAPI` hit with geo dimensions to GA4 |
| Google Analytics Configuration | Google Tag | All Pages | Standard GA4 config, environment-aware Stream ID routing |

### Triggers

| Name | Type | Condition |
|---|---|---|
| IPAPI Has Not Already Ran | Initialization | `_ipapi_ar` cookie does **not** match `^1$` |
| geoIPAPI Custom Event | Custom Event | Event name equals `geoIPAPI` |

### Key Variables

| Name | Type | Reads |
|---|---|---|
| IPAPI Has Already Ran | 1st Party Cookie | `_ipapi_ar` |
| geoCity dataLayer | Data Layer Variable | `geoCity` |
| geoISP dataLayer | Data Layer Variable | `geoISP` |
| geoPostal dataLayer | Data Layer Variable | `geoPostal` |
| geoTimeZone dataLayer | Data Layer Variable | `geoTimeZone` |
| geoUTCOffSet dataLayer | Data Layer Variable | `geoUTCOffSet` |
| Root Domain | URL Variable | `HOST` (strips `www`) |
| GTM Environment Name | Environment Name | Current GTM env |
| Environment Stream ID | Custom Template (If Else If) | Routes Dev vs Prod Stream ID |

### Custom Templates (bundled)

| Template | Purpose |
|---|---|
| **Cookie Writer** | Sandboxed JS tag template — writes multiple named cookies from a table parameter |
| **Get Root Domain** | Resolves root domain for cookie scoping |
| **Timestamp** | Returns current timestamp in milliseconds |
| **If Else If – Advanced Lookup Table** | Environment-aware regex routing (used for GA4 Stream ID) |

---

## The Core Tag — IPAPI Get Location

```html
<script type="text/javascript">
(function () {
  function getCookie(name) {
    var match = document.cookie.match(new RegExp('(?:^|;\\s*)' + name + '=([^;]*)'));
    return match ? match[1] : undefined;
  }

  function setCookie(name, value) {
    var host = window.location.hostname;
    var rootDomain = host.indexOf('.') !== -1
      ? '.' + host.split('.').slice(-2).join('.')
      : host;
    document.cookie = name + '=' + value + '; path=/; domain=' + rootDomain;
  }

  var COOKIE_NAME = '_ipapi_ar';

  // Bail if already run this session
  if (getCookie(COOKIE_NAME) === '1') { return; }

  // Stamp BEFORE the async fetch to prevent race conditions
  setCookie(COOKIE_NAME, '1');

  fetch("https://ipapi.co/json/")
    .then(function (response) {
      if (!response.ok) { console.log('ipapi: status ' + response.status); }
      return response.json();
    })
    .then(function (data) {
      window.dataLayer = window.dataLayer || [];
      window.dataLayer.push({
        'event':          'geoIPAPI',
        'geoPostal':      data.postal,
        'geoTimeZone':    data.timezone,
        'geoUTCOffSet':   data.utc_offset,
        'geoCallingCode': data.country_calling_code,
        'geoASN':         data.asn,
        'geoCity':        data.city,
        'geoISP':         data.org
        // Additional available fields (uncomment as needed):
        // 'geoIP':              data.ip,
        // 'geoRegion':          data.region,
        // 'geoRegionCode':      data.region_code,
        // 'geoCountryCode':     data.country_code,
        // 'geoCountry':         data.country_name,
        // 'geoInEU':            data.in_eu,
        // 'geoCurrency':        data.currency,
        // 'geoLatitude':        data.latitude,
        // 'geoLongitude':       data.longitude,
        // 'geoContinentCode':   data.continent_code,
      });
    })
    .catch(function (error) { console.log(error); });
})();
</script>
```

> **Guard cookie pattern:** The `_ipapi_ar` cookie is stamped *synchronously* before the `fetch()` call. This prevents a second page view or fast navigation from triggering a duplicate API call while the first response is still in flight.

---

## The Cookie Writer — Sandboxed JS Template

The **IPAPI Cookie Writer** tag uses a sandboxed GTM Custom Template instead of raw `document.cookie` manipulation. This keeps the implementation within GTM's permission model and makes cookie options (domain, expiry, SameSite, Secure) configurable through the GTM UI without touching code.

### Template parameters

| Parameter | Type | Default | Notes |
|---|---|---|---|
| `rootDomain` | Text | _(from `{{Root Domain}}` variable)_ | e.g. `.example.com` |
| `cookieDuration` | Select | `sessionCookie` | `sessionCookie` or `persistentCookie` |
| `persistentCookieExpires` | Text | `30` | Days; only active when `persistentCookie` is selected |
| `targetedCookies` | Table | — | Rows of `cookieName` / `cookieValue`; values accept `{{Variable}}` references |

### Sandboxed JavaScript

```javascript
const setCookie = require('setCookie');
const getTimestampMillis = require('getTimestampMillis');
const logToConsole = require('logToConsole');
const makeString = require('makeString');

const rootDomain              = data.rootDomain;
const cookieDuration          = data.cookieDuration;
const persistentCookieExpires = data.persistentCookieExpires;
const targetedCookies         = data.targetedCookies;

var cookieOptions = {
  domain  : rootDomain || 'auto',  // 'auto' lets GTM derive the domain
  path    : '/',
  secure  : true,
  samesite: 'Lax'
};

if (cookieDuration === 'persistentCookie') {
  var days = (persistentCookieExpires - 0) || 0;
  if (days > 0) {
    cookieOptions['max-age'] = days * 24 * 60 * 60;
  } else {
    logToConsole('GTM Cookie Writer: invalid expiry, falling back to session cookie.');
  }
}

if (targetedCookies && targetedCookies.length > 0) {
  var i = 0;
  while (i < targetedCookies.length) {
    var row    = targetedCookies[i];
    var cName  = makeString(row.cookieName  || '').trim();
    var cValue = makeString(row.cookieValue || '').trim();
    if (cName) {
      setCookie(cName, cValue, cookieOptions);
      logToConsole('GTM Cookie Writer: set "' + cName + '" = "' + cValue + '"');
    }
    i = i + 1;
  }
} else {
  logToConsole('GTM Cookie Writer: targetedCookies table is empty.');
}

data.gtmOnSuccess();
```

### Cookies written by this recipe

| Cookie Name | Value Source | Consent Required |
|---|---|---|
| `_i_geoCity` | `{{geoCity dataLayer}}` | `analytics_storage` + `functionality_storage` |
| `_i_geoISP` | `{{geoISP dataLayer}}` | `analytics_storage` + `functionality_storage` |
| `_i_geoPostal` | `{{geoPostal dataLayer}}` | `analytics_storage` + `functionality_storage` |
| `_i_geoTimeZone` | `{{geoTimeZone dataLayer}}` | `analytics_storage` + `functionality_storage` |
| `_i_geoUTCOffSet` | `{{geoUTCOffSet dataLayer}}` | `analytics_storage` + `functionality_storage` |

> The Cookie Writer tag has `consentStatus: NEEDED` and requires **both** `analytics_storage` and `functionality_storage` to be granted before it fires. The upstream `IPAPI Get Location` tag and the GA4 event tag are unaffected by this gate.

---

## GA4 Event Parameters

The `geoIPAPI` GA4 event carries these custom parameters. Register them as **Custom Dimensions** in your GA4 property (`Admin → Custom Definitions → Custom Dimensions`) to use them in reports and audiences.

| GA4 Parameter | Source Variable | ipapi.co Field |
|---|---|---|
| `GeoCity` | `{{geoCity dataLayer}}` | `city` |
| `GeoISP` | `{{geoISP dataLayer}}` | `org` |
| `GeoPostalCode` | `{{geoPostal dataLayer}}` | `postal` |
| `GeoTimeZone` | `{{geoTimeZone dataLayer}}` | `timezone` |
| `GeoUTCOffSet` | `{{geoUTCOffSet dataLayer}}` | `utc_offset` |

---

## How to Import

1. Download `ipapi-geolocation-recipe.json` from the [Google Drive link](https://drive.google.com/file/d/1pv5qOaLi4WcBccOwui6AhUsrGzGXY_1u/view?usp=drivesdk).
2. In GTM → **Admin** → **Import Container**.
3. Upload the JSON file.
4. Choose **Merge** (not Overwrite) to preserve your existing configuration.
5. Update the **Measurement Stream ID Development** and **Measurement Stream ID Production** constant variables with your actual GA4 Measurement IDs (format: `G-XXXXXXXXXX`).
6. In GA4 → **Admin → Custom Definitions → Custom Dimensions**, register `GeoCity`, `GeoISP`, `GeoPostalCode`, `GeoTimeZone`, and `GeoUTCOffSet` as event-scoped dimensions.
7. Open **GTM Preview mode**, load a page, and confirm:
   - The `IPAPI Get Location` tag fires on initialization.
   - A `geoIPAPI` event appears in the dataLayer.
   - The Cookie Writer and GA4 Event tags fire on the `geoIPAPI` trigger.
   - Cookies `_i_geo*` are visible in DevTools → Application → Cookies.
8. Publish the container.

---

## Extending the Recipe

### Add more geo fields

Uncomment any of the commented-out lines in the `IPAPI Get Location` tag and add a corresponding row to the Cookie Writer's `targetedCookies` table and a new parameter to the GA4 event tag.

### Switch to persistent cookies

In the **IPAPI Cookie Writer** tag's template parameters, change `cookieDuration` from `sessionCookie` to `persistentCookie` and set `persistentCookieExpires` to the number of days you want.

### Use the cookies server-side

The `_i_geo*` cookies are set on the root domain with `Secure; SameSite=Lax`, so they're readable by your origin server on subsequent requests — useful for server-side personalization without requiring a login.

### Environment routing

The container ships with **Development** and **Production** Stream IDs wired to the `GTM Environment Name` variable. Add your actual Measurement IDs to the two constant variables and the regex lookup table handles routing automatically.

---

## Requirements & Limits

- **ipapi.co free plan:** 30,000 requests/month. The guard cookie (`_ipapi_ar`) ensures at most one request per browser session, so this limit is only a concern at very high scale.
- **Browser support:** Uses `fetch()` — supported in all modern browsers. No polyfill is included; add one if you need IE11 support.
- **Consent:** Cookie writing requires `analytics_storage` + `functionality_storage` consent. The geolocation lookup itself and the GA4 event fire regardless.
- **GTM version:** Tested on GTM Web container (not server-side).

---

## License

MIT — use freely, attribution appreciated.

---

## Credits

- Geolocation data: [ipapi.co](https://ipapi.co)
- Cookie Writer custom template: *drewspen*
- Container design and integration: see blog post for full walkthrough
