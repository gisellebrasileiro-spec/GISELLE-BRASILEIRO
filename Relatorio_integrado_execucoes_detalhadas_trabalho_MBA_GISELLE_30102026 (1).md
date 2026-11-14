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

A pergunta do projeto é: é possível combinar indicadores de preço e notícias para gerar sinais simulados de compra (BUY), saída (SELL) ou manutenção (HOLD), e como esses sinais se comparam a comprar uma ação no início e mantê-la? A resposta deve ser vista como um experimento acadêmico, não como aconselhamento financeiro ou promessa de desempenho.

2. Vocabulário para acompanhar o relatório

| Termo | Explicação sem jargão |
| --- | --- |
| Ticker | Código usado para identificar a ação, por exemplo PETR4. |
| Feature | Um dado numérico que o modelo recebe: retorno, RSI, número de notícias etc. |
| Target / alvo | A resposta que queremos que o modelo estime. |
| Treino | Período antigo usado para ajustar o modelo. |
| Validação | Período usado para escolher a regra/limiares antes da avaliação final. |
| Teste | Período posterior reservado para medir a regra já escolhida. |
| Threshold / limiar | Um corte de probabilidade que decide quando entrar ou sair da posição. |
| Backtest | Simulação histórica de uma regra; não coloca ordens na bolsa. |
| Buy & Hold | Comprar no começo e manter; referência de comparação. |
| Overfitting | Quando escolhas se ajustam demais ao histórico e não generalizam. |
| Purga | Intervalo removido entre blocos para que rótulos futuros não atravessem a fronteira. |

3. Visão geral do código: por que foi dividido em funções

O código foi organizado em funções pequenas para que cada uma tenha uma responsabilidade fácil de testar. Isso melhora leitura e manutenção: se a fonte de preços mudar, pode-se revisar a função de preços sem reescrever a lógica do modelo; se a regra de decisão mudar, pode-se alterar o backtest sem misturá-lo ao parser RSS.

| Função/bloco | O que faz | Por que foi separado assim |
| --- | --- | --- |
| Configuração | Define tickers, horizonte, fonte, grids e custo. | Deixa as escolhas experimentais visíveis e fáceis de reproduzir. |
| get_brapi_token | Busca segredo do Colab ou solicita token escondido. | Evita escrever credencial diretamente no código público. |
| brapi_quote_historical | Baixa e limpa fechamentos diários. | Isola a integração de preço e trata ausência de dados. |
| rss_url / parse_rss | Monta consulta e extrai campos do XML RSS. | Separa busca de parsing; parser ainda depende do formato do feed. |
| simple_sentiment_pt / news_features | Cria score lexical e agrega notícia por data. | Transforma texto em colunas numéricas simples. |
| rsi / build_features_targets | Calcula features de preço e target futuro. | Torna explícita a janela dos indicadores e a variável a prever. |
| backtest_next_day | Aplica sinal da data t ao retorno de t até t+1. | Evita usar o retorno do dia antes de o sinal estar disponível. |
| choose_thresholds | Testa combinações na validação e guarda a melhor. | Mantém o bloco teste fora da seleção da regra. |
| run_one_ticker / run | Executa o fluxo por ativo e imprime JSON. | Permite comparar tickers e registrar a saída de forma estruturada. |

4. Construção dos dados passo a passo

4.1 Preços: por que BRAPI e o que é baixado

brapi_quote_historical monta uma chamada para a BRAPI usando o ticker e os parâmetros PRICE_RANGE="1y" e PRICE_INTERVAL="1d". A resposta contém registros históricos; o código conserva data e close, converte o preço para número, remove fechamentos ausentes e ordena as datas. Se a fonte não retornar uma lista de preços ou faltar uma coluna esperada, devolve uma tabela vazia, para evitar que os cálculos seguintes falhem silenciosamente.

O token é uma credencial de acesso. A versão revisada tenta ler um segredo chamado BRAPI_TOKEN do Google Colab ou solicita a entrada sem eco. O token não deve ser incluído no relatório, README, notebook público ou repositório.

4.2 Notícias: origem, parser e limite

rss_url cria uma consulta para Google News RSS com o ticker como termo. parse_rss usa um parser XML para percorrer itens e recuperar título, descrição, data e link. Se o XML não puder ser interpretado, retorna uma tabela vazia com as colunas esperadas. A configuração limita a 30 notícias por ticker para reduzir volume e tempo.

Essa escolha foi feita para manter o protótipo leve, mas tem consequência metodológica importante: o feed RSS não é uma base histórica completa de notícias para cada dia do ano. Os itens e as datas disponíveis podem variar com o tempo. Ao combinar as notícias recuperadas com todo o histórico de preço, muitas datas antigas ficam sem notícia associada e recebem zeros. Portanto, não se deve concluir que o modelo teve uma série de sentimento completa e comparável ao preço durante todo o ano.

4.3 Como o texto vira números

simple_sentiment_pt limpa pontuação, separa palavras e verifica se cada termo pertence a uma lista pequena de vocabulário positivo ou negativo. O escore é (quantidade positiva menos quantidade negativa) dividido pelo total reconhecido; sem palavras reconhecidas, o denominador mínimo evita divisão por zero e o escore é zero. news_features calcula esse escore para cada notícia e agrupa por data, gerando news_count e sentiment_mean.

