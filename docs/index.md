# QuantCrypto Chameleon Engine v6

## Visão geral

O **QuantCrypto Chameleon Engine** é uma arquitetura para trading sistemático
adaptativo na Binance Spot, com execução em janelas de 1h/4h, sem alavancagem e
com foco em preservação de capital. O sistema alterna sua atuação conforme o
regime de mercado, mas só avança de uma etapa operacional para outra depois de
passar por critérios estatísticos e operacionais explícitos.

As decisões centrais do projeto são:

- separar patrimônio de longo prazo, capital tático e reserva operacional;
- evitar look-ahead bias por alinhamento temporal estrito das velas;
- limitar risco por trade, por ordem e por drawdown;
- tratar ordens, rede, relógio e banco de dados como componentes de risco;
- validar custos, gaps, calibração e robustez antes do uso com dinheiro real;
- operar com gates `PASS`/`FAIL` e downgrade automático em caso de degradação.

## Navegação

Use a documentação para consultar cada parte da especificação:

1. **Arquitetura completa:** regras patrimoniais, regimes, risco, IA,
   execução e configuração.
2. **Metodologia e roadmap:** blocos de construção, gates de aprovação,
   transições de fase e kill-switches.
3. **Migração v5 → v6:** alterações que motivaram a versão atual.
4. **TODO:** triagem de riscos e melhorias pendentes.

> **Aviso:** este material é uma especificação técnica. Não constitui
> recomendação financeira e não autoriza operação real sem a validação dos
> gates descritos na metodologia.
