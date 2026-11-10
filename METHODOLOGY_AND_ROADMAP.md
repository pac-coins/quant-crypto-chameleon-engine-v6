# Metodologia de Engenharia, Governança Científica e Cronograma de Etapas — QuantCrypto Chameleon Engine (v6.0)

Este documento define o protocolo determinístico de construção, validação estatística, transição entre fases operacionais e resposta a incidentes do **QuantCrypto Chameleon Engine (v6.0)**. Nenhuma linha de código avança de bloco e nenhum centavo adicional é arriscado sem o cumprimento binário (`PASS` / `FAIL`) dos critérios aqui estabelecidos.

---

## 1. Visão Geral: A Diferença entre "Blocos de Construção" e "Fases de Operação"

Para que nunca haja confusão sobre onde o projeto está, separamos o ciclo de vida em duas réguas independentes:

* **Explicação Intuitiva (Como funciona na prática):**
Imagine que você está construindo um avião.
1. Os **6 Blocos de Construção (Estágios 1 a 6)** são a **fábrica no chão**: você monta o motor (Bloco 1), coloca as asas (Bloco 2), testa no túnel de vento (Bloco 3), instala o paraquedas (Bloco 4), liga na tomada de teste da oficina (Bloco 5) e faz um voo de teste sem passageiros com controle remoto (Bloco 6). Você ainda não colocou dinheiro real em risco.
2. As **3 Fases de Operação (Fases 1, 2 e 3)** começam **depois que o avião já está pronto e aprovado para voar**: na **Fase 1**, você só voa em dias de céu azul (`BULL`) e baixa altitude (risco fixo de 1%). Só depois de completar 60 voos reais com lucro comprovado é que você recebe a carteira de habilitação para a **Fase 2** (ligar o piloto automático de IA e o Cofre de Lucros). E só depois de 100 voos lucrativos você libera a **Fase 3** (voar em dias nublados / mercado lateral).



```text
================================================================================
ETAPA A: FÁBRICA E VALIDAÇÃO (ZERO DINHEIRO REAL EM RISCO)
================================================================================
[Bloco 1: Fundação] ➔ [Bloco 2: Clima MECE] ➔ [Bloco 3: Túnel de Vento (Colab)]
                                                       │ (Se PASS no Gate 3)
[Bloco 6: Shadow Live] ◄── [Bloco 5: Testnet OCO] ◄── [Bloco 4: Risco e Freios]
       │
       ▼ (Se PASS no Gate 6A e 6B)
================================================================================
ETAPA B: OPERAÇÃO COM CAPITAL REAL (EVOLUÇÃO POR MÉRITO COMPROVADO)
================================================================================
[FASE 1: MVP Tendência Pura] ──(Gate F1->F2)──► [FASE 2: +IA Silenciosa Validada + Cofre] ──(Gate F2->F3)──► [FASE 3: Multi-Clima]

```

---

## 2. Governança Científica Anti-Ilusão (Regras do Bloco 3 contra *Overfitting*)

O maior risco em trading algorítmico é testar dezenas de combinações no simulador até achar uma que deu lucro por pura sorte no passado (*p-hacking* / *Data Snooping*) e perder dinheiro no futuro. Para impedir isso, o Bloco 3 obedece a **4 Leis Inegociáveis**:

### 2.1. Partição Rígida do Histórico (`2022-01-01` até o presente)

Os dados históricos das 7 moedas (`BTCUSDT`, `ETHUSDT`, `SOLUSDT`, `XRPUSDT`, `ADAUSDT`, `LINKUSDT`, `AVAXUSDT`) são divididos cronologicamente em três cofres:

1. **In-Sample / Treino (Primeiros 60% do tempo):** Único período onde é permitido explorar dados, treinar modelos e ajustar parâmetros.
2. **Validation / Walk-Forward (25% seguintes do tempo):** Usado para validação cruzada com expurgo (*Purging* de 24h e *Embargo* de 2%) e calibração de probabilidades (*Platt Scaling*).
3. **Holdout Intocável (Últimos 15% do tempo — mínimo de 6 meses finais):** Este período fica trancado. Ele só pode ser executado **uma única vez** quando o código e os parâmetros estiverem 100% congelados.

