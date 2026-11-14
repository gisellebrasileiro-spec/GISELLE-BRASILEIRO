Indicadores Técnicos e Sentimento de Notícias para Sinais de Negociação: Um Experimento com Validação Temporal
Como o código foi construído, por que as escolhas foram feitas e o que os resultados significam

Relatório técnico detalhado em linguagem acessível, atualizado com a execução temporal compartilhada pela autora.

Informação	Descrição
Autora	Giselle de Araujo Brasileiro
Código revisado	pipeline_validacao_temporal_3tickers.py
Data	30/10/2026
Ativos analisados	ITUB4, PETR4 e VALE3
Linguagem	Python
Ambiente de execução	Google Colab
Sumário
Resumo executivo
Visão geral do código
Construção dos dados
Definição do alvo de previsão
Treinamento do modelo e escolha da estratégia
Funcionamento do backtest
Matriz de confusão do baseline
Linha do tempo e evolução experimental
Reconstrução da execução anterior
Comparação entre execução anterior e atual
Interpretação do JSON e conferência dos resultados
Resultados detalhados da nova execução
Comparação dos resultados
Como reproduzir no Google Colab
Conclusões, limitações e trabalhos futuros
Considerações finais
1. Resumo executivo

Este relatório explica não apenas quais números apareceram, mas também como o programa foi pensado e implementado.

O protótipo baixa preços e notícias, transforma essas fontes em variáveis, define um resultado futuro que deseja classificar, ajusta um modelo, escolhe uma regra de negociação na validação e mede essa regra em um bloco posterior de teste.

O fluxo foi aplicado a três ativos da bolsa brasileira:

ITUB4;
PETR4;
VALE3.

Na nova execução, ITUB4 e PETR4 ficaram abaixo do Buy & Hold no teste.

VALE3 não abriu posição e terminou em fator 1.0 durante uma queda do benchmark. Isso não demonstra uma estratégia ativa lucrativa.

A separação entre treino, validação e teste melhora a disciplina experimental em relação ao script anterior. Entretanto, permanece uma limitação importante: o modelo tenta prever o retorno de cinco pregões, enquanto o backtest revisado aplica o sinal ao retorno do dia seguinte.

O relatório explicita essa diferença para evitar interpretações equivocadas.

1.1. O que a pesquisa tentou responder

A pergunta central do projeto é:

É possível combinar indicadores de preço e notícias para gerar sinais simulados de compra (BUY), saída (SELL) ou manutenção (HOLD), e como esses sinais se comparam a comprar uma ação no início e mantê-la?

A resposta deve ser interpretada como um experimento acadêmico, não como aconselhamento financeiro ou promessa de desempenho.

1.2. Vocabulário para acompanhar o relatório
Termo	Explicação
Ticker	Código utilizado para identificar uma ação, por exemplo PETR4.
Feature	Dado numérico recebido pelo modelo, como retorno, RSI ou quantidade de notícias.
Target / alvo	Resultado que queremos que o modelo estime.
Treino	Período histórico utilizado para ajustar o modelo.
Validação	Período utilizado para escolher a regra e os limiares antes da avaliação final.
Teste	Período posterior reservado para medir a regra já escolhida.
Threshold / limiar	Corte de probabilidade que decide quando entrar ou sair de uma posição.
Backtest	Simulação histórica de uma regra de negociação. Não executa ordens na bolsa.
Buy & Hold	Estratégia de comprar no início e manter o ativo.
Overfitting	Quando as escolhas se ajustam excessivamente ao histórico e não generalizam.
Purga	Intervalo removido entre blocos para impedir que rótulos futuros atravessem suas fronteiras.
2. Visão geral do código: por que foi dividido em funções

O código foi organizado em funções menores para que cada uma tenha uma responsabilidade específica e possa ser compreendida, testada e modificada separadamente.

Essa organização melhora a leitura e a manutenção. Se a fonte de preços mudar, por exemplo, é possível revisar a função responsável pelos preços sem reescrever a lógica do modelo.

