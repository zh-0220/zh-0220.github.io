<!doctype html>
<html lang="zh-Hant">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>簡易加班費計算器</title>
<style>
  :root { --fg:#111; --muted:#666; --brand:#0ea5e9; --bg:#f7f7f8; }
  body{ font-family: ui-sans-serif,system-ui,-apple-system,Segoe UI,Roboto,Noto Sans,Arial;
        margin:0; background:var(--bg); color:var(--fg);}
  .wrap{ max-width:760px; margin:24px auto; padding:16px; }
  .card{ background:#fff; border-radius:16px; box-shadow:0 1px 8px rgba(0,0,0,.06); padding:16px; margin-bottom:16px; }
  h1{ font-size:20px; margin:0 0 8px; }
  .row{ display:grid; grid-template-columns:1fr 2fr; gap:8px; align-items:center; margin:10px 0; }
  .row > label{ color:var(--muted); font-size:14px; }
  input, select{ width:100%; padding:10px 12px; border:1px solid #e5e7eb; border-radius:10px; font-size:14px; }
  input[readonly]{ background:#fafafa; }
  .muted{ color:var(--muted); font-size:12px; }
  .btn{ padding:10px 14px; border:0; border-radius:10px; background:var(--brand); color:#fff; cursor:pointer; font-size:14px; }
  .btn:active{ transform:translateY(1px); }
  .result{ font-size:18px; font-weight:700; color:#059669; }
  .warn{ color:#dc2626; font-size:13px; margin-top:6px; }
  .grid-2{ display:grid; grid-template-columns:1fr 1fr; gap:8px; }
</style>
</head>
<body>
  <div class="wrap">
    <div class="card">
      <h1>簡易加班費計算器（臺灣常見倍數）</h1>
      <div class="muted">此工具僅供估算，實際依公司規章與法規為準。</div>
    </div>

    <div class="card">
      <h2 style="margin:0 0 8px;font-size:16px;">基本資料</h2>

      <div class="row">
        <label>計算模式</label>
        <select id="mode">
          <option value="hourly">以時薪計算</option>
          <option value="monthly">以月薪換算時薪（÷240）</option>
        </select>
      </div>

      <div class="grid-2">
        <div class="row">
          <label>時薪（元）</label>
          <input id="hourly" type="number" min="0" value="200" />
        </div>
        <div class="row">
          <label>月薪（元）</label>
          <input id="monthly" type="number" min="0" value="42000" />
        </div>
      </div>
      <div class="row">
        <label>換算時薪</label>
        <input id="hourlyComputed" type="number" readonly value="200" />
      </div>
      <div class="muted">換算時薪 = 月薪 ÷ 240（30 天 × 8 小時）</div>
    </div>

    <div class="card">
      <h2 style="margin:0 0 8px;font-size:16px;">加班條件</h2>

      <div class="row">
        <label>日期類型</label>
        <select id="dayType">
          <option value="workday">工作日（延長工時）</option>
          <option value="restday">休息日</option>
          <option value="holiday">國定假日/特休日</option>
        </select>
      </div>

      <div class="row">
        <label>加班時數（可 0.5）</label>
        <input id="hours" type="number" min="0" step="0.5" value="2" />
      </div>

      <div class="grid-2">
        <div class="row">
          <label>工作/休息日前2小時倍率</label>
          <input id="first2x" type="number" step="0.01" value="1.34" />
        </div>
        <div class="row">
          <label>工作/休息日後續倍率</label>
          <input id="nextx" type="number" step="0.01" value="1.67" />
        </div>
      </div>

      <div class="row">
        <label>假日倍率（全時段）</label>
        <input id="holidayx" type="number" step="0.01" value="2.00" />
      </div>

      <div class="muted">簡化規則：工作日最多計 4 小時；休息日與假日最多計 8 小時（超出部分不計）。</div>
    </div>

    <div class="card">
      <div class="row">
        <label></label>
        <button class="btn" id="calcBtn">計算加班費</button>
      </div>
      <div class="row">
        <label>結果</label>
        <div>
          <div class="result" id="result">$0</div>
          <div class="muted" id="detail"></div>
          <div class="warn" id="warn"></div>
        </div>
      </div>
    </div>

    <div class="card">
      <div class="muted">
        計算說明：<br>
        工作/休息日：前 2 小時用「前2小時倍率」，之後用「後續倍率」。<br>
        假日：全時段用「假日倍率」。<br>
        規則各公司可能不同，請自行調整倍率。
      </div>
    </div>
  </div>

<script>
  const $ = (id) => document.getElementById(id);

  function monthlyToHourly(m){ return m / 240; }
  function money(n){ return Math.round(n).toLocaleString('zh-TW', { style:'currency', currency:'TWD', maximumFractionDigits:0 }); }

  function currentHourly(){
    const mode = $('mode').value;
    if(mode === 'hourly'){
      return Number($('hourly').value || 0);
    }else{
      const m = Number($('monthly').value || 0);
      return monthlyToHourly(m);
    }
  }

  function updateHourlyComputed(){
    const h = currentHourly();
    $('hourlyComputed').value = Math.round(h);
  }

  $('mode').addEventListener('change', updateHourlyComputed);
  $('hourly').addEventListener('input', ()=>{ if($('mode').value==='hourly') updateHourlyComputed(); });
  $('monthly').addEventListener('input', ()=>{ if($('mode').value==='monthly') updateHourlyComputed(); });

  updateHourlyComputed();

  $('calcBtn').addEventListener('click', ()=>{
    const hWage = currentHourly();
    const type = $('dayType').value; // workday | restday | holiday
    let hours = Number($('hours').value || 0);
    const first2x = Number($('first2x').value || 1.34);
    const nextx   = Number($('nextx').value   || 1.67);
    const holidayx= Number($('holidayx').value|| 2.0);

    let cap = 8;
    if(type === 'workday') cap = 4;

    const extra = Math.max(0, hours - cap);
    const billable = Math.min(hours, cap);

    let pay = 0, detail = '';

    if(type === 'holiday'){
      pay = billable * hWage * holidayx;
      detail = `假日 ${billable} 小時 × ${holidayx} 倍 × 時薪 ${Math.round(hWage)} 元`;
    }else{
      const first2 = Math.min(2, billable);
      const rest = Math.max(0, billable - 2);
      pay = first2 * hWage * first2x + rest * hWage * nextx;
      const label = (type==='workday'?'工作日':'休息日');
      detail = `${label} 前2小時：${first2}h × ${first2x} + 後續：${rest}h × ${nextx} × 時薪 ${Math.round(hWage)} 元`;
    }

    $('result').textContent = money(pay);
    $('detail').textContent = detail;
    $('warn').textContent = extra>0 ? `超出可計算上限 ${cap} 小時（額外 ${extra} 小時不計）。` : '';
  });
</script>
</body>
</html>