### 2.2. Orçamento Máximo de Tentativas (*Trial Budget*)

* Toda vez que o notebook do Bloco 3 rodar uma simulação para testar uma ideia ou ajuste de parâmetro, o código grava automaticamente uma linha no arquivo `logs/backtest_trials_registry.csv` contendo: timestamp, hash dos parâmetros, Sharpe Ratio e retorno em $R$.
* **Limite de Tentativas ($K_{\max} = 20$):** São permitidas no máximo 20 execuções exploratórias no conjunto de Validação. O número total $K$ registrado nesse arquivo é inserido obrigatoriamente na fórmula do **Deflated Sharpe Ratio ($\text{DSR}$)** de Bailey & López de Prado (2014).

### 2.3. Critério de Abandono (O que fazer se a estratégia reprovar no Bloco 3?)

* **Se a estratégia reprovar no conjunto de Validação:** Você pode alterar a lógica da estratégia (ex.: exigir confirmação de volume ou mudar a regra de entrada) apenas dentro dos dados de Treino/Validação, desde que $K \le 20$.
* **Se a estratégia reprovar no cofre final de Holdout (últimos 6 meses):** É **estritamente proibido** voltar atrás, mudar um número no `strategies.yaml` (ex.: trocar a EMA de 20 para 25) e rodar o Holdout de novo! Se reprovar no Holdout, a estratégia atual está **reprovada e descartada**. Para testar uma nova estratégia do zero, será obrigatório aguardar novos dados ou mudar estruturalmente a família da estratégia, zerando o processo sob nova hipótese.

---

## 3. Cronograma Sequencial de Construção (Blocos 1 a 6 e Seus Gates)

Você (e a IA programadora) só podem escrever a primeira linha de código do Bloco $N+1$ quando o comando de teste do **Gate $N$** retornar `100% PASS`.

