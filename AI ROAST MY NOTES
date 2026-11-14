"""
buat_web.py  -  Generator proyek "AI Roast My Notes"

Cara pakai:
    python buat_web.py

Hasil: folder `ai-roast-my-notes/` berisi website lengkap
(HTML + CSS + JS + backend Python) yang siap di-push ke GitHub
dan di-deploy ke Vercel.
"""
from pathlib import Path

ROOT = Path("ai-roast-my-notes")

FILES = {}

# --------------------------------------------------------------------------
# BACKEND PYTHON (Vercel Serverless Function) -> api/roast.py
# --------------------------------------------------------------------------
FILES["api/roast.py"] = r'''
import json
import os
import re
import urllib.request
from http.server import BaseHTTPRequestHandler

API_URL = "https://api.anthropic.com/v1/messages"
API_KEY = os.environ.get("ANTHROPIC_API_KEY", "")
MODEL = os.environ.get("ANTHROPIC_MODEL", "claude-sonnet-5-5")

SYSTEM_ROAST = """Kamu adalah "Roast Master", teman belajar yang super sarkastik dan bermulut pedas.
Tugasmu: mengkritik, mengejek, dan menantang catatan atau pemahaman belajar siswa.
Aturan:
- Sindir kualitas TULISAN dan LOGIKA, bukan pribadi, fisik, agama, atau latar belakang siswa.
- Kritik harus akurat dan menunjuk bagian spesifik dari catatan (kutip singkat boleh).
- Selipkan 2 saran perbaikan yang nyata, dibungkus sarkasme.
- Bahasa Indonesia gaul, maksimal 170 kata. Jangan memuji.
Balas HANYA dengan JSON valid tanpa markdown, format:
{"roast": "teks roast", "skor": 0-100 (kualitas catatan, 100 = bagus),
 "burn": {"logika": 0-100, "struktur": 0-100, "kedalaman": 0-100, "bahasa": 0-100},
 "saran": ["saran 1", "saran 2"]}
Nilai burn = tingkat kehancuran (makin tinggi makin parah)."""

SYSTEM_COMEBACK = """Kamu adalah "Roast Master". Siswa sedang mendebat balik roast-mu.
Nilai argumennya: apakah logis, spesifik, dan didukung data/fakta/sumber yang valid?
- Jika argumen lemah, emosional, atau tanpa bukti: tetap sarkastik, tantang dia membawa data.
- Jika argumen logis dan berdata valid: melunak, akui dengan gaya tetap kocak (jangan lebay).
- Tetap sindir argumen, bukan pribadinya. Bahasa Indonesia gaul, maksimal 120 kata.
Balas HANYA dengan JSON valid tanpa markdown:
{"balasan": "teks balasan", "melunak": true/false, "skor_argumen": 0-100}"""


def clamp(v, lo=5, hi=98):
    return max(lo, min(hi, int(v)))


def call_claude(system, user_text):
    payload = {
        "model": MODEL,
        "max_tokens": 900,
        "system": system,
        "messages": [{"role": "user", "content": user_text}],
    }
    req = urllib.request.Request(
        API_URL,
        data=json.dumps(payload).encode("utf-8"),
        headers={
            "content-type": "application/json",
            "x-api-key": API_KEY,
            "anthropic-version": "2023-06-01",
        },
        method="POST",
    )
    with urllib.request.urlopen(req, timeout=50) as resp:
        data = json.loads(resp.read().decode("utf-8"))
    text = "".join(b.get("text", "") for b in data.get("content", []))
    text = re.sub(r"^```(?:json)?|```$", "", text.strip(), flags=re.M).strip()
    return json.loads(text)


# ---------- Mode offline (dipakai kalau API key belum diisi) ----------
def offline_roast(notes):
    words = re.findall(r"\w+", notes.lower())
    sentences = [s for s in re.split(r"[.!?\n]+", notes) if s.strip()]
    avg_len = len(words) / max(len(sentences), 1)
    uniq = len(set(words)) / max(len(words), 1)
    has_data = bool(re.search(r"\d", notes))
    has_logic = bool(re.search(r"\b(karena|sehingga|misalnya|contohnya|namun|oleh karena itu)\b", notes.lower()))

    burn = {
        "logika": clamp(85 - (25 if has_logic else 0) - (10 if has_data else 0)),
        "struktur": clamp(80 if len(sentences) < 4 else (35 if avg_len < 25 else 65)),
        "kedalaman": clamp(90 if len(words) < 60 else (55 if len(words) < 150 else 35)),
        "bahasa": clamp(30 + (1 - uniq) * 100),
    }
    worst = max(burn, key=burn.get)
    lines = {
        "logika": "Argumenmu berdiri tanpa 'karena' dan tanpa contoh. Itu bukan penalaran, itu pengumuman.",
        "struktur": "Struktur tulisanmu mirip lemari habis gempa: isinya ada, tapi nggak ada yang di tempatnya.",
        "kedalaman": "Segini doang? Ini bukan catatan, ini caption. Gali lebih dalam, bukan cuma permukaan.",
        "bahasa": "Kata-katamu diulang terus kayak kaset rusak. Kamus itu gratis, lho.",
    }
    skor = clamp(100 - sum(burn.values()) / 4)
    return {
        "roast": lines[worst] + " Catatan ini butuh lebih dari sekadar semangat.",
        "skor": skor,
        "burn": burn,
        "saran": [
            "Tambahkan alasan (karena/sehingga) di setiap klaim.",
            "Sertakan minimal satu contoh nyata atau angka per poin.",
        ],
        "sumber": "offline",
    }


def offline_comeback(argument):
    ok = len(argument) > 150 and bool(
        re.search(r"\b(karena|data|menurut|sumber|penelitian|contohnya|\d{2,})\b", argument.lower())
    )
    if ok:
        return {
            "balasan": "Hmm... argumenmu punya alasan dan bukti. Oke, aku akui. Sekali ini saja. Jangan GR dulu.",
            "melunak": True,
            "skor_argumen": 78,
            "sumber": "offline",
        }
    return {
        "balasan": "Itu pembelaan atau curhat? Bawa data, sumber, atau contoh nyata. Perasaan bukan bukti.",
        "melunak": False,
        "skor_argumen": 25,
        "sumber": "offline",
    }


class handler(BaseHTTPRequestHandler):
    def _send(self, code, data):
        body = json.dumps(data, ensure_ascii=False).encode("utf-8")
        self.send_response(code)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def do_POST(self):
        try:
            size = int(self.headers.get("Content-Length", 0))
            data = json.loads(self.rfile.read(size) or b"{}")
        except Exception:
            return self._send(400, {"error": "Format permintaan salah."})

        mode = data.get("mode", "roast")
        notes = (data.get("notes") or "").strip()[:6000]
        if len(notes) < 30:
            return self._send(400, {"error": "Catatannya terlalu pendek. Minimal 30 karakter."})

        try:
            if mode == "comeback":
                argument = (data.get("argument") or "").strip()[:2000]
                if len(argument) < 10:
                    return self._send(400, {"error": "Tulis argumenmu dulu."})
                if not API_KEY:
                    return self._send(200, offline_comeback(argument))
                prompt = (
                    "CATATAN SISWA:\n" + notes
                    + "\n\nROAST-MU SEBELUMNYA:\n" + (data.get("roast") or "")
                    + "\n\nARGUMEN BALASAN SISWA:\n" + argument
                )
                return self._send(200, call_claude(SYSTEM_COMEBACK, prompt))

            if not API_KEY:
                return self._send(200, offline_roast(notes))
            return self._send(200, call_claude(SYSTEM_ROAST, "CATATAN SISWA:\n" + notes))
        except Exception as exc:
            return self._send(500, {"error": "Roast Master lagi error: " + str(exc)[:120]})
'''

