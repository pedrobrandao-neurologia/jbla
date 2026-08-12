# JLBA — Jerk-Locked Back-Averaging

Aplicação para **back-averaging travado no abalo mioclônico** (jerk-locked back-averaging) de registros EEG–poliEMG, voltada à investigação neurofisiológica de mioclonias: detecção do correlato cortical pré-mioclônico (espícula pré-EMG da mioclonia cortical) e do **Bereitschaftspotential** (potencial de prontidão, buscado na mioclonia funcional).

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

Os dois achados clássicos que o programa ajuda a identificar:

| Achado | Padrão esperado | Significado usual |
|---|---|---|
| **Espícula pré-mioclônica** | Transiente bifásico (positivo–negativo) com pico ~20 ms antes do EMG em músculo distal da mão, máximo na região central contralateral | Mioclonia cortical |
| **Bereitschaftspotential (BP)** | Negatividade lenta crescente iniciando ≥ 400–1500 ms antes do EMG, máxima no vértice | Mioclonia funcional (movimento com preparação motora voluntária) |

A ausência de qualquer transiente significativo é igualmente informativa (sugere gerador subcortical, reticular ou espinhal — a interpretar junto à duração do burst e ao padrão de recrutamento muscular).

---

## Fluxo de trabalho (as 6 abas)

O programa é organizado como um pipeline sequencial. Cada aba corresponde a uma etapa.

### 1 · Dados

**Carregar um registro próprio** — formatos aceitos:

- **TXT/CSV/TSV**: uma coluna por canal; separador espaço, tabulação, vírgula ou ponto-e-vírgula; cabeçalho opcional (se houver, os rótulos são usados para adivinhar o tipo de cada canal). O formato de exportação do **BacAv** (separado por espaço, sem cabeçalho, 1ª coluna = tempo em segundos) é lido diretamente.
- **EDF/EDF+**: cabeçalho lido integralmente (rótulos, unidades, ganhos de calibração). Canais com taxas de amostragem diferentes são reamostrados por interpolação linear para a taxa mais alta do arquivo.

Depois de carregar, confira a **tabela de mapeamento**: cada coluna precisa de um tipo — `EEG`, `EMG`, `ACC` (acelerômetro), `TRIG`, `TIME` (coluna de tempo) ou `ignorar`. O programa tenta adivinhar pelos rótulos, mas a atribuição correta de EEG/EMG é sua responsabilidade e determina que filtro cada canal recebe. Se houver coluna de tempo, a taxa de amostragem é recalculada a partir dela.

**Registro sintético para validação** — três exemplos com verdade-terreno conhecida (9 canais EEG 10-20 + 3 EMG do antebraço direito, 1000 Hz, 90 s):

- *Mioclonia cortical*: espícula positivo-negativa com pico a **−20 ms** em C3. O pipeline deve recuperar essa latência.
- *Mioclonia funcional*: BP de rampa lenta iniciando ~1,6 s antes do abalo, máximo em Cz.
- *Controle negativo*: abalos EMG **sem** transiente EEG acoplado. O pipeline **não** deve produzir achado significativo.

Recomenda-se rodar os três sintéticos antes de analisar dados reais: é a forma de verificar que os parâmetros escolhidos não criam nem destroem o achado.

### 2 · Pré-processamento

Todos os filtros são **Butterworth de fase zero** (aplicação forward–backward, tipo *filtfilt*): a filtragem **não desloca a latência dos picos** — requisito central, já que a latência é o desfecho diagnóstico do exame. Um autoteste numérico roda a cada carregamento da página e confirma isso no console do navegador (F12).

- **EEG**: passa-altas / passa-baixas / ordem configuráveis. Presets: `1–70 Hz` (mioclonia) e `0,01–50 Hz` (BP — o passa-altas precisa ser baixíssimo para não atenuar a rampa lenta).
- **EMG**: presets `10–250 Hz` e `20–300 Hz`.
- **Notch** 50 ou 60 Hz, com harmônicos até o 2º ou 3º.
- **Re-referenciamento EEG**: original, média comum ou eletrodo único.
- **Detrend linear** por canal (opcional, ligado por padrão).
- **Envelope EMG** (retificado puro, RMS móvel ou passa-baixas): usado **apenas pelo detector de abalos** — a média de EMG exibida usa sempre o sinal retificado puro.

