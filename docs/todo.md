# Quadro geral de triagem

| Categoria | Item Validado | Quem Apontou | Onde Entra | Impacto Prático |
| --- | --- | --- | --- | --- |
| **1 — Catastrófico** | Filtro exato do erro `-2010` e Stop Pendurado | ChatGPT e DeepSeek | Bloco 5 (`oco_order_manager`) | Evita vender errado ou ficar sem stop |
| **1 — Catastrófico** | Arredondamento `close_time` (+1ms) no `merge_asof` | DeepSeek | Bloco 1 (`candle_collector`) | Evita perder a vela de 4h recém-fechada |
| **1 — Catastrófico** | Tratamento de `NaN` e divisão por zero no `CHOP` | ChatGPT | Bloco 2 (`indicators.py`) | Impede que o robô trave em moeda parada |
| **1 — Catastrófico** | Backup SQLite via `sqlite3.backup()` nativo | ChatGPT | Bloco 1 (`database.py`) | Evita arquivo de backup corrompido |
| **1 — Catastrófico** | Cálculo empírico de $\mathbb{E}[R]$ com Time-Stop no Gate 3 | ChatGPT e DeepSeek | Bloco 3 (Colab) | Mede o lucro real de quem sai por tempo |
| **1 — Catastrófico** | Trava de 20 trials (`trials_registry.csv`) e Holdout 6m | Gemini e DeepSeek | Bloco 3 (Colab) | Impede falso lucro por tentativa e erro |
| **1 — Catastrófico** | Kill-Switches (-12% DD, BNB min $2 e Manutenção API) | Gemini e DeepSeek | Bloco 6 (`main.py`) | Trava novas compras em pane ou crise |
| **2 — Refinamento** | Mock de `/sapi/*` na Testnet e Permissão `Universal Transfer` | ChatGPT e DeepSeek | Bloco 5 e `.env` | Testnet não aceita `/sapi/*` (evita erro 404) |
| **2 — Refinamento** | *High-Water Mark* (Topo Histórico) no Cofre de Lucros | ChatGPT | Bloco 6 (`profit_splitter`) | Só tira 30% quando recuperar prejuízos |
| **2 — Refinamento** | Teto de ordem em % da banca (`55%`) e BNB em Dólar (`$5`) | ChatGPT e DeepSeek | Bloco 4 (`strategies.yaml`) | Protege bancas pequenas desde o dia 1 |
| **2 — Refinamento** | Direção de arredondamento do `tickSize` na OCO | ChatGPT | Bloco 5 (`binance_client`) | Evita que `stopPrice` cole no `stopLimit` |
| **2 — Refinamento** | Trocar *Isotonic* por *Platt Scaling* para menos de 2.000 trades | DeepSeek | Bloco 3 (`ml_judge.py`) | Evita que a IA decore dados pequenos |
| **2 — Refinamento** | Mínimo de 150 trades, Profit Factor maior que 1,20 e Holdout | ChatGPT | Bloco 3 (Colab) | Mais segurança estatística antes do ao vivo |
| **2 — Refinamento** | Salvar Colunas 8 (`trades`) e 9 (`taker_buy_vol`) da vela | Gemini (Trader Ref) | Bloco 1 (`candle_collector`) | Captura Order Flow grátis da Binance |
| **2 — Refinamento** | Calcular *Order Flow Delta* e *Liquidity Sweep* em 1h | Gemini (Trader Ref) | Bloco 2 (`indicators.py`) | Mede agressão compradora e absorção |
| **2 — Refinamento** | Testar ablação Com vs. Sem *Order Flow* no Colab | Gemini (Trader Ref) | Bloco 3 (Colab) | Só ativa Order Flow se subir o $\mathbb{E}[R]$ |
| **2 — Refinamento** | IA em *Modo Observador Silencioso* na Fase 1 (60 trades) | Gemini | Bloco 6 (`ml_judge.py`) | Prova que a IA acerta ao vivo antes da Fase 2 |
| **3 — Cosmético** | Trocar o texto "Sobrevivência Determinística" no README | ChatGPT | Texto do README | Muda zero linhas de código |
| **3 — Cosmético** | Renomear `CPPI_DRAWDOWN_CONTROL` para `DRAWDOWN_SCALING` | ChatGPT | Nome no YAML | Muda zero linhas de matemática |
| **3 — Cosmético** | Reclamar que a Fase 1 é "simples demais / MVP" | DeepSeek | Nenhum | Contradição: simplificar a Fase 1 era a meta! |
| **3 — Cosmético** | Decidir se o Cofre em `FUNDING` guarda USDT ou XRP | DeepSeek | Operação manual | Guardar USDT já protege o lucro sem taxa extra |
| **3 — Cosmético** | Testar se `SIDEWAYS` deve vir antes de `BEAR` | DeepSeek | Nenhum | Ficar em dólar na queda (`BEAR` 1º) é mais seguro |
| **3 — Cosmético** | Ler *Order Book Level 2* (Livro de Ofertas) ao vivo | Gemini (Descartado) | Nenhum | Sem backtest grátis, sofre spoofing e pesa na VPS |

---

# Checklist de Execução por Bloco (Ordem Estrita de Construção)