# --------------------------------------------------------------------------
# FRONTEND
# --------------------------------------------------------------------------
FILES["index.html"] = r'''<!DOCTYPE html>
<html lang="id">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Roast My Notes - Teman belajar yang nggak ramah</title>
  <meta name="description" content="Tempel catatanmu. AI sarkastik akan mengkritiknya habis-habisan.">
  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:wght@400;600;800&family=Caveat:wght@500;700&display=swap" rel="stylesheet">
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <main class="page">
    <header class="top">
      <h1>Roast My Notes</h1>
      <p>Tempel catatanmu. Jangan harap dipuji.</p>
    </header>

    <section class="input-box">
      <label for="notes">Catatan, esai, atau rangkumanmu</label>
      <textarea id="notes" rows="9" placeholder="Contoh: Fotosintesis adalah proses tumbuhan membuat makanan..."></textarea>
      <div class="row">
        <span id="count">0 karakter</span>
        <button id="roastBtn" class="btn">Roast catatanku</button>
      </div>
      <p id="error" class="error" role="alert" hidden></p>
    </section>

    <section id="result" class="result" hidden>
      <div class="meter-wrap">
        <h2>Burn Meter</h2>
        <svg viewBox="0 0 200 115" class="gauge" role="img" aria-label="Burn meter">
          <path d="M10 100 A90 90 0 0 1 190 100" class="g-bg" pathLength="100"/>
          <path id="gaugeFill" d="M10 100 A90 90 0 0 1 190 100" class="g-fill" pathLength="100"/>
          <text id="gaugeNum" x="100" y="92" text-anchor="middle">0</text>
        </svg>
        <p id="gaugeLabel" class="gauge-label"></p>
        <div id="bars" class="bars"></div>
      </div>

      <div class="roast-card">
        <h2>Kata Roast Master</h2>
        <p id="roastText" class="roast"></p>
        <h3>Kalau mau tidak dihina lagi</h3>
        <ul id="tips"></ul>
        <p id="source" class="source"></p>
      </div>

      <div class="comeback">
        <h2>Mode Comeback</h2>
        <p class="hint">Merasa dizalimi? Bantah dengan logika dan data. Kalau valid, dia melunak.</p>
        <div id="chat" class="chat" aria-live="polite"></div>
        <textarea id="argument" rows="3" placeholder="Tulis pembelaanmu, lengkap dengan alasan dan data..."></textarea>
        <button id="replyBtn" class="btn ghost">Kirim bantahan</button>
      </div>
    </section>
  </main>
  <script src="script.js"></script>
</body>
</html>
'''

