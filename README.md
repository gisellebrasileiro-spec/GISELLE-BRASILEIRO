Indicadores técnicos e sentimento de notícias para sinais de negociação: um experimento com validação temporal

Como o código foi construído, por que as escolhas foram feitas e o que os resultados significam

Relatório detalhado em linguagem acessível, atualizado com a execução temporal compartilhada pela autora.

Autora: Giselle de Araujo Brasileiro
Código revisado: pipeline_validacao_temporal_3tickers.py
Data: 30/10/2026
Resumo executivo

Este relatório explica não apenas quais números apareceram, mas como o programa foi pensado e implementado. O protótipo baixa preços e notícias, transforma essas fontes em variáveis, define um resultado futuro que quer classificar, ajusta um modelo, escolhe uma regra de negociação na validação e mede essa regra em um bloco posterior de teste. O fluxo foi aplicado a ITUB4, PETR4 e VALE3.

Na nova execução, ITUB4 e PETR4 ficaram abaixo de Buy & Hold no teste. VALE3 não abriu posição e terminou em fator 1.0 durante uma queda do benchmark: isso não demonstra uma estratégia ativa lucrativa. A separação treino/validação/teste melhora a disciplina experimental em relação ao script anterior, mas ainda há uma limitação: o modelo tenta prever o retorno de cinco pregões, enquanto o backtest revisado aplica o sinal ao retorno do dia seguinte. O relatório explicita essa diferença.

1. O que a pesquisa tentou responder

A pergunta do projeto é: é possível combinar indicadores de preço e notícias para gerar sinais simulados de compra (BUY), saída (SELL) ou manutenção (HOLD), e como esses sinais se comparam a comprar uma ação no início e mantê-la?

A resposta deve ser vista como um experimento acadêmico, não como aconselhamento financeiro ou promessa de desempenho.

2. Vocabulário para acompanhar o relatório
Termo	Explicação sem jargão
Ticker	Código usado para identificar a ação, por exemplo PETR4.
Feature	Um dado numérico que o modelo recebe: retorno, RSI, número de notícias etc.
Target / alvo	A resposta que queremos que o modelo estime.
Treino	Período antigo usado para ajustar o modelo.
Validação	Período usado para escolher a regra/limiares antes da avaliação final.
Teste	Período posterior reservado para medir a regra já escolhida.
Threshold / limiar	Um corte de probabilidade que decide quando entrar ou sair da posição.
Backtest	Simulação histórica de uma regra; não coloca ordens na bolsa.
Buy & Hold	Comprar no começo e manter; referência de comparação.
Overfitting	Quando escolhas se ajustam demais ao histórico e não generalizam.
Purga	Intervalo removido entre blocos para que rótulos futuros não atravessem a fronteira.
3. Visão geral do código: por que foi dividido em funções

O código foi organizado em funções pequenas para que cada uma tenha uma responsabilidade fácil de testar. Isso melhora leitura e manutenção: se a fonte de preços mudar, pode-se revisar a função de preços sem reescrever a lógica do modelo; se a regra de decisão mudar, pode-se alterar o backtest sem misturá-lo ao parser RSS.

Função/bloco	O que faz	Por que foi separado assim
Configuração	Define tickers, horizonte, fonte, grids e custo.	Deixa as escolhas experimentais visíveis e fáceis de reproduzir.
get_brapi_token	Busca segredo do Colab ou solicita token escondido.	Evita escrever credencial diretamente no código público.
brapi_quote_historical	Baixa e limpa fechamentos diários.	Isola a integração de preço e trata ausência de dados.
rss_url / parse_rss	Monta consulta e extrai campos do XML RSS.	Separa busca de parsing; parser ainda depende do formato do feed.
simple_sentiment_pt / news_features	Cria score lexical e agrega notícia por data.	Transforma texto em colunas numéricas simples.
rsi / build_features_targets	Calcula features de preço e target futuro.	Torna explícita a janela dos indicadores e a variável a prever.
backtest_next_day	Aplica sinal da data t ao retorno de t até t+1.	Evita usar o retorno do dia antes de o sinal estar disponível.
choose_thresholds	Testa combinações na validação e guarda a melhor.	Mantém o bloco teste fora da seleção da regra.
run_one_ticker / run	Executa o fluxo por ativo e imprime JSON.	Permite comparar tickers e registrar a saída de forma estruturada.
4. Construção dos dados passo a passo
4.1 Preços: por que BRAPI e o que é baixado

brapi_quote_historical monta uma chamada para a BRAPI usando o ticker e os parâmetros PRICE_RANGE="1y" e PRICE_INTERVAL="1d".