O método lexical foi escolhido por simplicidade, velocidade e explicabilidade. Não compreende negação (por exemplo, “não subiu”), ironia, contexto, duplicatas, relevância da notícia nem relações semânticas. É uma baseline didática, não análise financeira de sentimentos validada.

4.4 Indicadores de preço

| Variável | Como ler | Decisão de projeto |
| --- | --- | --- |
| ret_1d | Quanto o fechamento mudou em relação ao pregão anterior. | Captura movimento recente de um dia. |
| sma_10 | Média dos dez fechamentos mais recentes. | Suaviza oscilações diárias curtas. |
| rsi_14 | Indicador de movimentos recentes de alta e baixa em janela de 14. | Acrescenta informação sintética de momentum. |
| news_count | Quantidade de notícias associadas à data. | Usa volume de cobertura como sinal simples. |
| sentiment_mean | Média dos escores do vocabulário no dia. | Usa orientação lexical rudimentar. |

5. Como foi definido o que prever

5.1 Baseline D+1

A primeira hipótese era prever se o próximo dia teria retorno positivo. A vantagem é ser fácil de entender; a desvantagem é que movimentos diários têm muito ruído. A matriz registrada [[171 147], [219 183]] tem 720 observações e ROC-AUC=0.5076895397227698, próxima do nível de discriminação 0,5. A accuracy reportada foi 0.49166666666666664. O resultado serviu como baseline para motivar outras formulações.

5.2 Horizonte H=5 e margem de 0,5%

Na versão H=5, o programa calcula o preço de fechamento cinco pregões à frente com shift(-HORIZON_DAYS). O retorno futuro é fechamento futuro dividido pelo fechamento atual, menos 1. O target recebe 1 se esse retorno exceder RET_THRESHOLD=0.005 (0,5%) e 0 caso contrário. FILTER_NEUTRAL=False significa que pequenas altas, estabilidade e quedas que não ultrapassam a margem não são retiradas; ficam na classe 0.

A hipótese era reduzir o ruído de tentar acertar exatamente o próximo dia. Isso não garante melhor previsão: altera a pergunta do experimento e muda a distribuição dos rótulos. As datas finais sem preço futuro ficam sem target e são removidas. A purga de cinco linhas antes da fronteira impede que o período usado para rotular uma observação invada a janela seguinte.

6. Como o modelo aprende e como se escolhe uma regra

6.1 Divisão temporal e padronização

Com as observações válidas em ordem cronológica, train_cut separa os primeiros 60% para treino e validation_cut marca o fim dos 20% seguintes. O restante compõe teste. Ao contrário de dividir aleatoriamente, essa abordagem respeita a ordem passado→futuro. A purga remove HORIZON_DAYS observações perto das fronteiras para reduzir sobreposição dos alvos.

StandardScaler transforma as features para escalas comparáveis, calculando média e desvio no conjunto de treino. O mesmo escalador é aplicado à validação e ao teste. Isso evita que estatísticas do futuro influenciem a preparação do treino. LogisticRegression é um classificador binário simples que aprende pesos e produz p_up, uma probabilidade estimada para a classe positiva.

6.2 Métricas de classe não são backtest

O código transforma p_up em classe prevista com corte 0,5 e compara com o target real. Accuracy mede proporção de acertos; precision mede acertos entre previsões positivas; recall mede positivos reais encontrados; F1 combina precision e recall; ROC-AUC mede ordenação de positivos versus negativos. Essas métricas avaliam a classificação, não o retorno financeiro da regra.

6.3 Thresholds BUY/SELL/HOLD

No backtest há dois limiares. Se probabilidade efetiva for pelo menos buy_thr, o programa abre posição. Se ficar em ou abaixo de sell_thr, zera posição. Entre os limites conserva a posição anterior. O grid BUY_GRID=[0.55,0.60,0.65] e SELL_GRID=[0.45,0.40,0.35] testa combinações e rejeita sell maior/igual a buy, que não deixa intervalo lógico de manutenção.

6.4 Inversão e seleção na validação

O código tenta p_up e 1-p_up. roc_auc_inverted mede o desempenho da probabilidade invertida. A função choose_thresholds calcula o backtest para cada combinação na validação e mantém a combinação com maior fator de patrimônio na validação. Essa regra, depois, é congelada e testada. A inversão é apenas uma hipótese para aquele histórico; não prova que a direção oposta persistirá.

7. O que faz o backtest revisado — e o que ainda não faz

backtest_next_day calcula next_return como close(t+1)/close(t)-1. O sinal calculado em t é aplicado a esse retorno seguinte. Se a posição muda, o patrimônio é multiplicado por (1-FEE_PER_TRADE); se a posição comprada permanece, o retorno do dia seguinte é aplicado. Buy & Hold multiplica todos os retornos da mesma janela. trades_approx conta mudanças de posição.

Correção importante em relação ao script antigo: o sinal não é aplicado a um retorno que já aconteceu antes do sinal. Limitação remanescente: o classificador tem target de cinco dias, mas a estratégia captura retorno de um dia seguinte. Embora o alinhamento t→t+1 evite look-ahead imediato, as duas definições não medem a mesma pergunta econômica. Próxima revisão deveria fazer a regra manter posição até o horizonte H ou redefinir o target para o retorno de um dia, e documentar a escolha.

