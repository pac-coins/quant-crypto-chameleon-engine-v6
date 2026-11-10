# Arquitetura sistemática, validação estatística e execução blindada

Este repositório contém a arquitetura de engenharia de software, o pipeline de validação econométrica e a metodologia de construção incremental em seis estágios para um robô de **Trading Sistemático Adaptativo (1h/4h)** na **Binance Spot**.

O sistema foi projetado para administrar um capital inicial de **300 XRP** sem uso de alavancagem, sob duas premissas inegociáveis:

1. **Rigor Epistemológico:** Nenhuma estratégia operacional, filtro de regime ou modelo de Inteligência Artificial é presumido como lucrativo a priori. Toda regra de decisão é tratada como uma **hipótese estatística** que permanece desativada (*Feature Flag OFF*) até superar os testes fora da amostra (*Out-of-Sample*) com custos reais, estresse de cauda e correção para múltiplos testes no Estágio 3.
2. **Sobrevivência Operacional Determinística:** Quedas de energia, perda de pacotes de rede (*Timeouts*), variações de relógio da máquina virtual, cobrança de taxas no ativo base ou saltos de preço (*Flash-Crashes*) jamais podem deixar uma posição aberta sem proteção de Stop Loss ou liquidar acidentalmente o patrimônio de longo prazo.

---

## 1. Arquitetura Patrimonial: Core-Satellite, Reserva de BNB e Segregação via Carteira Funding

Para separar o investimento passivo de longo prazo do giro tático do robô sem criar conflitos contábeis na corretora, o patrimônio inicial de **300 XRP** é dividido em três camadas fisicamente isoladas:

* **Carteira Core (50% — 150 XRP em Hold Direcional Intocável):** Mantida em carteira externa (ou subconta isolada fora do escopo da API Spot).
* *Nota de Transparência de Risco:* A Carteira Core possui **risco operacional zero** (o robô não consegue tocá-la), mas permanece **100% exposta ao risco direcional de mercado do XRP**. Em ciclos históricos de baixa (*bear markets* cripto), um ativo único pode sofrer quedas temporárias (*drawdowns*) de 70% a 90%. Portanto, a Carteira Core é uma alocação direcional consciente de longo prazo, enquanto a preservação ativa em dólar ocorre exclusivamente na Carteira Tática. O risco global é monitorado pelo *Value at Risk* (VaR) e *Conditional VaR* (CVaR a 99%) do portfólio combinado.


* **Carteira Tática Camaleão (48,5% — Equivalente a 145,5 XRP convertidos em USDT):** Constitui o caixa operacional do robô ($C_{\text{usdt, total}}$) na carteira `SPOT` (`MAIN`). Permite operar os pares mais líquidos do mercado e permanecer 100% protegido em dólar (`USDT`) durante períodos de queda ou indefinição.
* **Reserva Técnica Intocável de Taxas (1,5% — Equivalente a 4,5 XRP convertidos em BNB):** Mantida na carteira `SPOT` com a opção *Use BNB to pay fees* ativada na Binance (garantindo 25% de desconto nas taxas).
* *Por que o BNB é isolado?* Na versão anterior (v5), se você não tivesse BNB livre — ou se o robô operasse o par `BNBUSDT` e vendesse todo o seu BNB —, a Binance descontaria a taxa de compra da própria moeda comprada (ex.: você compraria 10 SOL e receberia 9,99 SOL). Ao tentar criar a ordem de Stop Loss (OCO) para os 10 SOL inteiros, a Binance recusaria a ordem por saldo insuficiente (`Erro -2010`) e sua posição ficaria **desprotegida**. Na v6.0, o par `BNBUSDT` é **proibido** na cesta de trading para nunca consumir a reserva de taxas, e o módulo de ordens calcula o Stop Loss sobre o saldo líquido real.


* **Cofre Divisor de Lucros com Isolamento Físico (`profit_splitter.py`):**
* Começa **desligado na Fase 1** (`enabled: false`) até que o robô comprove vantagem estatística real.
* Quando ativado na Fase 2, separa contabilmente 70% do lucro líquido de cada operação vencedora para reinvestimento no robô e 30% em uma variável do banco SQLite (`pending_hodl_bucket_usdt`).
* **Proteção de Caixa Livre:** O dimensionador de lotes (`position_sizer.py`) desconta obrigatoriamente o saldo do cofrinho antes de calcular qualquer compra:

$$C_{\text{usdt, operável}} = \max\left(0, \; \text{USDT}_{\text{free}} - \text{pending\_hodl\_bucket\_usdt}\right)$$



Isso impede que o robô gaste o dinheiro do cofrinho em novas operações enquanto ele ainda não atingiu 10,00 USDT.
* **Segregação Automática na Carteira `FUNDING`:** Assim que `pending_hodl_bucket_usdt` atinge pelo menos **10,00 USDT**, o robô não deixa moedas compradas paradas na carteira Spot (onde o reconciliador de boot acharia que é uma posição órfã sem Stop Loss e a venderia!). Em vez disso, o sistema transfere os 10,00 USDT **diretamente da carteira `SPOT` para a carteira `FUNDING**` da própria Binance via endpoint oficial `POST /sapi/v1/asset/transfer` (`type="MAIN_FUNDING"`). A carteira `FUNDING` é 100% invisível e intocável pelas ordens de trade Spot, eliminando qualquer condição de corrida sem precisar habilitar permissão de saque externo na chave de API.



---

## 2. O Cérebro "Camaleão": Partição MECE de 4 Estados e Causalidade Temporal Estrita

Imagine que o robô é um piloto de corrida que olha o boletim meteorológico de 4 em 4 horas (`4h`) para escolher o pneu certo antes de acelerar de hora em hora (`1h`). Na versão anterior (v5), havia dois defeitos graves nesse sensor:

1. **Horários que olhavam o futuro (*Look-Ahead Bias*):** Às 13h, a vela de 4h que abriu às 12h só vai fechar às 16h. Se o computador ler a vela das 12h durante o teste histórico, ele estará lendo às 13h o que aconteceu até as 16h!
2. **Regras com buracos (*Não-MECE*):** Se o índice de bagunça do mercado (`CHOP`) ficasse em 52 (entre 50 e 55), nenhuma regra funcionava e o sensor ficava cego.

### 2.1. Lei da Causalidade Temporal Estrita (Zero Look-Ahead Bias)