Função / bloco	O que faz	Por que foi separado
Configuração	Define tickers, horizonte, fonte, grids e custos.	Deixa as escolhas experimentais visíveis e reproduzíveis.
get_brapi_token	Busca segredo do Colab ou solicita token oculto.	Evita inserir credenciais diretamente no código público.
brapi_quote_historical	Baixa e limpa fechamentos diários.	Isola a integração de preços e trata ausência de dados.
rss_url / parse_rss	Monta consulta e extrai campos do XML RSS.	Separa busca de parsing.
simple_sentiment_pt / news_features	Cria score lexical e agrega notícias por data.	Transforma textos em colunas numéricas.
rsi / build_features_targets	Calcula indicadores e target futuro.	Torna explícitas as janelas e a variável prevista.
backtest_next_day	Aplica o sinal da data t ao retorno de t até t+1.	Evita utilizar retornos anteriores à disponibilidade do sinal.
choose_thresholds	Testa combinações na validação e guarda a melhor.	Mantém o teste fora da seleção da regra.
run_one_ticker / run	Executa o fluxo por ativo e imprime JSON.	Permite comparar ativos e registrar resultados estruturados.
3. Construção dos dados passo a passo
3.1. Preços: por que BRAPI e o que é baixado

A função brapi_quote_historical monta uma chamada para a BRAPI utilizando o ticker e os parâmetros:

python
PRICE_RANGE = "1y"
PRICE_INTERVAL = "1d"


A resposta contém registros históricos. O código conserva as informações de data e fechamento (close), converte os preços para valores numéricos, remove fechamentos ausentes e ordena as datas.

Se a fonte não retornar uma lista de preços ou faltar alguma coluna esperada, a função devolve uma tabela vazia, evitando que os cálculos seguintes falhem silenciosamente.

O token é uma credencial de acesso. A versão revisada tenta ler um segredo chamado BRAPI_TOKEN do Google Colab ou solicita a entrada sem exibir o conteúdo digitado.

Importante: o token não deve ser incluído no relatório, README, notebook público ou repositório.

3.2. Notícias: origem, parser e limitações

A função rss_url cria uma consulta para o Google News RSS utilizando o ticker como termo de pesquisa.

A função parse_rss utiliza um parser XML para percorrer os itens e recuperar:

Título;
Descrição;
Data;
Link.

Se o XML não puder ser interpretado, retorna uma tabela vazia com as colunas esperadas.

A configuração limita a coleta a 30 notícias por ticker, reduzindo o volume de dados e o tempo de processamento.

Essa escolha mantém o protótipo leve, mas apresenta uma consequência metodológica importante:

O feed RSS não é uma base histórica completa de notícias para cada dia do ano.

Os itens e as datas disponíveis podem variar ao longo do tempo.

Ao combinar as notícias recuperadas com todo o histórico de preços, muitas datas antigas ficam sem notícias associadas e recebem valores zero.

Portanto, não se deve concluir que o modelo teve uma série histórica de sentimento completa e comparável aos preços durante todo o período.

3.3. Como o texto vira números

A função simple_sentiment_pt limpa a pontuação, separa as palavras e verifica se cada termo pertence a uma pequena lista de vocabulário positivo ou negativo.

O escore é calculado por:

Sentimento=Quantidade positiva−Quantidade negativaTotal reconhecido

Quando nenhuma palavra é reconhecida, o denominador mínimo evita divisão por zero e o escore é definido como zero.

A função news_features calcula esse escore para cada notícia e agrupa os resultados por data, gerando:

news_count: quantidade de notícias associadas ao dia;
sentiment_mean: média dos escores de sentimento.

O método lexical foi escolhido por simplicidade, velocidade e facilidade de explicação.

Entretanto, ele não compreende adequadamente:

Negação, como em "não subiu";
Ironia;
Contexto;
Duplicatas;
Relevância da notícia;
Relações semânticas.

Trata-se de uma baseline didática, não de uma análise financeira de sentimentos validada.

