
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Binance Real-Time Short Strategy - GitHub Pages</title>
  <style>
    :root {
      --bg: #0b0e11;
      --card: #1e2329;
      --text: #eaecef;
      --green: #0ecb81;
      --red: #f6465d;
      --accent: #f0b90b;
      --subtext: #848e9c;
    }

    body {
      font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell, sans-serif;
      background-color: var(--bg);
      color: var(--text);
      margin: 0;
      padding: 20px;
      display: flex;
      justify-content: center;
    }

    .container {
      max-width: 700px;
      width: 100%;
      display: flex;
      flex-direction: column;
      gap: 16px;
    }

    .card {
      background-color: var(--card);
      border-radius: 8px;
      padding: 16px;
      border: 1px solid #2b313a;
    }

    .header-ticker {
      display: flex;
      justify-content: space-between;
      align-items: center;
    }

    .price-large {
      font-size: 2rem;
      font-weight: bold;
    }

    .status-badge {
      padding: 6px 12px;
      border-radius: 4px;
      font-weight: bold;
      font-size: 0.85rem;
    }

    .status-active { background: rgba(14, 203, 129, 0.15); color: var(--green); }
    .status-short { background: rgba(246, 70, 93, 0.15); color: var(--red); }

    .targets-list {
      display: flex;
      flex-direction: column;
      gap: 8px;
      margin-top: 12px;
    }

    .target-item {
      display: flex;
      justify-content: space-between;
      align-items: center;
      padding: 10px 14px;
      background: #14181d;
      border-radius: 6px;
      border-left: 4px solid var(--subtext);
    }

    .target-item.hit {
      border-left-color: var(--green);
      background: rgba(14, 203, 129, 0.08);
    }

    .target-item.pending {
      border-left-color: var(--accent);
    }

    .log-box {
      background: #14181d;
      border-radius: 6px;
      padding: 12px;
      height: 180px;
      overflow-y: auto;
      font-family: monospace;
      font-size: 0.82rem;
      color: #b7bdc6;
    }

    .log-entry { margin-bottom: 4px; }
    .log-success { color: var(--green); }
    .log-warn { color: var(--accent); }
  </style>
</head>
<body>

<div class="container">
  <!-- Header / Preço Real -->
  <div class="card header-ticker">
    <div>
      <div style="color: var(--subtext); font-size: 0.9rem;">BTC/USDT Binance Live</div>
      <div class="price-large" id="live-price">Conectando...</div>
    </div>
    <div id="position-status" class="status-badge status-short">POSIÇÃO: SHORT ABERTO</div>
  </div>

  <!-- Alvos da Estratégia -->
  <div class="card">
    <h3 style="margin-top: 0; color: var(--accent);">Seqüência de Alvos (Alvo Fixo -> Pivô -> Extremos -> Suportes)</h3>
    <div class="targets-list" id="targets-container">
      <!-- Gerado via JS -->
    </div>
  </div>

  <!-- Log de Execução -->
  <div class="card">
    <h3 style="margin-top: 0; font-size: 1rem; color: var(--subtext);">Log de Execução em Tempo Real</h3>
    <div class="log-box" id="log-box"></div>
  </div>
</div>

<script>
  // Configuração dos Níveis Extraídos
  const TARGETS = [
    { id: 'ultimo', name: 'Último Fechamento Fixo', price: 77442.0, hit: false },
    { id: 'pivo', name: 'Preço Pivô', price: 77601.0, hit: false },
    { id: 'maxima', name: 'Máxima do Período', price: 78039.83, hit: false },
    { id: 'minima', name: 'Mínima do Período', price: 77783.31, hit: false },
    { id: 's1', name: 'Suporte 1', price: 75712.0, hit: false },
    { id: 's2', name: 'Suporte 2', price: 75299.0, hit: false },
    { id: 's3', name: 'Suporte 3', price: 74887.0, hit: false }
  ];

  let currentTargetIndex = 0;
  let entryPrice = null;

  function log(msg, type = '') {
    const logBox = document.getElementById('log-box');
    const time = new Date().toLocaleTimeString();
    const entry = document.createElement('div');
    entry.className = `log-entry ${type}`;
    entry.innerText = `[${time}] ${msg}`;
    logBox.appendChild(entry);
    logBox.scrollTop = logBox.scrollHeight;
  }

  function renderTargets() {
    const container = document.getElementById('targets-container');
    container.innerHTML = '';
    TARGETS.forEach((target, index) => {
      const div = document.createElement('div');
      let statusClass = 'pending';
      let statusText = 'Aguardando';

      if (target.hit) {
        statusClass = 'hit';
        statusText = 'FECHADO ✓';
      } else if (index === currentTargetIndex) {
        statusText = 'EM BUSCA...';
      }

      div.className = `target-item ${statusClass}`;
      div.innerHTML = `
        <div>
          <strong>${target.name}</strong><br>
          <small style="color: var(--subtext)">Alvo: $${target.price.toLocaleString('en-US', {minimumFractionDigits: 2})}</small>
        </div>
        <span style="font-weight: bold;">${statusText}</span>
      `;
      container.appendChild(div);
    });
  }

  function initWebSocket() {
    // Conexão WebSocket oficial da Binance (Preço em Tempo Real)
    const ws = new WebSocket('wss://stream.binance.com:9443/ws/btcusdt@ticker');

    ws.onopen = () => {
      log('Conectado ao WebSocket da Binance em tempo real.', 'log-success');
    };

    ws.onmessage = (event) => {
      const data = JSON.parse(event.data);
      const currentPrice = parseFloat(data.c);

      // Atualiza o preço na tela
      document.getElementById('live-price').innerText = `$${currentPrice.toLocaleString('en-US', {minimumFractionDigits: 2})}`;

      // Abertura imediata no primeiro tick
      if (!entryPrice) {
        entryPrice = currentPrice;
        log(`ORDEM EXECUTADA: Venda (Short) imediata ao preço atual de $${entryPrice.toFixed(2)}`, 'log-warn');
      }

      // Verificação de atingimento de alvo
      if (currentTargetIndex < TARGETS.length) {
        const target = TARGETS[currentTargetIndex];

        // Lógica de gatilho para Short (preço atinge ou cruza para baixo do alvo)
        if (currentPrice <= target.price) {
          target.hit = true;
          log(`ALVO ATINGIDO: ${target.name} em $${currentPrice.toFixed(2)} (Alvo: $${target.price})`, 'log-success');
          currentTargetIndex++;
          renderTargets();

          if (currentTargetIndex === TARGETS.length) {
            log('TODAS AS ORDENS E SUPORTES FORAM FINALIZADOS!', 'log-success');
            document.getElementById('position-status').innerText = 'POSIÇÃO: FINALIZADA';
            document.getElementById('position-status').className = 'status-badge status-active';
          }
        }
      }
    };

    ws.onerror = (error) => {
      log('Erro na conexão WebSocket.', 'log-warn');
    };
  }

  // Inicializa o painel ao carregar a página
  window.onload = () => {
    renderTargets();
    initWebSocket();
  };
</script>

</body>
</html>