Tanto no coletor ao vivo (`candle_collector.py`) quanto nos notebooks do Google Colab, nenhuma série é alinhada pelo horário de abertura (`open_time`). Todas as velas de 1h, 4h e do Bitcoin são indexadas pelo seu horário efetivo de fechamento (`close_time_utc`). No instante de decisão $t$ (fechamento de uma vela de 1h), o estado macro de 4h só pode utilizar a última vela de 4h cujo fechamento $T_{\text{close}, 4h}$ satisfaça:


$$T_{\text{close}, 4h} \le t$$


No código, essa junção é executada exclusivamente via `pandas.merge_asof(df_1h, df_4h, on='close_time_utc', direction='backward', allow_exact_matches=True)`. Assim, às 01h, 02h e 03h UTC, o robô enxerga apenas a vela de 4h que fechou à 00h00 UTC; às 04h00 UTC, enxerga a vela de 4h que acabou de fechar às 04h00 UTC.

### 2.2. Definições Matemáticas Canônicas dos Indicadores de Regime

Para evitar distorções numéricas que invalidem os limiares da literatura:

* **True Range ($\text{TR}_t$):** $\text{TR}_t = \max\left(H_t - L_t, \; \vert{}H_t - C_{t-1}\vert{}, \; \vert{}L_t - C_{t-1}\vert{}\right)$.
* **ATR de 14 períodos ($\text{ATR}_{14}$):** Calculado com a suavização exponencial de Wilder (RMA com $\alpha = 1/14$).
* **ADX de 14 períodos ($\text{ADX}_{14}$):** Calculado estritamente com a suavização RMA de Wilder sobre $+\text{DM}$, $-\text{DM}$ e $\text{TR}$ (validado contra a biblioteca `TA-Lib` com erro máximo inferior a $10^{-10}$).
* **Choppiness Index ($\text{CHOP}_{14}$):** Calculado obrigatoriamente na base logarítmica 10 (Dreiss, 1993):

$$\text{CHOP}_{14, t} = 100 \cdot \frac{\log_{10}\left( \frac{\sum_{j=t-13}^{t} \text{TR}_j}{\max_{j \in [t-13, t]} H_j - \min_{j \in [t-13, t]} L_j} \right)}{\log_{10}(14)}$$



### 2.3. Partição MECE dos 4 Climas de Mercado (Ordem de Precedência Estrita)

A função `detect_market_regime()` avalia os indicadores da última vela fechada de 4h seguindo uma árvore de decisão **Mutuamente Exclusiva e Coletivamente Exaustiva (MECE)** onde a defesa do capital tem prioridade máxima:

| Ordem | Regime Detectado | Regra Matemática Exata no 4h | Estratégia em 1h | Status na Fase 1 |
| --- | --- | --- | --- | --- |
| **1º** | **BEAR (Tempestade)** | $\text{Close}_{4h} \lt \text{EMA}_{100, 4h}$ e $\Delta \text{EMA}_{100, 4h} \lt 0$ | Defesa em USDT (`bear_capitulation_snap` em quarentena) | **Ativo (Defesa 100% USDT)** |
| **2º** | **BULL (Sol Aberto)** | $\text{Close}_{4h} \gt \text{EMA}_{100, 4h}$ e $\Delta \text{EMA}_{100, 4h} \gt 0$ e $\text{CHOP}_{14, 4h} \lt 50$ | `bull_trend_pullback` (Recuo na $\text{EMA}_{20, 1h}$) | **Ativo (Fase 1)** |
| **3º** | **SIDEWAYS (Lateral)** | Não é `BEAR`, $\text{CHOP}_{14, 4h} \ge 55$ e $\text{ADX}_{14, 4h} \lt 22$ | `sideways_pingpong` (Banda Inf. + $\text{RSI}_{14} \lt 32$ + $\text{RR} \ge 1.5$) | **Desligado (Fase 2)** |
| **4º** | **TRANSITION (Neblina)** | Qualquer estado intermediário não capturado acima | Nenhuma (`NO_TRADE` — 100% em USDT) | **Ativo (Proteção)** |

* **Por que `sideways_pingpong` exige a trava $\text{RR} \ge 1.5$?** Como o alvo no mercado lateral é a média central da Banda de Bollinger ($\text{SMA}_{20}$) e o stop é $1.5 \times \text{ATR}_{14}$, quando as bandas se estreitam o alvo pode ficar mais curto que o próprio risco! O robô bloqueia a entrada se $\frac{\text{SMA}_{20, t} - P_{\text{entry}}}{1.5 \cdot \text{ATR}_{14, t}} \lt 1.5$.
* **Por que `bear_capitulation_snap` começa desligado em quarentena?** Na literatura quantitativa (Connors), comprar $\text{RSI}_2$ abaixo de 5 funciona quando o ativo está *acima* da média longa de 200 períodos. Comprar $\text{RSI}_2$ abaixo de 5 em tendência de baixa ($\text{Close}_{4h} \lt \text{EMA}_{100, 4h}$) no mercado cripto expõe a conta ao risco de comprar colapsos estruturais (como ocorreu com LUNA e FTT). Essa estratégia só pode ser ligada se provar expectância líquida positiva no teste de estresse com filtro adicional de rejeição de pavio inferior.

### 2.4. Cesta de 7 Ativos Líquidos (Mitigação de Survivorship Bias) e Fila "Single-Slot"

* **Cesta Congelada de 7 Pares:** `BTCUSDT`, `ETHUSDT`, `SOLUSDT`, `XRPUSDT`, `ADAUSDT`, `LINKUSDT` e `AVAXUSDT` (`BNBUSDT` excluído para proteger a reserva de taxas).
* **Controle de Viés de Sobrevivência (*Survivorship Bias*):** Como ativos como `SOL` e `AVAX` não possuíam liquidez consolidada em 2019–2020, o backtest multi-ativo padrão é restrito ao período a partir de `2022-01-01` (quando todos os 7 pares já integravam o topo de liquidez da Binance), além de exigir um teste separado de estresse incluindo ativos deslistados/colapsados da época se estratégias de reversão à média forem avaliadas.
* **Desempate Coerente na Fila Single-Slot (`MAX_OPEN_POSITIONS = 1`):**
* Na **Fase 1** (onde apenas `bull_trend_pullback` opera), se duas moedas derem sinal na mesma hora, o desempate utiliza a Força Relativa Ajustada por Volatilidade nas últimas 12 horas:

$$\text{RS}_{\text{score}, i} = \frac{\text{Close}_{i, t} - \text{Close}_{i, t-12}}{\text{ATR}_{14, i, t}}$$


* Na **Fase 2+** (se mais de um regime estiver habilitado com IA calibrada), é proibido comparar $\text{RS}_{\text{score}}$ bruto entre estratégias de tendência e estratégias de reversão. O desempate passa a escolher o ativo com maior **Valor Esperado Líquido em unidades de risco ($\widehat{\mathbb{E}}[R_i]$)**.



---

## 3. Gestão Quantitativa de Risco, Dimensionamento de Lote e Controle de Drawdown

### 3.1. Dimensionamento Fracionário Fixo com Teto Duplo e Trava de Expectância (Substituindo o Falso "Kelly")

Na versão anterior (v5), chamava-se de "Kelly Fracionado" uma regra que alocava até 95% do caixa nos 30% melhores sinais da IA, sem verificar se esses sinais tinham lucro esperado positivo e sem notar que um stop de $5\%$ com $95\%$ do caixa geraria uma perda de quase $6\%$ da carteira (violando o limite de $2,2\%$).

Na **v6.0**, o módulo `position_sizer.py` implementa uma arquitetura honesta em duas etapas:

1. **Fase 1 (Operação Base Sem IA):** Risco fracionário fixo de **1,0% da carteira tática por operação** (`risk_per_trade_pct = 0.010`), limitado ao teto máximo de **50% do caixa operável em USDT** (`max_cash_allocation_pct = 0.50`).
2. **Fase 2 (Com Juiz de IA Calibrado Validado no Bloco 3):**
* **Trava Obrigatória de Expectância Positiva:** Seja $R_{\text{eff}, i}$ o múltiplo R:R efetivo da estratégia e $C_{R, i} = \frac{f_{\text{ida+volta}} + s_{\text{stress}}}{d_{\text{stop}, i}}$ o custo friccional em unidades de risco. A probabilidade mínima de empate (*limiar de indiferença*) é:

$$\underline{p}_i = \frac{1 + C_{R, i}}{R_{\text{eff}, i} + 1}$$



Se a probabilidade calibrada prevista pelo modelo $\hat{p}_{\text{cal}, i}$ não superar $\underline{p}_i$ com margem de segurança ($\hat{p}_{\text{cal}, i} \le \underline{p}_i + 0.03$), o sinal recebe **Nota C (Alocação = 0%)**, mesmo que esteja no topo do ranking de quantis!
* **Equação de Teto Duplo (Proteção Contra Volatilidade e Caudas):** O volume financeiro $V_{\text{pos}}$ em USDT jamais pode violar o risco máximo parametrizado da carteira ($r_{\text{max}} = 1.0\%$ na Nota B ou $r_{\text{max}} = 2.0\%$ na Nota A+ sob limite de CVaR):

$$V_{\text{pos}} = \min\left( w_{\text{max}} \cdot C_{\text{usdt, operável}}, \; \frac{r_{\text{max}} \cdot C_{\text{usdt, total}}}{d_{\text{stop}, i} + f_{\text{ida+volta}} + s_{\text{stress}}} \right)$$


* **Piso Dinâmico da Binance (`NOTIONAL`):** O sistema lê o filtro `minNotional` (ou `notional`) real de cada moeda via `/api/v3/exchangeInfo`. Se $V_{\text{pos}}$ for menor que $\text{minNotional}_{\text{moeda}} \times 1.05$, a operação é **abortada** (jamais arredondada para cima, para não inflar o risco).



### 3.2. Política sobre o Breakeven em $+1.2R$ (Por Que Começa Desligado na Fase 1)

* **A Matemática:** Pelo Teorema da Parada Opcional de Doob, em um processo com deriva positiva ($\mu \gt 0$, que é exatamente o que ocorre quando acertamos uma tendência de alta!), encurtar o Stop Loss de $-1.0R$ para $0.0R$ quando o preço atinge $+1.2R$ reduz drasticamente o tempo médio de permanência na tendência ($\mathbb{E}[\tau_{\text{BE}}] \lt \mathbb{E}[\tau_{\text{orig}}]$), cortando a chance de atingir $+2.5R$ e aumentando as saídas prematuras por ruído (*whipsaws*) com pagamento de taxas e slippage. Além disso, mover o stop dentro da hora exige manter um **WebSocket assíncrono em tempo real** e cancelar/recriar a ordem OCO na Binance (criando uma janela de corrida se o preço cair rápido).
* **Decisão de Engenharia v6.0:** Na Fase 1, `breakeven_enabled: false` (a ordem OCO original vai até o alvo ou até o stop sem interferência). No Bloco 3 (Colab), o breakeven é testado como o **Degrau E** do estudo de ablação usando **velas de 1 minuto (1m)** para eliminar a ambiguidade intra-vela de 1h. Só será ligado na Fase 2 se aumentar comprovadamente o *Calmar Ratio* e a expectância líquida.

### 3.3. Controle de Drawdown (CPPI) e Reforma do "Modo Fantasma" (`shadow_manager.py`)

Na v5, o "Modo Fantasma" tinha dois defeitos: (a) religava o dinheiro real ao "acertar 2 trades ou somar +2,0R", mas como 1 único trade de alta paga $+2.5R$, ele religava com **1 única vitória**; e (b) 2 vitórias seguidas acontecem por pura sorte até em moedas viciadas perdedoras ($p^2 = 12,25\%$ quando $p = 0,35$), gerando falsos alarmes constantes.

Na **v6.0**, a defesa contra meses adversos possui dois modos configuráveis (testados no Notebook 03):

1. **Modo Padrão Recomendado — Controle Contínuo de Drawdown (`CPPI_DRAWDOWN_CONTROL`):** Em vez de desligar o robô completamente após 2 stops (e correr o risco de ficar de fora justamente do trade de recuperação da tendência!), sempre que o drawdown acumulado da Carteira Tática atingir $4,0\%$, o robô reduz o risco por operação pela metade ($r_{\text{max}} \rightarrow 0,5 \times r_{\text{max}}$, ou seja, $0,5\%$ por trade) até que a carteira recupere a máxima histórica (*High-Water Mark*).
2. **Modo Alternativo de Parada Sequencial (`CUSUM_SHADOW_MODE`, desligado na Fase 1):** Se ativado após validação empírica de agrupamento de volatilidade (*clustering* GARCH), substitui a regra arbitrária de "2 derrotas / 2 vitórias" por um filtro estatístico **CUSUM tabular** sobre os retornos em $R$, exigindo no mínimo 4 operações virtuais com média acumulada positiva e sem permitir atalho por um único trade isolado.