3.4. Indicadores de preço
Variável	Como interpretar	Decisão de projeto
ret_1d	Variação do fechamento em relação ao pregão anterior.	Captura movimento recente de um dia.
sma_10	Média dos dez fechamentos mais recentes.	Suaviza oscilações diárias curtas.
rsi_14	Indicador de movimentos recentes de alta e baixa em uma janela de 14 períodos.	Acrescenta informação sintética de momentum.
news_count	Quantidade de notícias associadas à data.	Utiliza o volume de cobertura como sinal simples.
sentiment_mean	Média dos escores lexicais das notícias do dia.	Utiliza uma orientação lexical rudimentar.
4. Como foi definido o que prever
4.1. Baseline D+1

A primeira hipótese era prever se o próximo dia teria retorno positivo.

A vantagem dessa abordagem é a facilidade de compreensão. A desvantagem é que movimentos diários possuem bastante ruído.

A matriz de confusão registrada foi:

text
[[171 147]
 [219 183]]


Resultados:

Métrica	Resultado
Observações	720
ROC-AUC	0.5076895397227698
Accuracy	0.49166666666666664

O resultado serviu como baseline para motivar outras formulações do problema.

4.2. Horizonte H=5 e margem de 0,5%

Na versão H=5, o programa calcula o preço de fechamento cinco pregões à frente utilizando shift(-HORIZON_DAYS).

O retorno futuro é:

Retornofuturo=FechamentofuturoFechamentoatual−1

O target recebe:

1 se o retorno futuro exceder RET_THRESHOLD = 0.005 (0,5%);
0 caso contrário.

A configuração FILTER_NEUTRAL=False significa que pequenas altas, estabilidade e quedas que não ultrapassam a margem não são retiradas. Elas permanecem na classe 0.

A hipótese era reduzir o ruído de tentar acertar exatamente o próximo dia.

Isso não garante uma previsão melhor: apenas altera a pergunta do experimento e modifica a distribuição dos rótulos.

As datas finais sem preço futuro ficam sem target e são removidas.

A purga de cinco linhas antes da fronteira impede que o período utilizado para rotular uma observação invada a janela seguinte.

5. Como o modelo aprende e como se escolhe uma regra
5.1. Divisão temporal e padronização

Com as observações válidas em ordem cronológica:

train_cut separa os primeiros 60% para treino;
validation_cut marca o fim dos 20% seguintes;
O restante compõe o teste.

Diferentemente de uma divisão aleatória, essa abordagem respeita a ordem passado → futuro.

A purga remove HORIZON_DAYS observações próximas às fronteiras para reduzir a sobreposição dos alvos.

O StandardScaler transforma as features para escalas comparáveis, calculando média e desvio-padrão no conjunto de treino.

O mesmo escalador é aplicado à validação e ao teste. Isso evita que estatísticas do futuro influenciem a preparação do treinamento.

A LogisticRegression é um classificador binário simples que aprende pesos e produz p_up, uma probabilidade estimada para a classe positiva.

5.2. Métricas de classificação não são backtest

O código transforma p_up em classe prevista com corte 0,5 e compara o resultado com o target real.

Métrica	Significado
Accuracy	Mede a proporção de acertos.
Precision	Mede os acertos entre as previsões positivas.
Recall	Mede os positivos reais encontrados.
F1	Combina precision e recall.
ROC-AUC	Mede a capacidade de ordenação entre positivos e negativos.

Essas métricas avaliam a classificação, não o retorno financeiro da regra.

5.3. Thresholds BUY / SELL / HOLD

No backtest existem dois limiares.

Se a probabilidade efetiva for pelo menos buy_thr, o programa abre posição.
Se ficar em ou abaixo de sell_thr, zera a posição.
Entre os limites, conserva a posição anterior.

O grid utilizado foi:

python
BUY_GRID = [0.55, 0.60, 0.65]
SELL_GRID = [0.45, 0.40, 0.35]


O programa testa combinações e rejeita casos em que sell >= buy, pois não haveria um intervalo lógico de manutenção.

5.4. Inversão e seleção na validação

O código tenta utilizar:

p_up;
1 - p_up.