A resposta contém registros históricos; o código conserva data e close, converte o preço para número, remove fechamentos ausentes e ordena as datas. Se a fonte não retornar uma lista de preços ou faltar uma coluna esperada, devolve uma tabela vazia, para evitar que os cálculos seguintes falhem silenciosamente.

O token é uma credencial de acesso. A versão revisada tenta ler um segredo chamado BRAPI_TOKEN do Google Colab ou solicita a entrada sem eco. O token não deve ser incluído no relatório, README, notebook público ou repositório.

4.2 Notícias: origem, parser e limite

rss_url cria uma consulta para Google News RSS com o ticker como termo. parse_rss usa um parser XML para percorrer itens e recuperar título, descrição, data e link. Se o XML não puder ser interpretado, retorna uma tabela vazia com as colunas esperadas. A configuração limita a 30 notícias por ticker para reduzir volume e tempo.

Essa escolha foi feita para manter o protótipo leve, mas tem consequência metodológica importante: o feed RSS não é uma base histórica completa de notícias para cada dia do ano. Os itens e as datas disponíveis podem variar com o tempo. Ao combinar as notícias recuperadas com todo o histórico de preço, muitas datas antigas ficam sem notícia associada e recebem zeros. Portanto, não se deve concluir que o modelo teve uma série de sentimento completa e comparável ao preço durante todo o ano.

4.3 Como o texto vira números

simple_sentiment_pt limpa pontuação, separa palavras e verifica se cada termo pertence a uma lista pequena de vocabulário positivo ou negativo.

O escore é:

(quantidade positiva - quantidade negativa) / total reconhecido

Sem palavras reconhecidas, o denominador mínimo evita divisão por zero e o escore é zero. news_features calcula esse escore para cada notícia e agrupa por data, gerando news_count e sentiment_mean.

O método lexical foi escolhido por simplicidade, velocidade e explicabilidade. Não compreende negação (por exemplo, “não subiu”), ironia, contexto, duplicatas, relevância da notícia nem relações semânticas. É uma baseline didática, não análise financeira de sentimentos validada.

4.4 Indicadores de preço
Variável	Como ler	Decisão de projeto
ret_1d	Quanto o fechamento mudou em relação ao pregão anterior.	Captura movimento recente de um dia.
sma_10	Média dos dez fechamentos mais recentes.	Suaviza oscilações diárias curtas.
rsi_14	Indicador de movimentos recentes de alta e baixa em janela de 14.	Acrescenta informação sintética de momentum.
news_count	Quantidade de notícias associadas à data.	Usa volume de cobertura como sinal simples.
sentiment_mean	Média dos escores do vocabulário no dia.	Usa orientação lexical rudimentar.
5. Como foi definido o que prever
5.1 Baseline D+1

A primeira hipótese era prever se o próximo dia teria retorno positivo. A vantagem é ser fácil de entender; a desvantagem é que movimentos diários têm muito ruído.

A matriz registrada [[171 147], [219 183]] tem 720 observações e ROC-AUC=0.5076895397227698, próxima do nível de discriminação 0,5. A accuracy reportada foi 0.49166666666666664. O resultado serviu como baseline para motivar outras formulações.

5.2 Horizonte H=5 e margem de 0,5%

Na versão H=5, o programa calcula o preço de fechamento cinco pregões à frente com shift(-HORIZON_DAYS). O retorno futuro é fechamento futuro dividido pelo fechamento atual, menos 1.

O target recebe 1 se esse retorno exceder RET_THRESHOLD=0.005 (0,5%) e 0 caso contrário. FILTER_NEUTRAL=False significa que pequenas altas, estabilidade e quedas que não ultrapassam a margem não são retiradas; ficam na classe 0.

A hipótese era reduzir o ruído de tentar acertar exatamente o próximo dia. Isso não garante melhor previsão: altera a pergunta do experimento e muda a distribuição dos rótulos. As datas finais sem preço futuro ficam sem target e são removidas. A purga de cinco linhas antes da fronteira impede que o período usado para rotular uma observação invada a janela seguinte.

6. Como o modelo aprende e como se escolhe uma regra
6.1 Divisão temporal e padronização

Com as observações válidas em ordem cronológica, train_cut separa os primeiros 60% para treino e validation_cut marca o fim dos 20% seguintes. O restante compõe teste.

Ao contrário de dividir aleatoriamente, essa abordagem respeita a ordem passado→futuro. A purga remove HORIZON_DAYS observações perto das fronteiras para reduzir sobreposição dos alvos.

