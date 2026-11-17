---
name: resumo-reuniao
description: Transcreve localmente a gravação de uma reunião (daily, standup, retro, planning, call com cliente) e gera um resumo estruturado — por pessoa, decisões, pendências e riscos — usando o vocabulário e as regras da empresa configurados em /transcritor:configurar, com publicação opcional em Confluence, Notion etc. Use sempre que o usuário enviar ou apontar um arquivo de áudio/vídeo de reunião, apontar a pasta das reuniões, ou pedir transcrição, resumo, ata ou notas de uma reunião — inclusive quando ele só anexar o áudio e disser algo curto como "daily de hoje" ou "resume aí", e mesmo que não mencione onde publicar.
---

# Resumo de reunião

Fluxo **automático de ponta a ponta até o rascunho**: transcrever, cruzar com as ferramentas da
empresa, atribuir quem falou, escrever. Há **uma única parada**: a revisão antes de publicar
(Fase 5).

Por que assim: perguntar no meio do fluxo (confirmar o mapa de quem falou, depois confirmar a
publicação, cada um com uma lista de dúvidas) cansa quem usa todo dia. As dúvidas viram
**palpites escritos no resumo**, e a pessoa corrige tudo de uma vez numa só leitura. Corrigir
um palpite errado no rascunho custa uma frase; parar o fluxo para perguntar custa uma ida e
volta a cada reunião.

**Regra geral: não pergunte durante as Fases 1–4.** Se faltar certeza, escolha o melhor
palpite, escreva com ele e anote em "Conferir" (Fase 5). Não termine o turno com pergunta antes
do rascunho pronto.

Scripts: `${CLAUDE_PLUGIN_ROOT}/skills/resumo-reuniao/scripts/` (abaixo, `$S`).

## Fase 0 — Carregar a configuração

```bash
S="${CLAUDE_PLUGIN_ROOT}/skills/resumo-reuniao/scripts"
bash "$S/config.sh" pasta
```

- **Vazio:** não há configuração. Diga em uma linha que dá para personalizar com
  `/transcritor:configurar` e siga com o padrão (`medium`, `pt`, resumo local ao lado do
  áudio). Não bloqueie.
- **Achou:** leia, dessa pasta, `config.json`, **`glossario.md` inteiro** e **`regras.md`** se
  existir. As regras da empresa (quem conduz a reunião, homônimos, onde e como publicar, como
  achar o apelido de quem usa) valem **acima** das regras genéricas deste arquivo. Campos e
  formato em `references/configuracao.md`.

O resumo e as conversas saem no `idioma` da config (padrão: português do Brasil), mantendo
termos técnicos em inglês quando é assim que o time fala.

O trabalho fica lado a lado em `pasta_saida` (em `DD-MM-AAAA/` quando `pasta_por_data`):
o áudio, o `.transcript.txt`, o `cards.txt`, o `-resumo.md` e o `publicado.txt`. Se o áudio
chegou solto, crie a pasta da data e mova-o para lá antes de começar.

## Fase 1 — Transcrever (e adiantar o resto em paralelo)

**Antes de transcrever, confira se o arquivo terminou de chegar.** Um `.m4a` copiado do
Downloads ainda no meio do download tem metade do tamanho e não tem o átomo `moov`, e o
decodificador falha (`InvalidDataError`). Se faltar o `moov` (`grep -c moov <arquivo>` dá 0)
ou se o tamanho ainda estiver mudando, espere o download acabar e confira de novo. Só vale
tentar reconstruir o áudio se o arquivo estiver parado e mesmo assim incompleto.

**Ao retomar uma reunião**, confira primeiro se o `.transcript.txt` e o `-resumo.md` da pasta
já existem — outra sessão pode ter adiantado.

```bash
LOG=$(bash "$S/iniciar.sh" <caminho-do-audio> [--tipo retro] [--modelo small])
```

