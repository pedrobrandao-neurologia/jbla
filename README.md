# JLBA — Jerk-Locked Back-Averaging

Aplicação para **back-averaging travado no abalo mioclônico** (jerk-locked back-averaging) de registros EEG–poliEMG, voltada à investigação neurofisiológica de mioclonias: detecção do correlato cortical pré-mioclônico (espícula pré-EMG da mioclonia cortical), do **Bereitschaftspotential** (potencial de prontidão) e da **ERD beta pré-abalo** (buscados na mioclonia funcional), além de **SPLA** (promediação travada no período silente) para mioclonia negativa/asterixis.

Todo o programa é um único arquivo — `index.html` — que roda inteiramente no navegador, **offline e sem dependências externas**. Nenhum dado sai da sua máquina: o sinal nunca é enviado a servidor algum.

> **Uso em pesquisa.** Ferramenta de apoio à análise. Não substitui o julgamento clínico e não constitui dispositivo médico certificado.

---

## Como executar

1. Baixe (ou clone) este repositório.
2. Abra o arquivo `index.html` em um navegador moderno (Chrome, Edge, Firefox ou Safari recentes).
3. Pronto — não há instalação, servidor nem conexão com a internet.

---

## O que o programa faz

O back-averaging responde a uma pergunta: **existe um evento EEG consistentemente acoplado no tempo ao abalo muscular?** Para isso, o programa:

1. marca o **início** (onset) de cada burst EMG do músculo escolhido como âncora;
2. recorta uma época de EEG em torno de cada abalo;
3. calcula a média dessas épocas — a atividade EEG aleatória se cancela e o transiente time-locked emerge;
4. testa estatisticamente se a onda média é real ou artefato do acaso;
5. gera um relatório com latências, amplitudes e uma interpretação de apoio.

Os principais achados que o programa ajuda a identificar:

| Achado | Padrão esperado | Significado usual |
|---|---|---|
| **Espícula pré-mioclônica** | Transiente bifásico (positivo–negativo) com pico ~20 ms antes do EMG em músculo distal da mão, máximo na região central contralateral, com inversão de polaridade através do escalpo | Mioclonia cortical |
| **Bereitschaftspotential (BP)** | Negatividade lenta crescente iniciando ≥ 400–1500 ms antes do EMG, máxima no vértice | Mioclonia funcional (movimento com preparação motora voluntária) |
| **ERD beta/gama pré-abalo** | Dessincronização (queda de potência 13–45 Hz) antes do movimento, no mapa tempo-frequência | Mioclonia funcional — mais sensível que o BP (65% vs 25%; Meppelink 2016, Beudel 2018) |
| **Onda aguda pré-silêncio (SPLA)** | Positividade precedendo o **início** do período silente em contração tônica | Mioclonia negativa de origem cortical (asterixis; Ugawa 1989) |

A ausência de qualquer transiente significativo é igualmente informativa (sugere gerador subcortical, reticular ou espinhal — a interpretar junto à duração do burst e ao padrão de recrutamento muscular), mas **ausência não exclui**: o laudo do programa lembra explicitamente as limitações do JLBA de EEG (Latorre 2023; Mima 1998).

---

## Fluxo de trabalho (as 6 abas)

O programa é organizado como um pipeline sequencial. Cada aba corresponde a uma etapa.

### 1 · Dados

**Carregar um registro próprio** — formatos aceitos:

- **TXT/CSV/TSV**: uma coluna por canal; separador espaço, tabulação, vírgula ou ponto-e-vírgula (a detecção do separador tolera decimais com vírgula, que são convertidos com aviso); cabeçalho opcional (se houver, os rótulos são usados para adivinhar o tipo de cada canal). Terminação CRLF ou LF, espaços múltiplos e linha final vazia são tolerados; linhas com contagem de colunas inconsistente ou valores não numéricos geram erro apontando a linha e a coluna.
- **Formato BacAv** (Vial et al., *Clin Neurophysiol Pract* 2020 — plataforma do NIH): separado por espaço, sem cabeçalho, 1ª coluna = tempo em segundos. É **detectado automaticamente**: a taxa de amostragem é inferida como 1/mediana(diff(tempo)) em dupla precisão, com validação de regularidade do passo (< 1% de jitter; se houver lacunas, o programa avisa e oferece continuar assumindo a mediana), e os tipos EEG/EMG de cada coluna são **sugeridos por análise espectral** (fração de potência < 5 Hz alta → EEG; potência 20–300 Hz alta ou curtose alta por bursts fásicos → EMG). A sugestão preenche os seletores do mapeamento, mas o usuário sempre pode remapear — a numeração exibida inclui a coluna de tempo como coluna 1. Arquivos grandes (o dataset público `mmc1.txt` tem ~23 MB e 304.011 linhas) são lidos **em partes, com barra de progresso e sem travar a interface**, direto para `Float32Array` por canal.
- **EDF/EDF+ e BDF (BioSemi, 24 bits)**: cabeçalho lido integralmente (rótulos, unidades, ganhos de calibração); canais de anotação são descartados. Canais com taxas de amostragem diferentes são reamostrados por interpolação linear para a taxa mais alta do arquivo.

Depois de carregar, confira a **tabela de mapeamento**: cada coluna precisa de um tipo — `EEG`, `EMG`, `ACC` (acelerômetro), `TRIG`, `TIME` (coluna de tempo) ou `ignorar`. O programa tenta adivinhar pelos rótulos, mas a atribuição correta de EEG/EMG é sua responsabilidade e determina que filtro cada canal recebe. Se houver coluna de tempo, a taxa de amostragem é recalculada a partir dela.

**Registro sintético para validação** — quatro exemplos com verdade-terreno conhecida (9 canais EEG 10-20 + 3 EMG do antebraço direito, 1000 Hz, 90 s):

- *Mioclonia cortical*: espícula positivo-negativa com pico a **−20 ms** em C3, com campo dipolar (inversão de polaridade no hemisfério oposto). O pipeline deve recuperar essa latência e a inversão.
- *Mioclonia funcional*: BP de rampa lenta iniciando ~1,6 s antes do abalo, máximo em Cz.
- *Mioclonia negativa (SPLA)*: contração tônica com períodos silentes de 120 ms, onda aguda positiva a **−25 ms** do início do silêncio e mioclonia positiva precedente em metade dos eventos.
- *Controle negativo*: abalos EMG **sem** transiente EEG acoplado. O pipeline **não** deve produzir achado significativo.

Recomenda-se rodar os três sintéticos antes de analisar dados reais: é a forma de verificar que os parâmetros escolhidos não criam nem destroem o achado.

### 2 · Pré-processamento

Todos os filtros são **Butterworth de fase zero** (aplicação forward–backward, tipo *filtfilt*): a filtragem **não desloca a latência dos picos** — requisito central, já que a latência é o desfecho diagnóstico do exame. Um autoteste numérico roda a cada carregamento da página e confirma isso no console do navegador (F12).

- **EEG**: passa-altas / passa-baixas / ordem configuráveis. Presets citados: `1–70 Hz` (mioclonia, Vial/Hallett), `0,01–50 Hz` (BP — o passa-altas precisa ser baixíssimo para não atenuar a rampa lenta), `0,05–50 Hz` (prática clássica, Shibasaki/Barrett) e `0,05–350 Hz` (banda larga de Meppelink para ERD/BP, limitada pela amostragem).
- **EMG**: presets `10–250 Hz` e `20–300 Hz` (o preset de Meppelink ajusta também o EMG para 25–1250 Hz).
- **Notch** 50 ou 60 Hz, com harmônicos até o 2º ou 3º.
- **Re-referenciamento EEG**: original, média comum, eletrodo único ou **Laplaciano de Hjorth** (subtração da média dos vizinhos 10-20 — derivação recomendada para estudos de coerência córtico-muscular e mais focal para o transiente).
- **Detrend linear** por canal (opcional, ligado por padrão).
- **Envelope EMG** (retificado puro, RMS móvel ou passa-baixas): usado **apenas pelo detector de abalos** — a média de EMG exibida usa sempre o sinal retificado puro.