FILES["style.css"] = r'''
:root {
  --paper: #eaf0f6;
  --line: #c9d6e4;
  --ink: #1b2a41;
  --pen: #d7263d;
  --mark: #ffe45e;
  --soft: #5b6b82;
}
* { box-sizing: border-box; }
html { scroll-behavior: smooth; }
body {
  margin: 0;
  font-family: "Bricolage Grotesque", system-ui, sans-serif;
  color: var(--ink);
  background-color: var(--paper);
  background-image: repeating-linear-gradient(to bottom, transparent 0 31px, var(--line) 31px 32px);
  line-height: 1.6;
}
.page {
  max-width: 760px;
  margin: 0 auto;
  padding: 40px 20px 80px 56px;
  border-left: 2px solid var(--pen);
  min-height: 100vh;
}
.top h1 {
  font-size: clamp(2.4rem, 8vw, 4.2rem);
  font-weight: 800;
  line-height: 1;
  margin: 0 0 8px;
  text-decoration: underline wavy var(--pen);
  text-underline-offset: 8px;
}
.top p { margin: 0 0 32px; color: var(--soft); font-size: 1.1rem; }

label { font-weight: 600; display: block; margin-bottom: 8px; }
textarea {
  width: 100%;
  font: inherit;
  padding: 14px;
  border: 2px solid var(--ink);
  border-radius: 6px;
  background: #fff;
  color: var(--ink);
  resize: vertical;
}
textarea:focus-visible, .btn:focus-visible { outline: 3px solid var(--mark); outline-offset: 2px; }
.row { display: flex; justify-content: space-between; align-items: center; margin-top: 10px; gap: 12px; }
#count { color: var(--soft); font-size: .9rem; }

.btn {
  font: inherit;
  font-weight: 800;
  padding: 12px 22px;
  background: var(--pen);
  color: #fff;
  border: 2px solid var(--ink);
  border-radius: 6px;
  cursor: pointer;
  box-shadow: 3px 3px 0 var(--ink);
  transition: transform .1s, box-shadow .1s;
}
.btn:hover { transform: translate(1px, 1px); box-shadow: 2px 2px 0 var(--ink); }
.btn:disabled { opacity: .6; cursor: wait; }
.btn.ghost { background: var(--mark); color: var(--ink); margin-top: 10px; }
.error { color: var(--pen); font-weight: 600; }

.result { margin-top: 48px; display: grid; gap: 36px; }
h2 { font-size: 1.5rem; margin: 0 0 12px; }
h3 { margin: 20px 0 6px; font-size: 1rem; }

.gauge { width: 100%; max-width: 340px; display: block; }
.g-bg, .g-fill { fill: none; stroke-width: 16; stroke-linecap: round; }
.g-bg { stroke: #fff; }
.g-fill { stroke: var(--pen); stroke-dasharray: 0 100; transition: stroke-dasharray 1.2s ease-out; }
#gaugeNum { font: 800 34px "Bricolage Grotesque", sans-serif; fill: var(--ink); }
.gauge-label { font-family: "Caveat", cursive; font-size: 1.7rem; color: var(--pen); margin: 0 0 12px; }

.bars { display: grid; gap: 10px; max-width: 420px; }
.bar { display: grid; grid-template-columns: 90px 1fr 36px; align-items: center; gap: 10px; font-size: .95rem; }
.track { height: 12px; background: #fff; border: 2px solid var(--ink); border-radius: 8px; overflow: hidden; }
.fill { height: 100%; width: 0; background: var(--pen); transition: width 1s ease-out; }

.roast-card { background: #fff; border: 2px solid var(--ink); border-radius: 6px; padding: 22px; }
.roast { font-family: "Caveat", cursive; font-size: 1.8rem; line-height: 1.3; color: var(--pen); margin: 0; white-space: pre-line; }
.roast-card ul { margin: 0; padding-left: 20px; }
.roast-card li { background: linear-gradient(transparent 60%, var(--mark) 60%); width: fit-content; }
.source { font-size: .8rem; color: var(--soft); margin: 16px 0 0; }

.hint { color: var(--soft); margin-top: 0; }
.chat { display: grid; gap: 10px; margin-bottom: 14px; }
.bubble { padding: 10px 14px; border: 2px solid var(--ink); border-radius: 6px; max-width: 88%; white-space: pre-line; }
.bubble.user { background: #fff; justify-self: end; }
.bubble.ai { background: var(--paper); font-family: "Caveat", cursive; font-size: 1.5rem; line-height: 1.25; color: var(--pen); }
.bubble.soft { background: var(--mark); color: var(--ink); }

@media (max-width: 560px) {
  .page { padding: 28px 14px 60px 24px; }
  .bar { grid-template-columns: 76px 1fr 32px; }
}
@media (prefers-reduced-motion: reduce) {
  .g-fill, .fill, .btn { transition: none; }
}
'''