O backtest também é simplificado: taxa fixa por mudança não representa todos os custos, slippage, spread, impostos, liquidez, execução parcial ou impacto de mercado. Não é simulação de ordens reais.

8. Confusion matrix do baseline: tratamento e interpretação

A matriz baseline [[171 147], [219 183]] é tratada como [[TN,FP],[FN,TP]]: TN=171, FP=147, FN=219, TP=183, total=720. TN são negativos corretamente previstos; FP são falsos alarmes positivos; FN são positivos reais que o modelo não encontrou; TP são positivos corretamente encontrados.

| Métrica derivada | Valor | Significado para leitor |
| --- | --- | --- |
| Accuracy | 0.491666666666667 | Fração de todas as observações classificadas corretamente. |
| Precision | 0.554545454545455 | Das previsões positivas, fração realmente positiva. |
| Recall | 0.455223880597015 | Dos positivos reais, fração encontrada. |
| Specificity | 0.537735849056604 | Dos negativos reais, fração corretamente reconhecida. |
| F1-score | 0.500000000000000 | Equilíbrio entre precision e recall. |
| Balanced accuracy | 0.496479864826809 | Média de recall e especificidade. |
| MCC | -0.007016598874448 | Correlação entre previsão e rótulo real; zero indica associação fraca. |
| FPR | 0.462264150943396 | Fração de negativos incorretamente marcados positivos. |
| FNR | 0.544776119402985 | Fração de positivos reais perdidos. |

Esta matriz é do baseline D+1 e não deve ser confundida com as matrizes H=5 da nova execução: ITUB4 [[14,6],[9,17]], PETR4 [[4,13],[13,16]] e VALE3 [[0,37],[0,9]].

9. Linha do tempo e evolução experimental

Cada versão alterou uma decisão para investigar uma dificuldade. O par de fatores é estratégia primeiro e Buy & Hold segundo. Como os períodos, tickers, thresholds e critérios mudaram, os números não constituem uma comparação controlada única.

| Versão | Hipótese/mudança | Resultados registrados | Aprendizado e ressalva |
| --- | --- | --- | --- |
| Baseline D+1 | Prever direção do retorno do próximo dia. | accuracy=0.49166666666666664; ROC-AUC=0.5076895397227698; confusion matrix [[171 147], [219 183]]. | A AUC próxima de 0,5 e a accuracy abaixo de 0,5 apontaram desempenho fraco no teste e motivaram testar outro alvo. |
| Migração de dados para BRAPI e Google News RSS | Encontrar fontes que permitissem uma execução leve no Colab após limitações nas tentativas anteriores. | Preços BRAPI; notícias Google News RSS. | A solução diminui barreiras de execução, mas o RSS não forma uma série histórica completa e o conteúdo recuperado pode variar. |
| Leve v2 — experimento ITUB4 | Alvo H=5, thresholds e diagnóstico de sinal invertido. | ROC-AUC=0.8556521739130435; final_strategy=1.348032657957878; Buy & Hold=1.1836686944314467; 1 trade. | Resultado promissor, mas limiares foram selecionados no teste do exemplo; portanto, não era avaliação independente. |
| V3 — walk-forward | Separar treino/validação/teste em ordem temporal; configuração 2y, RSS 60, min_trades 2. | ITUB4 1.0966828809115032 vs 1.2499999999999996; PETR4 1.1000219170377215 vs 1.2113459399332591; VALE3 1.111607702106226 vs 0.8429141958126577. | A validação temporal reduz ajuste direto ao teste; evidencia também diferenças entre ativos. |
| V4 — thresholds globais | Compartilhar uma regra pelo portfólio: buy_thr=0.55, sell_thr=0.45. | ITUB4 1.0955894782386104 vs 1.2487537387836487; PETR4 1.0390643808276419 vs 1.2086763070077864; VALE3 1.0813698715747515 vs 0.8430352172334491. | Regra comum é simples, mas pode não adaptar-se a cada ativo. |
| V5 — restrição para tentar não piorar ativos | Selecionar regras na validação com critério para não piorar nenhum ativo quando possível. | ITUB4 1.28447190843006 vs 1.2537387836490523; PETR4 1.0973956020842797 vs 1.2084538375973302; VALE3 1.111607702106226 vs 0.8475130098027339. | A condição não produziu superioridade em todos os ativos. |
| V6 — maximin | Priorizar na validação o pior resultado entre os tickers. | ITUB4 1.274768144509515 vs 1.244267198404785; PETR4 1.0895166572239543 vs 1.1997775305895437; VALE3 1.111607702106226 vs 0.8481181169066913. | Proteger o pior caso não garantiu superar Buy & Hold em todos os ativos. |
| V7/V7.1 — foco PETR4 | Explorar diferentes horizontes e limiares para PETR4; seleção principal pela validação. | Best V7.1: 1.081769689124918 vs 1.207119021134594; cenário alternativo H=10 visto no teste: 1.22902842297186 vs 1.207119021134594; 6 trades. | Uma configuração observada no teste não deve ser promovida a vencedora retrospectivamente. |
| Versão final anterior — leve 3 tickers | H=5 e grid de thresholds/inversão escolhidos pelo retorno no teste. | ITUB4 1.1847522449526409 vs 1.1912502988285931; PETR4 1.3649836662346402 vs 1.3413244718727741; VALE3 0.985232617679185 vs 0.8920281204403763. | Como o teste escolhia a regra, havia viés de seleção e potencial otimismo. |
| Versão revisada — validação temporal | 60/20/20 cronológico; purga H nas fronteiras; thresholds escolhidos na validação; backtest aplica sinal a retorno do dia seguinte. | Resultados detalhados na seção 10. | A seleção fica mais limpa, mas é necessário alinhar a definição do target de H=5 ao retorno diário do backtest. |