Ao usar passa-altas muito baixo (ex.: 0,01 Hz), o programa troca automaticamente a forma do filtro por uma cascata de seções de 1ª ordem, numericamente estável, e avisa que a constante de tempo é longa (descarte os primeiros segundos do registro).

O visualizador de traçado permite zoom (rolagem do mouse), deslocamento (arrastar), ajuste de ganho EEG/EMG e sobreposição do sinal bruto.

### 3 · Detecção de abalos

Escolha o **canal âncora** (o EMG com os bursts mais nítidos) e o algoritmo:

- **Limiar com validação de contexto (estilo BacAv)** — um ponto só vira marcador se cruzar o limiar **e** a janela anterior estiver silenciosa (linha de base < *Amplitude antes*) **e** a janela posterior confirmar burst substancial (> *Amplitude depois*). É o padrão recomendado.
- **Linha de base + k·DP (Hodges–Bui)** — limiar estatístico sobre a mediana + k desvios robustos (IQR/1,349), com duração mínima acima do limiar.
- **TKEO + limiar adaptativo** — operador de energia de Teager–Kaiser, sensível a bursts curtos.
- **Somente manual** — marque diretamente no traçado (clique adiciona, Alt+clique remove). A edição manual também pode ser usada para corrigir marcadores dos detectores automáticos.

Duas correções importantes desta etapa:

- **Compensação do envelope**: uma janela de suavização centrada faz o limiar ser cruzado *antes* do início real do burst (metade da largura da janela). A compensação (automática por padrão) torna a latência final independente do ajuste do envelope.
- **Realinhamento por correlação cruzada** (opcional): realinha cada abalo contra o modelo médio do EMG retificado. Reduz o *jitter* (a principal fonte de borramento do transiente cortical), mas não corrige viés sistemático.

A dica abaixo do botão orienta sobre o número de eventos: **< 20 abalos é insuficiente; o alvo prático é 40–50**, o que também permite a validação por metades. Intervalos muito curtos entre marcadores (< 250 ms em excesso) sugerem marcação dupla do mesmo burst.

A tabela de **latências entre músculos** (preenchida após a média) mostra início, pico, duração e amplitude de cada EMG em relação à âncora — útil para avaliar padrão de propagação (crânio-caudal rápida → reticular; lenta e bidirecional a partir de miótomo médio → propriospinal).

### 4 · Épocas e média

- **Janela de segmentação**: presets `−200/+100 ms` (mioclonia focal), `−500/+200 ms` (padrão) e `−2500/+500 ms` (BP).
- **Correção de linha de base**: subtração da média ou detrend linear, numa janela pré-evento configurável (aplicada só ao EEG).
- **Rejeição automática de épocas**: por amplitude pico-a-pico EEG (artefatos) e por atividade EMG na linha de base (abalos sobrepostos, que contaminariam o pré-evento).
- **Estimador**: média aritmética, mediana ou média aparada 20% (as duas últimas são mais robustas a artefatos residuais).
- **Referência de t = 0**: por padrão, o zero é realinhado ao **início do EMG retificado médio** (critério de % do pico, configurável). Isso elimina a dependência do limiar do detector — o marcador bruto sempre cai alguns ms *depois* do início real do burst, e esse atraso muda com o limiar escolhido.

O gráfico principal mostra o EEG médio (canal de análise em destaque, ±1 EPM opcional, modo *butterfly* opcional) e o EMG retificado médio, com t = 0 marcado. A convenção de polaridade **negativo para cima** pode ser ativada na legenda.

A **imagem de épocas** (raster) mostra cada abalo como uma linha colorida: um transiente real forma uma **faixa vertical** alinhada em t = 0; ruído aleatório não forma faixa. É a inspeção visual mais honesta de consistência época a época.

### 5 · Validação

A validação estatística é parte obrigatória do método, não um extra:

- **Reordenar e dividir / par-ímpar**: a média é recalculada em duas metades independentes. Uma onda real **sobrevive nas duas metades** (correlação alta entre elas); um artefato de poucas épocas, não.
- **Bootstrap** (reamostragem das épocas): gera o intervalo de confiança da própria média.
- **Surrogatos com gatilhos aleatórios**: repete todo o processo de promediação com marcadores posicionados ao acaso (mesmo n, mesmo espaçamento mínimo), construindo o **envelope nulo** — o que o acaso produziria. Trechos da média observada que saem do envelope por tempo suficiente (duração mínima configurável) são marcados como **significativos**.