A métrica roc_auc_inverted mede o desempenho da probabilidade invertida.

A função choose_thresholds calcula o backtest para cada combinação na validação e mantém a combinação com maior fator de patrimônio nesse bloco.

Depois disso, a regra é congelada e aplicada ao teste.

A inversão é apenas uma hipótese para aquele histórico. Não prova que a direção oposta continuará funcionando no futuro.

6. O que faz o backtest revisado

A função backtest_next_day calcula:

next_return=close(t+1)close(t)−1

O sinal calculado em t é aplicado ao retorno seguinte.

Se a posição muda, o patrimônio é multiplicado por:

1−FEE_PER_TRADE

Se a posição comprada permanece, o retorno do dia seguinte é aplicado.

O Buy & Hold multiplica todos os retornos da mesma janela.

A variável trades_approx conta mudanças de posição.

6.1. Correção em relação ao script antigo

O sinal não é aplicado a um retorno que já aconteceu antes de sua geração.

Isso representa uma melhoria no alinhamento temporal do backtest.

6.2. Limitação remanescente

O classificador tem target de cinco dias, mas a estratégia captura o retorno de um dia seguinte.

Embora o alinhamento t → t+1 evite look-ahead imediato, as duas definições não medem a mesma pergunta econômica.

A próxima revisão deveria:

Fazer a regra manter a posição até o horizonte H; ou
Redefinir o target para o retorno de um dia.

A escolha deve ser documentada.

O backtest também é simplificado. Uma taxa fixa por mudança não representa todos os custos envolvidos, como:

Slippage;
Spread;
Impostos;
Liquidez;
Execução parcial;
Impacto de mercado.

Portanto, não é uma simulação de ordens reais.

7. Confusion matrix do baseline: tratamento e interpretação

A matriz baseline foi:

text
[[171 147]
 [219 183]]


Sua organização é:

text
[[TN, FP],
 [FN, TP]]


Portanto:

Elemento	Valor	Interpretação
TN	171	Negativos corretamente previstos.
FP	147	Falsos alarmes positivos.
FN	219	Positivos reais não identificados.
TP	183	Positivos corretamente identificados.
Total	720	Total de observações.
7.1. Métricas derivadas
Métrica	Valor	Significado
Accuracy	0.491666666666667	Fração de todas as observações classificadas corretamente.
Precision	0.554545454545455	Das previsões positivas, fração realmente positiva.
Recall	0.455223880597015	Dos positivos reais, fração encontrada.
Specificity	0.537735849056604	Dos negativos reais, fração corretamente reconhecida.
F1-score	0.500000000000000	Equilíbrio entre precision e recall.
Balanced accuracy	0.496479864826809	Média de recall e especificidade.
MCC	-0.007016598874448	Correlação entre previsão e rótulo real; zero indica associação fraca.
FPR	0.462264150943396	Fração de negativos incorretamente marcados como positivos.
FNR	0.544776119402985	Fração de positivos reais perdidos.

Importante: essa matriz pertence ao baseline D+1. Ela não deve ser confundida com as matrizes H=5 da nova execução:

Ativo	Matriz de confusão H=5
ITUB4	[[14, 6], [9, 17]]
PETR4	[[4, 13], [13, 16]]
VALE3	[[0, 37], [0, 9]]
8. Linha do tempo e evolução experimental

Cada versão alterou uma decisão para investigar uma dificuldade específica.

O par de fatores apresenta primeiro a estratégia e depois o Buy & Hold.

Como os períodos, tickers, thresholds e critérios mudaram, os números não constituem uma comparação controlada única.