Os fatores de patrimônio começam em 1. Por exemplo, 1.20 equivale a 1,20 vezes o patrimônio inicial; 0.90 significa terminar abaixo do início. Não são porcentagens escritas diretamente.

10. Execução anterior: reconstrução completa, linha de processamento e interpretação dos resultados

Esta seção descreve em detalhe o arquivo pipeline_leve_janela_v2_3tickers.py, que foi executado antes do código temporal revisado. O objetivo é explicar o procedimento real, não apenas nomear as variáveis. A cada ticker, o programa executava as etapas abaixo.

Configuração: TICKERS continha ITUB4, PETR4 e VALE3; PRICE_RANGE era 1y; PRICE_INTERVAL era 1d; RSS_ITEMS_PER_TICKER era 30; HORIZON_DAYS era 5; RET_THRESHOLD era 0.005; TRAIN_FRAC era 0.80. Isso representa até um ano de fechamentos diários, no máximo 30 notícias processadas por ativo e um rótulo futuro de cinco pregões com margem de 0,5%.

Coleta do preço: brapi_quote_historical consultava o endpoint de cotação histórica da BRAPI, selecionava date e close, convertia timestamp para data e ordenava as observações. Se não existisse resultado/preço, a função retornava uma tabela vazia e run_one_ticker informava erro para o ticker.

Coleta de notícia: build_google_news_rss_url montava uma consulta do Google News RSS com o ticker; parse_rss tentava separar item, título, data, descrição e link por expressões regulares. Notícias acima do limite eram cortadas para manter 30 itens. Essa fonte era prática, mas não representava um arquivo completo de notícias do ano.

Sentimento: simple_sentiment_pt limpava sinais de pontuação, separava palavras e contava termos encontrados em POS_WORDS e NEG_WORDS. O escore (positivos menos negativos) dividido pelo total classificado era resumido por dia. Não havia interpretação de contexto, negação ou duplicidade.

Features/target: o código criava retorno diário ret_1d, média móvel sma_10 e RSI rsi_14; associava news_count e sentiment_mean por data; criava future_ret_H deslocando close cinco linhas para a frente; e marcava target_up_H=1 quando o retorno excedia 0.005. Uma ressalva: future_close inexistente no final da série podia produzir comparação falsa e classe 0; o script anterior não preservava explicitamente o rótulo como ausente nessas linhas.

Divisão: depois de retirar linhas com features ausentes, n era o número de linhas; cut=floor(n*0.80). As primeiras 80% iam para treino e as últimas 20% para teste. Não havia bloco de validação independente.

Treino e métricas: StandardScaler era ajustado no treino e transformava treino/teste; LogisticRegression era ajustada com target_up_H do treino. No teste, predict_proba gerava p_up. O corte fixo 0.5 gerava pred para accuracy, F1, precision, recall e confusion_matrix; roc_auc media p_up e roc_auc_inverted media 1-p_up.

Busca da regra: o código varria invert False/True, buy_grid [0.55,0.60,0.65] e sell_grid [0.45,0.40,0.35], descartava sell >= buy e executava backtest_probs para cada combinação na própria tabela test_bt. Guardava a combinação de maior final_strategy.

Consequência metodológica: como a regra foi escolhida pelo retorno do conjunto de teste e depois reportada nesse mesmo conjunto, o resultado não era uma medição imparcial fora da amostra. Entre muitas combinações, alguma pode vencer por acaso naquele período; escolher a vencedora depois de olhar o teste produz viés de seleção/otimismo.

Backtest anterior: test_bt guardava date, close e p_up. A função backtest_probs calculava ret_1d na mesma linha e atualizava a posição pelo limiar. Descontava FEE_PER_TRADE=0.0005 quando a posição mudava, contava essa mudança como trades_approx e comparava o patrimônio simulado com Buy & Hold. Essa mecânica simplificada não incluía slippage, spread, liquidez nem execução real.

10.1 Resultados completos da execução anterior

| Ticker | Accuracy | Precision | Recall | F1 | ROC-AUC | AUC invertida | Matriz de teste | Invert / buy / sell | Estratégia anterior | Buy & Hold anterior | Trades |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| ITUB4 | 0.723404255319149 | 0.9444444444444444 | 0.5862068965517241 | 0.723404255319149 | 0.7547892720306514 | 0.24521072796934865 | [[17, 1], [12, 17]] | true / 0.65 / 0.45 | 1.1847522449526409 | 1.1912502988285931 | 3 |
| PETR4 | 0.40425531914893614 | 0.5294117647058824 | 0.3103448275862069 | 0.391304347826087 | 0.30459770114942525 | 0.6954022988505748 | [[10, 8], [20, 9]] | true / 0.55 / 0.4 | 1.3649836662346402 | 1.3413244718727741 | 3 |
| VALE3 | 0.3617021276595745 | 0.2571428571428571 | 0.6923076923076923 | 0.375 | 0.49547511312217196 | 0.504524886877828 | [[8, 26], [4, 9]] | true / 0.55 / 0.4 | 0.985232617679185 | 0.8920281204403763 | 5 |