Ao usar passa-altas muito baixo (ex.: 0,01 Hz), o programa troca automaticamente a forma do filtro por uma cascata de seções de 1ª ordem, numericamente estável, e avisa que a constante de tempo é longa (descarte os primeiros segundos do registro).

O visualizador de traçado permite zoom (rolagem do mouse), deslocamento (arrastar), ajuste de ganho EEG/EMG e sobreposição do sinal bruto.

### 3 · Detecção de abalos

Escolha o **canal âncora** (o EMG com os bursts mais nítidos) e o algoritmo:

- **Limiar com validação de contexto (estilo BacAv)** — um ponto só vira marcador se cruzar o limiar **e** a janela anterior estiver silenciosa (linha de base < *Amplitude antes*) **e** a janela posterior confirmar burst substancial (> *Amplitude depois*). É o padrão recomendado.
- **Linha de base + k·DP (Hodges–Bui)** — limiar estatístico sobre a mediana + k desvios robustos (IQR/1,349), com duração mínima acima do limiar.
- **TKEO + limiar adaptativo** — operador de energia de Teager–Kaiser, sensível a bursts curtos.
- **SPLA — período silente (mioclonia negativa)** — o detector espelhado: durante contração tônica mantida, procura quedas sustentadas do envelope abaixo de uma fração do nível tônico (duração entre 50 e 500 ms por padrão), exigindo atividade tônica imediatamente antes. O gatilho pode ser o **início** do silêncio (onde está o correlato cortical — Ugawa 1989) ou o **fim** (ao qual o movimento visível do asterixis se relaciona). Mioclonia positiva imediatamente precedente é contada e sinalizada, pois pode mascarar a mioclonia negativa ao exame clínico (Pollini 2024).
- **Somente manual** — marcação de precisão diretamente no traçado, para quando o detector automático não basta. Com a **edição manual** ligada: uma **linha-guia tracejada** segue o cursor mostrando o tempo exato; o **clique** marca exatamente nessa posição (arrastar continua deslocando o traçado sem marcar); **clique direito** (ou Alt+clique) remove o marcador mais próximo; **← / →** fazem o ajuste fino do último marcador em 1 ms (Shift = 10 ms); **"Desfazer último"** remove o recém-colocado. A resolução do clique em ms/pixel é exibida ao lado do intervalo — amplie com a rolagem até ≤ 1 ms/px para latências corticais. Marque sempre a **primeira deflexão que sai da linha de base silenciosa** (o início do burst, nunca o pico). O **ímã de onset** opcional ajusta cada clique ao início detectado do burst mais próximo (±50 ms), mantendo critério objetivo com velocidade manual. A edição manual também corrige marcadores dos detectores automáticos, e o relatório registra quantos marcadores foram editados manualmente.

Duas correções importantes desta etapa:

- **Compensação do envelope**: uma janela de suavização centrada faz o limiar ser cruzado *antes* do início real do burst (metade da largura da janela). A compensação (automática por padrão) torna a latência final independente do ajuste do envelope.
- **Realinhamento por correlação cruzada** (opcional): realinha cada abalo contra o modelo médio do EMG retificado. Reduz o *jitter* (a principal fonte de borramento do transiente cortical), mas não corrige viés sistemático.

A dica abaixo do botão orienta sobre o número de eventos: **< 20 abalos é insuficiente; o alvo prático é 40–50**, o que também permite a validação por metades. Intervalos muito curtos entre marcadores (< 250 ms em excesso) sugerem marcação dupla do mesmo burst.

A tabela de **latências entre músculos** (preenchida após a média) mostra início, pico, duração e amplitude de cada EMG em relação à âncora — útil para avaliar padrão de propagação (crânio-caudal rápida → reticular; lenta e bidirecional a partir de miótomo médio → propriospinal). Se outro músculo for recrutado **antes** da âncora escolhida, o programa sugere trocá-la e refaz a análise em um clique — a literatura recomenda ancorar no músculo ativado mais precocemente.