---

## 4. Ciência de Dados, Calibração de Probabilidades e Validação Estatística (Bloco 3)

Nenhuma peça é ligada no escuro. O simulador no Google Colab (`notebooks/`) aplica sete travas científicas contra ilusões de backtest:

1. **Pesos Exatos de Unicidade Amostral de López de Prado (AFML Snippet 4.2):**
Duas operações em criptomoedas que abrem às 10h e às 11h e ficam abertas por 20 horas compartilham o mesmo movimento do mercado. Em vez de olhar apenas o segundo da entrada, o peso de cada evento $i$ (ativo entre as barras $t_{i,0}$ e $t_{i,1}$) é a **média da sua unicidade ao longo de toda a sua duração**:

$$\bar{u}_i = \frac{1}{t_{i,1} - t_{i,0} + 1} \sum_{t=t_{i,0}}^{t_{i,1}} \frac{1}{c_t}, \quad \text{onde } c_t = \sum_{j=1}^{N} \mathbb{I}\left(t \in [t_{j,0}, t_{j,1}]\right)$$



Na validação cruzada temporal (*Purged Walk-Forward*), aplica-se **Purging** de 24 horas (horizonte máximo da barreira de tempo) e **Embargo** de 2% entre cada bloco de treino e teste.
2. **Alinhamento Cruzado Causal das 5 Features Estacionárias:**
Calculadas apenas sobre velas 100% fechadas:
* `atr_pct`: $\text{ATR}_{14, t} / \text{Close}_t$.
* `dist_ema100_atr`: $(\text{Close}_t - \text{EMA}_{100, t}) / \text{ATR}_{14, t}$.
* `rvol_20`: $\text{Volume}_t / \text{Mediana}(\text{Volume}_{t-19 \dots t})$.
* `chop_14`: $\text{CHOP}_{14, t}$ (base $\log_{10}$).
* `btc_regime_ret`: Retorno logarítmico de 12h do `BTCUSDT`, alinhado estritamente com `close_time_btc <= close_time_asset` (zero vazamento cruzado).


3. **Meta-Labeling Segregado por Regime e Calibração Real de Probabilidade (`ECE`):**
* É proibido misturar sinais de `BULL` (alvo $+2.5R$) com sinais de `SIDEWAYS` (alvo curto na média) dentro de um mesmo ranking de probabilidade bruta $P_{\text{win}}$. Cada estratégia possui seu próprio modelo secundário.
* As probabilidades brutas do classificador passam por **Calibração Isotônica / Platt Scaling** em conjunto de validação fora da amostra, exigindo **Expected Calibration Error ($\text{ECE}$) menor que $0,05$** nos *Reliability Diagrams* antes de liberar o uso dos quantis.


4. **Três Cenários de Custos e Estresse de Gaps:**
Todo backtest é reportado sob três matrizes de atrito:
* *Cenário Otimista:* Saída no Alvo como `LIMIT_MAKER` (0,075% com BNB) e Stop como Taker (0,075% com BNB) + 0,03% de slippage.
* *Cenário Base (Padrão de Aprovação):* Taxa Taker de 0,15% ida e volta (com BNB) + 0,06% de slippage base + modelo de slippage proporcional à volatilidade da vela.
* *Cenário Pessimista (Estresse de Cauda):* Taxa sem BNB (0,20% ida e volta) + 0,12% de slippage + choques de *gap* adverso no Stop Loss calibrados na distribuição histórica de caudas (medindo o CVaR a 99%).


5. **Estudo de Ablação em 5 Degraus + Deflated Sharpe Ratio ($\text{DSR}$):**
Comparamos fora da amostra:
* **Degrau A:** Benchmark passivo (50% XRP + 50% USDT e Buy & Hold BTC).
* **Degrau B:** Apenas `bull_trend_pullback` sem filtro de clima 4h (risco fixo 1%).
* **Degrau C (Candidato da Fase 1):** Cérebro Camaleão MECE (Clima `BULL` operando com risco fixo 1% + `BEAR` e `TRANSITION` 100% em USDT).
* **Degrau D:** Degrau C + Juiz de IA calibrado filtrando Nota C e modulando Nota B / A+.
* **Degrau E:** Teste de impacto de ativar ou não o Breakeven em $+1.2R$ (resolvido com velas de 1 minuto) e o controle de drawdown CPPI.
* *Critério Estatístico de Aprovação:* Registro obrigatório do número total de configurações testadas ($K$) para cálculo do **Deflated Sharpe Ratio ($\text{DSR} \gt 0,95$)** e do **Minimum Track Record Length ($\text{MinTRL}$)** de Bailey & López de Prado.



---

## 5. Engenharia de Resiliência e Integração com a Binance Spot API (Blocos 1, 5 e 6)

Para garantir que nenhuma falha de hardware, rede ou corretora deixe posições órfãs:

1. **Aritmética Exata com `decimal.Decimal` e Cache Dinâmico do `exchangeInfo`:**
* O uso de `float` binário para preços e quantidades é proibido no envio de ordens. O módulo `binance_client.py` baixa e atualiza a cada 1 hora os filtros `PRICE_FILTER` (`tickSize`), `LOT_SIZE` (`stepSize`, `minQty`) e `NOTIONAL` (`minNotional`) de `/api/v3/exchangeInfo`.
* Preços são truncados/arredondados para múltiplos exatos de `tickSize` e quantidades são sempre arredondadas para baixo (`ROUND_DOWN`) para múltiplos exatos de `stepSize` usando `decimal.Decimal` serializado como `str`.


2. **Sincronização Monotônica de Relógio (Algoritmo de Cristian) e Buffer de Virada de Vela:**
* Máquinas virtuais gratuitas sofrem deriva de relógio que causa o erro `-1021 (Timestamp ahead/behind server)`. O cliente calcula o offset em relação ao relógio monotônico local descontando metade da latência de ida e volta ($\text{RTT}/2$) via `GET /api/v3/time`:

$$\text{Offset}_{\text{mono}} = T_{\text{binance}} - \left(T_{\text{mono\_req}} + \frac{T_{\text{mono\_resp}} - T_{\text{mono\_req}}}{2}\right), \quad T_{\text{assinatura}}(t) = \lfloor T_{\text{mono}}(t) + \text{Offset}_{\text{mono}} - 200\text{ ms} \rfloor$$