10.2 Interpretação da execução anterior por ativo

ITUB4: a matriz [[17,1],[12,17]] mostra 17 negativos corretamente encontrados, 1 falso positivo, 12 falsos negativos e 17 positivos encontrados. A precision 0.9444444444444444 era alta, mas recall 0.5862068965517241 deixava muitos positivos sem identificação. ROC-AUC=0.7547892720306514 parecia relativamente favorável, porém a estratégia final 1.1847522449526409 ficou ligeiramente abaixo de Buy & Hold 1.1912502988285931. Portanto, a classificação não se traduziu em vantagem de patrimônio no backtest.

PETR4: ROC-AUC original 0.30459770114942525 ficou abaixo de 0.5 e a AUC de 1-p_up foi 0.6954022988505748. O grid usou inversão; contudo, esse diagnóstico e a escolha dos thresholds foram avaliados no próprio teste. O patrimônio da estratégia foi 1.3649836662346402 versus 1.3413244718727741 do benchmark. Como a regra foi escolhida para maximizar esse mesmo teste, a diferença não pode ser tomada como confirmação independente de superioridade.

VALE3: AUC original 0.49547511312217196 e invertida 0.504524886877828 ficaram próximas de 0.5. O fator de estratégia 0.985232617679185 terminou abaixo do capital inicial 1, embora fosse superior ao Buy & Hold 0.8920281204403763. Logo, dizer apenas que superou o benchmark esconderia que a estratégia também perdeu valor em termos absolutos.

10.3 O que é diretamente comparável e o que não é

O resultado anterior e o atual usam relatórios e código diferentes. O anterior escolhia thresholds e inversão no teste; o atual escolhe na validação. O atual também remove corretamente rótulos sem futuro conhecido, faz purga H nas fronteiras e aplica sinal em t ao retorno t+1. A mudança simultânea desses elementos impede atribuir toda diferença a apenas uma decisão. A nova execução deve ser descrita como novo protocolo, não como simples correção numérica da execução anterior.

10.4 Explicação sem abreviações da execução anterior, passo a passo

Nesta subseção, a execução antiga é explicada como se a pessoa estivesse acompanhando o programa pela primeira vez. O resultado não surgiu de uma única previsão: para cada ticker, o script coletou os dados, formou linhas de análise, treinou o modelo, calculou as probabilidades, procurou a regra que mais rendia dentro do teste e calculou o fator final. Cada fase influencia a seguinte.

O programa começava com um ticker, por exemplo ITUB4. O valor PRICE_RANGE="1y" pedia cerca de um ano de histórico de preços diários; não significa que todos os dias corridos tenham pregão. A fonte entregava close, o preço de fechamento usado na análise. Se a resposta não tinha resultados, o script devolvia erro daquele ticker e passava ao próximo.

Na mesma função, o programa buscava notícias pelo ticker no feed RSS. A configuração permitia processar no máximo 30 itens. Isso não significa 30 notícias para cada dia do ano; são até 30 itens retornados pela consulta naquele momento. A data e o texto eram extraídos e notícias da mesma data agrupadas.

Para cada notícia, o programa formava um texto com título e descrição. simple_sentiment_pt contava ocorrências de palavras presentes nas listas positivas e negativas. Por exemplo, se reconhecesse duas palavras positivas e uma negativa, o escore seria (2-1)/(2+1)=1/3. Se não reconhecesse nenhuma, o escore seria zero. Isso não interpreta se uma palavra aparece negada ou em ironia.

O histórico de preços virava uma tabela diária. pct_change calculava ret_1d. sma_10 calculava a média móvel de 10 fechamentos. rsi calculava um indicador com altas e baixas recentes em 14 observações. Para cada dia com notícia casada, o programa acrescentava news_count e sentiment_mean. Dias sem notícia casada recebiam zero nessas variáveis.

O target era construído comparando fechamento atual e fechamento cinco pregões à frente: future_ret_H=close(t+5)/close(t)-1. Se esse valor fosse maior que 0.005, target_up_H recebia 1; caso contrário, 0. FILTER_NEUTRAL=False não removia retornos pequenos. No código anterior, o preço futuro ausente nas últimas cinco linhas não era preservado corretamente como ausente antes do teste booleano; essas linhas podiam virar classe 0. A versão revisada corrigiu isso.

A tabela final de treino/teste só mantinha linhas com valores para as cinco features e para target. O script antigo calculava cut=floor(n*0.80), onde n era o tamanho dessa tabela. Linhas antes do corte iam para treino; linhas do corte até o fim iam para teste. Portanto, o teste era cronologicamente posterior, mas não havia terceiro bloco de validação para selecionar parâmetros.