StandardScaler transforma as features para escalas comparáveis, calculando média e desvio no conjunto de treino. O mesmo escalador é aplicado à validação e ao teste. Isso evita que estatísticas do futuro influenciem a preparação do treino.

LogisticRegression é um classificador binário simples que aprende pesos e produz p_up, uma probabilidade estimada para a classe positiva.

6.2 Métricas de classe não são backtest

O código transforma p_up em classe prevista com corte 0,5 e compara com o target real.

Accuracy: mede proporção de acertos.
Precision: mede acertos entre previsões positivas.
Recall: mede positivos reais encontrados.
F1: combina precision e recall.
ROC-AUC: mede ordenação de positivos versus negativos.

Essas métricas avaliam a classificação, não o retorno financeiro da regra.

6.3 Thresholds BUY/SELL/HOLD

No backtest há dois limiares. Se probabilidade efetiva for pelo menos buy_thr, o programa abre posição. Se ficar em ou abaixo de sell_thr, zera posição. Entre os limites conserva a posição anterior.

O grid BUY_GRID=[0.55,0.60,0.65] e SELL_GRID=[0.45,0.40,0.35] testa combinações e rejeita sell maior/igual a buy, que não deixa intervalo lógico de manutenção.

6.4 Inversão e seleção na validação

O código tenta p_up e 1-p_up. roc_auc_inverted mede o desempenho da probabilidade invertida.

A função choose_thresholds calcula o backtest para cada combinação na validação e mantém a combinação com maior fator de patrimônio na validação. Essa regra, depois, é congelada e testada.

A inversão é apenas uma hipótese para aquele histórico; não prova que a direção oposta persistirá.

7. O que faz o backtest revisado — e o que ainda não faz

backtest_next_day calcula next_return como close(t+1)/close(t)-1. O sinal calculado em t é aplicado a esse retorno seguinte.

Se a posição muda, o patrimônio é multiplicado por (1-FEE_PER_TRADE); se a posição comprada permanece, o retorno do dia seguinte é aplicado. Buy & Hold multiplica todos os retornos da mesma janela. trades_approx conta mudanças de posição.

Correção importante em relação ao script antigo: o sinal não é aplicado a um retorno que já aconteceu antes do sinal.

Limitação remanescente: o classificador tem target de cinco dias, mas a estratégia captura retorno de um dia seguinte. Embora o alinhamento t→t+1 evite look-ahead imediato, as duas definições não medem a mesma pergunta econômica.

Próxima revisão deveria fazer a regra manter posição até o horizonte H ou redefinir o target para o retorno de um dia, e documentar a escolha.

O backtest também é simplificado: taxa fixa por mudança não representa todos os custos, slippage, spread, impostos, liquidez, execução parcial ou impacto de mercado. Não é simulação de ordens reais.

8. Confusion matrix do baseline: tratamento e interpretação

A matriz baseline [[171 147], [219 183]] é tratada como [[TN,FP],[FN,TP]]: TN=171, FP=147, FN=219, TP=183, total=720.

TN são negativos corretamente previstos; FP são falsos alarmes positivos; FN são positivos reais que o modelo não encontrou; TP são positivos corretamente encontrados.

Métrica derivada	Valor	Significado para leitor
Accuracy	0.491666666666667	Fração de todas as observações classificadas corretamente.
Precision	0.554545454545455	Das previsões positivas, fração realmente positiva.
Recall	0.455223880597015	Dos positivos reais, fração encontrada.
Specificity	0.537735849056604	Dos negativos reais, fração corretamente reconhecida.
F1-score	0.500000000000000	Equilíbrio entre precision e recall.
Balanced accuracy	0.496479864826809	Média de recall e especificidade.
MCC	-0.007016598874448	Correlação entre previsão e rótulo real; zero indica associação fraca.
FPR	0.462264150943396	Fração de negativos incorretamente marcados positivos.
FNR	0.544776119402985	Fração de positivos reais perdidos.

Esta matriz é do baseline D+1 e não deve ser confundida com as matrizes H=5 da nova execução: ITUB4 [[14,6],[9,17]], PETR4 [[4,13],[13,16]] e VALE3 [[0,37],[0,9]].

9. Linha do tempo e evolução experimental

Cada versão alterou uma decisão para investigar uma dificuldade. O par de fatores é estratégia primeiro e Buy & Hold segundo. Como os períodos, tickers, thresholds e critérios mudaram, os números não constituem uma comparação controlada única.

