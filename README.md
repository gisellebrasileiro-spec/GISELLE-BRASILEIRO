# GISELLE-BRASILEIRO
Projeto final BI MASTER: pipeline reprodutível no Colab integrando indicadores técnicos, notícias (RSS) e sentimento para prever direção/retorno (D+1) e comparar estratégias de trading via backtest.
GISELLE-BRASILEIRO — SIAD para Compra/Venda de Ações (BRAPI + RSS + Sentimento + ML)

Projeto final (BI MASTER/PUC-Rio) de um Sistema Inteligente de Apoio à Decisão (SIAD) para apoiar decisões de compra/venda de ações na B3 por meio de:

Séries históricas de preços (OHLCV) via BRAPI

Indicadores técnicos (SMA/EMA/RSI/MACD/Bollinger)

Notícias financeiras via Google News RSS

Análise de sentimento (baseline) por léxico (heurística) e agregação diária

Modelos de Machine Learning:

Classificação: prever se o retorno do dia seguinte será positivo (alta)

Regressão: prever o retorno do dia seguinte

Backtest para comparar estratégias derivadas dos modelos vs Buy & Hold

1. Objetivo

Construir um pipeline reprodutível que:

integre dados de preço e notícias;

transforme as informações em features;

treine modelos preditivos;

gere recomendações objetivas (COMPRA/NEUTRO/SAIR);

avalie o impacto prático por meio de backtesting.

2. Universo de ativos

Por limitações e instabilidades observadas durante a execução no Google Colab e validação de fontes públicas, o universo operacional final deste repositório ficou:

ITUB4

PETR4

VALE3

Observação: o projeto originalmente considerava 6 ativos com base em liquidez/movimentação, mas a disponibilidade de histórico consistente via APIs públicas no ambiente Colab exigiu ajustes para garantir reprodutibilidade.

3. Fontes de dados

3.1 Preços (BRAPI)

Endpoint utilizado: https://brapi.dev/api/quote/{TICKER}?range=5y&interval=1d&token=...

Dados extraídos: open, high, low, close, volume (OHLCV)

Periodicidade: diária

3.2 Notícias (Google News RSS)

Fonte: RSS público do Google News por consulta:

ITUB4: Itaú Unibanco OR ITUB4

PETR4: Petrobras OR PETR4

VALE3: Vale S.A. OR VALE3 mineração

Retorno típico: ~100 itens por consulta (variável conforme relevância/atualidade)

Campos usados: headline, datetime, source, url

3.3 Por que não Finnhub (company-news)?

Durante testes, o endpoint company-news retornou 403 Forbidden (restrição do plano/credenciais), e market-news não trouxe menções suficientes aos ativos brasileiros no lote analisado. Para não travar a etapa de notícias/sentimento, foi adotado Google News RSS como alternativa reprodutível sem chave.

4. Definições do SIAD (alvos e decisões)

4.1 Alvo de Classificação

target_up_next:

1 se o retorno do dia seguinte (D+1) for positivo

0 caso contrário

Esse alvo é diretamente interpretável como “probabilidade de alta amanhã” → útil para COMPRA/NEUTRO/SAIR.

4.2 Alvo de Regressão

target_ret_1d_next:

retorno (log-return) do dia seguinte (D+1)

Esse alvo permite decisões baseadas em magnitude do retorno previsto (com thresholds).

5. Features

5.1 Indicadores técnicos

Calculados por ticker:

SMA(20), SMA(50)

EMA(12)

RSI(14)

MACD(12, 26, 9): macd, macd_signal, macd_hist

Bollinger Bands(20, 2σ): bb_mid, bb_upper, bb_lower

Retorno 1D: ret_1d

Volume: volume

5.2 Sentimento (baseline)

Heurística por léxico (PT-BR), computando sentiment_score por notícia:

positivo: palavras como “alta”, “lucro”, “recorde” etc.

negativo: “queda”, “prejuízo”, “crise” etc.

Agregação diária por ticker:

news_count