Versão	Hipótese / mudança	Resultados registrados	Aprendizado e ressalva
Baseline D+1	Prever direção do retorno do próximo dia.	Accuracy = 0.49166666666666664; ROC-AUC = 0.5076895397227698; matriz [[171,147],[219,183]].	A AUC próxima de 0,5 e a accuracy abaixo de 0,5 apontaram desempenho fraco.
Migração para BRAPI e Google News RSS	Encontrar fontes para uma execução leve no Colab.	Preços BRAPI; notícias Google News RSS.	Diminui barreiras de execução, mas RSS não forma uma série histórica completa.
Leve v2 — ITUB4	Alvo H=5, thresholds e diagnóstico de sinal invertido.	ROC-AUC = 0.8556521739130435; estratégia = 1.348032657957878; Buy & Hold = 1.1836686944314467; 1 trade.	Resultado promissor, mas limiares foram selecionados no teste.
V3 — walk-forward	Separar treino, validação e teste; configuração 2y, RSS 60, min_trades 2.	ITUB4: 1.0966828809115032 vs. 1.2499999999999996; PETR4: 1.1000219170377215 vs. 1.2113459399332591; VALE3: 1.111607702106226 vs. 0.8429141958126577.	A validação temporal reduz ajuste direto ao teste.
V4 — thresholds globais	Compartilhar uma regra pelo portfólio: buy = 0.55 e sell = 0.45.	ITUB4: 1.0955894782386104 vs. 1.2487537387836487; PETR4: 1.0390643808276419 vs. 1.2086763070077864; VALE3: 1.0813698715747515 vs. 0.8430352172334491.	Uma regra comum pode não se adaptar a cada ativo.
V5 — restrição para tentar não piorar ativos	Selecionar regras na validação com critério de proteção entre ativos.	ITUB4: 1.28447190843006 vs. 1.2537387836490523; PETR4: 1.0973956020842797 vs. 1.2084538375973302; VALE3: 1.111607702106226 vs. 0.8475130098027339.	A condição não produziu superioridade em todos os ativos.
V6 — maximin	Priorizar o pior resultado entre os tickers.	ITUB4: 1.274768144509515 vs. 1.244267198404785; PETR4: 1.0895166572239543 vs. 1.1997775305895437; VALE3: 1.111607702106226 vs. 0.8481181169066913.	Proteger o pior caso não garantiu superar Buy & Hold em todos os ativos.
V7/V7.1 — PETR4	Explorar horizontes e limiares diferentes.	Best V7.1: 1.081769689124918 vs. 1.207119021134594; cenário alternativo H=10 visto no teste: 1.22902842297186 vs. 1.207119021134594; 6 trades.	Uma configuração observada no teste não deve ser promovida retrospectivamente a vencedora.
Versão final anterior — três tickers	H=5 e grid escolhidos pelo retorno no teste.	ITUB4: 1.1847522449526409 vs. 1.1912502988285931; PETR4: 1.3649836662346402 vs. 1.3413244718727741; VALE3: 0.985232617679185 vs. 0.8920281204403763.	O teste escolhia a regra, gerando viés de seleção.
Versão revisada — validação temporal	60/20/20 cronológico, purga H e thresholds escolhidos na validação.	Resultados detalhados na seção 12.	Seleção mais limpa, mas ainda é necessário alinhar o target H=5 ao retorno diário do backtest.

Os fatores de patrimônio começam em 1.

Exemplos:

1.20 equivale a 1,20 vezes o patrimônio inicial.
0.90 significa terminar abaixo do capital inicial.
Esses fatores não são porcentagens escritas diretamente.
9. Execução anterior: reconstrução completa

Esta seção descreve em detalhe o arquivo pipeline_leve_janela_v2_3tickers.py, executado antes do código temporal revisado.

O objetivo é explicar o procedimento real, não apenas nomear as variáveis.

A cada ticker, o programa executava as etapas abaixo.

9.1. Configuração
Parâmetro	Valor
TICKERS	ITUB4, PETR4 e VALE3
PRICE_RANGE	1y
PRICE_INTERVAL	1d
RSS_ITEMS_PER_TICKER	30
HORIZON_DAYS	5
RET_THRESHOLD	0.005
TRAIN_FRAC	0.80

Isso representa até um ano de fechamentos diários, no máximo 30 notícias processadas por ativo e um rótulo futuro de cinco pregões com margem de 0,5%.

9.2. Coleta do preço

brapi_quote_historical consultava o endpoint de cotação histórica da BRAPI, selecionava date e close, convertia timestamp para data e ordenava as observações.