Versão	Hipótese/mudança	Resultados registrados	Aprendizado e ressalva
Baseline D+1	Prever direção do retorno do próximo dia.	accuracy=0.49166666666666664; ROC-AUC=0.5076895397227698; confusion matrix [[171 147], [219 183]].	A AUC próxima de 0,5 e a accuracy abaixo de 0,5 apontaram desempenho fraco no teste e motivaram testar outro alvo.
Migração de dados para BRAPI e Google News RSS	Encontrar fontes que permitissem uma execução leve no Colab após limitações nas tentativas anteriores.	Preços BRAPI; notícias Google News RSS.	A solução diminui barreiras de execução, mas o RSS não forma uma série histórica completa e o conteúdo recuperado pode variar.
Leve v2 — experimento ITUB4	Alvo H=5, thresholds e diagnóstico de sinal invertido.	ROC-AUC=0.8556521739130435; final_strategy=1.348032657957878; Buy & Hold=1.1836686944314467; 1 trade.	Resultado promissor, mas limiares foram selecionados no teste do exemplo; portanto, não era avaliação independente.
V3 — walk-forward	Separar treino/validação/teste em ordem temporal; configuração 2y, RSS 60, min_trades 2.	ITUB4 1.0966828809115032 vs 1.2499999999999996; PETR4 1.1000219170377215 vs 1.2113459399332591; VALE3 1.111607702106226 vs 0.8429141958126577.	A validação temporal reduz ajuste direto ao teste; evidencia também diferenças entre ativos.
V4 — thresholds globais	Compartilhar uma regra pelo portfólio: buy_thr=0.55, sell_thr=0.45.	ITUB4 1.0955894782386104 vs 1.2487537387836487; PETR4 1.0390643808276419 vs 1.2086763070077864; VALE3 1.0813698715747515 vs 0.8430352172334491.	Regra comum é simples, mas pode não adaptar-se a cada ativo.
V5 — restrição para tentar não piorar ativos	Selecionar regras na validação com critério para não piorar nenhum ativo quando possível.	ITUB4 1.28447190843006 vs 1.2537387836490523; PETR4 1.0973956020842797 vs 1.2084538375973302; VALE3 1.111607702106226 vs 0.8475130098027339.	A condição não produziu superioridade em todos os ativos.
V6 — maximin	Priorizar na validação o pior resultado entre os tickers.	ITUB4 1.274768144509515 vs 1.244267198404785; PETR4 1.0895166572239543 vs 1.1997775305895437; VALE3 1.111607702106226 vs 0.8481181169066913.	Proteger o pior caso não garantiu superar Buy & Hold em todos os ativos.
V7/V7.1 — foco PETR4	Explorar diferentes horizontes e limiares para PETR4; seleção principal pela validação.	Best V7.1: 1.081769689124918 vs 1.207119021134594; cenário alternativo H=10 visto no teste: 1.22902842297186 vs 1.207119021134594; 6 trades.	Uma configuração observada no teste não deve ser promovida a vencedora retrospectivamente.
Versão final anterior — leve 3 tickers	H=5 e grid de thresholds/inversão escolhidos pelo retorno no teste.	ITUB4 1.1847522449526409 vs 1.1912502988285931; PETR4 1.3649836662346402 vs 1.3413244718727741; VALE3 0.985232617679185 vs 0.8920281204403763.	Como o teste escolhia a regra, havia viés de seleção e potencial otimismo.
Versão revisada — validação temporal	60/20/20 cronológico; purga H nas fronteiras; thresholds escolhidos na validação; backtest aplica sinal a retorno do dia seguinte.	Resultados detalhados na seção 10.	A seleção fica mais limpa, mas é necessário alinhar a definição do target de H=5 ao retorno diário do backtest.

Os fatores de patrimônio começam em 1. Por exemplo, 1.20 equivale a 1,20 vezes o patrimônio inicial; 0.90 significa terminar abaixo do início. Não são porcentagens escritas diretamente.

10. Execução anterior: reconstrução completa, linha de processamento e interpretação dos resultados

Esta seção descreve em detalhe o arquivo pipeline_leve_janela_v2_3tickers.py, que foi executado antes do código temporal revisado. O objetivo é explicar o procedimento real, não apenas nomear as variáveis. A cada ticker, o programa executava as etapas abaixo.