O `iniciar.sh` prepara o ambiente Python (`~/.cache/transcritor/venv`, só na primeira vez —
**é o único ponto que baixa algo**: nem a instalação do plugin nem o `/transcritor:configurar`
baixam o Whisper; não chame `preparar_ambiente.sh` fora de um pedido de transcrição),
aplica o vocabulário da config e dispara a transcrição **desacoplada da sessão**: sessões do
Claude Code encerram processos filhos comuns, e uma transcrição longa morreria no meio. O
script também grava linha a linha, então uma interrupção preserva o que já saiu. O log é um por
áudio — nunca reaproveite o log de outra reunião.

Estimativa: `medium` em CPU roda a ~2× tempo real (26 min de áudio ≈ 13 min). Na primeira
execução ele ainda instala o faster-whisper e baixa ~1,5 GB de modelo — avise. Acompanhe o
`$LOG` com um `Monitor` até a última linha ser `DONE`; não fique reportando progresso. Se o log
terminar sem `DONE`, mostre as últimas linhas e diagnostique.

**Enquanto transcreve, adiante o que não depende do texto:**

1. Leia o `glossario.md`, o `regras.md` e `references/integracoes.md`.
2. Se houver rastreador configurado, rode a busca de itens (ver `references/integracoes.md`) e
   salve a tabela resumida (chave · responsável · status · título) em `cards.txt` na pasta da
   data. É a principal fonte de atribuição na Fase 2.

## Fase 2 — Atribuir quem falou (sem parar)

Não há diarização automática. A atribuição sai do conteúdo e da ordem da conversa. **Não
apresente o mapa para confirmação**: monte-o internamente e siga direto para a escrita.

1. **Segmente por mudança de assunto.** Numa daily cada pessoa fala um bloco contínuo.
   Marcadores de virada: "então vamos começar", "e você, fulano?", "ok, show", "onde a gente
   parou?". Em reuniões livres, segmente por tema.
2. **Colha as auto-referências.** Quem cita outro em terceira pessoa ("o fulano me passou")
   não é essa pessoa. Quem é interpelado pelo nome ("boa explicação, fulano") falou no bloco
   anterior. Se dois sinais se contradizem, o problema costuma ser a fronteira do bloco, não a
   identidade.
3. **Cruze com o `cards.txt`.** O responsável pelo card que a pessoa descreve resolve a maior
   parte das ambiguidades e é evidência mais dura que qualquer pista do áudio. **Não use
   eliminação nem "ontem eu não vim" para decidir identidade**: conteúdo técnico que bate com o
   card de alguém vale mais que qualquer eliminação.
4. **Bloco que pode ser de quem está usando: confira o git dele.** O apelido de quem usa vem do
   `regras.md`; os commits dele entre a véspera e o horário da gravação
   (`git log --all --author="$(git config --global user.name)" --since=… --until=…` em cada
   repositório da pasta de projetos) são evidência dura, de graça.

Ordem de força dos sinais, do mais forte para o mais fraco: fala em primeira pessoa sobre um
card → interpelação pelo nome → responsável do card pelo assunto → papel no glossário → ordem
da rodada.

Resolva direto, **sem nem anotar em "Conferir"**:

- **Variação fonética de nome ou termo já no glossário.**
- **Responsável do card diverge, mas a fala é em primeira pessoa.** "Eu fiz…" vale mais que o
  responsável — o card muitas vezes fica com quem criou ou com quem revisa.
- **Fala de condução** ("próximo", "beleza, fulano?", perguntas de priorização) é de quem
  conduz a reunião (está no `regras.md`).
- **Interjeições e respostas curtas** ("tá pra teste", "show") são de quem está no assunto;
  status dito no áudio vai para o resumo como dito.
- **Termo com leitura óbvia pela frase.**
- **Card cujo título descreve o que foi dito**: cite o card.
- **Conversa livre depois da rodada**: quem fala em primeira pessoa de mergear a branch de
  outra pessoa é quem faz os merges, não o revisor do card.

