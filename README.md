<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover" />
<meta name="theme-color" content="#0f1419" />
<meta name="apple-mobile-web-app-capable" content="yes" />
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
<meta name="apple-mobile-web-app-title" content="Gunluk" />
<link rel="manifest" href="manifest.json" />
<title>Kripto Günlük</title>
<style>
  :root { --bg:#0f1419; --card:#1a2332; --line:#2a3a4d; --tx:#e8eef5; --mut:#8aa0b8; --ok:#3dd68c; --bad:#ff6b6b; }
  * { box-sizing:border-box; }
  body { margin:0; font-family:system-ui,sans-serif; background:var(--bg); color:var(--tx); padding-bottom:32px; }
  header { padding:18px 16px 12px; border-bottom:1px solid var(--line); position:sticky; top:0; background:var(--bg); }
  h1 { margin:0; font-size:18px; }
  main { padding:12px; }
  .grid { display:grid; grid-template-columns:1fr 1fr; gap:8px; margin:12px 0; }
  .card { background:var(--card); border:1px solid var(--line); border-radius:14px; padding:12px; }
  .k { color:var(--mut); font-size:12px; }
  .v { font-size:20px; font-weight:700; margin-top:4px; }
  form { display:grid; grid-template-columns:1fr 1fr; gap:8px; }
  label { font-size:12px; color:var(--mut); display:flex; flex-direction:column; gap:4px; }
  input, select, textarea { background:#0d131b; color:var(--tx); border:1px solid var(--line); border-radius:10px; padding:10px; font-size:16px; }
  textarea, .full { grid-column:1/-1; }
  button { border:0; border-radius:10px; padding:12px; font-weight:700; font-size:15px; }
  .go { background:var(--ok); color:#073; }
  .ghost { background:transparent; color:var(--tx); border:1px solid var(--line); }
  .row { display:flex; gap:8px; flex-wrap:wrap; margin-top:8px; }
  table { width:100%; border-collapse:collapse; font-size:13px; }
  th, td { border-bottom:1px solid var(--line); padding:8px 4px; text-align:left; }
  .bad { color:var(--bad); } .ok { color:var(--ok); }
</style>
</head>
<body>
<header>
  <h1>Kripto Günlük</h1>
  <div class="k">Sweep + onay olmadan işlem yok</div>
</header>
<main>
  <div class="grid" id="ozet"></div>
  <div class="row">
    <button class="ghost" onclick="exportJSON()">JSON yedek</button>
    <button class="ghost" onclick="exportCSV()">CSV</button>
  </div>
  <div class="card" style="margin:12px 0;">
    <b>Yeni işlem</b>
    <form id="f">
      <label>Tarih<input name="tarih" type="date" required></label>
      <label>Saat<input name="saat" type="time" required></label>
      <label>Sembol<select name="sembol"><option>BTCUSDT</option><option>ETHUSDT</option></select></label>
      <label>Yön<select name="yon"><option>LONG</option><option>SHORT</option></select></label>
      <label>Setup<select name="setup"><option>SWEEP_RETEST</option><option>BOX_KIRILIM</option><option>DIGER</option></select></label>
      <label>Sweep<select name="sweep"><option>E</option><option>H</option></select></label>
      <label>Onay mumu<select name="onay"><option>E</option><option>H</option></select></label>
      <label>Muma mı bastım<select name="muma"><option>H</option><option>E</option></select></label>
      <label>Giriş<input name="giris" type="number" step="any" required></label>
      <label>Stop<input name="stop" type="number" step="any" required></label>
      <label>Hedef<input name="hedef" type="number" step="any" required></label>
      <label>Risk USDT<input name="risk" type="number" step="any" required></label>
      <label>Sonuç USDT<input name="sonuc" type="number" step="any" required></label>
      <label>MFE R<input name="mfe" type="number" step="any" value="0"></label>
      <label>1R geldi<select name="r1"><option>H</option><option>E</option></select></label>
      <label>Yarım kesildi<select name="yarim"><option>H</option><option>E</option><option>NA</option></select></label>
      <label>Stop kaydırıldı<select name="kaydir"><option>H</option><option>E</option></select></label>
      <label>Stop sonrası 15dk<select name="intikam"><option>H</option><option>E</option></select></label>
      <label>Duygu<select name="duygu"><option>SAGIN</option><option>ACELE</option><option>KORKU</option><option>INTIKAM</option><option>SIKILMA</option></select></label>
      <label class="full">Not<textarea name="not" placeholder="Tek cümle"></textarea></label>
      <button class="go full" type="submit">Kaydet</button>
    </form>
  </div>
  <div class="card"><b>Geçmiş</b><div id="tablo"></div></div>
</main>
<script>
if ("serviceWorker" in navigator) navigator.serviceWorker.register("sw.js");
const KEY="kripto-gunluk-v1";
const load=()=>JSON.parse(localStorage.getItem(KEY)||"[]");
const save=r=>localStorage.setItem(KEY,JSON.stringify(r));
const ihlal=t=>t.muma==="E"||t.kaydir==="E"||t.intikam==="E"||t.sweep==="H"||t.onay==="H";
const tip=t=>t.intikam==="E"?"INTIKAM":t.kaydir==="E"?"STOP_KAYDIR":(t.muma==="E"||t.sweep==="H"||t.onay==="H")?"ERKEN_GIRIS":"YOK";
function render(){
  const rows=load(), n=rows.length;
  const netR=rows.reduce((s,t)=>s+(t.risk?t.sonuc/t.risk:0),0);
  const ihl=rows.filter(ihlal).length, ihlOran=n?Math.round(100*ihl/n):0;
  const yesilEksi=rows.filter(t=>t.mfe>0&&t.sonuc<0).length;
  let karar="VERI_YOK";
  if(n>=10) karar=ihlOran>40?"1_HAFTA_DUR":(ihlOran>=20||netR<0)?"RISK_DUSUR":"DEVAM";
  document.getElementById("ozet").innerHTML=
    `<div class="card"><div class="k">İşlem</div><div class="v">${n}</div></div>
     <div class="card"><div class="k">Net R</div><div class="v">${netR.toFixed(2)}</div></div>
     <div class="card"><div class="k">İhlal</div><div class="v ${ihlOran>=30?"bad":"ok"}">%${ihlOran}</div></div>
     <div class="card"><div class="k">Karar / yeşil-eksi</div><div class="v">${karar} · ${yesilEksi}</div></div>`;
  document.getElementById("tablo").innerHTML=n?`<table><thead><tr><th>Tarih</th><th>Çift</th><th>R</th><th>İhlal</th></tr></thead><tbody>${
    rows.slice().reverse().map(t=>{
      const r=t.risk?(t.sonuc/t.risk).toFixed(2):"0";
      const ih=ihlal(t);
      return `<tr><td>${t.tarih}</td><td>${t.sembol} ${t.yon}</td><td class="${r<0?"bad":"ok"}">${r}</td><td class="${ih?"bad":"ok"}">${ih?tip(t):"YOK"}</td></tr>`;
    }).join("")
  }</tbody></table>`:"<p class='k'>Henüz işlem yok.</p>";
}
document.getElementById("f").onsubmit=e=>{
  e.preventDefault();
  const t=Object.fromEntries(new FormData(e.target).entries());
  ["risk","sonuc","mfe","giris","stop","hedef"].forEach(k=>t[k]=+t[k]);
  const rows=load(); rows.push(t); save(rows);
  e.target.reset();
  e.target.tarih.value=new Date().toISOString().slice(0,10);
  render();
};
function exportJSON(){const a=document.createElement("a");a.href=URL.createObjectURL(new Blob([JSON.stringify(load(),null,2)],{type:"application/json"}));a.download="gunluk.json";a.click();}
function exportCSV(){const rows=load();if(!rows.length)return;const cols=Object.keys(rows[0]);const csv=[cols.join(","),...rows.map(r=>cols.map(c=>`"${r[c]??""}"`).join(","))].join("\n");const a=document.createElement("a");a.href=URL.createObjectURL(new Blob([csv],{type:"text/csv"}));a.download="gunluk.csv";a.click();}
document.getElementById("f").tarih.value=new Date().toISOString().slice(0,10);
render();
</script>
</body>
</html>