FILES["script.js"] = r'''
const $ = (s) => document.querySelector(s);
const state = { notes: "", roast: "" };
const LABELS = { logika: "Logika", struktur: "Struktur", kedalaman: "Kedalaman", bahasa: "Bahasa" };

$("#notes").addEventListener("input", (e) => {
  $("#count").textContent = e.target.value.length + " karakter";
});

async function post(payload) {
  try {
    const res = await fetch("/api/roast", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload),
    });
    // Di GitHub Pages tidak ada backend -> pakai mode lokal
    if (res.status === 404 || res.status === 405) return localFallback(payload);
    const data = await res.json();
    if (!res.ok) throw new Error(data.error || "Terjadi kesalahan.");
    return data;
  } catch (err) {
    if (err instanceof TypeError) return localFallback(payload); // jaringan/backend tidak ada
    throw err;
  }
}

/* ---------- Mode lokal (tanpa AI) supaya web tetap hidup di GitHub Pages ---------- */
function localFallback(p) {
  if (p.mode === "comeback") {
    const a = p.argument.toLowerCase();
    const ok = a.length > 150 && /(karena|data|menurut|sumber|penelitian|contohnya|\d{2,})/.test(a);
    return ok
      ? { balasan: "Oke, argumenmu ada alasan dan buktinya. Aku akui. Sekali ini saja.", melunak: true, skor_argumen: 78, sumber: "lokal" }
      : { balasan: "Itu pembelaan atau curhat? Bawa data dan contoh nyata.", melunak: false, skor_argumen: 25, sumber: "lokal" };
  }
  const words = p.notes.toLowerCase().match(/\w+/g) || [];
  const sentences = p.notes.split(/[.!?\n]+/).filter((s) => s.trim());
  const uniq = new Set(words).size / Math.max(words.length, 1);
  const hasLogic = /(karena|sehingga|misalnya|contohnya|namun)/.test(p.notes.toLowerCase());
  const clamp = (v) => Math.max(5, Math.min(98, Math.round(v)));
  const burn = {
    logika: clamp(hasLogic ? 55 : 85),
    struktur: clamp(sentences.length < 4 ? 80 : 50),
    kedalaman: clamp(words.length < 60 ? 90 : words.length < 150 ? 55 : 35),
    bahasa: clamp(30 + (1 - uniq) * 100),
  };
  const worst = Object.keys(burn).sort((a, b) => burn[b] - burn[a])[0];
  const lines = {
    logika: "Argumenmu berdiri tanpa 'karena' dan contoh. Itu bukan penalaran, itu pengumuman.",
    struktur: "Strukturnya mirip lemari habis gempa: isinya ada, tapi nggak ada yang di tempatnya.",
    kedalaman: "Segini doang? Ini bukan catatan, ini caption.",
    bahasa: "Kata-katamu diulang terus kayak kaset rusak.",
  };
  return {
    roast: lines[worst] + " Catatan ini butuh lebih dari sekadar semangat.",
    skor: clamp(100 - Object.values(burn).reduce((a, b) => a + b, 0) / 4),
    burn,
    saran: ["Tambahkan alasan di setiap klaim.", "Sertakan satu contoh nyata per poin."],
    sumber: "lokal",
  };
}

/* ---------- Render ---------- */
function burnLabel(v) {
  if (v >= 85) return "Gosong total";
  if (v >= 65) return "Matang merata";
  if (v >= 40) return "Setengah matang";
  return "Cuma kena asap";
}

function render(data) {
  const burn = data.burn;
  const avg = Math.round(Object.values(burn).reduce((a, b) => a + b, 0) / 4);

  $("#result").hidden = false;
  $("#gaugeNum").textContent = avg;
  $("#gaugeLabel").textContent = burnLabel(avg);
  requestAnimationFrame(() => {
    $("#gaugeFill").style.strokeDasharray = avg + " 100";
  });

  $("#bars").innerHTML = Object.entries(burn)
    .map(([k, v]) => `<div class="bar"><span>${LABELS[k] || k}</span>
      <div class="track"><div class="fill" data-w="${v}"></div></div><span>${v}</span></div>`)
    .join("");
  requestAnimationFrame(() => {
    document.querySelectorAll(".fill").forEach((el) => (el.style.width = el.dataset.w + "%"));
  });

  $("#roastText").textContent = data.roast;
  $("#tips").innerHTML = (data.saran || []).map((t) => `<li></li>`).join("");
  document.querySelectorAll("#tips li").forEach((li, i) => (li.textContent = data.saran[i]));
  $("#source").textContent = data.sumber
    ? "Mode " + data.sumber + ": pasang ANTHROPIC_API_KEY di server untuk roast dari AI sungguhan."
    : "";

  $("#chat").innerHTML = "";
  $("#result").scrollIntoView({ behavior: "smooth" });
}

function bubble(text, cls) {
  const div = document.createElement("div");
  div.className = "bubble " + cls;
  div.textContent = text;
  $("#chat").appendChild(div);
}

function showError(msg) {
  const el = $("#error");
  el.textContent = msg;
  el.hidden = !msg;
}

/* ---------- Aksi ---------- */
$("#roastBtn").addEventListener("click", async () => {
  const notes = $("#notes").value.trim();
  showError("");
  if (notes.length < 30) return showError("Catatannya terlalu pendek. Minimal 30 karakter.");
  const btn = $("#roastBtn");
  btn.disabled = true;
  btn.textContent = "Lagi dibakar...";
  try {
    const data = await post({ mode: "roast", notes });
    state.notes = notes;
    state.roast = data.roast;
    render(data);
  } catch (e) {
    showError(e.message);
  } finally {
    btn.disabled = false;
    btn.textContent = "Roast catatanku";
  }
});

$("#replyBtn").addEventListener("click", async () => {
  const argument = $("#argument").value.trim();
  if (argument.length < 10) return;
  const btn = $("#replyBtn");
  btn.disabled = true;
  bubble(argument, "user");
  $("#argument").value = "";
  try {
    const data = await post({ mode: "comeback", notes: state.notes, roast: state.roast, argument });
    bubble(data.balasan, data.melunak ? "ai soft" : "ai");
    if (data.melunak) bubble("Roast Master melunak. Skor argumenmu: " + data.skor_argumen + "/100", "soft");
  } catch (e) {
    bubble(e.message, "ai");
  } finally {
    btn.disabled = false;
  }
});
'''

FILES["README.md"] = r'''# AI Roast My Notes

Teman belajar sarkastik: tempel catatan, dapat roast + Burn Meter, lalu bantah balik di Mode Comeback.

## Struktur
- `index.html`, `style.css`, `script.js` : frontend
- `api/roast.py` : backend Python (Vercel Serverless Function)

## Deploy (Vercel + GitHub)
1. Push folder ini ke repository GitHub.
2. vercel.com -> Add New Project -> pilih repository -> Deploy.
3. Settings -> Environment Variables -> tambah `ANTHROPIC_API_KEY` -> Redeploy.

Tanpa API key, web tetap jalan dengan mode roast sederhana (offline).
GitHub Pages hanya hosting statis: frontend tetap jalan dengan mode lokal, tapi tanpa AI sungguhan.
'''

FILES[".gitignore"] = ".env\n__pycache__/\n.vercel/\n"


def main():
    for rel_path, content in FILES.items():
        target = ROOT / rel_path
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_text(content.lstrip("\n"), encoding="utf-8")
        print("dibuat:", target)
    print("\nSelesai! Lanjut: cd ai-roast-my-notes && git init && git add . && git commit -m 'first commit'")


if __name__ == "__main__":
    main()