### 4 · Épocas e média

- **Janela de segmentação**: presets `−200/+100 ms` (mioclonia focal), `−500/+200 ms` (padrão) e `−2500/+500 ms` (BP).
- **Correção de linha de base**: subtração da média ou detrend linear, numa janela pré-evento configurável (aplicada só ao EEG).
- **Rejeição automática de épocas**: por amplitude pico-a-pico EEG (artefatos) e por atividade EMG na linha de base (abalos sobrepostos, que contaminariam o pré-evento).
- **Estimador**: média aritmética, mediana ou média aparada 20% (as duas últimas são mais robustas a artefatos residuais).
- **Referência de t = 0**: por padrão, o zero é realinhado ao **início do EMG retificado médio** (critério de % do pico, configurável). Isso elimina a dependência do limiar do detector — o marcador bruto sempre cai alguns ms *depois* do início real do burst, e esse atraso muda com o limiar escolhido.

O gráfico principal mostra o EEG médio (canal de análise em destaque, ±1 EPM opcional, modo *butterfly* opcional) e o EMG retificado médio, com t = 0 marcado. A convenção de polaridade **negativo para cima** pode ser ativada na legenda.

A **imagem de épocas** (raster) mostra cada abalo como uma linha colorida: um transiente real forma uma **faixa vertical** alinhada em t = 0; ruído aleatório não forma faixa. É a inspeção visual mais honesta de consistência época a época.

A **curva de convergência** plota a amplitude dos picos em função do número acumulado de épocas: se estabilizou, mais abalos não agregam; se ainda oscila, vale prolongar o registro. Responde empiricamente à pergunta "quantos eventos bastam?" — sobre a qual não há consenso na literatura (de 50 a 200 trials; Latorre 2023). Também são medidos a **duração do burst** classificada contra faixas normativas por região corporal (com a advertência explícita de que < 50 ms **não** é específico — controles normais geram bursts balísticos curtos) e o **coeficiente de variação da duração** entre épocas, cuja variabilidade alta favorece origem funcional.

### 5 · Validação

A validação estatística é parte obrigatória do método, não um extra:

- **Reordenar e dividir / par-ímpar**: a média é recalculada em duas metades independentes. Uma onda real **sobrevive nas duas metades** (correlação alta entre elas); um artefato de poucas épocas, não.
- **Bootstrap** (reamostragem das épocas): gera o intervalo de confiança da própria média.
- **Surrogatos com gatilhos aleatórios**: repete todo o processo de promediação com marcadores posicionados ao acaso (mesmo n, mesmo espaçamento mínimo), construindo o **envelope nulo** — o que o acaso produziria. Trechos da média observada que saem do envelope por tempo suficiente (duração mínima configurável) são marcados como **significativos**.
- **ERD 13–45 Hz**: decomposição tempo-frequência por Morlet (7 ciclos) em épocas de ±6 s, normalização em log contra a linha de base (−5 a −4 s) e **teste t de uma amostra com correção por permutação no nível de cluster**. O mapa tempo×frequência destaca os clusters significativos e a curva 31–45 Hz ± EPM acompanha o eixo temporal. A ERD pré-abalo favorece origem funcional com sensibilidade superior à do BP (Meppelink 2016; Beudel 2018); requer intervalos longos entre abalos — o programa avisa quando as épocas se sobrepõem.

### 6 · Relatório