Vai para "Conferir" (com o palpite já aplicado no texto) só o que **muda o resumo e se apoiou
em sinal fraco**: bloco atribuído apenas pela ordem da rodada, ou dois sinais fortes em
contradição. Bloco sem nenhuma pista entra no resumo sob o palpite mais provável, nunca como
seção "não identificado".

## Fase 3 — Normalizar nomenclatura

A transcrição erra nomes próprios de forma previsível. O glossário cobre os casos conhecidos;
para o que não estiver lá, nesta ordem:

1. Títulos do `cards.txt` (grafia oficial de clientes, campos e produtos).
2. Busca na base de conhecimento conectada (ver `references/integracoes.md`).
3. Leitura óbvia pelo contexto.

Se nada resolver, escreva `⟨termo como soou⟩` e anote em "Conferir". **Não marque `⟨…⟩`** em
primeiro nome de pessoa sem sobrenome conhecido, nem em termo comum do time (branch, build,
homolog): só em nome de cliente, produto, campo ou pessoa que seria publicado errado — um nome
inventado publicado é pior que uma lacuna assumida.

## Fase 4 — Escrever o resumo

Escolha o modelo em `assets/modelos/` pelo tipo da reunião (`daily`, `retro`, `cliente`;
`geral` para planning e o resto). Se o tipo real não bater com o padrão da config, use o real.
Salve como `<pasta-do-áudio>/<nome-do-áudio>-resumo.md`.

O que separa um resumo útil de uma transcrição encurtada:

- **Registre a decisão, não o debate.** O estado final da regra: o que passa a valer, para
  quem, controlado por qual configuração ou flag.
- **Cite cards como link clicável** (`[ABC-123](url)`, montado com `url_card` da config) em
  todo lugar do resumo — destaques, atualizações, tabelas —, nunca a chave solta ou em
  `código`. Vale para cada chave de um par.
- **Cabeçalho enxuto.** Depois do título vem só a **Duração** e já entra "Destaques". Sem
  linha de participantes nem aviso de nomenclatura.
- **Preserve a causa raiz.** "Corrigiu bug de notificação" não ajuda ninguém; "o ícone não
  aparecia porque faltava o recurso no manifesto" ajuda quem cair no mesmo problema.
- **Riscos merecem seção própria.** Regra que diverge entre clientes, configuração que quebra
  outros ambientes, uma pessoa sozinha segurando uma área.
- **Números e nomes conforme ditos.** Prazos, percentuais, quantidades e horários vêm do áudio;
  não arredonde nem "melhore".
- **Trecho ilegível** (áudio cortado, gente falando junto): diga qual intervalo ficou ruim em
  vez de preencher com inferência.

## Fase 5 — Revisão única e publicação

Esta é a **única** parada. Publicar é visível para o time inteiro; não publique sem o OK de
quem está usando. Apresente, nesta ordem e sem mais nada:

1. O caminho do `-resumo.md` (e o resumo em si, para ler no chat).
2. **Conferir** — no máximo ~5 linhas, só o que entrou por palpite fraco ou ficou com `⟨…⟩`.
   Uma linha por item, já no formato de correção fácil:

   > - **02:02 · João** — atribuído só pela ordem ("fiquei mexendo no módulo de relatórios")
   > - **Destaques** — `⟨Coperfil⟩` não achei no rastreador nem na base

   Sem nada a conferir, diga "Nada a conferir".
3. **Candidatas a regra** — só se houver: decisões ditas na reunião que valem além do card e
   que um agente ou um colega "corrigiria" por engano depois. Sinais: "é intencional", "deixa
   assim", "não é bug", "o cliente X usa Y", "isso é contagem, não quantidade", "a partir de
   agora…". No máximo 3, uma linha cada, com o destino que o `regras.md` indicar (ou
   `decisoes.md` da config, se ele não indicar nada):

   > - **Regra?** "o cliente Alfa ainda envia pedidos pela API antiga" (Ana, 07:40) → regras do app de vendas

   Não é regra: status de tarefa, prazo, opinião, combinado só daquele card.