* Na virada de cada hora, o coletor aguarda **3 segundos de segurança (`XX:00:03 UTC`)** e confere se o `close_time` da vela `[-2]` corresponde exatamente à hora que acabou de fechar (fazendo até 5 retentativas de 1 segundo se o agregador da Binance atrasar).


3. **Protocolo Idempotente contra Timeouts HTTP (`Erro -1007`):**
* Antes de enviar uma compra, o robô grava no SQLite (configurado com `PRAGMA journal_mode=WAL;` e `PRAGMA synchronous=FULL;`) o estado `PENDING_ENTRY` com `client_order_id = f"CHM_{symbol}_{epoch_ms}"`.
* Se a requisição `POST /api/v3/order` sofrer *Timeout* ou queda de conexão, **o robô é proibido de reenviar o POST cegamente** (pois na Binance, repetir um `newClientOrderId` após a primeira ordem já ter sido executada gera uma segunda compra!). O sistema consulta obrigatoriamente `GET /api/v3/order?symbol=...&origClientOrderId=...` para descobrir se a ordem executou antes de dar o próximo passo.


4. **Ordem OCO Moderna (`POST /api/v3/orderList/oco`) com Quantidade Líquida e Fallback a Mercado:**
* A quantidade enviada para a ordem OCO desconta qualquer comissão eventualmente cobrada no ativo base e limita-se ao saldo livre real:

$$Q_{\text{oco}} = \text{floor\_step}\left( \min\left( Q_{\text{executed}} - \sum_{k \text{ where asset}=A} \text{commission}_k, \; \text{FreeBalance}(A) \right), \; \text{stepSize}_A \right)$$


* Utiliza o endpoint atual `/api/v3/orderList/oco` (com `aboveType="LIMIT_MAKER"` para o Take Profit e `belowType="STOP_LOSS_LIMIT"` com buffer de **2,5%** abaixo do `stopPrice`, ou `belowType="STOP_LOSS"` a mercado conforme parametrizado).
* **Fallback Salva-Vidas contra `Erro -2010` ("Stop price would trigger immediately"):** Se durante o envio da OCO (ou durante a recuperação de uma queda de energia no `state_reconciler.py`) o preço atual da moeda já tiver despencado para igual ou abaixo do `stopPrice`, a Binance recusará a criação da OCO. Nesse exato milissegundo, o `oco_order_manager.py` captura a condição ($P_{\text{bid}} \le P_{\text{stop}} \times 1.0005$ ou erro `-2010`) e dispara imediatamente uma **venda total a mercado (`MARKET`)**, zerando a posição desprotegida.


5. **Reconciliação de Execuções Parciais e Fechamento Atômico no Time Stop:**
* No boot e a cada ciclo, o `state_reconciler.py` soma o saldo `free + locked` de cada um dos 7 ativos operáveis e compara com a quantidade coberta por uma `orderList` OCO ativa. Se uma OCO tiver executado parcialmente e deixado resíduo desprotegido acima de `minNotional`, ele protege ou liquida o resíduo.
* Quando uma operação atinge o limite de tempo (`time_barrier_candles_1h`), o robô executa uma sequência atômica de 4 passos: (1) Cancela a `orderList` OCO (`DELETE /api/v3/orderList`); (2) Lê as quantidades já executadas nas pernas da OCO cancelada; (3) Consulta o saldo `free` atualizado; (4) Envia a venda `MARKET` exclusivamente para a quantidade residual não executada.


6. **Backup Automático do SQLite e Disjuntor de Segurança:**
* Como discos de VPS gratuitas podem ser reciclados pelo provedor, um *cronjob* diário envia cópia criptografada do arquivo SQLite (`.db` + `.wal`) para armazenamento externo (Object Storage / Telegram privado).
* O cliente possui travas rígidas no código (`max_single_order_usdt`, `max_daily_trades`) e verificação de manutenção programada da corretora (`/sapi/v1/system/status`) para bloquear novas entradas antes de janelas de manutenção.



---

## 6. Metodologia de Construção Incremental (Os 6 Estágios e Seus Gates de Aprovação)

```text
[Bloco 1: Dados, Relógio e SQLite WAL] ──► [Bloco 2: Indicadores TA-Lib e Clima MECE] ──► [Bloco 3: Backtest Causal e Ablação no Colab]
                                                                                                                  │
[Bloco 6: Produção + Cofre FUNDING]   ◄── [Bloco 5: Binance Testnet, OCO e Fallback] ◄── [Bloco 4: Sizing Teto Duplo + CPPI Drawdown]

```

* **Estágio 1 — Fundação, Relógio Monotônico, SQLite WAL e Coletor Causal (`src/core/` e `src/data/`):**
* *Escopo:* Banco SQLite com WAL ativo, leitor de `strategies.yaml`, cliente Binance com aritmética `Decimal`, cache de `exchangeInfo`, sincronização monotônica de relógio (`RTT/2`) e coletor de velas fechadas de 1h e 4h das 7 moedas + BTC.
* *Gate 1 (Teste de Aprovação):* Truncar dados históricos às `02:00 UTC` e provar que o alinhamento `merge_asof` por `close_time_utc` gera vetores 100% idênticos aos gerados com a tabela completa (zero vazamento de futuro), além de validar que toda quantidade gerada em 10.000 simulações respeita `stepSize` e `tickSize` exatos via `Decimal`.


* **Estágio 2 — Indicadores Canônicos e Sensor MECE de 4 Estados (`src/features/` e `src/regimes/`):**
* *Escopo:* Cálculo de EMA, ATR (Wilder), RSI (Wilder), Bollinger, ADX (Wilder) e CHOP ($\log_{10}$), mais a árvore `detect_market_regime()` retornando `BEAR`, `BULL`, `SIDEWAYS` ou `TRANSITION`.
* *Gate 2 (Teste de Aprovação):* Testes `pytest` comparando ADX, ATR e RSI contra `TA-Lib` (erro máximo menor que $10^{-10}$) e teste de fronteira provando que 100% do espaço de estados cai em exatamente 1 único regime MECE.