StandardScaler era ajustado usando X_train: calculava médias e dispersões do treino e transformava as features. Depois aplicava a mesma transformação ao teste. LogisticRegression recebia X_train escalado e os rótulos y_train. Após o ajuste, predict_proba(X_test) devolvia, para cada linha de teste, uma probabilidade p_up entre 0 e 1.

Com corte fixo de 0.5, o código marcava cada exemplo com previsão positiva se p_up>=0.5. A comparação da previsão com y_test gerava quatro contagens: verdadeiros negativos TN, falsos positivos FP, falsos negativos FN e verdadeiros positivos TP. A partir daí calculava accuracy, F1, precision, recall e matriz. ROC-AUC não depende do corte 0.5 do mesmo modo: avalia a ordenação das probabilidades ao variar o limiar.

Em seguida, o código testava 18 possibilidades de regra: três thresholds de compra vezes três de venda, para cada um dos dois estados invert (sem inversão e com 1-p_up), ignorando pares em que sell>=buy. Em cada possibilidade, a regra percorria o período de teste: probabilidade acima do buy threshold abria posição; abaixo do sell threshold fechava; no intervalo mantinha a posição anterior.

Quando a posição mudava, o simulador multiplicava o patrimônio por 1-FEE_PER_TRADE, com fee=0.0005. Quando estava comprado, multiplicava também pelo fator de retorno diário. Equidade começava em 1.0. O benchmark Buy & Hold multiplicava todos os fatores diários de preço no mesmo segmento. trades_approx somava mudanças de posição, não ordens reais completas.

Ao final, o loop comparava final_strategy das regras e guardava a maior. Como essa busca rodava sobre test_bt, aquele período era simultaneamente o lugar de escolha e o lugar de avaliação. Essa é a razão específica para dizer que os resultados antigos poderiam ser otimistas: dentre várias regras testadas, foi reportada a melhor justamente no conjunto que depois era descrito como teste.

10.5 Cálculos concretos para interpretar os resultados antigos

Para ITUB4, o fator de estratégia 1.1847522449526409 indica o valor final simulado dividido pelo valor inicial; a referência terminou em fator 1.1912502988285931. A estratégia não superou a referência. Para PETR4, 1.3649836662346402 excedeu 1.3413244718727741 nessa amostra, mas a seleção da regra no próprio teste impede tratar a diferença como teste independente. Para VALE3, 0.985232617679185 significa terminar cerca de 1.48% abaixo do valor inicial (fator 1), enquanto 0.8920281204403763 significa benchmark cerca de 10.80% abaixo do valor inicial. A estratégia caiu menos, mas também perdeu valor absoluto.

Na matriz de baseline D+1, accuracy=(TN+TP)/total=(171+183)/720=0.49166666666666664. Precision=TP/(TP+FP)=183/(183+147)=0.5545454545454546. Recall=TP/(TP+FN)=183/(183+219)=0.4552238805970149. F1=2*precision*recall/(precision+recall)=0.5. Esses cálculos pertencem ao baseline D+1 e não são as métricas das matrizes da versão H=5.

10.6 O que a professora pode perguntar sobre a execução anterior

| Pergunta provável | Resposta explícita baseada no procedimento |
| --- | --- |
| O modelo escolheu thresholds antes de ver o teste? | Não. O código antigo calculava os retornos das combinações diretamente em test_bt e escolhia a maior final_strategy. |
| Então o teste era independente? | Não para o backtest: os mesmos dados influenciaram a escolha da regra. As métricas classificatórias com corte 0.5 e o retorno da regra respondem análises diferentes. |
| Por que há ROC-AUC original e invertida? | Uma usa p_up; a outra usa 1-p_up. O modelo antigo testou ambas no próprio teste ao escolher a regra. |
| As 30 notícias cobrem o ano inteiro? | Não. É um limite de itens RSS consultados; não é um arquivo histórico diário anual. |
| A versão antiga removeu o futuro inexistente no fim? | Não de forma explícita e segura; a versão revisada mantém o target ausente e descarta essas linhas. |
| O retorno do backtest é uma ordem real? | Não. É uma simulação simplificada com custo fixo por mudança de posição. |
| Os resultados antigos e atuais podem ser comparados como experimento isolado? | Não diretamente. Mudaram seleção, tratamento de targets, purga e alinhamento temporal do backtest; também podem ter mudado os dados consultados. |

11. Comparação explicada: execução anterior versus execução atual

A tabela resume cada fase do processamento em paralelo. Ela é necessária porque “execução atualizada” não quer dizer apenas trocar alguns números: o protocolo mudou em pontos que afetam diretamente como o resultado deve ser interpretado.

