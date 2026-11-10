# QuantCrypto Chameleon Engine v6

O **QuantCrypto Chameleon Engine** é um projeto de engenharia para um robô de
trading sistemático adaptativo na Binance Spot, operando em 1h/4h e sem
alavancagem. A arquitetura prioriza preservação de capital, validação
estatística, execução idempotente e operação gradual por gates.

## O que existe neste repositório

- **Arquitetura:** separação Core/Satellite, regimes de mercado, gestão de
  risco, execução OCO e resiliência operacional.
- **Metodologia:** construção em seis blocos, critérios binários de aprovação,
  validação fora da amostra e protocolos de downgrade.
- **Migração v5 → v6:** diferenças estruturais, matemáticas e operacionais
  entre as versões.
- **Triagem:** itens críticos e refinamentos que ainda orientam a implementação.

## Documentação

A documentação completa está publicada em
[pac-coins.github.io/quant-crypto-chameleon-engine-v6](https://pac-coins.github.io/quant-crypto-chameleon-engine-v6/).

Para executar localmente:

```bash
python -m pip install -r requirements-docs.txt
mkdocs serve
```

O site é publicado automaticamente no GitHub Pages a cada atualização da
branch `main`.

## Estado do projeto

O repositório contém a especificação e o roadmap auditado do sistema. O robô
não deve operar com capital real antes que os gates definidos na metodologia
sejam aprovados e registrados.

## Licença

Consulte os arquivos e configurações do repositório para informações de
licenciamento.
