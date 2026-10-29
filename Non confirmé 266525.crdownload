// api/check.js — fonction Vercel (Node.js)
// Détecte les zones OB / Supply / Demand (M15) sur 6 actifs et envoie une
// notification (ntfy) quand le prix S'APPROCHE d'une zone.
// Variables d'environnement à définir dans Vercel :
//   NTFY_TOPIC   (nom secret du canal ntfy)
//   CRON_SECRET  (clé secrète passée dans l'adresse ?key=...)
//   APPROACH_ATR (facultatif, défaut 0.5 : distance d'alerte en multiple de l'ATR M15)

const SYMBOLS = [
  { name: "XAUUSD", yahoo: "XAUUSD=X", dec: 2, maxPts: 6 },
  { name: "BTCUSD", yahoo: "BTC-USD", dec: 1 },
  { name: "USDJPY", yahoo: "USDJPY=X", dec: 3 },
  { name: "EURGBP", yahoo: "EURGBP=X", dec: 5 },
  { name: "EURUSD", yahoo: "EURUSD=X", dec: 5 },
  { name: "GBPUSD", yahoo: "GBPUSD=X", dec: 5 }
];

// Mêmes réglages que l'indicateur TradingView
const P = { atrLen: 14, majLen: 10, obMinATR: 2.0, obRunMax: 6, obMaxATR: 1.0, impATR: 1.2, baseATR: 0.5, keepDays: 3, maxZones: 6 };

// ───────── données ─────────
async function fetchM5(sym) {
  const url = "https://query1.finance.yahoo.com/v8/finance/chart/" + encodeURIComponent(sym) + "?interval=5m&range=5d";
  const r = await fetch(url, { headers: { "User-Agent": "Mozilla/5.0" } });
  if (!r.ok) throw new Error("Yahoo " + r.status);
  const j = await r.json();
  const res = j.chart && j.chart.result && j.chart.result[0];
  if (!res || !res.timestamp) throw new Error("pas de données");
  const q = res.indicators.quote[0], out = [];
  for (let i = 0; i < res.timestamp.length; i++) {
    if (q.open[i] == null || q.high[i] == null || q.low[i] == null || q.close[i] == null) continue;
    out.push({ t: res.timestamp[i], o: q.open[i], h: q.high[i], l: q.low[i], c: q.close[i] });
  }
  return out;
}

function toM15(m5, nowSec) {
  const out = []; let cur = null;
  for (const b of m5) {
    const k = Math.floor(b.t / 900);
    if (!cur || cur.k !== k) { if (cur) out.push(cur); cur = { k, t: k * 900, o: b.o, h: b.h, l: b.l, c: b.c }; }
    else { cur.h = Math.max(cur.h, b.h); cur.l = Math.min(cur.l, b.l); cur.c = b.c; }
  }
  if (cur) out.push(cur);
  if (out.length && (out[out.length - 1].k + 1) * 900 > nowSec) out.pop(); // bougie M15 pas encore terminée
  return out;
}

function atrSeries(c, len) {
  const a = new Array(c.length).fill(null); let sum = 0;
  for (let i = 0; i < c.length; i++) {
    const tr = i === 0 ? c[i].h - c[i].l : Math.max(c[i].h - c[i].l, Math.abs(c[i].h - c[i - 1].c), Math.abs(c[i].l - c[i - 1].c));
    if (i < len) { sum += tr; if (i === len - 1) a[i] = sum / len; } else a[i] = (a[i - 1] * (len - 1) + tr) / len;
  }
  return a;
}

// ───────── détection des zones (port de l'indicateur) ─────────
function pivotHigh(c, p, L) { for (let k = 1; k <= L; k++) if (c[p - k].h >= c[p].h || c[p + k].h > c[p].h) return false; return true; }
function pivotLow(c, p, L) { for (let k = 1; k <= L; k++) if (c[p - k].l <= c[p].l || c[p + k].l < c[p].l) return false; return true; }

