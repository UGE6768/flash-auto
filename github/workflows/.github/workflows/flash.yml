"""Flash Auto : récupère l'actualité (Google Actualités), la fait trier par Gemini, écrit brief.json."""
import json, os, re, time, email.utils, urllib.request, urllib.parse, urllib.error
import xml.etree.ElementTree as ET
from datetime import datetime, timezone, timedelta
from zoneinfo import ZoneInfo

CFG = json.load(open("config.json", encoding="utf-8"))
KEY = os.environ.get("GEMINI_API_KEY", "").strip()
MODEL = CFG.get("modele", "gemini-3.6-flash")
NB = int(CFG.get("infos_par_rubrique", 4))
PARIS = ZoneInfo("Europe/Paris")
NOW = datetime.now(PARIS)


def http_get(url):
    req = urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0 (FlashAuto)"})
    with urllib.request.urlopen(req, timeout=30) as r:
        return r.read()


def google_news(query):
    url = "https://news.google.com/rss/search?" + urllib.parse.urlencode(
        {"q": query + " when:2d", "hl": "fr", "gl": "FR", "ceid": "FR:fr"})
    root = ET.fromstring(http_get(url))
    out = []
    for it in root.iter("item"):
        titre = (it.findtext("title") or "").strip()
        source = (it.findtext("source") or "").strip()
        if source and titre.endswith(" - " + source):
            titre = titre[: -len(source) - 3].strip()
        try:
            dt = email.utils.parsedate_to_datetime(it.findtext("pubDate") or "")
        except Exception:
            dt = None
        out.append({"titre": titre, "source": source, "url": (it.findtext("link") or "").strip(),
                    "ts": dt.timestamp() if dt else 0,
                    "date": dt.astimezone(PARIS).strftime("%d/%m") if dt else ""})
    return out


def norm(t):
    return re.sub(r"[^a-z0-9]", "", t.lower())[:60]


def collect(rub):
    seen, items = set(), []
    for q in rub.get("recherches", []):
        try:
            for a in google_news(q):
                k = norm(a["titre"])
                if a["titre"] and k not in seen:
                    seen.add(k)
                    items.append(a)
        except Exception as e:
            print("Recherche en échec :", q, e)
        time.sleep(1)
    items.sort(key=lambda a: a["ts"], reverse=True)
    return items[:30]


def gemini(prompt):
    url = f"https://generativelanguage.googleapis.com/v1beta/models/{MODEL}:generateContent?key={KEY}"
    body = {"contents": [{"role": "user", "parts": [{"text": prompt}]}],
            "generationConfig": {"temperature": 0.3, "responseMimeType": "application/json"}}
    for essai in range(3):
        try:
            req = urllib.request.Request(url, data=json.dumps(body).encode(),
                                         headers={"Content-Type": "application/json"})
            with urllib.request.urlopen(req, timeout=180) as r:
                data = json.loads(r.read())
            text = "".join(p.get("text", "") for p in data["candidates"][0]["content"]["parts"])
            return parse_json(text)
        except urllib.error.HTTPError as e:
            msg = e.read().decode(errors="ignore")[:300]
            print(f"Gemini HTTP {e.code} (essai {essai + 1}) :", msg)
            if e.code in (429, 500, 503) and essai < 2:
                time.sleep(40)
                continue
            raise
    raise RuntimeError("Gemini indisponible")


def parse_json(text):
    t = text.replace("```json", "").replace("```", "").strip()
    try:
        return json.loads(t)
    except Exception:
        a, b = min([i for i in (t.find("["), t.find("{")) if i >= 0] or [0]), max(t.rfind("]"), t.rfind("}"))
        return json.loads(t[a:b + 1])


def select(rub, articles):
    liste = "\n".join(f"{i}. [{a['date']}] {a['titre']} ({a['source']})" for i, a in enumerate(articles))
    prompt = f"""Tu prépares le flash d'actualité automobile du matin d'un directeur de concessions.
Contexte du lecteur : {CFG.get('contexte', '')}
Rubrique : {rub['nom']} — {rub.get('consigne', '')}

Voici les titres d'articles publiés ces dernières 48 heures :
{liste}

Choisis les {NB} articles les plus utiles pour ce lecteur et cette rubrique, sans doublon (un même sujet traité par plusieurs médias = un seul article), du plus important au moins important. Écarte ce qui est hors sujet.
Pour chacun, écris "enjeu" : une phrase en français qui explique l'information et pourquoi elle compte. Appuie-toi uniquement sur le titre : n'invente aucun chiffre ni détail absent du titre.
Réponds avec un tableau JSON : [{{"n": numéro de l'article, "enjeu": "...", "important": true si impact direct pour un distributeur Mercedes-Benz en France, sinon false}}]"""
    choix = gemini(prompt)
    out = []
    for c in choix if isinstance(choix, list) else []:
        try:
            a = articles[int(c["n"])]
        except Exception:
            continue
        out.append({**{k: a[k] for k in ("titre", "source", "url", "date")},
                    "enjeu": str(c.get("enjeu", "")).strip(), "important": bool(c.get("important"))})
    return out[:NB]


def main():
    brief = {"date": NOW.strftime("%Y-%m-%d"), "genere": NOW.isoformat(timespec="minutes"),
             "mode": "ia" if KEY else "titres", "essentiel": [], "rubs": []}
    for rub in CFG["rubriques"]:
        entry = {"id": rub["id"], "nom": rub["nom"], "items": [], "error": None}
        articles = collect(rub)
        print(rub["nom"], ":", len(articles), "articles trouvés")
        if articles:
            try:
                if not KEY:
                    raise RuntimeError("pas de clé")
                entry["items"] = select(rub, articles)
                time.sleep(8)
            except Exception as e:
                print("Tri IA impossible, titres bruts :", e)
                brief["mode"] = "titres"
                entry["items"] = [{**{k: a[k] for k in ("titre", "source", "url", "date")},
                                   "enjeu": "", "important": False} for a in articles[:NB]]
        brief["rubs"].append(entry)

    lignes = "\n".join(f"[{r['nom']}] {it['titre']} : {it['enjeu']}" for r in brief["rubs"] for it in r["items"])
    if KEY and lignes:
        try:
            ess = gemini(f"""Voici l'actualité automobile du jour :\n{lignes}\n\nContexte du lecteur : {CFG.get('contexte', '')}
Rédige les 3 points à retenir en priorité pour ce lecteur, une phrase chacun, en français, factuels, sans rien inventer au-delà de ces informations.
Réponds avec un tableau JSON de 3 chaînes.""")
            brief["essentiel"] = [str(x) for x in ess][:3] if isinstance(ess, list) else []
        except Exception as e:
            print("Essentiel impossible :", e)

    json.dump(brief, open("brief.json", "w", encoding="utf-8"), ensure_ascii=False, indent=1)
    print("brief.json écrit :", sum(len(r["items"]) for r in brief["rubs"]), "infos, mode", brief["mode"])


if __name__ == "__main__":
    main()