10.1 Resultados completos da execução anterior
Ticker	Accuracy	Precision	Recall	F1	ROC-AUC	AUC invertida	Matriz de teste	Invert / buy / sell	Estratégia anterior	Buy & Hold anterior	Trades
ITUB4	0.723404255319149	0.9444444444444444	0.5862068965517241	0.723404255319149	0.7547892720306514	0.24521072796934865	[[17, 1], [12, 17]]	true / 0.65 / 0.45	1.1847522449526409	1.1912502988285931	3
PETR4	0.40425531914893614	0.5294117647058824	0.3103448275862069	0.391304347826087	0.30459770114942525	0.6954022988505748	[[10, 8], [20, 9]]	true / 0.55 / 0.4	1.3649836662346402	1.3413244718727741	3
VALE3	0.3617021276595745	0.2571428571428571	0.6923076923076923	0.375	0.49547511312217196	0.504524886877828	[[8, 26], [4, 9]]	true / 0.55 / 0.4	0.985232617679185	0.8920281204403763	5
10.2 Interpretação da execução anterior por ativo
ITUB4

A matriz [[17,1],[12,17]] mostra 17 negativos corretamente encontrados, 1 falso positivo, 12 falsos negativos e 17 positivos encontrados.

A precision 0.9444444444444444 era alta, mas recall 0.5862068965517241 deixava muitos positivos sem identificação. ROC-AUC=0.7547892720306514 parecia relativamente favorável, porém a estratégia final 1.1847522449526409 ficou ligeiramente abaixo de Buy & Hold 1.1912502988285931.

Portanto, a classificação não se traduziu em vantagem de patrimônio no backtest.

PETR4

ROC-AUC original 0.30459770114942525 ficou abaixo de 0.5 e a AUC de 1-p_up foi 0.6954022988505748.

O grid usou inversão; contudo, esse diagnóstico e a escolha dos thresholds foram avaliados no próprio teste. O patrimônio da estratégia foi 1.3649836662346402 versus 1.3413244718727741 do benchmark.

Como a regra foi escolhida para maximizar esse mesmo teste, a diferença não pode ser tomada como confirmação independente de superioridade.

VALE3

AUC original 0.49547511312217196 e invertida 0.504524886877828 ficaram próximas de 0.5.

O fator de estratégia 0.985232617679185 terminou abaixo do capital inicial 1, embora fosse superior ao Buy & Hold 0.8920281204403763.

Logo, dizer apenas que superou o benchmark esconderia que a estratégia também perdeu valor em termos absolutos.

10.3 O que é diretamente comparável e o que não é

O resultado anterior e o atual usam relatórios e código diferentes. O anterior escolhia thresholds e inversão no teste; o atual escolhe na validação.

O atual também remove corretamente rótulos sem futuro conhecido, faz purga H nas fronteiras e aplica sinal em t ao retorno t+1.

A mudança simultânea desses elementos impede atribuir toda diferença a apenas uma decisão. A nova execução deve ser descrita como novo protocolo, não como simples correção numérica da execução anterior.

10.4 Explicação sem abreviações da execução anterior, passo a passo

Nesta subseção, a execução antiga é explicada como se a pessoa estivesse acompanhando o programa pela primeira vez.

O resultado não surgiu de uma única previsão: para cada ticker, o script coletou os dados, formou linhas de análise, treinou o modelo, calculou as probabilidades, procurou a regra que mais rendia dentro do teste e calculou o fator final. Cada fase influencia a seguinte.

Configuração

O programa começava com um ticker, por exemplo ITUB4. O valor PRICE_RANGE="1y" pedia cerca de um ano de histórico de preços diários; não significa que todos os dias corridos tenham pregão.

A fonte entregava close, o preço de fechamento usado na análise. Se a resposta não tinha resultados, o script devolvia erro daquele ticker e passava ao próximo.

Coleta de notícias

Na mesma função, o programa buscava notícias pelo ticker no feed RSS. A configuração permitia processar no máximo 30 itens.

Isso não significa 30 notícias para cada dia do ano; são até 30 itens retornados pela consulta naquele momento. A data e o texto eram extraídos e notícias da mesma data agrupadas.

Cálculo do sentimento

Para cada notícia, o programa formava um texto com título e descrição. simple_sentiment_pt contava ocorrências de palavras presentes nas listas positivas e negativas.

Por exemplo, se reconhecesse duas palavras positivas e uma negativa, o escore seria:

(2-1)/(2+1)=1/3

Se não reconhecesse nenhuma, o escore seria zero. Isso não interpreta se uma palavra aparece negada ou em ironia.

Construção das features

O histórico de preços virava uma tabela diária. pct_change calculava ret_1d. sma_10 calculava a média móvel de 10 fechamentos. rsi calculava um indicador com altas e baixas recentes em 14 observações.

Para cada dia com notícia casada, o programa acrescentava news_count e sentiment_mean. Dias sem