function refine(b, bull, maxH) {
  let top = b.h, bot = b.l;
  if (top - bot > maxH) { top = Math.max(b.o, b.c); bot = Math.min(b.o, b.c); }
  if (top - bot > maxH) { if (bull) top = bot + maxH; else bot = top - maxH; }
  return { top, bot };
}

function detectZones(c, atr, cfg) {
  const zones = [];
  let lastMH = null, lastML = null, lastMHi = -1, lastMLi = -1, brkH = -1, brkL = -1;
  for (let t = 0; t < c.length; t++) {
    const a = atr[t]; if (a == null) continue;
    const p = t - P.majLen;
    if (p >= P.majLen) {
      if (pivotHigh(c, p, P.majLen)) { lastMH = c[p].h; lastMHi = p; }
      if (pivotLow(c, p, P.majLen)) { lastML = c[p].l; lastMLi = p; }
    }
    const maxH = Math.min(P.obMaxATR * a, cfg.maxPts != null ? cfg.maxPts : Infinity);

    // BOS haussier -> OB Bull (dernière bougie baissière avant l'impulsion)
    if (lastMH != null && c[t].c > lastMH && brkH !== lastMHi) {
      brkH = lastMHi;
      let i0 = 0; while (i0 <= P.obRunMax && t - i0 >= 0 && c[t - i0].c >= c[t - i0].o) i0++;
      if (i0 >= 1 && i0 <= P.obRunMax && t - i0 >= 0 && c[t].c - c[t - i0].l >= P.obMinATR * a) {
        const z = refine(c[t - i0], true, maxH); let fresh = true;
        for (let j = 0; j < i0; j++) if (c[t - j].c < z.bot) fresh = false;
        if (fresh) zones.push({ kind: "OB", buy: true, top: z.top, bot: z.bot, t0: c[t - i0].t, created: t });
      }
    }
    // BOS baissier -> OB Bear (dernière bougie haussière avant l'impulsion)
    if (lastML != null && c[t].c < lastML && brkL !== lastMLi) {
      brkL = lastMLi;
      let i1 = 0; while (i1 <= P.obRunMax && t - i1 >= 0 && c[t - i1].c <= c[t - i1].o) i1++;
      if (i1 >= 1 && i1 <= P.obRunMax && t - i1 >= 0 && c[t - i1].h - c[t].c >= P.obMinATR * a) {
        const z = refine(c[t - i1], false, maxH); let fresh = true;
        for (let j = 0; j < i1; j++) if (c[t - j].c > z.top) fresh = false;
        if (fresh) zones.push({ kind: "OB", buy: false, top: z.top, bot: z.bot, t0: c[t - i1].t, created: t });
      }
    }
    // Supply / Demand : base + bougie d'impulsion
    if (t >= 2) {
      const body = Math.abs(c[t].c - c[t].o), rng = c[t].h - c[t].l;
      const base1 = Math.abs(c[t - 1].c - c[t - 1].o) <= P.baseATR * a;
      const base2 = Math.abs(c[t - 2].c - c[t - 2].o) <= P.baseATR * a;
      if (base1 && body >= P.impATR * a && body >= 0.6 * rng) {
        if (c[t].c > c[t].o && c[t].c > c[t - 1].h) {
          let top = Math.max(c[t - 1].o, c[t - 1].c), bot = c[t - 1].l, t0 = c[t - 1].t;
          if (base2) { top = Math.max(top, c[t - 2].o, c[t - 2].c); bot = Math.min(bot, c[t - 2].l); t0 = c[t - 2].t; }
          zones.push({ kind: "DEMAND", buy: true, top, bot, t0, created: t });
        } else if (c[t].c < c[t].o && c[t].c < c[t - 1].l) {
          let bot = Math.min(c[t - 1].o, c[t - 1].c), top = c[t - 1].h, t0 = c[t - 1].t;
          if (base2) { bot = Math.min(bot, c[t - 2].o, c[t - 2].c); top = Math.max(top, c[t - 2].h); t0 = c[t - 2].t; }
          zones.push({ kind: "SUPPLY", buy: false, top, bot, t0, created: t });
        }
      }
    }
  }
  // zones non mitigées (clôture au travers = zone morte)
  const alive = zones.filter(z => {
    for (let k = z.created + 1; k < c.length; k++) { if (z.buy ? c[k].c < z.bot : c[k].c > z.top) return false; }
    return true;
  });
  const minT = c[c.length - 1].t - P.keepDays * 86400;
  const recent = alive.filter(z => z.t0 >= minT);
  const ob = recent.filter(z => z.kind === "OB").slice(-P.maxZones);
  const sd = recent.filter(z => z.kind !== "OB").slice(-P.maxZones);
  return ob.concat(sd);
}

