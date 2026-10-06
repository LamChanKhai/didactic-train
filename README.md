<!doctype html>
<html lang="vi">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>CSV → Google Sheets</title>
  <style>
    :root { --ink:#17202a; --muted:#667085; --line:#d9dee7; --paper:#f5f7fa; --accent:#0f766e; --accent-dark:#115e59; }
    * { box-sizing:border-box; }
    body { margin:0; color:var(--ink); background:var(--paper); font:15px/1.5 system-ui,-apple-system,"Segoe UI",sans-serif; }
    main { max-width:1180px; margin:0 auto; padding:42px 24px 64px; }
    .eyebrow { color:var(--accent); font-size:12px; font-weight:800; letter-spacing:.14em; text-transform:uppercase; }
    h1 { margin:8px 0 6px; font-size:clamp(28px,4vw,48px); letter-spacing:-.04em; line-height:1.05; }
    .intro { max-width:710px; margin:0 0 26px; color:var(--muted); }
    .panel { background:#fff; border:1px solid var(--line); border-radius:16px; padding:20px; box-shadow:0 10px 30px #17202a0b; }
    .controls { display:flex; align-items:center; flex-wrap:wrap; gap:12px; }
    input[type=file] { max-width:100%; padding:10px; border:1px dashed #aeb8c5; border-radius:10px; background:#fbfcfd; }
    button { border:0; border-radius:9px; padding:10px 16px; color:#fff; background:var(--accent); font:inherit; font-weight:700; cursor:pointer; }
    button:hover { background:var(--accent-dark); }
    button:disabled { cursor:not-allowed; opacity:.45; }
    .status { margin:14px 0 0; color:var(--muted); min-height:24px; }
    .status.error { color:#b42318; font-weight:650; }
    .status.ok { color:var(--accent-dark); font-weight:650; }
    .table-wrap { margin-top:20px; overflow:auto; border:1px solid var(--line); border-radius:12px; background:#fff; }
    table { width:100%; min-width:850px; border-collapse:collapse; }
    th, td { padding:11px 13px; border-right:1px solid var(--line); border-bottom:1px solid var(--line); text-align:left; vertical-align:top; }
    th { color:#fff; background:#193b3a; font-size:12px; letter-spacing:.04em; text-transform:uppercase; white-space:nowrap; }
    td { white-space:pre-line; }
    tr:last-child td { border-bottom:0; }
    th:last-child, td:last-child { border-right:0; }
    .empty { padding:42px; text-align:center; color:var(--muted); }
    .hint { margin-top:16px; color:var(--muted); font-size:13px; }
    code { padding:2px 5px; border-radius:4px; background:#eef2f4; }
  </style>
</head>
<body>
  <main>
    <div class="eyebrow">Local CSV transformer</div>
    <h1>CSV → bảng để dán vào Google Sheets</h1>
    <p class="intro">Chọn CSV. Công cụ sẽ gộp các dòng theo <code>Name</code>, giữ <code>Risk</code>, thêm hai cột <code>N/A</code>, và đánh số từng cặp Host/IP trong một ô.</p>
 
    <section class="panel">
      <div class="controls">
        <input id="file" type="file" accept=".csv,text/csv">
        <button id="copy" type="button" disabled>Copy bảng có định dạng</button>
      </div>
      <div id="status" class="status">Chưa chọn file.</div>
      <div id="output" class="table-wrap"><div class="empty">Bảng kết quả sẽ xuất hiện ở đây.</div></div>
      <div class="hint">Sau khi copy, mở Google Sheets và paste bình thường. Nội dung xuống dòng trong cột Hosts sẽ được giữ trong cùng một ô.</div>
    </section>
  </main>
 
  <script>
    const fileInput = document.getElementById('file');
    const copyButton = document.getElementById('copy');
    const status = document.getElementById('status');
    const output = document.getElementById('output');
    const headers = ['Name', 'Risk', 'Temp1', 'Temp2', 'Hosts'];
    let resultRows = [];
 
    function parseCsv(text) {
      const rows = [];
      let row = [], cell = '', quoted = false;
      text = text.replace(/^\uFEFF/, '');
      for (let i = 0; i < text.length; i += 1) {
        const char = text[i], next = text[i + 1];
        if (char === '"' && quoted && next === '"') { cell += '"'; i += 1; }
        else if (char === '"') quoted = !quoted;
        else if (char === ',' && !quoted) { row.push(cell); cell = ''; }
        else if ((char === '\n' || char === '\r') && !quoted) {
          if (char === '\r' && next === '\n') i += 1;
          row.push(cell); cell = '';
          if (row.some(value => value.trim() !== '')) rows.push(row);
          row = [];
        } else cell += char;
      }
      if (cell || row.length) { row.push(cell); if (row.some(value => value.trim() !== '')) rows.push(row); }
      return rows;
    }
 
    function columnIndex(columns, wanted) {
      const target = wanted.toLowerCase().replace(/\s+/g, ' ').trim();
      return columns.findIndex(value => value.toLowerCase().replace(/\s+/g, ' ').trim() === target);
    }
 
    function render(rows) {
      const table = document.createElement('table');
      const thead = document.createElement('thead');
      const headRow = document.createElement('tr');
      headers.forEach(header => { const th = document.createElement('th'); th.textContent = header; headRow.appendChild(th); });
      thead.appendChild(headRow); table.appendChild(thead);
      const tbody = document.createElement('tbody');
      rows.forEach(row => {
        const tr = document.createElement('tr');
        row.forEach(value => { const td = document.createElement('td'); td.textContent = value; tr.appendChild(td); });
        tbody.appendChild(tr);
      });
      table.appendChild(tbody); output.replaceChildren(table);
    }
 
    fileInput.addEventListener('change', async () => {
      const file = fileInput.files[0];
      if (!file) return;
      try {
        const rows = parseCsv(await file.text());
        if (rows.length < 2) throw new Error('CSV không có dòng dữ liệu.');
        const sourceHeaders = rows[0];
        const indexes = Object.fromEntries(['Name', 'Risk', 'Host', 'IP address'].map(name => [name, columnIndex(sourceHeaders, name)]));
        const missing = Object.entries(indexes).filter(([, index]) => index < 0).map(([name]) => name);
        if (missing.length) throw new Error(`Thiếu cột bắt buộc: ${missing.join(', ')}.`);
 
        const groups = new Map();
        rows.slice(1).forEach(sourceRow => {
          const name = (sourceRow[indexes.Name] || '').trim();
          if (!name) return;
          if (!groups.has(name)) groups.set(name, { risk: (sourceRow[indexes.Risk] || '').trim(), hosts: [] });
          const group = groups.get(name);
          const host = (sourceRow[indexes.Host] || '').trim();
          const ip = (sourceRow[indexes['IP address']] || '').trim();
          const pair = [host, ip].filter(Boolean).join(' | ');
          if (pair && !group.hosts.includes(pair)) group.hosts.push(pair);
        });
        resultRows = [...groups].map(([name, group]) => [name, group.risk, 'N/A', 'N/A', group.hosts.map((pair, index) => `${index + 1}. ${pair}`).join('\n')]);
        if (!resultRows.length) throw new Error('Không tìm thấy dòng có Name.');
        render(resultRows); copyButton.disabled = false;
        status.className = 'status ok'; status.textContent = `Đã xử lý ${resultRows.length} Name từ ${file.name}.`;
      } catch (error) {
        resultRows = []; copyButton.disabled = true; status.className = 'status error'; status.textContent = error.message;
        output.innerHTML = '<div class="empty">Không tạo được bảng kết quả.</div>';
      }
    });
 
    copyButton.addEventListener('click', async () => {
      const table = output.querySelector('table');
      if (!table) return;
      const html = `<!doctype html><html><body>${table.outerHTML}</body></html>`;
      const plain = [headers, ...resultRows].map(row => row.map(value => String(value).replace(/\n/g, '\n')).join('\t')).join('\n');
      try {
        await navigator.clipboard.write([new ClipboardItem({ 'text/html': new Blob([html], { type: 'text/html' }), 'text/plain': new Blob([plain], { type: 'text/plain' }) })]);
        status.className = 'status ok'; status.textContent = 'Đã copy bảng có định dạng. Paste vào Google Sheets.';
      } catch {
        const area = document.createElement('textarea'); area.value = plain; document.body.appendChild(area); area.select(); document.execCommand('copy'); area.remove();
        status.className = 'status ok'; status.textContent = 'Đã copy dữ liệu. Nếu định dạng không đi theo, hãy paste vào vùng trống trong Google Sheets.';
      }
    });
  </script>
</body>
</html>