A tabela abaixo resume todas as diferenças estruturais, matemáticas e operacionais entre a versão original (**v5.0**) e a versão auditada (**v6.0**).

| Área | Componente | Versão v5.0 | Versão v6.0 | Impacto no Bolso |
| --- | --- | --- | --- | --- |
| **Patrimônio** | Divisão dos 300 XRP | 150 Core e 150 Tática | 150 Core, 145,5 Tática, 4,5 BNB | Evita falta de BNB |
| **Patrimônio** | Risco do Core Hold | Tratado como risco zero | Medido via VaR e CVaR | Transparência de queda |
| **Patrimônio** | Par `BNBUSDT` | Operado normalmente | Proibido na cesta | Protege BNB de taxas |
| **Patrimônio** | Cofre de Lucros | Compra Spot misturada | Transfere para `FUNDING` | Reconciliador não vende Hold |
| **Patrimônio** | Caixa do Cofrinho | Misturado no USDT livre | Subtraído do caixa operável | Robô não gasta o cofre |
| **Dados e Clima** | Alinhamento 4h e 1h | Indefinido (risco de futuro) | Estrito por `close_time_utc` | Fim de lucro falso no teste |
| **Dados e Clima** | Indicadores CHOP e ADX | Sem fórmula padronizada | Base 10 e RMA de Wilder | Matemática igual ao TA-Lib |
| **Dados e Clima** | Regimes de Mercado | 3 estados com buracos | 4 estados MECE sem lacunas | Fica em dólar na dúvida |
| **Estratégias** | Ativação Inicial | As 3 ligadas juntas | Fase 1 opera apenas `BULL` | Foco no que tem vantagem |
| **Estratégias** | Lateral (`Sideways`) | Entra sem checar distância | Exige alvo mínimo de 1,5R | Não arrisca muito por pouco |
| **Estratégias** | Tempestade (`Bear`) | Compra $\text{RSI}_2$ abaixo de 5 | Desligada em quarentena | Não compra moedas falindo |
| **Estratégias** | Desempate de Moedas | Sempre por força ($\text{RS}$) | Por lucro esperado $\mathbb{E}[R]$ | Escolhe o melhor ganho |
| **Risco e Lote** | Cálculo de Lote | Falso Kelly de até 95% | Teto duplo fixo (1% a 2%) | Nunca perde mais que o limite |
| **Risco e Lote** | Trava de Qualidade | Top 30% ganha mão máxima | Exige lucro esperado positivo | Não aposta alto em sinal ruim |
| **Risco e Lote** | Breakeven em $+1,2R$ | Sempre ligado em 1h | Desligado na Fase 1 | Não corta tendências cedo |
| **Risco e Lote** | Defesa em Mês Ruim | Modo Fantasma (2 stops) | Corte gradual CPPI (-50%) | Não perde a retomada |
| **Risco e Lote** | Religamento Fantasma | 2 vitórias ou $+2,0R$ | Mínimo 4 trades via CUSUM | Corrige bug de 1 trade só |
| **IA e Backtest** | Modelo de IA | 1 juiz único para os 3 climas | 1 modelo separado por clima | Não mistura alvos diferentes |
| **IA e Backtest** | Peso de Treino | Olha só a hora de entrada | Média durante todo o trade | Treino sem sinais repetidos |
| **IA e Backtest** | Calibração da IA | Apenas divisão por tercis | Calibração Isotônica (ECE) | Probabilidade real confiável |
| **IA e Backtest** | Simulação de Custos | Taxa única fixa de 0,20% | 3 cenários mais gaps de cauda | Testa sobrevivência em pânico |
| **IA e Backtest** | Validação Estatística | Escolhe o melhor gráfico | Exige Deflated Sharpe ($\text{DSR}$) | Evita sorte de backtest |
| **Execução** | Preços e Quantidades | Números `float` comuns | `Decimal` e `exchangeInfo` vivo | Zero erro de arredondamento |
| **Execução** | Quantidade da OCO | Usa total bruto comprado | Desconta comissão recebida | Evita recusa da ordem OCO |
| **Execução** | Preço Abaixo do Stop | OCO falha e fica sem stop | Vende a `MARKET` na hora | Salva a conta em quedas |
| **Execução** | Saída por Tempo | Vende direto no Time Stop | Cancela OCO e checa parciais | Não trava por saldo preso |
| **Infraestrutura** | Queda de Rede na Compra | Risco de comprar duplicado | Consulta `origClientOrderId` | Compra exatamente uma vez |
| **Infraestrutura** | Relógio e Banco SQLite | Hora local e banco simples | Clock monotônico, WAL e backup | Zero erro `-1021` ou perda |