| Bloco | Arquivos Construídos | Regras Obrigatórias Embutidas (Categorias 1 e 2) | Gate de Aprovação (Critério Binário) |
| --- | --- | --- | --- |
| **Bloco 1: Dados, Relógio e Banco** | `config.py`, `database.py`, `binance_client.py`, `candle_collector.py`, `test_block1.py` | 1. SQLite com `WAL`, `synchronous=FULL` e rotina nativa `sqlite3.backup()`. 2. `Decimal` em preços (`tickSize` direcional) e lotes (`stepSize` `ROUND_DOWN`). 3. Sincronização monotônica (`RTT/2`). 4. Normalização do `close_time` para o segundo exato seguinte (`+1ms`) antes do `merge_asof`. 5. Mock automático para chamadas `/sapi/*` quando `TESTNET=True`. | **Gate 1:** `pytest tests/test_block1.py` aprova: (a) teste de truncamento às `02:00 UTC` com diferença zero no vetor alinhado; (b) 10.000 preços e lotes aleatórios múltiplos exatos de `tickSize`/`stepSize`; (c) restauração real de um arquivo gerado via `sqlite3.backup()`. |
| **Bloco 2: Indicadores e Clima MECE** | `indicators.py`, `regime_detector.py`, `test_block2.py` | 1. `ATR`, `RSI` e `ADX` com suavização RMA de Wilder. 2. `CHOP` na base $\log_{10}$ com proteção contra divisão por zero ($\text{High} == \text{Low} \rightarrow \text{NaN}$). 3. Qualquer `NaN`, infinito ou histórico menor que 120 velas força retorno imediato para `TRANSITION` (100% em USDT). 4. Ordem MECE: `BEAR` $\rightarrow$ `BULL` $\rightarrow$ `SIDEWAYS` $\rightarrow$ `TRANSITION`. | **Gate 2:** `pytest tests/test_block2.py` aprova: (a) correlação maior que $0,9999$ com `TA-Lib` após período de aquecimento (*warm-up*); (b) série artificial de preços constantes (divisão por zero) retorna `TRANSITION` sem levantar exceção; (c) 100% de cobertura disjunta nos testes de fronteira. |
| **Bloco 3: Simulador Causal e Ablação** | `01_download.ipynb`, `02_ablation_dsr.ipynb`, `03_stress.ipynb` | 1. Simulação da fila Single-Slot descontando taxas assimétricas (*maker* no alvo, *taker* no stop/tempo). 2. Cálculo empírico de $\mathbb{E}[R]$ incluindo o resultado real das saídas por tempo (`24h`). 3. Resolução de 1 minuto (`1m`) para testar se o Breakeven (Degrau E) ajuda ou atrapalha. 4. Registro automático no `trials_registry.csv`. | **Gate 3 (No Colab):** Para aprovar a ida do Degrau C (`bull_trend_pullback` puro) para os próximos blocos: (a) Mínimo de **150 trades** fora da amostra; (b) Expectância líquida $\bar{R} \gt +0,10R$ com limite inferior do Intervalo de Confiança de 95% (*Block Bootstrap*) acima de $0,0R$; (c) *Profit Factor* maior ou igual a $1,20$ no cenário de custo Base e maior que $1,05$ no cenário Pessimista (slippage dobrado); (d) Lucro positivo em pelo menos 3 de 4 subperíodos e no Holdout de 6 meses; (e) $\text{DSR} \ge 0,95$. |
| **Bloco 4: Cérebro de Risco e Lote** | `position_sizer.py`, `shadow_manager.py`, `profit_splitter.py`, `test_block4.py` | 1. Teto duplo: risco de $1,0\%$ no stop e máximo de $50\%$ do caixa operável. 2. Disjuntor de ordem máxima proporcional: $\min(500 \text{ USDT}, 55\% \text{ do caixa})$. 3. Reserva mínima de `$5,00 USDT` em BNB verificada via `/api/v3/account/commission`. 4. Desconto de `pending_hodl_bucket_usdt` do caixa livre. 5. Regra de *High-Water Mark* no Cofre (só separa 30% do lucro *acima* do topo histórico da carteira tática). | **Gate 4:** `pytest tests/test_block4.py` aprova: (a) stop largo de $6\%$ limita a compra a $\sim 16\%$ do caixa (jamais furando $1\%$ de perda); (b) lote menor que `1,05 * minNotional` retorna `0.0` (abortado); (c) lucro após um drawdown que ainda não recuperou o topo histórico gera transferência zero para o Cofre; (d) queda de $4\%$ reduz o lote seguinte exatamente pela metade ($0,5\%$). |
| **Bloco 5: Execução OCO e Cura no Boot** | `oco_order_manager.py`, `state_reconciler.py`, `test_block5.py` | 1. Gravação `PENDING_ENTRY` antes do envio e consulta obrigatória por `origClientOrderId` em caso de Timeout (`-1007`). 2. Quantidade da OCO descontando comissão no ativo base. 3. Tratamento cirúrgico do `-2010`: vende a `MARKET` imediatamente se $P_{\text{bid}} \le P_{\text{stop}} \times 1,0005$ ou mensagem `"trigger immediately"`. 4. *Watchdog* de Stop Pendurado: se a perna de stop ativou mas o preço pulou o `stopLimitPrice`, cancela e vende a `MARKET`. 5. Fechamento serializado de 4 passos no Time-Stop e cura de OCO parcialmente executada. | **Gate 5 (Simulação de Falhas):** `pytest tests/test_block5.py` + Teste na Binance Spot Testnet aprovam os **4 Testes de Tortura**: (1) Timeout simulado na compra recupera a ordem via `origClientOrderId` sem comprar duplicado; (2) Compra sem BNB desconta a taxa do ativo e cria a OCO sem erro `-2010`; (3) Queda de energia simulada com preço abaixo do `stopPrice` dispara venda a `MARKET` no 1º segundo do boot; (4) Execução parcial de 40% da OCO faz o reconciliador proteger os 60% restantes. |
| **Bloco 6: Observabilidade e Voo Assistido** | `telegram_reporter.py`, `main.py`, `Dockerfile`, `docker-compose.yml` | 1. Alertas claros no Telegram (clima atual, distância da EMA, tamanho do lote, status do relógio e reserva de BNB). 2. Verificação de manutenção da Binance (`/sapi/v1/system/status`) bloqueando entradas 2h antes. 3. Container Docker com reinício automático e backup diário consistente. | **Gate 6A (Engenharia — 7 dias na Testnet)** e **Gate 6B (Shadow Live — 30 dias/trades em dados reais)** descritos na Seção 4 abaixo. |