* **Estágio 3 — Simulador Causal de Fila Única e Estudo de Ablação no Google Colab (`notebooks/`):**
* *Escopo:* Execução dos Degraus A, B, C, D e E (incluindo resolução intra-vela de 1 minuto para testar o breakeven), pesos de unicidade $\bar{u}_i$, 3 cenários de custo e cálculo de DSR e CVaR 99%.
* *Gate 3 (Teste de Aprovação):* Aprovação estatística da Fase 1 (Degrau C superando custos base e pessimistas com $\mathbb{E}[R] \gt 0$ e $\text{DSR} \gt 0.95$). Qualquer módulo (IA, Breakeven, Sideways) que não agregar valor estatístico líquido permanece desligado em produção.


* **Estágio 4 — Dimensionador de Teto Duplo e Controle de Drawdown CPPI (`src/risk/`):**
* *Escopo:* `position_sizer.py` (descontando `pending_hodl_bucket_usdt`, aplicando a equação de teto duplo e abortando se menor que `minNotional`) e `shadow_manager.py` (redução CPPI de 50% do risco em drawdown de 4% e máquina de estados virtual).
* *Gate 4 (Teste de Aprovação):* Provar em testes unitários que um stop largo de 6% jamais aloca 95% do caixa (sendo travado pelo teto de risco da carteira) e que um sinal com probabilidade abaixo do limiar de indiferença $\underline{p}_i$ recebe lote zero.


* **Estágio 5 — Execução Blindada, OCO Líquida e Reconciliador na Testnet (`src/execution/`):**
* *Escopo:* Envio de ordens com `origClientOrderId`, criação de `/api/v3/orderList/oco` descontando comissão no ativo base, fechamento atômico de 4 passos no Time Stop e `state_reconciler.py`.
* *Gate 5 ("Teste do Puxão de Tomada e Preço Violado"):* Simular na Testnet (e via mocks de falha): (1) Timeout `-1007` na compra — provar que o bot consulta `origClientOrderId` e não compra duplicado; (2) Desligar o bot após a compra, esperar o preço cair *abaixo* do `stopPrice` e religar — provar que o reconciliador recebe o erro `-2010` da OCO e vende imediatamente a `MARKET` no primeiro segundo!


* **Estágio 6 — Alertas no Telegram, Segregação `MAIN_FUNDING` e Deploy (`src/notifications/`):**
* *Escopo:* Relatórios transparentes no Telegram, transferência automática do cofre para a carteira `FUNDING` e container Docker com backup diário do SQLite.
* *Gate 6 (Teste de Aprovação):* Operar 7 dias contínuos na Binance Spot Testnet com zero erros não tratados e reconciliação limpa em reinicializações forçadas.



---

## 7. Mapa de Diagnóstico Rápido (Onde Olhar Quando Algo Der Errado)

| Sintoma Observado | Arquivo Responsável | Causa Técnica e Diagnóstico Imediato |
| --- | --- | --- |
| Rejeição `PRICE_FILTER`, `LOT_SIZE` ou `NOTIONAL` | `src/core/binance_client.py` ou `src/risk/position_sizer.py` | Cache do `/api/v3/exchangeInfo` expirado ou conversão fora do wrapper `Decimal`. Rodar `pytest tests/test_block1_data.py`. |
| Erro `-2010` ao criar ordem OCO logo após a compra | `src/execution/oco_order_manager.py` | Ou o preço caiu abaixo do `stopPrice` (deve acionar fallback `MARKET`), ou a reserva de BNB acabou (a comissão comeu o ativo base). |
| Servidor reiniciou após queda de internet na compra | `src/core/state_reconciler.py` | Verificar status `PENDING_ENTRY` resolvido via `GET /api/v3/order?origClientOrderId=...`. |
| Erro `-1021 Timestamp for this request was ahead/behind` | `src/core/binance_client.py` | Deriva de clock na VPS. Verificar log de `sync_monotonic_clock()` e latência RTT com a Binance. |
| Robô não abriu operações durante várias horas seguidas | `src/regimes/regime_detector.py` | Verificar no Telegram se os ativos estão em regime `BEAR` ou na zona de proteção `TRANSITION` (50 a 55 de CHOP). |

---

## 8. Árvore de Arquivos do Projeto (v6.0)

```text
quant-crypto-chameleon/
├── config/
│   └── strategies.yaml                     # Regras MECE, tetos de risco, custos e Feature Flags (Fase 1 vs Fase 2)
├── notebooks/                              # Bloco 3: Validação Científica no Google Colab
│   ├── 01_download_causal_1h_4h_1m.ipynb   # Baixa histórico alinhado por close_time_utc (1h, 4h e 1m)
│   ├── 02_ablation_and_dsr_sim.ipynb       # Compara Degraus A, B, C, D e E + Deflated Sharpe Ratio e 3 Custos
│   └── 03_tail_stress_cppi_cusum.ipynb     # Testa gaps (CVaR 99%), CPPI Drawdown Control e CUSUM Shadow Mode
├── models/
│   └── calibrated_bull_meta_model.joblib   # Modelo segregado por regime com calibração Isotônica (ECE <= 0.05)
├── src/                                    # Motor Modular de Produção
│   ├── core/
│   │   ├── config.py                       # Validador de schema (Pydantic) para .env e strategies.yaml
│   │   ├── database.py                     # SQLite em modo WAL + synchronous=FULL + rotina de backup externo
│   │   ├── binance_client.py               # Ed25519, Clock Monotônico (RTT/2), cache exchangeInfo e Decimal
│   │   └── state_reconciler.py             # Cura no boot: origClientOrderId, OCO parcial e fallback MARKET
│   ├── data/
│   │   └── candle_collector.py             # Coleta em XX:00:03 UTC e merge_asof estrito por close_time_utc
│   ├── features/
│   │   ├── indicators.py                   # EMA, ATR (Wilder), RSI (Wilder), Bollinger, ADX (Wilder), CHOP (log10)
│   │   └── ml_judge.py                     # Calcula features causais, ECE e Valor Esperado Líquido E[R]
│   ├── regimes/
│   │   └── regime_detector.py              # Partição MECE em 4 estados: BEAR, BULL, SIDEWAYS, TRANSITION
│   ├── strategies/
│   │   ├── base_strategy.py                # Interface padrão (Entrada, Stop, Alvo, R:R Efetivo)
│   │   ├── bull_trend_pullback.py          # Clima BULL (Ativo na Fase 1): Recuo na EMA 20
│   │   ├── sideways_pingpong.py            # Clima SIDEWAYS (Fase 2): Banda Inf. + RSI < 32 + filtro RR >= 1.5
│   │   └── bear_capitulation_snap.py       # Clima BEAR (Quarentena): Exige filtro de rejeição + validação OOS
│   ├── risk/
│   │   ├── position_sizer.py               # Teto Duplo, trava de expectância p > p_min e desconto do hodl_bucket
│   │   ├── shadow_manager.py               # CPPI Drawdown Control (reduz lote em 50%) e CUSUM Shadow Mode
│   │   └── profit_splitter.py              # Acumula 10 USDT e transfere direto para carteira FUNDING (MAIN_FUNDING)
│   ├── execution/
│   │   └── oco_order_manager.py            # POST /api/v3/orderList/oco, quantidade líquida e Time-Stop atômico
│   ├── notifications/
│   │   └── telegram_reporter.py            # Transparência de regime, tamanho do lote, offset de clock e saúde
│   └── main.py                             # Maestro de ciclo horário + verificador de manutenção da corretora
├── tests/
│   ├── test_block1_causal_and_decimal.py
│   ├── test_block2_talib_and_mece.py
│   ├── test_block4_dual_cap_and_cppi.py
│   └── test_block5_reconciler_and_oco.py
├── .env.example
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md

```

