# Configuração do transcritor

Gerada por `/transcritor:configurar`. Duas camadas, a primeira que existir vence:

| Escopo | Pasta | Para quê |
|---|---|---|
| Projeto | `./.transcritor/` (no diretório de trabalho) | um time, cliente ou projeto específico |
| Global | `~/.config/transcritor/` | o padrão da pessoa |

`bash <scripts>/config.sh pasta` diz qual está ativa. Cada pasta tem:

- `config.json` — os campos abaixo.
- `glossario.md` — grafias oficiais, erros recorrentes da transcrição e o **papel observado**
  de cada pessoa (quem cuida de qual assunto). Cresce com o uso: toda correção que valha para
  as próximas reuniões entra aqui (Fase 6 da skill).
- `regras.md` (opcional) — regras da empresa que não são vocabulário e valem acima das regras
  genéricas da skill: quem conduz a reunião, homônimos ("quando falam X, é sempre a pessoa Y"),
  como achar o apelido de quem está usando, onde e como publicar (título, pasta por ano),
  padrão de título dos cards. Também cresce com o uso.

A pasta pode ser um link simbólico para um repositório do time (`ln -s <repo>/transcritor
~/.config/transcritor`): assim o glossário e as regras ficam versionados e quem corrige um nome
corrige para todos.

## `config.json`

```json
{
  "empresa": "Acme",
  "time": "Mobile",
  "idioma": "pt",
  "tipo_reuniao_padrao": "daily",
  "modelo_whisper": "medium",
  "dispositivo": "cpu",
  "pasta_saida": "~/reunioes",
  "pasta_por_data": true,
  "participantes": [
    { "nome": "Maria Souza", "apelidos": ["Mari"], "papel": "Tech Lead" }
  ],
  "produtos": ["Acme Vendas"],
  "clientes": ["Cliente X"],
  "termos": ["deploy", "homologação"],
  "integracoes": {
    "publicar_em": "arquivo",
    "destino": {},
    "rastreador": "nenhum",
    "rastreador_config": {},
    "revisar_antes_de_publicar": true
  }
}
```

| Campo | Observação |
|---|---|
| `idioma` | código ISO do Whisper (`pt`, `en`, `es`…). Também é o idioma do resumo. |
| `tipo_reuniao_padrao` | `daily`, `retro`, `planning`, `cliente` ou `geral`. Escolhe o modelo em `assets/modelos/`. |
| `modelo_whisper` | `small` (rápido, erra mais), `medium` (equilíbrio), `large-v3` (melhor, ~2× mais lento e ~3 GB). |
| `dispositivo` | `cpu` sempre funciona. `cuda` só com GPU NVIDIA e cuDNN instalados. |
| `pasta_saida` | onde ficam áudio, transcrição e resumo. `pasta_por_data` cria `DD-MM-AAAA/` dentro. |
| `participantes`, `produtos`, `clientes`, `termos` | viram o prompt inicial do Whisper — é o que faz ele acertar nomes próprios. O prompt aproveita ~800 caracteres; ponha primeiro o que mais erra. |
| `integracoes.publicar_em` | `arquivo`, `confluence`, `notion`, `google_docs` ou `outro`. |
| `integracoes.destino` | coordenadas do destino, livres por ferramenta. Ex. Confluence: `{"site": "acme.atlassian.net", "espaco": "TIME", "pagina_pai": "Dailys"}`, ou com a pasta do ano já resolvida: `{"site": "…", "espaco": "TIME", "pasta_ano": "123456", "titulo_pasta_ano": "AAAA — Dailys"}`; Notion: `{"pagina_pai": "Reuniões"}`. IDs descobertos na primeira publicação podem ser gravados aqui para pular a busca. |
| `integracoes.rastreador` | `nenhum`, `jira`, `linear`, `asana`, `github`, `outro`. Usado para cruzar quem fez o quê e citar cards. |
| `integracoes.rastreador_config` | ex. Jira: `{"site": "acme.atlassian.net", "cloud_id": "…", "projeto": "ABC", "url_card": "https://acme.atlassian.net/browse/{chave}", "janela": "gravacao"}`. `janela: "gravacao"` fecha a busca no horário da gravação (padrão); `"agora"` deixa aberta. |
| `integracoes.revisar_antes_de_publicar` | mantenha `true`. Publicar é visível para o time inteiro. |
