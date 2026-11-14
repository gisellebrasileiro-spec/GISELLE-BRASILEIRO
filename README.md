# README — Projeto Final BI MASTER (PUC-Rio)

## Sistema de Apoio à Decisão para Compra e Venda de Ações da B3

**Autora:** Giselle de Araujo Brasileiro  
**Código:** `pipeline_validacao_temporal_3tickers.py`

> Projeto acadêmico e exploratório; não é recomendação financeira nem garantia de desempenho.

## 1. O que o projeto faz

O programa usa preços históricos e notícias para construir indicadores, estimar a probabilidade de uma alta definida e simular uma regra de compra, saída ou manutenção. A regra é comparada com Buy & Hold: comprar no início e manter a ação. A versão foi aplicada a ITUB4, PETR4 e VALE3.

## 2. Termos essenciais

| Termo | Significado simples |
|---|---|
| Feature | Informação numérica de entrada, como retorno, RSI ou notícias. |
| Target | Resultado que o modelo tenta prever. |
| Treino | Dados mais antigos usados para ajustar o modelo. |
| Validação | Dados usados para escolher limiares e inversão. |
| Teste | Dados posteriores usados para avaliar uma regra já escolhida. |
| Threshold | Corte de probabilidade que muda a posição. |
| Backtest | Simulação histórica, não operação real. |
| Buy & Hold | Comprar no começo e manter até o fim. |

## 3. Metodologia em linguagem acessível

1. **Preços:** `brapi_quote_historical` solicita fechamentos diários à BRAPI (`PRICE_RANGE="1y"`, `PRICE_INTERVAL="1d"`), ordena as datas e verifica se há dados.
2. **Notícias:** o código procura notícias pelo Google News RSS, tentando extrair título, descrição, data e link; considera até 30 itens por ticker.
3. **Sentimento:** `simple_sentiment_pt` usa um vocabulário curto de palavras positivas e negativas. As contagens são resumidas por dia em `news_count` e `sentiment_mean`. É um baseline simples, incapaz de compreender contexto, ironia e negação; o RSS não é um arquivo histórico completo.
4. **Features de preço:** `ret_1d` mede a variação diária; `sma_10` é a média dos últimos dez pregões; `rsi_14` resume movimentos recentes de alta/baixa.
5. **Target:** `HORIZON_DAYS=5` e `RET_THRESHOLD=0.005`: classe positiva se o retorno até cinco pregões à frente superar 0,5%. Datas sem preço futuro conhecido são removidas. A purga na fronteira ajuda a evitar que o futuro de um bloco invada o próximo.
6. **Divisão temporal:** 60% treino, 20% validação e 20% teste; sem embaralhar a sequência cronológica.
7. **Modelo:** StandardScaler é ajustado no treino; regressão logística produz `p_up`.
8. **Regra:** `buy_thr` abre posição, `sell_thr` zera posição e a faixa intermediária mantém a posição (HOLD). O grid inclui possibilidade de usar `1-p_up` (`invert_signal`).
9. **Avaliação:** a regra é escolhida na validação e então congelada para o teste. O backtest revisado aplica o sinal de t ao retorno de t para t+1 e desconta `FEE_PER_TRADE=0.0005` em mudanças de posição.

## 4. Resultados da execução recente

Cada ticker teve 133 linhas de treino, 41 de validação e 46 de teste. O backtest contabilizou 45 retornos, pois requer um preço no dia seguinte.

| Ativo | ROC-AUC | Matriz de confusão | Regra selecionada na validação | Estratégia no teste | Buy & Hold no teste | Trades |
|---|---:|---|---|---:|---:|---:|
| ITUB4 | 0.8442307692307692 | `[[14, 6], [9, 17]]` | sem inversão; buy 0.55; sell 0.45 | 1.075753936237348 | 1.258795934323691 | 4 |
| PETR4 | 0.35902636916835695 | `[[4, 13], [13, 16]]` | invertida; buy 0.55; sell 0.4 | 1.3218431606508754 | 1.3611691022964503 | 4 |
| VALE3 | 0.4054054054054054 | `[[0, 37], [0, 9]]` | invertida; buy 0.55; sell 0.45 | 1.0 | 0.8581727436572849 | 0 |

`final_strategy` e `final_buy_hold` são fatores começando em 1. ITUB4 e PETR4 ficaram abaixo de Buy & Hold. Em VALE3, o classificador apontou classe positiva em todas as observações e o backtest não abriu posição; fator 1.0 superou o benchmark durante uma queda, mas isso não é evidência de trades vencedores.

## 5. Limitações metodológicas relevantes

- O target olha cinco pregões à frente, mas o backtest aplica o sinal ao retorno do próximo dia. É preciso alinhar esses horizontes para uma análise coerente.
- Uma única divisão com 46 linhas de teste por ativo é pequena; recomenda-se walk-forward em várias datas.
- O RSS é limitado, o sentimento é lexical e a cobertura histórica de notícias é incompleta.
- O backtest não representa integralmente spread, slippage, liquidez e execução real.
- Os resultados são históricos e não garantem comportamento futuro.

## 6. Como replicar no Google Colab

1. Abra [Google Colab](https://colab.research.google.com/) e crie um notebook.
2. Adicione o token BRAPI como segredo chamado `BRAPI_TOKEN`; não o publique em código, Word ou GitHub.
3. Carregue ou cole `pipeline_validacao_temporal_3tickers.py`.
4. Se necessário, instale `pandas`, `numpy`, `scikit-learn` e `requests`.
5. Execute e salve `RESULTS_JSON`, parâmetros e data de execução.
6. Confira os tamanhos dos blocos e erros antes de interpretar a saída.