## Bloco 1: Fundação, Banco SQLite WAL, Relógio e Coletor Causal
- [ ] Criar `config/strategies.yaml` (v6.0) e validador em `src/core/config.py`.
- [ ] Implementar `src/core/database.py` com `PRAGMA journal_mode=WAL`, `synchronous=FULL`, rotina nativa `sqlite3.backup()` e colunas `taker_buy_base_volume` e `number_of_trades` na tabela `candles`.
- [ ] Implementar `src/core/binance_client.py` com assinatura Ed25519, sincronização monotônica (`RTT/2`), cache dinâmico de `/api/v3/exchangeInfo`, arredondamento `Decimal` direcional (`tickSize` / `stepSize`) e mock automático para endpoints `/sapi/*` na Testnet.
- [ ] Implementar `src/data/candle_collector.py` normalizando `close_time` para o segundo exato seguinte (`+1ms`) e fazendo junção 4h/1h via `merge_asof(..., direction='backward', allow_exact_matches=True)`.
- [ ] Passar 100% no **Gate 1** (`pytest tests/test_block1_causal_and_decimal.py`).

## Bloco 2: Indicadores Canônicos (TA-Lib), Order Flow e Clima MECE
- [ ] Implementar `src/features/indicators.py` com `EMA`, `ATR` (Wilder), `RSI` (Wilder), `Bollinger`, `ADX` (Wilder), `CHOP` (base 10 com proteção contra divisão por zero) e variáveis de fluxo (`order_flow_delta`, `taker_buy_ratio`, `liquidity_sweep_bull`).
- [ ] Implementar `src/regimes/regime_detector.py` com partição MECE de 4 estados (`BEAR` -> `BULL` -> `SIDEWAYS` -> `TRANSITION`) e fallback automático para `TRANSITION` em caso de `NaN` ou histórico menor que 120 velas.
- [ ] Passar 100% no **Gate 2** (`pytest tests/test_block2_talib_and_mece.py`).

## Bloco 3: Simulador Causal Single-Slot e Ablação no Google Colab
- [ ] Criar `notebooks/01_download_causal_1h_4h_1m.ipynb` baixando dados desde `2022-01-01` das 7 moedas (`BTC`, `ETH`, `SOL`, `XRP`, `ADA`, `LINK`, `AVAX`) incluindo volume de agressão compradora.
- [ ] Criar `notebooks/02_ablation_and_dsr_sim.ipynb` com partição `60% Treino / 25% Validação / 15% Holdout (6 meses)`, cálculo empírico de $\mathbb{E}[R]$ (incluindo saídas por tempo de 24h), comparação com e sem filtro de *Order Flow Delta* e registro automático em `trials_registry.csv`.
- [ ] Passar no **Gate 3**: mínimo de 150 trades fora da amostra, $\bar{R}$ maior que $+0,10R$ (IC 95% positivo via *Block Bootstrap*), *Profit Factor* maior ou igual a $1,20$ no custo Base e maior que $1,05$ no custo Pessimista, e $\text{DSR}$ maior ou igual a $0,95$.

## Bloco 4: Dimensionamento de Teto Duplo, Drawdown Scaling e Cofre
- [ ] Implementar `src/risk/position_sizer.py` com teto duplo (risco de 1% e máximo de 50% do caixa operável), trava proporcional $\min(500\text{ USDT}, 55\%\text{ da banca})$, desconto de `pending_hodl_bucket_usdt` e aborto se menor que $1,05 \times \text{minNotional}$.
- [ ] Implementar `src/risk/shadow_manager.py` com corte de risco pela metade em drawdown de 4% e gatilhos de *Auto-Downgrade* e *Hard Stop* (-12%).
- [ ] Implementar `src/risk/profit_splitter.py` com regra de *High-Water Mark* e transferência `MAIN_FUNDING`.
- [ ] Passar 100% no **Gate 4** (`pytest tests/test_block4_dual_cap_and_cppi.py`).

## Bloco 5: Execução OCO, Watchdog de Stop e Reconciliador de Boot
- [ ] Implementar `src/execution/oco_order_manager.py` descontando comissão no ativo base, tratando erro `-2010` com verificação de preço/mensagem para fallback `MARKET`, *watchdog* de stop pendurado e fechamento de 4 passos no Time-Stop.
- [ ] Implementar `src/core/state_reconciler.py` resolvendo `PENDING_ENTRY` via `origClientOrderId` (sem duplicar compra em caso de timeout `-1007`) e curando execuções parciais de OCO.
- [ ] Passar 100% no **Gate 5** (`pytest tests/test_block5_reconciler_and_oco.py` + 4 Testes de Tortura na Testnet).

## Bloco 6: Observabilidade, Voo Assistido e Graduação de Fases
- [ ] Implementar `src/notifications/telegram_reporter.py` e `src/main.py` (com checagem de manutenção `/sapi/v1/system/status` e reserva de BNB).
- [ ] Aprovar **Gate 6A** (7 dias na Testnet + teste de restauração do backup SQLite) e **Gate 6B** (30 dias ou 25 trades em *Shadow Live* com dados reais).
- [ ] Iniciar **Fase 1** com capital real (apenas `BULL`, risco fixo de 1%, IA anotando notas em modo silencioso) até completar 60 trades reais para avaliar subida à **Fase 2**.