---

## 9. Arquivo Central de Configuração (`config/strategies.yaml` — v6.0)

Copie o conteúdo abaixo para `config/strategies.yaml`:

```yaml
# ==============================================================================
# QUANTCRYPTO CHAMELEON ENGINE (v6.0) - CONFIGURAÇÃO CENTRAL AUDITADA
# ==============================================================================

version: "6.0.0"
rollout_phase: "PHASE_1_CORE_VALIDATION"    # Fase 1: Apenas 1 estratégia validada, risco fixo 1%, sem complexidade prematura

portfolio:
  core_hold_asset: "XRP"
  core_hold_amount: 150.0                   # 150 XRP fora da chave Spot (Risco operacional zero; exposto ao risco de mercado do XRP)
  tactical_quote_asset: "USDT"              # ~145.5 XRP convertidos em USDT para giro tático na carteira Spot (MAIN)
  fee_reserve_asset: "BNB"
  min_bnb_fee_reserve_usdt_equiv: 5.00      # Reserva intocável de BNB (~4.5 XRP) para garantir taxa de 0.075% e evitar erro -2010 na OCO
  max_concurrent_positions: 1               # Modo Sniper Single-Slot: 1 posição aberta por vez
  min_notional_safety_buffer: 1.05          # Exige lote >= 1.05 * minNotional dinâmico lido de /api/v3/exchangeInfo (aborta se menor)
  max_single_order_usdt_cap: 500.00         # Disjuntor de segurança máximo por ordem individual
  max_daily_trades_cap: 6                   # Disjuntor contra loop infinito de ordens em caso de anomalia

execution_and_resilience:
  clock_sync_interval_minutes: 15           # Ressincroniza offset monotônico (Cristian RTT/2) a cada 15 min
  clock_safety_margin_ms: 200               # Subtrai 200ms para evitar erro -1021 (Timestamp ahead of server)
  recv_window_ms: 5000                      # Janela de validade de assinatura Ed25519
  candle_close_fetch_delay_seconds: 3       # Aguarda XX:00:03 UTC para garantir fechamento completo da vela no agregador Binance
  candle_fetch_max_retries: 5               # Retentativas de 1s caso close_time da vela [-2] ainda não tenha virado
  oco_endpoint: "/api/v3/orderList/oco"     # Endpoint oficial atual da Binance Spot
  oco_below_type: "STOP_LOSS_LIMIT"         # Opções: "STOP_LOSS_LIMIT" (com buffer) ou "STOP_LOSS" (stop a mercado puro)
  oco_stop_limit_buffer_pct: 0.025          # 2.5% abaixo do stopPrice para mitigar salto de livro em flash-crashes
  fallback_to_market_on_stop_breach: true   # Vende a MARKET no mesmo segundo se preço atual <= stopPrice (-2010)
  sqlite_wal_mode: true                     # PRAGMA journal_mode=WAL; PRAGMA synchronous=FULL;
  daily_sqlite_backup_enabled: true         # Backup criptografado diário contra perda de disco em VPS efêmera

position_sizing:                            # Dimensionamento Fracionário Fixo com Teto Duplo e Trava de Expectância
  mode: "FIXED_FRACTIONAL_DUAL_CAP"         # Substitui o antigo rótulo impreciso de "Kelly"
  default_grade_b_risk_pct: 0.010           # Fase 1 / Sinal Nota B: Arrisca estritamente 1.0% da Carteira Tática no Stop
  default_grade_b_max_cash_pct: 0.50        # Teto máximo de alocação de caixa para Sinal Nota B = 50% do USDT operável
  conviction_sizing_enabled: false          # Começa DESLIGADO na Fase 1; só liga na Fase 2 se Degrau D vencer com DSR > 0.95
  grade_a_plus_risk_pct: 0.020              # Fase 2 (Sinal Nota A+): Arrisca no máximo 2.0% da Carteira Tática (sujeito a CVaR 99% <= 5%)
  grade_a_plus_max_cash_pct: 0.85           # Reduzido de 95% para 85% para absorver gaps de cauda e preservar margem
  min_expectancy_margin_over_breakeven: 0.03 # Exige P_calibrada > p_indiferenca + 3% para aprovar qualquer operação
  roundtrip_fee_assumption_pct: 0.0015      # 0.15% ida e volta (Taker/Taker com BNB)
  stress_slippage_assumption_pct: 0.0006    # 0.06% de slippage base embutido no cálculo da distância efetiva do stop

mid_trade_management:
  breakeven_enabled: false                  # DESLIGADO na Fase 1 (evita violino sob deriva positiva e race condition sem WebSocket)
  breakeven_trigger_r: 1.2                  # Avaliado exclusivamente como Degrau E no Bloco 3 com velas de 1 minuto (1m)
  require_realtime_websocket_for_be: true   # Proíbe ligar breakeven se o loop assíncrono de bookTicker não estiver ativo

drawdown_and_shadow_protection:             # Defesa contra meses adversos (substitui a regra frágil de 2 vitórias / +2.0R)
  protection_mode: "CPPI_DRAWDOWN_CONTROL"  # Opções: "CPPI_DRAWDOWN_CONTROL", "CUSUM_SHADOW_MODE", "DISABLED"
  cppi_drawdown_threshold_pct: 0.04         # Se a Carteira Tática cair 4.0% abaixo do topo (High-Water Mark)...
  cppi_risk_reduction_factor: 0.50          # ...corta o risco por operação pela metade (de 1.0% para 0.5%) até recuperar o topo
  shadow_mode_enabled: false                # Modo Fantasma binário desligado na Fase 1 (testado via CUSUM no Notebook 03)
  shadow_max_consecutive_live_losses: 3     # Se modo fantasma for usado no futuro, exige 3 stops ou gatilho CUSUM
  shadow_virtual_wins_to_reactivate: 4      # Unificado: exige 4 trades virtuais com E[R] > 0 (removido o atalho de +2.0R em 1 trade)

profit_splitter:                            # Alimentador blindado da carteira de longo prazo
  enabled: false                            # Começa DESLIGADO na Fase 1 até comprovar lucro real fora da amostra
  reinvest_bot_pct: 0.70                    # 70% do lucro líquido permanece no caixa tático (juros compostos)
  send_to_hodl_pct: 0.30                    # 30% acumula em pending_hodl_bucket_usdt (descontado do caixa operável do robô!)
  min_bucket_to_transfer_usdt: 10.00        # Ao somar >= 10 USDT, transfere direto para a carteira FUNDING
  transfer_method: "MAIN_TO_FUNDING_USDT"   # Usa POST /sapi/v1/asset/transfer (type="MAIN_FUNDING") — zero saldo órfão na Spot!
  hodl_target_reference_asset: "XRP"        # Ativo de referência contábil para conversão posterior na carteira Funding

universe:
  backtest_min_start_date: "2022-01-01"     # Mitiga survivorship bias garantindo liquidez consolidada nos 7 pares
  phase_1_tie_breaker: "rs_vol_adjusted_12h" # Fase 1 (apenas Bull ativo): (Close[t] - Close[t-12]) / ATR_14[t]
  phase_2_tie_breaker: "net_expected_r"     # Fase 2 (múltiplos regimes): E[R_i] = P_cal * R_target - (1 - P_cal) - C_R
  symbols:                                  # BNBUSDT removido propositalmente para jamais liquidar a reserva de taxas em BNB
    - "BTCUSDT"
    - "ETHUSDT"
    - "SOLUSDT"
    - "XRPUSDT"
    - "ADAUSDT"
    - "LINKUSDT"
    - "AVAXUSDT"

regimes_and_strategies:
  macro_compass_timeframe: "4h"             # Alinhado estritamente por close_time_utc (backward merge_asof)
  execution_timeframe: "1h"                 # Operado no fechamento confirmado da vela de 1h
  intracandle_sim_timeframe: "1m"           # Resolução obrigatória no Colab para eliminar viés de trajetória intra-vela

  regime_thresholds_4h:                     # Partição MECE de 4 Estados: BEAR (1º) -> BULL (2º) -> SIDEWAYS (3º) -> TRANSITION (4º)
    ema_macro_period: 100
    chop_period: 14                         # Calculado na base log10 canônica
    chop_bull_max: 50.0                     # Abaixo de 50 = tendência limpa
    chop_sideways_min: 55.0                 # Acima ou igual a 55 = consolidação lateral (entre 50 e 55 cai em TRANSITION)
    adx_period: 14                          # Calculado com suavização RMA de Wilder (TA-Lib)
    adx_sideways_max: 22.0                  # Abaixo de 22 confirma ausência de força direcional

  bull_trend_pullback:                      # CLIMA BULL (Sol Aberto) — ÚNICA ESTRATÉGIA ATIVA NA FASE 1
    enabled: true
    ema_fast_1h: 20
    ema_slow_1h: 100
    stop_atr_mult: 2.0                      # Stop inicial em 2.0 * ATR_14 (-1.0R)
    target_atr_mult: 5.0                    # Alvo em 5.0 * ATR_14 (+2.5R bruto)
    time_barrier_candles_1h: 24             # Saída atômica por tempo após 24 horas (validada contra decaimento de E[R])

  sideways_pingpong:                        # CLIMA SIDEWAYS (Pista Ensaboada) — DESLIGADO NA FASE 1
    enabled: false
    bollinger_period: 20
    bollinger_std: 2.0
    rsi_period: 14
    rsi_oversold_entry: 32.0
    stop_atr_mult: 1.5
    target_band: "middle_bollinger"         # Alvo fixado no valor da SMA_20 no instante da entrada
    min_initial_rr_ratio: 1.50              # TRAVA GEOMÉTRICA v6: Bloqueia entrada se (SMA_20 - Entry) / (1.5 * ATR_14) < 1.5
    time_barrier_candles_1h: 12

  bear_capitulation_snap:                   # CLIMA BEAR (Tempestade) — EM QUARENTENA (Alto risco de continuação em Bear Market)
    enabled: false                          # Mantém 100% em USDT durante Bear Market até provar alpha OOS com filtro de exaustão
    rsi_fast_period: 2
    rsi_panic_threshold: 5.0
    relative_volume_min: 3.0
    min_lower_wick_ratio: 0.40              # Exige pavio inferior >= 40% da vela de 1h provando rejeição compradora real
    stop_atr_mult: 1.8
    target_atr_mult: 3.6                    # Alvo de repique (+2.0R)
    time_barrier_candles_1h: 8

ml_judge_and_validation:
  use_ml_filter: false                      # Começa DESLIGADO na Fase 1; exige vencer Degrau C no Colab
  segregate_models_by_regime: true          # Proíbe misturar estratégias com razões R:R diferentes no mesmo modelo
  sample_uniqueness_method: "afml_mean_uniqueness" # Média de 1/c_t ao longo de toda a janela [t_entry, t_exit]
  purge_window_hours: 24                    # Expurgo de 24h entre treino e validação/teste
  embargo_pct: 0.02                         # Embargo de 2% após cada fold de teste
  calibration_method: "isotonic"            # Calibração explícita de probabilidade (exige ECE <= 0.05 fora da amostra)
  max_allowed_ece: 0.05
  ranking_metric: "net_expected_r"          # Classifica tercis por E[R] líquido, nunca por P(win) bruto
  reject_bottom_quantile: 0.33              # Tercil inferior (ou E[R] <= 0) = Nota C (Ignora)
  grade_a_plus_top_quantile: 0.70           # Top 30% de E[R] (com P_cal > p_indiferenca + 0.03) = Nota A+
  min_deflated_sharpe_ratio: 0.95           # Exige DSR >= 0.95 descontando o número total de trials testados no Colab

```
