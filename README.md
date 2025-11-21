# guzelolsaydın.github.com
<!doctype html>
<html lang="tr">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Gizli Fotoğraf Bulmaca — Demo</title>
  <style>
    :root{--w:900px;--h:600px}
    body{font-family: Inter, system-ui, Arial;display:flex;align-items:center;justify-content:center;min-height:100vh;background:#0f172a;color:#e6eef8;margin:0}
    .wrap{width:clamp(320px,90vw,var(--w));max-width:100%;}
    .toolbar{display:flex;gap:.5rem;align-items:center;margin-bottom:.5rem}
    .toolbar button{padding:.45rem .6rem;border-radius:8px;border:0;background:#111827;color:#fff;cursor:pointer}
    .board{position:relative;aspect-ratio:var(--w)/var(--h);background:#111827;overflow:hidden;border-radius:12px;box-shadow:0 8px 30px rgba(2,6,23,.6)}
    .board img{width:100%;height:100%;object-fit:cover;display:block;user-select:none}
    svg.overlay{position:absolute;inset:0;width:100%;height:100%;pointer-events:none}
    .hud{position:absolute;left:8px;top:8px;background:rgba(0,0,0,.45);padding:.4rem .6rem;border-radius:8px;font-size:14px}
    .hint{position:absolute;right:8px;top:8px;background:rgba(0,0,0,.45);padding:.3rem .5rem;border-radius:8px;font-size:13px}
    .found-marker{stroke:#34d399;stroke-width:3;fill:rgba(52,211,153,0.08)}
    .pulse{animation:pulse 900ms ease-out}
    @keyframes pulse{from{opacity:1;transform:scale(.6)}to{opacity:0;transform:scale(1.6)}}
    .small{font-size:13px;color:#cbd5e1}
    .credits{margin-top:.5rem;font-size:13px;color:#93c5fd}
  </style>
</head>
<body>
  <div class="wrap">
    <div class="toolbar">
      <button id="resetBtn">Yeniden Başlat</button>
      <button id="showHintsBtn">İpuçlarını Göster (test)</button>
      <div style="margin-left:auto" class="small">Bulunan: <strong id="foundCount">0</strong>/<span id="totalCount">0</span></div>
    </div>

    <div class="board" id="board">
      <!-- Replace the image src with your base image that "gizli yüzleri" içerir -->
      <img id="mainImg" src="https://images.unsplash.com/photo-1503264116251-35a269479413?q=80&w=1600&auto=format&fit=crop&ixlib=rb-4.0.3&s=placeholder" alt="gizli bulmaca" />

      <!-- SVG overlay for drawing circles -->
      <svg class="overlay" id="overlay" viewBox="0 0 1000 1000" preserveAspectRatio="xMidYMid slice">
        <g id="markers"></g>
      </svg>

      <div class="hud" id="hud">Tıklayıp gizli fotoğrafı bul → Ses + daire</div>
      <div class="hint" id="hintArea" style="display:none">(İpuçları gösteriliyor)</div>
    </div>

    <div class="credits">Not: Fotoğrafı ve hotspot koordinatlarını aşağıdaki <code>hotspots</code> dizisinde değiştir. Koordinatlar % (0-100) cinsinden.</div>
  </div>

  <script>
    // ---------- AYARLAR (burayı değiştir) ----------
    // Ana görseli değiştirin: <img id="mainImg"> içinde src'yi değiştirin

    // hotspot dizisi: her obje x,y (yüzde 0-100) ve radius (yüzde, görsel genişliğinin yüzdesi olarak)
    const hotspots = [
      {id: 'yuz1', x: 22, y: 34, r: 6},
      {id: 'yuz2', x: 66, y: 42, r: 6},
      {id: 'yuz3', x: 48, y: 72, r: 5}
    ];

    // Ses ayarları: WebAudio ile kısa bir bip sesi oluşturuyoruz (harici dosya gerekmez)
    function playBeep(){
      try{
        const ctx = new (window.AudioContext || window.webkitAudioContext)();
        const o = ctx.createOscillator();
        const g = ctx.createGain();
        o.type = 'sine';
        o.frequency.value = 880;
        g.gain.value = 0.12;
        o.connect(g); g.connect(ctx.destination);
        o.start();
        o.frequency.exponentialRampToValueAtTime(1320, ctx.currentTime + 0.16);
        g.gain.exponentialRampToValueAtTime(0.001, ctx.currentTime + 0.22);
        setTimeout(()=>{try{o.stop();ctx.close()}catch(e){}},300);
      }catch(e){ console.warn('Ses oynatılamadı:', e) }
    }

    // ---------- OYUN KODU ----------
    const board = document.getElementById('board');
    const overlay = document.getElementById('overlay');
    const markersGroup = document.getElementById('markers');
    const foundCountEl = document.getElementById('foundCount');
    const totalCountEl = document.getElementById('totalCount');
    const hintArea = document.getElementById('hintArea');
    const showHintsBtn = document.getElementById('showHintsBtn');
    const resetBtn = document.getElementById('resetBtn');

    let found = new Set();
    totalCountEl.textContent = hotspots.length;

    // SVG koordinat sistemi: viewBox 0..1000 (kullanışlı yüzde dönüşümü için)
    function pctToView(xPct, yPct){
      return {x: xPct*10, y: yPct*10}; // yüzde -> 0..1000
    }

    function createCircleMarker(h){
      const {x,y} = pctToView(h.x,h.y);
      const r = h.r * 10; // r yüzde -> viewbox scale
      const g = document.createElementNS('http://www.w3.org/2000/svg','g');
      g.setAttribute('data-id',h.id);

      const circ = document.createElementNS('http://www.w3.org/2000/svg','circle');
      circ.setAttribute('cx',x); circ.setAttribute('cy',y); circ.setAttribute('r',r);
      circ.setAttribute('class','found-marker'); circ.style.opacity = 0; // başlangıçta görünmez

      // Pulse efekt için aynı konumda şeffaf bir daire daha
      const pulse = document.createElementNS('http://www.w3.org/2000/svg','circle');
      pulse.setAttribute('cx',x); pulse.setAttribute('cy',y); pulse.setAttribute('r',r);
      pulse.setAttribute('class','found-marker pulse'); pulse.style.opacity = 0;

      g.appendChild(pulse); g.appendChild(circ);
      markersGroup.appendChild(g);
      return {g,circ,pulse};
    }

    // Başlangıçta SVG içinde marker öğelerini oluştur
    const markerMap = new Map();
    hotspots.forEach(h=>{
      const m = createCircleMarker(h);
      markerMap.set(h.id,{data:h,el:m.g, circle:m.circ, pulse:m.pulse});
    });

    // Tıklama konumunu hotspotlarla karşılaştırma (görsel boyutu değişse de % ile çalışıyoruz)
    function handleClick(clientX, clientY){
      const rect = board.getBoundingClientRect();
      const relX = clientX - rect.left; const relY = clientY - rect.top;
      const xPct = (relX / rect.width) * 100; const yPct = (relY / rect.height) * 100;

      // hangi hotspot içindeyse onu bul
      for(const h of hotspots){
        if(found.has(h.id)) continue;
        const dx = xPct - h.x; const dy = yPct - h.y; const dist = Math.sqrt(dx*dx+dy*dy);
        if(dist <= h.r){
          // bulundu
          found.add(h.id);
          revealMarker(h.id);
          playBeep();
          updateHUD();
          return {found:true,id:h.id};
        }
      }
      // Bulunamadı
      flashMiss(xPct,yPct);
      return {found:false};
    }

    function revealMarker(id){
      const m = markerMap.get(id);
      if(!m) return;
      m.circle.style.transition = 'opacity .18s ease-in'; m.circle.style.opacity = 1;
      m.pulse.style.opacity = 1;
      // pulse kaldır
      setTimeout(()=>{try{m.pulse.style.opacity=0}catch(e){}},700);
    }

    function flashMiss(xPct,yPct){
      const pt = pctToView(xPct,yPct);
      const p = document.createElementNS('http://www.w3.org/2000/svg','circle');
      p.setAttribute('cx',pt.x); p.setAttribute('cy',pt.y); p.setAttribute('r',20);
      p.setAttribute('stroke','rgba(255,120,120,.95)'); p.setAttribute('fill','none'); p.setAttribute('stroke-width',3);
      p.style.opacity = 1; p.style.transition = 'opacity .9s ease-out, transform .9s ease-out';
      markersGroup.appendChild(p);
      setTimeout(()=>{p.style.opacity = 0; p.style.transform='translateY(-20px)';},30);
      setTimeout(()=>{try{markersGroup.removeChild(p)}catch(e){}},1000);
    }

    function updateHUD(){
      foundCountEl.textContent = found.size;
      if(found.size === hotspots.length){
        document.getElementById('hud').textContent = 'Tebrikler! Tüm gizli fotoğrafları buldun 🎉';
      }else{
        document.getElementById('hud').textContent = 'Bulunan: ' + found.size + '/' + hotspots.length;
      }
    }

    // Board tıklamalarını dinle
    board.addEventListener('click', (ev)=>{
      // Eğer kullanıcı bir link veya butona tıkladıysa engelle
      const tag = ev.target.tagName.toLowerCase();
      if(tag === 'button' || tag === 'a') return;
      handleClick(ev.clientX, ev.clientY);
    });

    // Touch desteği (mobil)
    board.addEventListener('touchend', (ev)=>{
      const t = ev.changedTouches[0];
      handleClick(t.clientX, t.clientY);
    });

    // İpuçlarını göster (test amaçlı) -> gerçekten şakaya başlamadan önce koordinatları buradan kontrol et
    let hintsShown = false;
    showHintsBtn.addEventListener('click', ()=>{
      hintsShown = !hintsShown; hintArea.style.display = hintsShown ? 'block' : 'none';
      showHintsBtn.textContent = hintsShown ? 'İpuçlarını Gizle' : 'İpuçlarını Göster (test)';
      // ipuçlarını SVG üzerinde göstermek -> küçük yarı saydam noktalar
      markersGroup.querySelectorAll('.hint-dot').forEach(n=>n.remove());
      if(hintsShown){
        hotspots.forEach(h=>{
          const {x,y} = pctToView(h.x,h.y);
          const dot = document.createElementNS('http://www.w3.org/2000/svg','circle');
          dot.setAttribute('cx',x); dot.setAttribute('cy',y); dot.setAttribute('r',6);
          dot.setAttribute('fill','rgba(255,255,255,.6)'); dot.setAttribute('class','hint-dot');
          markersGroup.appendChild(dot);
        });
      }
    });

    resetBtn.addEventListener('click', ()=>{
      found.clear();
      markersGroup.querySelectorAll('g').forEach(g=>{ // reset opacity
        g.querySelectorAll('circle').forEach(c=>c.style.opacity=0);
      });
      markersGroup.querySelectorAll('.hint-dot').forEach(n=>n.remove());
      hintArea.style.display = 'none'; hintsShown=false; showHintsBtn.textContent='İpuçlarını Göster (test)';
      document.getElementById('hud').textContent = 'Tıklayıp gizli fotoğrafı bul → Ses + daire';
      updateHUD();
    });

    // İlk HUD güncellemesi
    updateHUD();

    // İPUCU: gerçek oyunda hotspot koordinatlarını küçük fotoğrafların merkezlerine göre ayarla.
    // Örnek: {id:'yuz1', x:22, y:34, r:6} -> x,y = yüzde (görsel genişliğine göre), r = yüzdelik yarıçap
  </script>
</body>
</html>