// Distance du prix à la zone, du bon côté seulement (null = prix de l'autre côté / zone traversée)
function distance(z, p) {
  if (z.buy) { if (p < z.bot) return null; return p > z.top ? p - z.top : 0; }
  if (p > z.top) return null; return p < z.bot ? z.bot - p : 0;
}

async function check(sym, approachATR, nowSec) {
  const m5 = await fetchM5(sym.yahoo);
  if (m5.length < 3) throw new Error("pas assez de données");
  const last = m5[m5.length - 1], prev = m5[m5.length - 2];
  if (nowSec - last.t > 1200) return { name: sym.name, closed: true, zones: [], alerts: [] }; // marché fermé
  const c = toM15(m5, nowSec), atr = atrSeries(c, P.atrLen);
  let a = null; for (let i = atr.length - 1; i >= 0; i--) if (atr[i] != null) { a = atr[i]; break; }
  if (a == null) throw new Error("ATR indisponible");
  const zones = detectZones(c, atr, sym), th = approachATR * a, p0 = last.c, pPrev = prev.c;
  const alerts = [];
  for (const z of zones) {
    const d0 = distance(z, p0), dPrev = distance(z, pPrev);
    if (d0 != null && d0 <= th && (dPrev == null || dPrev > th)) {
      const entry = (z.top + z.bot) / 2;
      alerts.push({
        title: (z.buy ? "🟢 ACHAT " : "🔴 VENTE ") + sym.name,
        message: "Le prix approche d'une zone " + z.kind + " · niveau " + entry.toFixed(sym.dec) + " · prix " + p0.toFixed(sym.dec) + " (à " + d0.toFixed(sym.dec) + ")",
        tag: z.buy ? "green_circle" : "red_circle"
      });
    }
  }
  return {
    name: sym.name, price: p0, atr: a, threshold: th,
    zones: zones.map(z => ({ kind: z.kind, side: z.buy ? "ACHAT" : "VENTE", top: +z.top.toFixed(sym.dec), bot: +z.bot.toFixed(sym.dec), dist: distance(z, p0) })),
    alerts
  };
}

async function notify(topic, title, message, tag) {
  const r = await fetch("https://ntfy.sh", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ topic, title, message, priority: 4, tags: tag ? [tag] : [] })
  });
  if (!r.ok) throw new Error("ntfy " + r.status);
}

module.exports = async (req, res) => {
  const q = req.query || {};
  if (!process.env.CRON_SECRET || q.key !== process.env.CRON_SECRET) return res.status(401).json({ error: "clé invalide" });
  const topic = process.env.NTFY_TOPIC;
  if (!topic) return res.status(500).json({ error: "NTFY_TOPIC manquant" });
  try {
    if (q.test) { await notify(topic, "🔔 Test", "Les notifications fonctionnent.", "bell"); return res.json({ ok: true, test: true }); }
    const approach = parseFloat(process.env.APPROACH_ATR || "0.5") || 0.5, nowSec = Math.floor(Date.now() / 1000);
    const results = await Promise.all(SYMBOLS.map(s => check(s, approach, nowSec).catch(e => ({ name: s.name, error: e.message, zones: [], alerts: [] }))));
    let sent = 0;
    if (!q.dry) for (const r of results) for (const a of r.alerts) { await notify(topic, a.title, a.message, a.tag); sent++; }
    res.json({ ok: true, dry: !!q.dry, sent, results });
  } catch (e) { res.status(500).json({ error: e.message }); }
};
module.exports._test = { detectZones, atrSeries, toM15, distance };