Se não existisse resultado ou preço, a função retornava uma tabela vazia e run_one_ticker informava erro para aquele ticker.

9.3. Coleta de notícias

build_google_news_rss_url montava uma consulta do Google News RSS com o ticker.

parse_rss tentava separar item, título, data, descrição e link por expressões regulares.

Notícias acima do limite eram cortadas para manter 30 itens.

Essa fonte era prática, mas não representava um arquivo completo de notícias do ano.

9.4. Sentimento

simple_sentiment_pt limpava sinais de pontuação, separava palavras e contava termos encontrados em POS_WORDS e NEG_WORDS.

O escore era:

Sentimento=Positivos−NegativosTotal classificado

O resultado era resumido por dia.

Não havia interpretação de contexto, negação ou duplicidade.

9.5. Features e target

O código criava:

Retorno diário ret_1d;
Média móvel sma_10;
RSI rsi_14;
news_count e sentiment_mean associados por data;
future_ret_H, deslocando o fechamento cinco linhas para a frente;
target_up_H = 1 quando o retorno excedia 0.005.

Uma ressalva: future_close inexistente no final da série podia produzir comparação falsa e classe 0. O script anterior não preservava explicitamente o rótulo como ausente nessas linhas.

9.6. Divisão

Depois de retirar linhas com features ausentes:

python
n = len(df)
cut = floor(n * 0.80)


As primeiras 80% das observações iam para treino e as últimas 20% para teste.

Não havia um bloco de validação independente.

9.7. Treinamento e métricas

O StandardScaler era ajustado no treino e transformava treino e teste.

A LogisticRegression era ajustada com target_up_H do treino.

No teste, predict_proba gerava p_up.

O corte fixo de 0.5 gerava previsões para:

Accuracy;
F1;
Precision;
Recall;
Confusion matrix;
ROC-AUC;
ROC-AUC invertida.
9.8. Busca da regra

O código varria:

python
invert = [False, True]

buy_grid = [0.55, 0.60, 0.65]
sell_grid = [0.45, 0.40, 0.35]


Descartava sell >= buy e executava backtest_probs para cada combinação diretamente na tabela test_bt.

Guardava a combinação de maior final_strategy.

Consequência metodológica: como a regra foi escolhida pelo retorno do conjunto de teste e depois reportada nesse mesmo conjunto, o resultado não era uma medição imparcial fora da amostra.

Entre muitas combinações, alguma pode vencer por acaso naquele período. Escolher a vencedora depois de olhar o teste produz viés de seleção e potencial otimismo.

9.9. Backtest anterior

test_bt guardava:

date;
close;
p_up.

A função backtest_probs calculava ret_1d na mesma linha e atualizava a posição pelo limiar.

Descontava:

python
FEE_PER_TRADE = 0.0005


quando a posição mudava, contava essa mudança como trades_approx e comparava o patrimônio simulado com Buy & Hold.

Essa mecânica simplificada não incluía slippage, spread, liquidez nem execução real.

9.10. Resultados completos da execução anterior
Ticker	Accuracy	Precision	Recall	F1	ROC-AUC	AUC invertida	Matriz de teste	Invert / Buy / Sell	Estratégia	Buy & Hold	Trades
ITUB4	0.723404255319149	0.9444444444444444	0.5862068965517241	0.723404255319149	0.7547892720306514	0.24521072796934865	[[17, 1], [12, 17]]	true / 0.65 / 0.45	1.1847522449526409	1.1912502988285931	3
PETR4	0.40425531914893614	0.5294117647058824	0.3103448275862069	0.391304347826087	0.30459770114942525	0.6954022988505748	[[10, 8], [20, 9]]	true / 0.55 / 0.40	1.3649836662346402	1.3413244718727741	3
VALE3	0.3617021276595745	0.2571428571428571	0.6923076923076923	0.375	0.49547511312217196	0.504524886877828	[[8, 26], [4, 9]]	true / 0.55 / 0.40	0.985232617679185	0.8920281204403763	5
9.11
