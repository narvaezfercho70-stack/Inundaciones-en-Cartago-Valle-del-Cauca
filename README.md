<html lang="es">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Base de Datos Noticias (2010-2025)</title>

  <!-- Chart.js -->
  <script src="https://cdn.jsdelivr.net/npm/chart.js@3.9.1/dist/chart.min.js"></script>

  <style>
    :root{--blue:#3498db;--muted:#6c757d}
    body{font-family:Inter,system-ui,Arial,Helvetica,sans-serif;margin:0;background:#f6f8fb;color:#222}
    .container{max-width:1100px;margin:18px auto;padding:18px}
    header{text-align:center;margin-bottom:14px}
    h1{margin:.2rem 0;color:#2b3a42}
    p.small{color:var(--muted);margin:0 0 12px}
    .grid{display:grid;gap:16px}
    @media(min-width:900px){.grid{grid-template-columns:1fr 380px}}
    .card{background:#fff;padding:14px;border-radius:10px;box-shadow:0 6px 18px rgba(12,35,60,0.06)}
    table{width:100%;border-collapse:collapse;margin-top:10px}
    th,td{padding:8px;border-bottom:1px solid #eee;text-align:left;font-size:14px}
    th{background:var(--blue);color:#fff}
    .controls{display:flex;gap:8px;flex-wrap:wrap;margin-top:10px}
    button{background:var(--blue);color:#fff;border:0;padding:8px 10px;border-radius:7px;cursor:pointer}
    input,textarea,select{width:100%;padding:8px;border-radius:6px;border:1px solid #ddd;margin-bottom:8px}
    .chart-wrap{height:320px}
    .small-note{font-size:13px;color:#666}
    .muted{color:#666;font-size:13px}
    a.link{color:var(--blue)}
    .legend { display:flex; flex-wrap:wrap; gap:8px; margin-top:8px; }
    .legend-item { display:flex; align-items:center; gap:6px; font-size:13px; }
    .swatch { width:14px; height:14px; border-radius:3px; display:inline-block; }
  </style>
</head>
<body>
  <div class="container">
    <header>
      <h1>Base de Datos Noticias (2010-2025)</h1>
      <p class="small">Inundaciones en Cartago, base de datos de noticias y visualización de distribución por año (2010–2025).</p>
    </header>

    <div class="grid">
      <!-- main -->
      <div>
        <div class="card chart-wrap">
          <canvas id="chart"></canvas>
        </div>

        <div class="card" style="margin-top:12px">
          <div style="display:flex;justify-content:space-between;align-items:center">
            <strong>Registros</strong>
            <div class="controls">
              <button id="btnDownloadChart">Descargar gráfico</button>
              <button id="btnLoadSheet">Cargar desde Google Sheets</button>
            </div>
          </div>

          <table id="table" aria-live="polite">
            <thead>
              <tr><th>Año</th><th>Título</th><th>Fuente</th><th>Enlace</th></tr>
            </thead>
            <tbody></tbody>
          </table>

          <div id="legend" class="legend" aria-hidden="false" style="margin-top:10px"></div>
        </div>
      </div>

      <!-- sidebar -->
      <aside>
        <div class="card">
          <strong>Añadir noticia</strong>
          <form id="formAdd" style="margin-top:8px">
            <input id="fieldYear" type="number" placeholder="Año (2010 - 2025)" required />
            <input id="fieldTitle" type="text" placeholder="Título de la noticia" required />
            <input id="fieldSource" type="text" placeholder="Fuente (ej: El Tiempo)" />
            <input id="fieldLink" type="url" placeholder="Enlace (https://...)" />
            <button type="submit">Añadir</button>
          </form>

          <hr style="margin:12px 0" />

          <div>
            <div class="muted">Google Sheets (CSV público)</div>
            <input id="sheetUrl" value="https://docs.google.com/spreadsheets/d/e/2PACX-1vR6INDiwVFalUs0QvGpfprhgH6vRICZHSkc-XC6GhILtrlpyZ6hI5MoRtYd-NWRqRPA5xypB4bCVwxm/pub?output=csv" />
            <div class="small-note">El enlace debe terminar en <code>?output=csv</code>. Pulsa "Cargar desde Google Sheets".</div>

            <hr style="margin:10px 0" />

            <div style="display:flex;gap:8px;margin-top:8px">
              <button id="btnClearLocal">Borrar datos locales</button>
            </div>
            <div class="small-note" style="margin-top:8px">Los registros añadidos manualmente se guardan en este equipo (localStorage) y se mezclarán con los de la hoja cuando la cargues.</div>
          </div>
        </div>
      </aside>
    </div>
  </div>

<script>
(function(){
  // keys
  const KEY_NEWS = 'bdnoticias_news_v1';
  const KEY_COUNTS = 'bdnoticias_counts_v1';
  const KEY_COLORS = 'bdnoticias_colors_v1';

  // DOM
  const tbody = document.querySelector('#table tbody');
  const chartCtx = document.getElementById('chart').getContext('2d');
  const form = document.getElementById('formAdd');
  const fieldYear = document.getElementById('fieldYear');
  const fieldTitle = document.getElementById('fieldTitle');
  const fieldSource = document.getElementById('fieldSource');
  const fieldLink = document.getElementById('fieldLink');
  const sheetUrlInput = document.getElementById('sheetUrl');
  const btnLoadSheet = document.getElementById('btnLoadSheet');
  const btnDownloadChart = document.getElementById('btnDownloadChart');
  const btnClearLocal = document.getElementById('btnClearLocal');
  const legendDiv = document.getElementById('legend');

  // state
  let savedNews = load(KEY_NEWS) || [];
  let newsCounts = load(KEY_COUNTS) || {};
  // mapping year->color (string hex)
  let colorMap = load(KEY_COLORS) || {};

  // pre-fill colorMap for 2010-2025 if missing using deterministic pastel generator
  for(let y=2010; y<=2025; y++){
    const s = String(y);
    if(!colorMap[s]) colorMap[s] = generateDeterministicColor(s);
  }
  save(KEY_COLORS, colorMap);

  // create chart
  const chart = new Chart(chartCtx, {
    type: 'bar',
    data: { labels: Object.keys(newsCounts), datasets: [{ label:'Número de noticias', data: Object.values(newsCounts), backgroundColor: generateColors(Object.keys(newsCounts)) }] },
    options: { responsive:true, maintainAspectRatio:false, plugins:{ legend:{display:false}, tooltip:{mode:'index',intersect:false}, title:{display:true,text:'Base de Datos Noticias — registros por año'} }, scales:{ x:{beginAtZero:true}, y:{beginAtZero:true,precision:0} } }
  });

  // helpers: local storage
  function load(k){ try{ const v = localStorage.getItem(k); return v? JSON.parse(v): null; }catch(e){ return null; } }
  function save(k,v){ try{ localStorage.setItem(k, JSON.stringify(v)); }catch(e){ console.warn('No se pudo guardar', e); } }

  // deterministic pastel color for a string (so same year => same color)
  function generateDeterministicColor(year){
  const baseHue = 15;     // hue inicial
  const step = 23;        // separación entre colores (garantiza que no se repitan)
  const hue = (baseHue + (Number(year) - 2010) * step) % 360;
  return hslToHex(hue, 65, 65);  // pastel bonito
}
  function hslToHex(h,s,l){
    s/=100; l/=100;
    const k = n => (n + h/30) % 12;
    const a = s * Math.min(l, 1 - l);
    const f = n => l - a * Math.max(Math.min(k(n) - 3, 9 - k(n), 1), -1);
    const toHex = x => Math.round(x * 255).toString(16).padStart(2,'0');
    return `#${toHex(f(0))}${toHex(f(8))}${toHex(f(4))}`;
  }

  // generate colors array for labels (ensures colorMap contains each year)
  function generateColors(labels){
    return labels.map(label => {
      if(!colorMap[label]){
        colorMap[label] = generateDeterministicColor(label);
        save(KEY_COLORS, colorMap);
      }
      return colorMap[label];
    });
  }

  // render legend
  function renderLegend(){
    legendDiv.innerHTML = '';
    const labels = Object.keys(newsCounts);
    labels.forEach(lbl=>{
      const item = document.createElement('div');
      item.className = 'legend-item';
      const sw = document.createElement('span');
      sw.className = 'swatch';
      sw.style.background = colorMap[lbl] || '#ccc';
      const txt = document.createElement('span');
      txt.textContent = `${lbl} — ${newsCounts[lbl] || 0}`;
      item.appendChild(sw);
      item.appendChild(txt);
      legendDiv.appendChild(item);
    });
  }

  // update chart dataset/colors and legend
  function refreshChart(){
    const labels = Object.keys(newsCounts).sort();
    chart.data.labels = labels;
    chart.data.datasets[0].data = labels.map(l => newsCounts[l] || 0);
    chart.data.datasets[0].backgroundColor = generateColors(labels);
    chart.update();
    renderLegend();
    save(KEY_NEWS, savedNews);
    save(KEY_COUNTS, newsCounts);
    save(KEY_COLORS, colorMap);
  }

  // render table
  function renderTable(){
    tbody.innerHTML = '';
    // sort savedNews by createdAt ascending
    const sorted = savedNews.slice().sort((a,b)=> new Date(a.createdAt) - new Date(b.createdAt));
    sorted.forEach(item=>{
      const tr = document.createElement('tr');
      tr.innerHTML = `<td>${escapeHtml(item.year)}</td><td>${escapeHtml(item.title)}</td><td>${escapeHtml(item.source||'')}</td><td>${item.link? '<a href="'+escapeAttr(item.link)+'" target="_blank">Ver</a>':''}</td>`;
      tbody.appendChild(tr);
    });
  }

  // add entry form
  form.addEventListener('submit', (e)=>{
    e.preventDefault();
    const year = String(fieldYear.value).trim();
    const title = (fieldTitle.value||'').trim();
    if(!year || !title){ alert('Año y título son obligatorios'); return; }
    const yearNum = parseInt(year, 10);
    if(isNaN(yearNum) || yearNum < 2010 || yearNum > 2025){ alert('El año debe estar entre 2010 y 2025'); return; }
    const source = (fieldSource.value||'').trim();
    const link = (fieldLink.value||'').trim();
    const entry = { year:String(yearNum), title, source, link, createdAt: new Date().toISOString() };
    savedNews.push(entry);
    newsCounts[entry.year] = (newsCounts[entry.year]||0) + 1;
    fieldYear.value = ''; fieldTitle.value = ''; fieldSource.value = ''; fieldLink.value = '';
    renderTable();
    refreshChart();
  });

  // load from Google Sheets (CSV public)
  btnLoadSheet.addEventListener('click', ()=>{
    const url = (sheetUrlInput.value||'').trim();
    if(!url){ alert('Pega el enlace CSV público de Google Sheets'); return; }
    loadFromSheet(url);
  });

  async function loadFromSheet(url){
    try{
      const res = await fetch(url);
      if(!res.ok) throw new Error('HTTP '+res.status);
      const text = await res.text();
      parseCsvAndMerge(text);
    }catch(err){
      alert('Error cargando la hoja: '+err.message + '. Si el problema persiste, abre la consola del navegador y copia el error.');
      console.error(err);
    }
  }

  function parseCsvAndMerge(text){
    const lines = text.split(/\r?\n/).map(l=>l.trim()).filter(Boolean);
    if(lines.length < 2){ alert('CSV vacío o sin filas.'); return; }
    // split respecting quoted commas
    const rows = lines.map(l => l.split(/,(?=(?:[^"]*"[^"]*")*[^"]*$)/).map(c => c.replace(/^"|"$/g,'')));
    const header = rows.shift().map(h => h.toLowerCase().normalize('NFD').replace(/[\u0300-\u036f]/g,''));
    const idxYear = header.findIndex(h => /ano|año|year/.test(h));
    const idxTitle = header.findIndex(h => /titulo|title/.test(h));
    const idxSource = header.findIndex(h => /fuente|source/.test(h));
    const idxLink = header.findIndex(h => /enlace|link|url/.test(h));

    let added = 0;
    rows.forEach(r=>{
      const year = idxYear>=0 ? (r[idxYear]||'').trim() : (r[0]||'').trim();
      const title = idxTitle>=0 ? (r[idxTitle]||'').trim() : (r[1]||'').trim();
      const source = idxSource>=0 ? (r[idxSource]||'').trim() : (r[2]||'').trim();
      const link = idxLink>=0 ? (r[idxLink]||'').trim() : (r[3]||'').trim();
      if(!year || !title) return;
      const yearNum = parseInt(year, 10);
      if(isNaN(yearNum) || yearNum < 2010 || yearNum > 2025) return; // ignore out-of-range years
      // avoid duplicates: same year + title
      const exists = savedNews.some(s => s.year === String(yearNum) && s.title === title);
      if(!exists){
        const entry = { year:String(yearNum), title, source, link, createdAt: new Date().toISOString() };
        savedNews.push(entry);
        newsCounts[entry.year] = (newsCounts[entry.year]||0) + 1;
        added++;
      }
    });
    if(added > 0){
      renderTable();
      refreshChart();
      alert('Se añadieron ' + added + ' registros desde la hoja.');
    } else {
      alert('No se agregaron registros nuevos desde la hoja (posible duplicación o años fuera de 2010-2025).');
    }
  }

  // download chart image
  btnDownloadChart.addEventListener('click', ()=>{
    try{
      const url = chart.toBase64Image();
      const a = document.createElement('a'); a.href = url; a.download = 'grafico_noticias.png'; a.click();
    }catch(e){ alert('Error generando imagen: '+e.message); }
  });

  // clear local
  btnClearLocal.addEventListener('click', ()=>{
    if(confirm('Borrar todos los datos locales? (esto no afecta la hoja de Google)')){
      savedNews = []; newsCounts = {}; colorMap = {}; // reset all
      // reinitialize colors for 2010-2025
      for(let y=2010; y<=2025; y++){ colorMap[String(y)] = generateDeterministicColor(String(y)); }
      save(KEY_COLORS, colorMap);
      renderTable(); refreshChart();
    }
  });

  // helpers
  function escapeHtml(s){ if(!s) return ''; return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); }
  function escapeAttr(s){ if(!s) return ''; return String(s).replace(/"/g,'%22'); }

  // initial render
  renderTable();
  refreshChart();

})();
</script>
</body>
</html>

