# MindRift — API de Integração para UI

> Documentação para desenvolvedores front-end que querem consumir os dados do MindRift.

---

## Visão Geral

O MindRift é um coach de League of Legends em tempo real. Ele lê dados do jogo via Riot Live Client API (localhost:2999) e processa 36 engines de análise em paralelo para emitir sinais de coaching via TTS.

A `mindrift_api_server.py` expõe todos esses dados via REST + WebSocket para que uma UI externa possa consumi-los.

```
Riot API (localhost:2999, 20Hz)
    ↓
MindRift Backend (Python)
    ↓
mindrift_api_server.py (FastAPI, porta 8000)
    ↓ REST / WebSocket
Sua UI (React, Vue, Figma prototipado, etc.)
```

---

## Instalação e execução

### Demo standalone (sem jogo ativo, dados mockados)
```bash
pip install fastapi uvicorn
python mindrift_api_server.py
# → http://localhost:8000
# → http://localhost:8000/docs   (Swagger interativo)
# → http://localhost:8000/demo  (UI demo visual)
```

### Integrado ao MindRift (dados reais durante uma partida)
Em `main.py`, adicione após criar o SharedContext:
```python
from mindrift_api_server import launch_api_thread, set_context
set_context(ctx)          # injeta referência ao SharedContext
launch_api_thread()       # inicia servidor em thread daemon na porta 8000
```

---

## Endpoints REST

Base URL: `http://localhost:8000`

### `GET /`
Status e mapa de endpoints.
```json
{
  "status": "ok",
  "mode": "live",
  "version": "1.0",
  "docs": "/docs",
  "demo_ui": "/demo"
}
```

---

### `GET /game`
Estado atual do jogo.
```json
{
  "game_active": true,
  "mode": "live",
  "phase": "MID",
  "game_time_s": 847.2,
  "game_time_display": "14:07",
  "champion": "Jinx",
  "player": {
    "hp": 1240,
    "max_hp": 1850,
    "hp_pct": 0.670,
    "gold": 3200,
    "cs": 112,
    "cs_per_min": 7.9,
    "kills": 4,
    "deaths": 1,
    "assists": 3,
    "is_dead": false,
    "level": 11,
    "team": "ORDER"
  }
}
```

**`phase`** pode ser: `"EARLY"` | `"MID"` | `"LATE"`  
**`team`** pode ser: `"ORDER"` | `"CHAOS"`

---

### `GET /signals?limit=10`
Sinais de coaching recentes. `limit` padrão: 10.
```json
{
  "signals": [
    {
      "type": "vision_ward_timing",
      "severity": 6,
      "intent": "WARD",
      "message": "Hora de rewardar a jungle. Dragon spawn em 47s.",
      "timestamp": "14:05",
      "source": "vision_engine"
    }
  ],
  "count": 5
}
```

**`severity`**: 1–10 (1=info, 5=aviso, 8+=crítico)  
**`intent`**: `PLAY_SAFE` | `RETREAT` | `PUSH_WAVE` | `ROAM` | `TAKE_OBJECTIVE` | `GROUP_FIGHT` | `SPLIT_PUSH` | `RECALL` | `WARD` | `IDLE`

---

### `GET /profile`
Perfil histórico do jogador (acumulado entre partidas).
```json
{
  "total_games": 23,
  "cs_avg_pm": 6.8,
  "cs_best_pm": 9.1,
  "deaths_avg_per_game": 4.2,
  "avg_pct_dead": 12.3,
  "most_played_champion": "Jinx",
  "top_death_causes": [
    { "cause": "isolamento", "count": 14 },
    { "cause": "fog_of_war", "count": 8 }
  ],
  "weaknesses": ["farm_consistencia", "morte_excessiva"],
  "cs_trend": "improving",
  "death_rate_trend": "stable",
  "last_updated": "2026-05-25 20:14"
}
```

---

### `GET /objectives`
Timers dos grandes objetivos.
```json
{
  "dragon":  { "spawn_in_s": 47,   "label": "Dragon",  "urgent": true  },
  "baron":   { "spawn_in_s": null, "label": "Baron",   "urgent": false },
  "herald":  { "spawn_in_s": null, "label": "Herald",  "urgent": false }
}
```
`spawn_in_s: null` = objetivo não disponível ainda.

