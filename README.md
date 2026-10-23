<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Kesit — Yapay zeka ile arka plan silici</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link href="https://fonts.googleapis.com/css2?family=Bricolage+Grotesque:wght@500;700;800&family=Instrument+Sans:wght@400;500;600&display=swap" rel="stylesheet">
<style>
:root{--bg:#eef0ec;--ink:#14231e;--mute:#5d6b65;--line:#cfd6d0;--card:#fbfcfa;--acc:#0f6b57;--acc2:#e8f3ee;--r:14px}
*{box-sizing:border-box}
body{margin:0;background:var(--bg);color:var(--ink);font:16px/1.55 'Instrument Sans',system-ui,sans-serif}
h1,h2{font-family:'Bricolage Grotesque',system-ui,sans-serif;margin:0}
header{max-width:1100px;margin:0 auto;padding:28px 20px 0;display:flex;justify-content:space-between;align-items:center}
.logo{font:800 24px 'Bricolage Grotesque',sans-serif;letter-spacing:-.02em}
.tag{font-size:14px;color:var(--mute)}
main{max-width:1100px;margin:0 auto;padding:32px 20px 60px}
h1{font-size:clamp(34px,6vw,64px);line-height:1.02;letter-spacing:-.03em;max-width:14em}
.lead{color:var(--mute);max-width:36em;margin:16px 0 28px}
#drop{border:2px dashed var(--acc);background:var(--acc2);border-radius:var(--r);padding:44px 20px;text-align:center;cursor:pointer;transition:background .15s,transform .15s}
#drop.over{background:#d3eadf;transform:scale(1.01)}
#drop b{font:700 20px 'Bricolage Grotesque',sans-serif;display:block;margin-bottom:4px}
#drop span{color:var(--mute);font-size:14px}
#status{margin:14px 0;font-size:14px;color:var(--mute);min-height:22px}
#list{display:grid;gap:18px}
.item{background:var(--card);border:1px solid var(--line);border-radius:var(--r);padding:14px;display:grid;grid-template-columns:1fr 1fr;gap:14px}
@media(max-width:700px){.item{grid-template-columns:1fr}}
.pane{position:relative;border-radius:10px;overflow:hidden;min-height:180px;display:flex;align-items:center;justify-content:center;background:#e4e8e4}
.pane img{display:block;max-width:100%;max-height:420px;object-fit:contain}
.pane.out{background-color:#fff;background-image:linear-gradient(45deg,#d9ddd9 25%,transparent 25%,transparent 75%,#d9ddd9 75%),linear-gradient(45deg,#d9ddd9 25%,transparent 25%,transparent 75%,#d9ddd9 75%);background-size:20px 20px;background-position:0 0,10px 10px}
.pane small{position:absolute;left:8px;top:8px;background:rgba(20,35,30,.8);color:#fff;padding:2px 9px;border-radius:99px;font-size:12px}
.bar{grid-column:1/-1;display:flex;flex-wrap:wrap;gap:10px;align-items:center;justify-content:space-between}
.meta{font-size:13px;color:var(--mute)}
.ctrl{display:flex;gap:8px;align-items:center;flex-wrap:wrap}
.sw{width:26px;height:26px;border-radius:50%;border:2px solid var(--line);cursor:pointer;padding:0}
.sw.on{border-color:var(--ink)}
.sw.tr{background:conic-gradient(#ccc 25%,#fff 0 50%,#ccc 0 75%,#fff 0) 0 0/10px 10px}
input[type=color]{width:30px;height:30px;border:0;background:none;padding:0;cursor:pointer}
button.btn{font:600 15px 'Instrument Sans',sans-serif;background:var(--acc);color:#fff;border:0;border-radius:10px;padding:10px 18px;cursor:pointer}
button.btn:hover{background:#0b5846}
:focus-visible{outline:3px solid var(--acc);outline-offset:2px}
.prog{height:6px;background:var(--line);border-radius:9px;overflow:hidden;grid-column:1/-1}
.prog i{display:block;height:100%;width:0;background:var(--acc);transition:width .2s}
.note{margin-top:36px;font-size:13px;color:var(--mute);border-top:1px solid var(--line);padding-top:16px}
</style>
</head>
<body>
<header><div class="logo">Kesit</div><div class="tag">Sınırsız · Ücretsiz · Cihazında çalışır</div></header>
<main>
<h1>Arka planı saniyeler içinde sil.</h1>
<p class="lead">Fotoğraflarını yükle, yapay zeka konuyu ayırsın. Sonuç, orijinal çözünürlükte şeffaf PNG olarak iner. Hiçbir dosya sunucuya gönderilmez, yükleme sınırı yoktur.</p>

<div id="drop" tabindex="0" role="button" aria-label="Fotoğraf seç">
<b>Fotoğrafları buraya bırak</b><span>veya tıklayıp seç · JPG, PNG, WebP · birden fazla dosya olabilir</span>
<input id="file" type="file" accept="image/*" multiple hidden>
</div>
<div id="status"></div>
<div id="list"></div>
<p class="note">İlk kullanımda yapay zeka modeli (yaklaşık 80 MB) bir kez indirilir ve tarayıcıda saklanır. Sonraki işlemler çok daha hızlıdır. İşlem tarayıcıda gerçekleştiği için hız cihazına bağlıdır.</p>
</main>

<script type="module">
import { removeBackground } from "https://cdn.jsdelivr.net/npm/@imgly/background-removal@1.5.5/+esm";

const $ = s => document.querySelector(s);
const drop = $('#drop'), fileIn = $('#file'), list = $('#list'), statusEl = $('#status');
let busy = false; const queue = [];

drop.onclick = () => fileIn.click();
drop.onkeydown = e => { if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); fileIn.click(); } };
fileIn.onchange = () => add(fileIn.files);
['dragenter','dragover'].forEach(t => drop.addEventListener(t, e => { e.preventDefault(); drop.classList.add('over'); }));
['dragleave','drop'].forEach(t => drop.addEventListener(t, e => { e.preventDefault(); drop.classList.remove('over'); }));
drop.addEventListener('drop', e => add(e.dataTransfer.files));
addEventListener('paste', e => { const f = [...(e.clipboardData?.files||[])]; if (f.length) add(f); });

function add(files){
  [...files].filter(f => f.type.startsWith('image/')).forEach(f => {
    const el = mk(f); list.prepend(el.root); queue.push({f, el});
  });
  fileIn.value = ''; run();
}

function mk(f){
  const url = URL.createObjectURL(f);
  const root = document.createElement('div'); root.className = 'item';
  root.innerHTML = `
    <div class="pane"><small>Orijinal</small><img src="${url}" alt=""></div>
    <div class="pane out"><small>Sonuç</small><img alt="" hidden></div>
    <div class="prog"><i></i></div>
    <div class="bar">
      <div class="meta">${f.name.replace(/</g,'&lt;')} · sırada bekliyor</div>
      <div class="ctrl" hidden>
        <button class="sw tr on" data-c="" title="Şeffaf" aria-label="Şeffaf arka plan"></button>
        <button class="sw" data-c="#ffffff" style="background:#fff" title="Beyaz" aria-label="Beyaz"></button>
        <button class="sw" data-c="#000000" style="background:#000" title="Siyah" aria-label="Siyah"></button>
        <input type="color" value="#0f6b57" title="Özel renk" aria-label="Özel arka plan rengi">
        <button class="btn dl">PNG indir</button>
      </div>
    </div>`;
  return {root, url, name: f.name.replace(/\.[^.]+$/, ''),
    pane: root.querySelector('.out img'), prog: root.querySelector('.prog i'),
    meta: root.querySelector('.meta'), ctrl: root.querySelector('.ctrl')};
}

async function run(){
  if (busy) return; busy = true;
  while (queue.length){
    const {f, el} = queue.shift();
    statusEl.textContent = queue.length ? `İşleniyor… (${queue.length} dosya sırada)` : 'İşleniyor…';
    el.meta.textContent = 'İşleniyor…';
    try{
      const blob = await removeBackground(f, {
        model: 'isnet',
        output: { format: 'image/png', quality: 1 },
        progress: (key, cur, tot) => {
          if (tot) el.prog.style.width = Math.round(cur/tot*100) + '%';
          if (key.startsWith('fetch')) el.meta.textContent = 'Model indiriliyor (yalnızca ilk seferde)…';
          else if (key.startsWith('compute')) el.meta.textContent = 'Konu ayrılıyor…';
        }
      });
      el.blob = blob; el.cur = blob;
      el.pane.src = URL.createObjectURL(blob); el.pane.hidden = false;
      el.prog.style.width = '100%';
      const im = await createImageBitmap(blob);
      el.w = im.width; el.h = im.height; im.close();
      el.meta.textContent = `${el.w} × ${el.h} px · orijinal çözünürlük`;
      el.ctrl.hidden = false; wire(el);
    }catch(err){
      console.error(err);
      el.meta.textContent = 'İşlem başarısız. Sayfa internete bağlı mı? Dosyayı tekrar dene.';
    }
  }
  statusEl.textContent = 'Hazır. Yeni fotoğraf ekleyebilirsin.'; busy = false;
}

function wire(el){
  const sw = el.ctrl.querySelectorAll('.sw'), picker = el.ctrl.querySelector('input');
  const paint = async color => {
    if (!color){ el.cur = el.blob; el.pane.src = URL.createObjectURL(el.blob); return; }
    const img = await createImageBitmap(el.blob);
    const c = document.createElement('canvas'); c.width = el.w; c.height = el.h;
    const x = c.getContext('2d'); x.fillStyle = color; x.fillRect(0,0,c.width,c.height); x.drawImage(img,0,0);
    el.cur = await new Promise(r => c.toBlob(r, 'image/png')); el.pane.src = URL.createObjectURL(el.cur);
  };
  sw.forEach(b => b.onclick = () => { sw.forEach(s => s.classList.remove('on')); b.classList.add('on'); paint(b.dataset.c); });
  picker.oninput = () => { sw.forEach(s => s.classList.remove('on')); paint(picker.value); };
  el.ctrl.querySelector('.dl').onclick = () => {
    const a = document.createElement('a'); a.href = URL.createObjectURL(el.cur);
    a.download = el.name + '-kesit.png'; a.click();
  };
}
</script>
</body>
</html>