- Tabela completa de **medidas e parâmetros** (picos e latências na janela de busca, duração do burst EMG e seu CV, **BP clássico e BP quantificado** — teste t da inclinação pré-movimento por época e ponto de inflexão BP precoce/tardio —, jitter, todos os filtros e critérios usados — tudo o que é preciso para reproduzir a análise).
- **Topografia e salvaguardas contra artefato**: mapas esquemáticos 10-20 nos dois picos com verificação de **inversão de polaridade** através do escalpo (a ausência de inversão sugere artefato — Pollini 2024), correlação entre amplitude do transiente e amplitude do burst através das épocas (dependência sugere contaminação miogênica) e checagem de que o transiente precede o EMG por margem maior que o jitter.
- **Interpretação de apoio**: um veredito heurístico com as ressalvas pertinentes, seguido de uma **matriz de achados** (presente / ausente / não avaliado, com o peso que a literatura atribui a cada um) e da regra explícita de que **ausência não exclui** — incluindo o lembrete de que SEP gigante, reflexo C e coerência córtico-muscular completam os achados definitivos e ainda não são cobertos pelo programa. É apoio à leitura, não diagnóstico.
- **Exportação**:
  - Médias em **CSV longo** (`channel,time_ms,mean_uV,sem_uV,n`) e épocas individuais em CSV — prontos para R/Python;
  - Figura em **SVG** e **PNG**;
  - **Sessão em JSON** — parâmetros, marcadores (incluindo períodos silentes do SPLA), resultados de BP/ERD, **nunca o sinal bruto** (privacidade por construção);
  - **Relatório em HTML** autocontido, com figura, tabelas, matriz de achados e texto de métodos;
  - **Texto de métodos para manuscrito (TXT)**: parágrafo gerado dos parâmetros efetivamente usados, no nível de detalhamento exigido por periódicos de neurofisiologia — resposta direta à crítica de subespecificação do JLBA (Latorre 2023).

---

## Receita rápida

**Mioclonia cortical (suspeita):**
1. Carregue o registro → confirme o mapeamento EEG/EMG.
2. Filtros: preset *mioclonia 1–70 Hz*; notch conforme a rede local.
3. Detecção: algoritmo BacAv no músculo com bursts mais nítidos; confira os marcadores no traçado e corrija manualmente se preciso; alvo de 40–50 abalos.
4. Épocas: preset *padrão −500/+200 ms* (ou *focal*); segmentar e promediar.
5. Validação: dividir metades + executar bootstrap/surrogatos.
6. Relatório: janela de busca de pico −100 a +20 ms; exportar.

**Mioclonia funcional (busca de BP e ERD):** use os presets *BP* no filtro (0,01–50 Hz) **e** na segmentação (−2500/+500 ms) — o preset de filtro já ajusta a segmentação e a janela de busca automaticamente. Rode também a **ERD** na aba Validação. São necessários intervalos longos entre abalos (> 3 s para o BP; idealmente > 12 s para ERD com janela de ±6 s) para que a análise pré-evento não seja contaminada pelo abalo anterior.

**Mioclonia negativa / asterixis (SPLA):** com o paciente em contração tônica mantida, escolha o algoritmo *SPLA* na detecção, gatilho no **início** do silêncio para buscar o correlato cortical (e no **fim** para estudar o movimento visível). O sintético "mioclonia negativa" permite validar os parâmetros antes do dado real.

---

## Suíte de teste manual — importação BacAv (dataset `mmc1.txt` de Vial et al. 2020)

1. **Parse**: carregar o `mmc1.txt` (suplemento do artigo, PMC7033354). A interface não deve congelar; o mapeamento deve reportar 304.011 amostras, fs = 1000 Hz (inferida), duração 304,0 s e "formato BacAv detectado".
2. **Classificação**: os seletores devem sugerir coluna 1 = TIME, colunas 2–3 = EEG, colunas 4–9 = EMG (numeração contando o tempo como coluna 1).
3. **Detecção**: com o preset de filtro BP aplicado, retificação → média móvel de 50 ms → limiar 6× a mediana do envelope → refratário de 3 s no canal da coluna 5 deve gerar ≈ 49 triggers (aceitável 45–55).
4. **Back-averaging**: janela −2,5 a +1,0 s, linha de base nos 0,5 s iniciais: as médias das colunas 2 e 3 devem exibir rampa lenta iniciando ~1–1,2 s antes do onset com pico próximo de t = 0 (Bereitschaftspotential, Fig. 4 de Vial et al. 2020). Atenção: neste dataset a rampa tem **polaridade positiva** — convenção invertida, que o BP quantificado sinaliza com a ressalva apropriada.
5. **Robustez**: arquivo pequeno com CRLF, espaços duplos e linha final vazia deve ser aceito; colunas inconsistentes ou valores não numéricos geram erro claro com número da linha; decimais com vírgula são convertidos com aviso.
6. **Recarga**: carregar o `mmc1.txt` duas vezes seguidas deve substituir o dataset sem crescimento perceptível de memória.