### 6 · Relatório

- Tabela completa de **medidas e parâmetros** (picos e latências na janela de busca, duração do burst EMG, jitter, todos os filtros e critérios usados — tudo o que é preciso para reproduzir a análise).
- **Interpretação de apoio**: um veredito heurístico (espícula cortical / BP / sem transiente significativo / achado atípico) com as ressalvas pertinentes — n baixo, jitter alto, significância não executada. É apoio à leitura, não diagnóstico.
- **Exportação**:
  - Médias em **CSV longo** (`channel,time_ms,mean_uV,sem_uV,n`) e épocas individuais em CSV — prontos para R/Python;
  - Figura em **SVG** e **PNG**;
  - **Sessão em JSON** — parâmetros, marcadores e épocas rejeitadas, **nunca o sinal bruto** (privacidade por construção);
  - **Relatório em HTML** autocontido, com figura, tabelas e interpretação.

---

## Receita rápida

**Mioclonia cortical (suspeita):**
1. Carregue o registro → confirme o mapeamento EEG/EMG.
2. Filtros: preset *mioclonia 1–70 Hz*; notch conforme a rede local.
3. Detecção: algoritmo BacAv no músculo com bursts mais nítidos; confira os marcadores no traçado e corrija manualmente se preciso; alvo de 40–50 abalos.
4. Épocas: preset *padrão −500/+200 ms* (ou *focal*); segmentar e promediar.
5. Validação: dividir metades + executar bootstrap/surrogatos.
6. Relatório: janela de busca de pico −100 a +20 ms; exportar.

**Mioclonia funcional (busca de BP):** use os presets *BP* no filtro (0,01–50 Hz) **e** na segmentação (−2500/+500 ms) — o preset de filtro já ajusta a segmentação e a janela de busca automaticamente. São necessários intervalos longos entre abalos (> 3 s) para que a rampa pré-evento não seja contaminada pelo abalo anterior.

---

## Notas e limitações conhecidas

- **BDF (BioSemi, 24 bits)**: a extensão é aceita, mas o arquivo é atualmente lido como EDF de 16 bits — o resultado será incorreto. Prefira converter para EDF/TXT. (Correção prevista.)
- **Coluna de tempo em TXT/CSV**: assume-se **segundos**. Tempo em ms fará a taxa de amostragem ser recalculada errada — confira o campo "Amostragem" após carregar.
- **Decimal com vírgula**: suportado, mas em arquivos separados por espaço com decimais em vírgula a detecção automática de separador pode errar; prefira decimais com ponto.
- O processamento é síncrono no navegador: registros muito longos (> ~30 min multicanal em alta taxa) podem deixar a interface lenta durante a filtragem.
- A "Interpretação de apoio" é heurística e calibrada para os padrões clássicos (mão distal, ~20 ms); latências maiores são esperadas em músculos proximais e membros inferiores.

## Referências metodológicas

- Shibasaki H, Kuroiwa Y. *Electroencephalographic correlates of myoclonus.* Electroencephalogr Clin Neurophysiol 1975;39:455-63.
- Barrett G. *Jerk-locked averaging: technique and application.* J Clin Neurophysiol 1992;9:495-508.
- Terada K, et al. *Presence of Bereitschaftspotential preceding psychogenic myoclonus.* JNNP 1995;58:745-7.
- Hodges PW, Bui BH. *A comparison of computer-based methods for the determination of onset of muscle contraction using electromyography.* Electroencephalogr Clin Neurophysiol 1996;101:511-9.
- Tassinari CA, Rubboli G, Shibasaki H. *Neurophysiology of positive and negative myoclonus.* Electroencephalogr Clin Neurophysiol 1998;107:181-95.
- Shibasaki H, Hallett M. *Electrophysiological studies of myoclonus.* Muscle Nerve 2005;31:157-74.
- Shibasaki H, Hallett M. *What is the Bereitschaftspotential?* Clin Neurophysiol 2006;117:2341-56.
- Vial F, Attaripour S, McGurrin P, Hallett M. *BacAv, a new free online platform for clinical back-averaging.* Clin Neurophysiol Pract 2020;5:38-42.
- Latorre A, et al. *Diagnostic utility of clinical neurophysiology in jerky movement disorders.* Mov Disord Clin Pract 2025;12:272-84.
