# Quadro Geral de Triagem

| Categoria | Item Validado | Quem Apontou | Onde Entra | Impacto Prático |
| --- | --- | --- | --- | --- |
| **1 — Catastrófico** | Filtro exato do erro `-2010` e Stop Pendurado | ChatGPT e DeepSeek | Bloco 5 (`oco_order_manager`) | Evita vender errado ou ficar sem stop |
| **1 — Catastrófico** | Arredondamento `close_time` (+1ms) no `merge_asof` | DeepSeek | Bloco 1 (`candle_collector`) | Evita perder a vela de 4h recém-fechada |
| **1 — Catastrófico** | Tratamento de `NaN` e divisão por zero no `CHOP` | ChatGPT | Bloco 2 (`indicators.py`) | Impede que o robô trave em moeda parada |
| **1 — Catastrófico** | Backup SQLite via `sqlite3.backup()` nativo | ChatGPT | Bloco 1 (`database.py`) | Evita arquivo de backup corrompido |
| **1 — Catastrófico** | Cálculo empírico de $\mathbb{E}[R]$ com Time-Stop no Gate 3 | ChatGPT e DeepSeek | Bloco 3 (Colab) | Mede o lucro real de quem sai por tempo |
| **2 — Refinamento** | Mock de `/sapi/*` na Testnet e Permissão `Universal Transfer` | ChatGPT e DeepSeek | Bloco 5 e `.env` | Testnet não aceita `/sapi/*` (evita erro 404) |
| **2 — Refinamento** | *High-Water Mark* (Topo Histórico) no Cofre de Lucros | ChatGPT | Bloco 6 (`profit_splitter`) | Só tira 30% quando recuperar prejuízos |
| **2 — Refinamento** | Teto de ordem em % da banca (`55%`) e BNB em Dólar (`$5`) | ChatGPT e DeepSeek | Bloco 4 (`strategies.yaml`) | Protege bancas pequenas desde o dia 1 |
| **2 — Refinamento** | Direção de arredondamento do `tickSize` na OCO | ChatGPT | Bloco 5 (`binance_client`) | Evita que `stopPrice` cole no `stopLimit` |
| **2 — Refinamento** | Trocar *Isotonic* por *Platt Scaling* para $<2.000$ trades | DeepSeek | Bloco 3 (`ml_judge.py`) | Evita que a IA decore dados pequenos |
| **2 — Refinamento** | Mínimo de 150 trades, Profit Factor $>1,20$ e Holdout | ChatGPT | Bloco 3 (Colab) | Mais segurança estatística antes do ao vivo |
| **3 — Cosmético** | Trocar o texto "Sobrevivência Determinística" no README | ChatGPT | Texto do README | Muda zero linhas de código |
| **3 — Cosmético** | Renomear `CPPI_DRAWDOWN_CONTROL` para `DRAWDOWN_SCALING` | ChatGPT | Nome no YAML | Muda zero linhas de matemática |
| **3 — Cosmético** | Reclamar que a Fase 1 é "simples demais / MVP" | DeepSeek | Nenhum | Contradição: simplificar a Fase 1 era a meta! |
| **3 — Cosmético** | Decidir se o Cofre em `FUNDING` guarda USDT ou XRP | DeepSeek | Operação manual | Guardar USDT já protege o lucro sem taxa extra |
| **3 — Cosmético** | Testar se `SIDEWAYS` deve vir antes de `BEAR` | DeepSeek | Nenhum | Ficar em dólar na queda (`BEAR` 1º) é mais seguro |