| Fase | Execução anterior — o que fazia | Execução atual — o que faz | Por que a mudança importa |
| --- | --- | --- | --- |
| Dados e ativos | BRAPI e Google News RSS; três tickers; preço de um ano; até 30 itens RSS. | Mantém BRAPI/RSS, três tickers, preço de um ano e limite de 30 itens. | A fonte e a limitação de notícias continuam; RSS ainda não é histórico completo anual. |
| Features e target | Retorno diário, SMA10, RSI14, notícias; target H=5 e limite 0.005; datas sem preço futuro podiam virar classe 0. | Mesmo objetivo geral; target sem futuro fica NaN e é removido; purga cinco linhas nas fronteiras temporais. | Evita contar como queda um período cujo resultado futuro não é conhecido e reduz sobreposição de rótulos entre blocos. |
| Divisão | 80% treino e 20% teste; sem bloco de validação. | 60% treino, 20% validação e 20% teste em ordem temporal. | A validação fornece um local próprio para escolher regra; teste fica reservado para avaliação. |
| Escolha de thresholds/inversão | Testava várias combinações no próprio teste e selecionava a que maximizava final_strategy. | Escolhe combinação pela final_strategy na validação; congela parâmetros e aplica no teste. | Reduz viés de seleção no teste; não elimina incerteza de amostra, RSS ou demais limitações. |
| Métricas de classificação | p_up convertido em classe com corte 0.5 no bloco de teste. | Também usa corte 0.5 para métricas no teste. | Valores classificatórios entre execuções só comparam se dados e rótulos forem equivalentes. |
| Retorno do backtest | Usava retorno diário da linha em que a regra era avaliada. | Calcula close(t+1)/close(t)-1 e aplica o sinal observado em t ao retorno seguinte. | Melhora o alinhamento temporal sinal→retorno, mas o target H=5 continua diferente do retorno de um dia usado no backtest. |
| Avaliação/benchmark | Estratégia e Buy & Hold simulados no trecho final; escolha de regra influenciada pelo mesmo trecho. | Regra congelada pela validação; estratégia e Buy & Hold calculados no trecho de teste. | A avaliação atual é metodologicamente mais defensável, mas usa um protocolo diferente e os números não isolam o efeito de uma única correção. |

11.1 Como ler o RESULTS_JSON atual, campo por campo

parameters.horizon_days=5 e parameters.ret_threshold=0.005 registram qual target foi construído; price_range="1y" e rss_items_limit=30 indicam configurações de entrada.

train_rows=133, validation_rows=41 e test_rows=46 contam observações disponíveis em cada bloco após features, target conhecido e purga. Essas contagens são por ticker; a matriz do ticker soma 46.

test_metrics contém previsões binárias usando corte fixo 0.5, não usando buy_thr/sell_thr do backtest. confusion_matrix no formato [[TN,FP],[FN,TP]] resume os erros e acertos.

rule_selected_on_validation informa qual inversão e thresholds venceram na validação e também o patrimônio da regra e Buy & Hold nesse bloco. Esses valores de validação não são o resultado final de teste.

test_backtest contém a avaliação final no teste, já com regra escolhida na validação. n_returns=45 porque a última das 46 datas do teste não tem um fechamento seguinte dentro do próprio bloco para calcular close(t+1)/close(t).

interpretation_note lembra que se trata de simulação histórica. Neste caso, além disso, target de cinco dias e retorno diário do backtest devem ser alinhados antes de conclusão definitiva.

11.2 Fórmulas úteis para conferir os valores do teste

A accuracy é (TN+TP)/(TN+FP+FN+TP). Precision é TP/(TP+FP). Recall é TP/(TP+FN). A matriz do scikit-learn segue linhas como classe real e colunas como classe prevista quando labels=[0,1]. AUC usa probabilidades contínuas, portanto não pode ser recalculada apenas com quatro contagens da matriz.

| Ticker | Contagem da matriz | Conferência de accuracy | Fator de retorno da estratégia | Fator do benchmark | Comparação |
| --- | --- | --- | --- | --- | --- |
| ITUB4 | TN=14; FP=6; FN=9; TP=17 | (14+17)/46 = 0.6739130434782609 | 1.075753936237348 | 1.258795934323691 | Estratégia abaixo do benchmark |
| PETR4 | TN=4; FP=13; FN=13; TP=16 | (4+16)/46 = 0.43478260869565216 | 1.3218431606508754 | 1.3611691022964503 | Estratégia abaixo do benchmark |
| VALE3 | TN=0; FP=37; FN=0; TP=9 | (0+9)/46 = 0.1956521739130435 | 1.0 | 0.8581727436572849 | Sem posição; preservou valor durante queda do benchmark |

12. Nova execução: resultados detalhados por ativo

A saída informada tem 133 linhas de treino, 41 de validação e 46 de teste por ticker. A matriz soma 46 classificações; o backtest tem 45 retornos porque o sinal precisa de close do próximo dia.

ITUB4

Classificação do teste: accuracy=0.6739130434782609; precision=0.7391304347826086; recall=0.6538461538461539; F1=0.6938775510204082; ROC-AUC=0.8442307692307692; ROC-AUC invertida=0.1557692307692308. Matriz [[14, 6], [9, 17]] = TN 14, FP 6, FN 9, TP 17.

Regra escolhida na validação: invert_signal=false, buy_thr=0.55, sell_thr=0.45. Estratégia e Buy & Hold na validação: 1.065435618912237 e 0.9641532756489496; trades de validação=2.

No teste: final_strategy=1.075753936237348; final_buy_hold=1.258795934323691; trades_approx=4.

Interpretação: ROC-AUC=0.8442307692307692 sugere boa ordenação neste conjunto, mas estratégia 1.075753936237348 ficou abaixo de Buy & Hold 1.258795934323691. Precisão classificatória não garante retorno superior.

PETR4