sentiment_mean, sentiment_median

pos_pct, neg_pct

Este baseline serve como referência (baseline) e permite comparar posteriormente com uma versão LLM (conforme recomendação da orientadora).

6. Modelos

6.1 Classificação — Regressão Logística (baseline)

Padronização: StandardScaler

Modelo: LogisticRegression

Split temporal por ticker: 80% treino / 20% teste (sem vazamento)

Métricas obtidas (teste):

Accuracy: 0.49166666666666664

F1: 0.5

Precision: 0.5545454545454546

Recall: 0.4552238805970149

ROC-AUC: 0.5076895397227698

Confusion matrix: [[171 147], [219 183]]

6.2 Regressão — Ridge (baseline)

Padronização: StandardScaler

Modelo: Ridge

A regressão é convertida em estratégia via thresholds no retorno previsto.

7. Estratégias e Backtest

7.1 Estratégia baseada em Classificação (probabilidade)

O modelo produz p_up = probabilidade de alta no dia seguinte.

Foi usada uma política de histerese (evita “churning”):

COMPRA se p_up >= buy_thr

ZERA se p_up <= sell_thr

caso contrário: mantém posição anterior

7.2 Estratégia baseada em Regressão (retorno previsto)

A regressão produz ret_pred (retorno previsto D+1):

COMPRA se ret_pred >= buy_ret

ZERA se ret_pred <= sell_ret

caso contrário: mantém posição anterior

7.3 Comparação final com thresholds moderados + custo

Parâmetros (moderado + custo):

Classificação: buy_thr=0.53, sell_thr=0.47

Regressão: buy_ret=0.0010, sell_ret=-0.0010

Custo por trade: fee_per_trade=0.0005

Período do backtest: 2025-09-15 a 2026-08-31 (240 dias)

Resultados (valor acumulado final):

ITUB4:

final_cls: 0.885259

final_reg: 1.426421

buy&hold: 1.088966

max_dd_cls: -0.240227

max_dd_reg: -0.153273

trades_cls: 1

trades_reg: 10

PETR4:

final_cls: 1.451064

final_reg: 1.312062

buy&hold: 1.443874

max_dd_cls: -0.241262

max_dd_reg: -0.097180

trades_cls: 1

trades_reg: 12

VALE3:

final_cls: 1.025030

final_reg: 1.172570

buy&hold: 1.365789

max_dd_cls: -0.042491

max_dd_reg: -0.062190

trades_cls: 2

trades_reg: 6

Interpretação: não houve um vencedor único. Com thresholds moderados e custo, a regressão foi superior em ITUB4 e VALE3; a classificação foi superior em PETR4 (ligeiramente acima do buy&hold). A avaliação final deve considerar retorno, drawdown e número de trades.

8. Como executar (Google Colab)

Recomendação: rodar em ordem:

Semana 3 (RSS + sentimento) → gera agg

Semana 4 (preços + indicadores + merge) → gera df_ml

Semana 5 (classificação e regressão + backtests)

Dependências

pandas, numpy, requests, matplotlib

scikit-learn

Observação: no Colab, evite pip install -U pandas numpy pois pode quebrar compatibilidade do ambiente.

9. Limitações conhecidas

Dependência de fontes públicas sujeitas a instabilidade (BRAPI, RSS).

Sentimento baseline por léxico é simples e não captura contexto/ironia; é uma referência para comparação futura com LLM.

Backtest não inclui slippage realista; custo por trade é aproximado.

10. Próximos passos recomendados

Substituir sentimento baseline por LLM e comparar resultados (mesmo prompt) conforme recomendação da orientadora.

Testar modelos alternativos (RandomForest, Gradient Boosting, XGBoost/LightGBM).

Fazer calibração sistemática de thresholds (por percentis, otimização por métrica de risco).

Incluir custos mais realistas (corretagem/spread) e regras de posicionamento (stop/take profit).

Contato

Aluna: Giselle de Araujo Brasileiro

Projeto: BI MASTER / PUC-Rio