---

## 4. Como Saber Exatamente Quando Mudar de Fase (Critérios de Graduação)

Esta seção responde de forma matemática e sem subjetividade à pergunta: **"Quando eu viro a chave de uma fase para a outra?"**

### 4.1. Transição 0 $\rightarrow$ Fase 1: Da Testnet para o Dinheiro Real (Os 145,5 XRP em USDT)

Você **não** coloca dinheiro real logo após terminar de programar o Bloco 6. A passagem para a Fase 1 exige dois carimbos sequenciais:

1. **Gate 6A — Estabilidade de Engenharia (7 Dias Contínuos na Binance Spot Testnet):**
* Zero exceções não tratadas (`Unhandled Exceptions`) nos logs do Docker em 168 ciclos horários consecutivos.
* Offset do relógio monotônico mantido sempre dentro de $\pm 500\text{ ms}$.
* **Teste de Restauração de Desastre:** Ao final do 7º dia, você apaga o container, restaura o arquivo de backup do SQLite em uma pasta limpa e inicia o robô — ele deve reconhecer todo o histórico e estado sem erros.


2. **Gate 6B — Estabilidade de Microestrutura em Produção (*Paper Trading / Shadow Live* por 30 Dias ou 25 Trades):**
* O robô é ligado na VPS apontando para os **dados reais da Binance Produção**, mas com a chave `SHADOW_PAPER_ONLY=True` (ou operando o lote mínimo de $6 USDT por trade apenas para medir execução).
* **Critério de Aprovação:** A diferença entre o preço de execução esperado pelo simulador e o preço real do livro de ofertas da Binance (*slippage* realizado) deve ser, na média, menor ou igual a $0,08\%$, e a taxa de trades gerados por semana deve ser compatível com a distribuição histórica do Bloco 3.



---

### 4.2. O Que Roda na Fase 1 (Configuração Congelada do MVP Real)

Quando o Gate 6B é aprovado, você vira a chave para a **Fase 1** com os seus ~145,5 XRP em USDT e ~4,5 XRP em BNB:

* **Ativo:** Apenas o sensor de 4h (`BEAR`, `BULL`, `SIDEWAYS`, `TRANSITION`) e **apenas 1 estratégia** (`bull_trend_pullback` quando o clima é `BULL`). Nos outros 3 climas, o robô fica 100% protegido em USDT.
* **Lote:** Risco fixo de **1,0% da banca tática no Stop Loss** (ou 0,5% se o drawdown passar de 4%), limitado a no máximo 50% do caixa.
* **Desligados para proteção:** Breakeven ($+1,2R$) desligado, Cofre de Lucros (`profit_splitter`) desligado, e Juiz de IA (`ml_judge`) rodando **apenas em modo observador silencioso** (ele calcula a probabilidade e grava a nota C, B ou A+ numa coluna do SQLite, mas **não bloqueia nem altera o tamanho do lote**).

---

### 4.3. Transição Fase 1 $\rightarrow$ Fase 2: Quando Ligar o Cofre de Lucros e o Juiz de IA?

*Explicação simples:* Enquanto o robô opera na Fase 1, o Juiz de IA fica sentado no banco do passageiro anotando num caderno: *"Nesse trade eu teria dado Nota C (bloqueado); naquele eu teria dado Nota A+"*. Quando você completar 60 trades reais, você abre o caderno do Juiz: se os trades que ele chamou de "Nota C" realmente deram prejuízo na vida real e os "Nota A+" deram lucro, o Juiz provou que sabe o que está fazendo no mercado ao vivo e ganha permissão para dirigir na Fase 2!

Você **só** muda o `strategies.yaml` da Fase 1 para a Fase 2 quando **TODOS os 5 requisitos abaixo** forem verdadeiros ao mesmo tempo:

| # | Requisito para Subir para a Fase 2 | Métrica Exata Exigida (Lida do Banco SQLite) |
| --- | --- | --- |
| **1** | **Amostra Mínima Ao Vivo** | Pelo menos **60 operações reais fechadas** na Fase 1 (e mínimo de 60 dias corridos de operação). |
| **2** | **Crescimento Patrimonial Real** | A Carteira Tática em USDT está acima do seu valor inicial (lucro líquido acumulado positivo após todas as taxas de BNB) e com *Profit Factor* real $\ge 1,15$. |
| **3** | **Expectância Real Positiva** | Média realizada por operação $\bar{R}_{\text{live}} \ge +0,10R$, com *Max Drawdown* real registrado menor que $10\%$. |
| **4** | **Zero Falhas Graves de Execução** | Zero casos de posições órfãs não recuperadas pelo reconciliador e slippage médio real dentro do limite de estresse ($s_{\text{real}} \le 0,10\%$). |
| **5** | **Prova Ao Vivo da IA Silenciosa** | Nos 60+ trades reais gravados no SQLite, o grupo que a IA marcou silenciosamente como **Nota B/A+** teve média $\bar{R}$ superior ao grupo marcado como **Nota C**, com erro de calibração $\text{ECE} \le 0,05$ (*Platt Scaling*). |

**O que muda na Fase 2 (Em duas subetapas de segurança):**

* **Fase 2A (Primeiros 30 trades da Fase 2):** Liga o **Cofre de Lucros (`profit_splitter.enabled: true`)** com regra de *High-Water Mark* (transferindo 30% dos novos lucros acima do topo histórico para a carteira `FUNDING` via `MAIN_FUNDING` a cada 10 USDT) e liga o **Filtro Defensivo da IA (`ml_judge.use_ml_filter: true`)** apenas para **bloquear os sinais Nota C** (0% de lote), mantendo os sinais Nota B e Nota A+ ainda com risco conservador de **1,0%**.
* **Fase 2B (Após +30 trades mantendo $\bar{R} \gt +0,15R$ na Fase 2A):** Libera o escalonamento de lote da **Nota A+** para arriscar **1,5% da banca** (e até **2,0%** após mais 30 trades, desde que o CVaR 99% empírico da carteira não ultrapasse 5,0% de perda em um único evento de cauda).

---

### 4.4. Transição Fase 2 $\rightarrow$ Fase 3: Quando Ligar o Clima Lateral (`sideways_pingpong`) ou o Breakeven?

A Fase 3 adiciona novas armas ao robô. Cada peça extra tem seu próprio cadeado individual e **só** pode ser ligada se cumprir seu requisito:

1. **Para ligar `sideways_pingpong` (Operar Mercado Lateral):**
* O robô já deve estar na Fase 2 com pelo menos **100 trades reais acumulados** no histórico total.
* A estratégia `sideways_pingpong` isolada deve ter passado por todos os critérios do **Gate 3** no Colab (mínimo de 150 trades OOS, filtro $\text{RR} \ge 1,5$ líquido de custos, *Profit Factor* $\ge 1,20$ e $\text{DSR} \ge 0,95$).
* A estratégia deve ter rodado por pelo menos **30 operações virtuais em modo *Shadow Live* ao vivo** durante períodos reais de lateralização, apresentando expectância líquida simulada $\bar{R}_{\text{shadow}} \ge +0,10R$.


2. **Para ligar o Breakeven em $+1,2R$ (`breakeven_enabled: true`):**
* Só pode ser ligado se o **Degrau E** do Bloco 3 (simulado com velas de 1 minuto) tiver provado matematicamente que ligar o breakeven aumenta o *Calmar Ratio* e o retorno líquido em comparação ao Degrau C/D sem breakeven, **e** o módulo WebSocket assíncrono (`bookTicker`) tiver rodado por 14 dias sem desconexões não tratadas.


3. **Sobre `bear_capitulation_snap` (Tempestade):**
* Permanece **permanentemente desligada por padrão** (defesa 100% em USDT no regime `BEAR`), a menos que um estudo dedicado futuro com dados ponto-no-tempo (incluindo moedas que sofreram colapso) comprove vantagem estatística líquida.



---

## 5. Protocolo de Regressão Automática (Downgrade) e Disjuntores de Emergência (Kill-Switches)

Um sistema profissional não precisa apenas saber quando acelerar; ele precisa saber quando **reduzir a marcha sozinho** se o mercado mudar de comportamento, e quando **puxar o freio de mão** e chamar você no Telegram.