(Esses seis passos estão automatizados na suíte de desenvolvimento com Playwright; os números acima foram validados contra o arquivo real.)

## Notas e limitações conhecidas

- **Coluna de tempo em TXT/CSV**: assume-se **segundos**. Tempo em ms fará a taxa de amostragem ser recalculada errada — confira o campo "Amostragem" após carregar.
- O processamento é síncrono no navegador: registros muito longos (> ~30 min multicanal em alta taxa) podem deixar a interface lenta durante a filtragem; a ERD com muitas épocas leva alguns segundos (barra de progresso).
- A ERD exige intervalos longos entre eventos; com épocas sobrepostas o programa executa mas sinaliza a ressalva.
- A "Interpretação de apoio" é heurística e calibrada para os padrões clássicos (mão distal, ~20 ms); latências maiores são esperadas em músculos proximais e membros inferiores — e uma ampla variedade de relações temporais é descrita (Barrett 1992).
- **Módulos ainda não cobertos** (constam como "não avaliado" na matriz de achados): SEP gigante, reflexo C / respostas de longa latência, coerência córtico-muscular e intermuscular, comparação de condições (involuntário vs voluntário) e sincronização com vídeo.

## Referências metodológicas

- Ugawa Y, Shimpo T, Mannen T. *Physiological analysis of asterixis: silent period locked averaging.* JNNP 1989;52:89-93.
- Meppelink AM, et al. *Event related desynchronisation predicts functional propriospinal myoclonus.* Parkinsonism Relat Disord 2016;31:116-8.
- Beudel M, et al. *Improving neurophysiological biomarkers for functional myoclonic movements.* Parkinsonism Relat Disord 2018;51:3-8.
- Latorre A, et al. *Rethinking the neurophysiological concept of cortical myoclonus.* Clin Neurophysiol 2023;156:2-14.
- Pollini L, et al. *Negative myoclonus: neurophysiological study and clinical impact in progressive myoclonus ataxia.* Mov Disord 2024.
- Mima T, et al. *Pathogenesis of cortical myoclonus studied by magnetoencephalography.* Ann Neurol 1998;43:598-607.
- Shibasaki H, Kuroiwa Y. *Electroencephalographic correlates of myoclonus.* Electroencephalogr Clin Neurophysiol 1975;39:455-63.
- Barrett G. *Jerk-locked averaging: technique and application.* J Clin Neurophysiol 1992;9:495-508.
- Terada K, et al. *Presence of Bereitschaftspotential preceding psychogenic myoclonus.* JNNP 1995;58:745-7.
- Hodges PW, Bui BH. *A comparison of computer-based methods for the determination of onset of muscle contraction using electromyography.* Electroencephalogr Clin Neurophysiol 1996;101:511-9.
- Tassinari CA, Rubboli G, Shibasaki H. *Neurophysiology of positive and negative myoclonus.* Electroencephalogr Clin Neurophysiol 1998;107:181-95.
- Shibasaki H, Hallett M. *Electrophysiological studies of myoclonus.* Muscle Nerve 2005;31:157-74.
- Shibasaki H, Hallett M. *What is the Bereitschaftspotential?* Clin Neurophysiol 2006;117:2341-56.
- Vial F, Attaripour S, McGurrin P, Hallett M. *BacAv, a new free online platform for clinical back-averaging.* Clin Neurophysiol Pract 2020;5:38-42.
- Latorre A, et al. *Diagnostic utility of clinical neurophysiology in jerky movement disorders.* Mov Disord Clin Pract 2025;12:272-84.