4. Uma linha final: "Responda **pode publicar** ou mande as correções." (Com candidatas: "…e diga
   quais regras entram.")

Não faça perguntas abertas, não liste o que já está certo e não peça confirmação item a item.
Se vierem correções, aplique, registre o aprendizado (Fase 6) e **publique em seguida sem pedir
novo OK**, a menos que a correção mude a atribuição de um bloco inteiro — aí mostre só o trecho
alterado.

Com o OK:

- `publicar_em` = `arquivo` (ou sem config): o `.md` já é a entrega. Devolva o caminho.
- Outro destino: publique conforme `references/integracoes.md` e o `regras.md`, salve o link em
  `publicado.txt` na pasta da data e devolva o link. Se o conector não estiver disponível,
  entregue o arquivo e diga como ligar a integração.
- Pedido de alteração depois de publicado: **edite a página existente**, sem apagar o que outras
  pessoas mudaram nela (ver "Editar uma página já publicada" em `references/integracoes.md`).

## Fase 6 — Registrar o aprendizado (obrigatória)

É o que faz o transcritor perguntar menos a cada reunião. Sem esta fase a sessão não está
encerrada, mesmo com a página publicada. Toda correção de quem usa, e todo palpite que ele
**aprovou sem corrigir**, vira texto na mesma sessão:

| O quê | Onde |
|---|---|
| Nome, apelido, grafia errada da transcrição, termo de negócio, cliente | `glossario.md` da config ativa, com a data e o trecho de origem. Correção de entrada existente substitui a antiga |
| Quem cuida de qual assunto/card | coluna "Papel observado" do glossário — é o que permite atribuir sem perguntar da próxima vez |
| Nome que o Whisper precisa ouvir certo | `participantes`/`produtos`/`clientes`/`termos` do `config.json` |
| Regra da empresa (condução, homônimos, publicação) | `regras.md` da config |
| Candidata a regra **aprovada** por quem usa (Fase 5, item 3) | o destino que o `regras.md` indica; sem indicação, `decisoes.md` da config, no formato `- **<frase curta>** (<AAAA-MM-DD>, reunião <tipo>, <quem disse>): <o que é e por quê>.` Candidata sem resposta não entra |
| Regra genérica de como montar, escrever ou publicar | este `SKILL.md`, na fase correspondente, com o motivo |
| Preferência de quem usa que vale fora do transcritor | memória do projeto |

Antes de qualquer alteração, faça backup `<arquivo>.bak-DD-MM`. Ao terminar, diga numa linha
o que entrou e onde ("glossário: +5 termos; regras: condução"). Se a config estiver num
repositório compartilhado, lembre de sincronizar.

## Quando algo não se encaixa

- **Outra reunião** (retro, planning, call com cliente): o fluxo vale, mas ajuste a estrutura
  ao formato real da conversa. Forçar "o que fez / o que fará" numa retro produz um documento
  estranho.
- **Reunião sem áudio, enviada por escrito:** pule as Fases 1 e 2; o texto de cada pessoa já
  vem atribuído. Troque a linha **Duração** por "**Formato:** sem reunião; atualizações enviadas
  por escrito." Se quem usa pedir para completar a parte dele, reconstrua pelo git (`--since` =
  horário da reunião anterior), pelas sessões do Claude Code (`~/.claude/projects/*/*.jsonl`,
  timestamps em UTC) e pelo rastreador. Se a reunião seguinte já foi publicada, o que ela
  registra como feito entra aqui como "O que fará", para não aparecer duas vezes como concluído.
- **Alguém não falou:** não invente seção. Ausência é informação.