### 5.1. Tabela de Rebaixamento Automático de Fase (*Auto-Downgrade*)

O módulo `shadow_manager.py` monitora a saúde estatística da operação ao final de cada trade fechado:

| Condição Detectada em Produção | Ação Automática do Robô | Como Recuperar o Status Anterior |
| --- | --- | --- |
| **Drawdown Tático atinge $-4,0\%$** abaixo do Topo Histórico (*High-Water Mark*) | **Ativa `DRAWDOWN_RISK_SCALING`:** Corta o risco de todas as novas operações pela metade (ex.: de $1,0\%$ para $0,5\%$) e pausa transferências do Cofre. | Automático assim que o saldo da Carteira Tática voltar a superar o Topo Histórico. |
| **Na Fase 2:** A média móvel dos últimos **30 trades reais** fica negativa ($\bar{R}_{30} \lt 0,0R$) ou a IA apresenta descalibração ($\text{ECE}_{30} \gt 0,08$) | **Downgrade Automático para Fase 1:** Desliga a alocação extra da Nota A+ (volta tudo para risco base de $0,5\%$ a $1,0\%$) e notifica no Telegram. | Exige nova sequência de 30 trades com $\bar{R}_{30} \ge +0,10R$ e auditoria do modelo no Colab. |
| **Na Fase 3:** A estratégia `sideways_pingpong` acumula **$-3,0R$ líquidos** nos últimos 20 trades laterais | **Desarme Isolado de Clima:** Desliga `sideways_pingpong` (`enabled: false`), voltando a ficar 100% em USDT no clima `SIDEWAYS`, sem interromper o clima `BULL`. | Reavaliação completa no Bloco 3. |

### 5.2. Disjuntores de Parada Total (*Hard Kill-Switches* — Estado `EMERGENCY_HALT`)

O robô interrompe imediatamente **qualquer nova compra** (mantendo a vigilância e proteção das posições que já estiverem abertas) e envia alerta sonoro prioritário no Telegram se ocorrer qualquer um dos 4 eventos abaixo:

1. **Limite Máximo de perda da Carteira Tática (*Hard Stop Patrimonial* de $-12,0\%$):** Se a Carteira Tática cair $12\%$ abaixo do topo histórico (ou abaixo do capital inicial), o robô entra em `HALT` para revisão humana, impedindo qualquer risco de destruição da banca.
2. **Esgotamento da Reserva de Taxas em BNB:** Se o saldo livre de `BNB` cair abaixo do equivalente a **$2,00 USDT**, o robô bloqueia novas entradas até que você recarregue o BNB, garantindo que nenhuma ordem OCO falhe por comissão descontada no ativo base.
3. **Falha Repetida de Reconciliação ou API:** Se `state_reconciler.py` encontrar uma divergência de saldo na corretora que não consiga curar em 3 tentativas consecutivas, ou se a Binance retornar erro de permissão/assinatura (`-1002`, `-1022`, `-2015`).
4. **Anomalia de Frequência (*Runaway Loop Protection*):** Se o sistema tentar disparar mais do que `max_daily_trades_cap` (6 ordens de entrada em 24 horas).

### 5.3. Regra de Ouro da Supervisão Humana (O Que Você Deve e Não Deve Fazer)

* **Proibido intervir por tédio:** Em meses de queda ou mercado travado (`BEAR` ou `TRANSITION`), é normal e esperado que o robô passe **10 a 20 dias seguidos sem fazer nenhuma compra**, mantendo 100% do seu dinheiro protegido em dólar (USDT). **Não diminua os limiares do `strategies.yaml` para "forçar o robô a operar"** — ficar fora de mercados ruins é exatamente a fonte da preservação de capital do Chameleon.
* **Rito de Manutenção Mensal (15 minutos no dia 1º de cada mês):**
1. Verificar na mensagem mensal do Telegram se a reserva de `BNB` está acima de $5,00 USDT.
2. Verificar se o backup automático do SQLite foi salvo corretamente e se o espaço em disco da VPS está abaixo de 70%.
3. Se houver saldo de lucro acumulado na carteira `FUNDING` (na Fase 2), decidir se deseja mantê-lo em dólar ou converter manualmente para `XRP` para somar à sua Carteira Core.



---