---

### `GET /enemies`
Inimigos sumidos do mapa.
```json
{
  "missing": [
    { "champion": "Zed",     "missing_for_s": 12, "last_seen": "mid lane" },
    { "champion": "Lee Sin", "missing_for_s": 28, "last_seen": "jungle top" }
  ]
}
```

---

## WebSocket

**URL:** `ws://localhost:8000/ws`

O servidor envia um payload completo a cada **1 segundo**:

```json
{
  "event": "state_update",
  "timestamp": "2026-05-25T20:14:33.821",
  "game": { /* mesmo payload do GET /game */ },
  "signals": [ /* últimos 5 sinais */ ],
  "objectives": { /* mesmo payload do GET /objectives */ },
  "missing": [ /* mesmo payload do GET /enemies */ ]
}
```

### Exemplo de uso (JavaScript)
```javascript
const ws = new WebSocket('ws://localhost:8000/ws');

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  
  // Dados do jogo
  const { phase, champion, game_time_display, player } = data.game;
  
  // Novo sinal de coaching
  const latestSignal = data.signals[data.signals.length - 1];
  if (latestSignal) {
    console.log(`[${latestSignal.severity}] ${latestSignal.message}`);
  }
  
  // Objetivo chegando
  const dragon = data.objectives.dragon;
  if (dragon.urgent) {
    console.log(`Dragon em ${dragon.spawn_in_s}s!`);
  }
};
```

---

## Estrutura de dados dos sinais

Cada sinal tem uma **severidade** (1–10) e uma **intent** que descreve a ação recomendada:

| Intent | Significado |
|---|---|
| `PLAY_SAFE` | Jogar conservador, evitar trades |
| `RETREAT` | Recuar imediatamente |
| `PUSH_WAVE` | Empurrar a wave |
| `ROAM` | Rotacionar para outra lane |
| `TAKE_OBJECTIVE` | Ir ao objetivo |
| `GROUP_FIGHT` | Agrupar com o time |
| `SPLIT_PUSH` | Dividir pressão no mapa |
| `RECALL` | Resetar base |
| `WARD` | Colocar ward / visão |
| `IDLE` | Nenhuma ação específica |

### Cores recomendadas por severidade
```
1–2  →  #8b949e  (cinza, informativo)
3–4  →  #58a6ff  (azul, dica)
5–6  →  #d29922  (amarelo, aviso)
7–8  →  #db6d28  (laranja, importante)
9–10 →  #f85149  (vermelho, crítico)
```

---

## Engines ativos (fontes dos sinais)

| Engine | Sinais emitidos |
|---|---|
| `vision_engine` | Ward timing, facecheck risk, map clear |
| `matchup_engine` | Power spike warnings, dicas por fase |
| `fundamentals_engine` | CS/min, gestão de gold, death timer |
| `composition_engine` | Análise da comp inimiga |
| `recall_tracker_engine` | Janelas de push, rotações inimigas |
| `wave_engine` | Gerenciamento de minion wave |
| `objective_timer_engine` | Contagem regressiva Dragon/Baron/Herald |
| `threat_engine` | Ameaça imediata de inimigo |
| `death_recap_engine` | Análise de causa de morte |
| `economy_engine` | Sugestão de item |
| `win_condition_engine` | Elder, Barão, siege, avoid open fights |
| `jungle_invasion_engine` | Invasão de jungle detectada |
| `heuristic_macro_engine` | Rotações, split push, item spikes |
| `player_state_engine` | Gold spike, estado do jogador |
| `map_engine` | Awareness de mapa |
| + 21 mais | ... |

---

## Notas para o desenvolvedor UI

- O servidor inicia na porta **8000** por padrão (configurável)
- CORS está habilitado para `*` — qualquer origem pode consumir
- Swagger interativo disponível em `/docs` (útil para explorar payloads)
- A UI demo (`demo_ui.html`) se conecta ao WebSocket automaticamente e atualiza a UI quando o servidor real está disponível
- Em modo demo (sem jogo), todos os dados são mockados com valores realistas