Classificação do teste: accuracy=0.43478260869565216; precision=0.5517241379310345; recall=0.5517241379310345; F1=0.5517241379310345; ROC-AUC=0.35902636916835695; ROC-AUC invertida=0.640973630831643. Matriz [[4, 13], [13, 16]] = TN 4, FP 13, FN 13, TP 16.

Regra escolhida na validação: invert_signal=true, buy_thr=0.55, sell_thr=0.4. Estratégia e Buy & Hold na validação: 1.10019730948507 e 1.0809548521017127; trades de validação=3.

No teste: final_strategy=1.3218431606508754; final_buy_hold=1.3611691022964503; trades_approx=4.

Interpretação: AUC invertida 0.640973630831643 é maior que a AUC original 0.35902636916835695; a validação escolheu inversão. Mesmo assim, estratégia 1.3218431606508754 ficou abaixo de Buy & Hold 1.3611691022964503.

VALE3

Classificação do teste: accuracy=0.1956521739130435; precision=0.1956521739130435; recall=1.0; F1=0.32727272727272727; ROC-AUC=0.4054054054054054; ROC-AUC invertida=0.5945945945945945. Matriz [[0, 37], [0, 9]] = TN 0, FP 37, FN 0, TP 9.

Regra escolhida na validação: invert_signal=true, buy_thr=0.55, sell_thr=0.45. Estratégia e Buy & Hold na validação: 1.0 e 0.9135559921414531; trades de validação=0.

No teste: final_strategy=1.0; final_buy_hold=0.8581727436572849; trades_approx=0.

Interpretação: a matriz tem TN=0 e FN=0: o classificador previu classe positiva nas 46 observações (37 falsos positivos e 9 verdadeiros positivos). Recall=1.0 não é sinal de bom classificador aqui; accuracy=0.1956521739130435 e AUC=0.4054054054054054 apontam problema. O backtest não operou (0 trades), ficou em 1.0 e superou Buy & Hold 0.8581727436572849 por permanecer fora da posição durante a queda, não por obter ganhos em trades.

11. Comparação com execução anterior e cautela

| Ticker | Anterior: estratégia | Anterior: Buy & Hold | Nova: estratégia | Nova: Buy & Hold |
| --- | --- | --- | --- | --- |
| ITUB4 | 1.1847522449526409 | 1.1912502988285931 | 1.075753936237348 | 1.258795934323691 |
| PETR4 | 1.3649836662346402 | 1.3413244718727741 | 1.3218431606508754 | 1.3611691022964503 |
| VALE3 | 0.985232617679185 | 0.8920281204403763 | 1.0 | 0.8581727436572849 |

A execução anterior escolhia thresholds no próprio teste; a atual os escolhe na validação. Mudar seleção, alinhamento de retorno, data dos preços e itens RSS altera o resultado. Não se deve interpretar a diferença como efeito causal de apenas uma mudança.

12. Passo a passo para repetir no Colab

Abrir Google Colab e criar notebook.

Armazenar token como Secret chamado BRAPI_TOKEN; conceder acesso do notebook ao segredo.

Carregar ou colar pipeline_validacao_temporal_3tickers.py.

Instalar dependências se necessário: pandas, numpy, scikit-learn e requests.

Executar e aguardar RESULTS_JSON dos três tickers.

Guardar data, código exato, parâmetros, mensagens de erro e JSON.

Conferir contagens de treino/validação/teste; se bloco pequeno, ampliar PRICE_RANGE para 2y.

Repetir walk-forward em diferentes datas antes de concluir sobre robustez.

13. Conclusões, limitações e próximos passos

A versão revisada melhora a disciplina de seleção ao escolher thresholds na validação e avaliar no teste. Nesta rodada, não houve superioridade ao Buy & Hold em ITUB4 e PETR4; o resultado de VALE3 correspondeu a nenhuma exposição durante uma queda. Logo, não há demonstração de uma estratégia que supere de forma consistente os três ativos.

Alinhar target de cinco pregões ao horizonte de exposição do backtest.

Avaliar várias janelas com walk-forward; uma divisão e 46 linhas de teste são limitadas.

Construir ou obter uma série de notícias por data coerente com todo o período.

Adicionar custos realistas, liquidez e regras operacionais antes de qualquer interpretação econômica.

Remover token do código compartilhável e manter segredo no ambiente.

Apresentar fatores e métricas como resultados históricos do experimento, não como promessa futura.

Trabalhos futuros:
Os resultados obtidos não demonstraram superioridade consistente em relação ao Buy & Hold, mas o experimento evidenciou limitações específicas cuja correção pode melhorar o desempenho e, principalmente, a confiabilidade da avaliação. Como trabalhos futuros, propõe-se: (i) alinhar o horizonte do alvo ao período de exposição do backtest; (ii) ampliar o histórico e adotar validação walk-forward com múltiplas janelas; (iii) construir uma base histórica de notícias coerente com todo o período analisado; (iv) substituir o sentimento lexical simples por métodos que considerem contexto, como léxicos financeiros ou modelos de linguagem; (v) realizar ablação para medir a contribuição isolada das notícias; (vi) comparar a Regressão Logística com outros classificadores e baselines ingênuos; e (vii) incorporar custos operacionais realistas e métricas de risco, como Sharpe e drawdown máximo.
