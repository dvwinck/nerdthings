# nerdquotes — documento de design

Data: 2026-10-26 · Repositório: `nerdthings` (até aqui só documentos; nenhum código)

## Intenção

Um site público, em inglês, que mostra citações nerd. O primeiro bloco é **Star Trek** (55 quotes coladas pelo dono do projeto); outros universos virão depois como novos blocos. O usuário mais importante é **uma pessoa no navegador**. O site também é um **servidor MCP remoto** (uma URL pública que qualquer cliente MCP adiciona sem instalar nada) e uma **API JSON** — mesmo deploy, mesmos dados.

O diferencial pedido pelo dono: **cada quote é verificada** e traz a fonte (série/filme, episódio, ano). Um agente de IA vai repetir o que o MCP devolver como fato; atribuição errada é o pior defeito possível do produto.

## Decisões do dono (não reabrir)

| Tema | Decisão |
|---|---|
| Público principal | Pessoa no navegador; MCP é extra que já vem pronto |
| MCP | Remoto, URL pública, sem instalação |
| Hospedagem/stack | Cloudflare Workers, TypeScript |
| Dados | Arquivo(s) no repositório; quote/bloco novo = editar arquivo + deploy. Sem banco, login ou moderação |
| Funções v1 (site e MCP) | Quote aleatória (qualquer bloco); quote aleatória de um bloco; quotes de um personagem; quote do dia (igual para todos no dia); listar blocos e personagens. **Sem busca por texto** |
| Idioma | Site e quotes em inglês |
| Curadoria | Verificar cada quote e anotar a fonte (episódio/filme/livro). Sem fonte confirmada → fica fora do site |
| Fontes aceitas | Série, filme e livro/quadrinho do universo. Frase de pessoa real fora da obra não entra |
| Home | Uma quote grande + botão "outra" |
| Visual | Design aprovado no canvas (ver "Visual") |

Suposições apresentadas ao dono e confirmadas ("tudo correto"):

- Quote do dia troca às 00:00 UTC.
- Personagem identificado por id normalizado (`spock`, `jean-luc-picard`).
- Página separada explicando como conectar o MCP.
- Sem chave de API e sem limite de uso na v1.
- Sem domínio por enquanto; primeiro deploy em `*.workers.dev`.

## Metas, não-metas, anti-metas

**Metas**
- Abrir a home e ler uma quote bonita em < 1 s; trocar de quote com um clique.
- Agente conecta o MCP só com a URL e consegue as 5 operações.
- Adicionar um bloco novo = criar um arquivo de dados e fazer deploy, sem tocar em código.

**Não-metas (v1)**
- Busca por texto; tradução; envio de quotes por usuários; contas/login; painel admin; banco de dados; analytics; rate limit; domínio próprio.

**Anti-metas (falha mesmo que funcione)**
- Publicar atribuição errada ou inventada como fato.
- Exigir instalação, chave ou cadastro para usar o MCP ou a API.
- Página pesada/JS-dependente: o conteúdo principal precisa aparecer sem JavaScript.
- Imitar a identidade visual oficial de Star Trek (LCARS, logos) — visual próprio.

## Superfícies

### Páginas (HTML)

Design aprovado: canvas https://claude.ai/artifact/X6L8iuSCmE3S2DaAnVKEUy (privado). O fonte de cada página está em `.design-canvas/project/` (`Main.dc.html`, `Character.dc.html`, `Connect.dc.html`) — cores, fontes, espaçamentos e textos exatos estão lá. O formato `.dc.html` é do editor de design; serve só de referência visual, não é código para reaproveitar.

1. **Home (`/`)** — header com wordmark `nerd`+`quotes` (destaque âmbar) e links *Characters*, *Quote of the day*, *Use with MCP*; rótulo do bloco; quote grande (serifada) com personagem (link para a página dele) e fonte; botões **Another quote** e **Copy quote**; seção *Quote of the day* com data e "changes at 00:00 UTC"; rodapé com link para a página do MCP.
2. **Personagem (`/characters/<id>`)** — coluna esquerda com a lista de personagens do bloco e contagem; à direita nome, bloco, contagem e todas as quotes do personagem, cada uma com a fonte.
3. **Use with MCP (`/mcp-docs` ou similar)** — URL do servidor com botão copiar; comando do Claude Code (`claude mcp add --transport http nerdquotes https://<host>/mcp`); config JSON genérica; tabela das ferramentas; endpoints da API; exemplo de quote em JSON. A URL mostrada deve ser a do host real em que a página foi servida, não um placeholder.

Visual (do design aprovado): fundo `#0E1116`, texto `#EDEBE4`, secundário `#9BA3AE`/`#C9CDD3`, linhas `#232832`/`#39404C`, destaque `#F2A33A` com texto `#14100A` sobre ele. Fontes Google: Newsreader (quotes), Space Grotesk (interface), JetBrains Mono (fontes, rótulos, código). Funciona em largura de celular (menus quebram, tabela rola dentro de uma caixa). Alvos de toque ≥ 44 px, foco visível, contraste ≥ 4,5:1.

### Ferramentas MCP

Transporte Streamable HTTP em `/mcp`, sem estado (stateless).

| Ferramenta | Argumentos | Retorna |
|---|---|---|
| `random_quote` | `block` opcional (`startrek`) | Uma quote |
| `quote_of_the_day` | — | A quote do dia (UTC) |
| `quotes_by_character` | `character` (id) | Todas as quotes do personagem |
| `list_blocks` | — | Blocos e contagem de quotes |
| `list_characters` | `block` opcional | Personagens, ids e contagens |

Bloco ou personagem inexistente → erro claro que diz como descobrir os ids válidos (`list_blocks`, `list_characters`).

### API JSON (GET)

`/api/random?block=`, `/api/today`, `/api/characters/<id>`, `/api/blocks`, `/api/characters?block=`. Mesma semântica das ferramentas MCP; 404 com JSON de erro para id inexistente. CORS aberto (`*`) — é pública e só leitura.

### Forma de uma quote

```json
{
  "id": "startrek-spock-live-long-and-prosper",
  "text": "Live long and prosper.",
  "character": { "id": "spock", "name": "Spock" },
  "block": "startrek",
  "source": { "kind": "screen", "title": "Star Trek: The Original Series", "episode": "Amok Time", "year": 1967 }
}
```

`episode` ausente para filmes. Mesma forma no MCP, na API e no arquivo de dados (o arquivo pode omitir `block`, implícito pelo arquivo).

**Livros entram** (decisão do dono): romances e quadrinhos licenciados do universo também valem como fonte. Por isso:

- `source` ganha `kind`: `"screen"` (série/filme: `title`, `episode?`, `year`) ou `"book"` (romance/quadrinho: `title`, `author`, `year`).
- `character` passa a ser **opcional**. Num livro, quando quem fala é um personagem, ele é preenchido como numa série. Quando a frase é da narração ou do autor, `character` fica ausente, e o site e o MCP mostram o autor como atribuição, sem link de personagem.
- `quotes_by_character` e `list_characters` só consideram quotes com `character`. `random_quote` e `quote_of_the_day` sorteiam entre todas.
- Frase de pessoa real fora de uma obra (entrevista, discurso) **não** entra.

Exemplo de quote de livro sem personagem:

```json
{
  "id": "startrek-<slug>",
  "text": "…",
  "block": "startrek",
  "source": { "kind": "book", "title": "<título do livro>", "author": "<autor>", "year": 0 }
}
```

## Dados

- Um arquivo por bloco no repositório (ex.: `data/blocks/startrek.json`), com metadados do bloco (id, nome) e a lista de quotes.
- Uma checagem automática (teste) valida todos os arquivos: campos obrigatórios por `source.kind` (`screen` exige `title` e `year`; `book` exige `title`, `author` e `year`), ids únicos, ids de personagem consistentes (mesmo id → mesmo nome), sem aspas/espaços sobrando no texto.
- Quote do dia: escolha determinística a partir da data UTC sobre a lista em ordem estável (por `id`).

## Curadoria do bloco Star Trek

A lista original, exatamente como o dono colou (55 linhas): `docs/superpowers/specs/2026-10-26-startrek-quotes-pasted.txt`. É a única fonte de conteúdo; as quotes que aparecem nos arquivos do design são amostras dessa lista.

Para cada uma das 55 quotes:
1. Confirmar falante e obra (episódio/filme/livro e ano) em fonte confiável (ex.: Memory Alpha, transcrições; para livros, Memory Beta ou o próprio texto).
2. Se o texto colado difere da fala original, usar a fala original. Já se sabe: "Captain Cathryn Janeway" → Kathryn Janeway; aspa sem fechar na quote do Data sobre sobremesa; "Curious, how often you humans manage that which you do not want" provavelmente é "…manage to obtain that which you do not want".
3. Normalizar nomes de personagem (ex.: "Captain Kirk", "Dr. McCoy", "Lt. Worf" → decidir uma forma e usá-la sempre).
4. Registrar o resultado da verificação de cada quote num relatório de curadoria (fonte usada, o que mudou), fora do arquivo de dados.

**Quote sem fonte confirmada sai do site** (decisão do dono). Ela não entra no arquivo de dados e fica no relatório de curadoria como "pendente", com o que foi tentado. Volta ao site quando for confirmada. Consequências no design aprovado:

- O selo "Source not yet confirmed" da página de personagem não é implementado.
- A contagem mostra só quotes publicadas (no design, "13 quotes · 8 with a confirmed source" vira só "N quotes").
- O rodapé "Every quote is checked against the episode or film it comes from" fica, ajustado para incluir livros, por exemplo "…against the episode, film or book it comes from".

**Critério de "verificado"** *(decisão do Claude)*: a fala aparece numa transcrição ou numa página de episódio, filme ou livro (Memory Alpha, Memory Beta ou equivalente) que nomeia a obra e, se houver, quem fala. Fala parafraseada → usar o texto da transcrição. Duas versões da mesma cena (ex.: corte do diretor) → a versão de lançamento.

**Ponto de parada antes do deploy** *(decisão do Claude)*: com os dados e o relatório prontos, o dono revisa o relatório antes do deploy. O deploy já depende do login dele na Cloudflare, então a revisão não atrasa nada.

## Restrições

- Cloudflare Workers, plano grátis; um único Worker serve páginas, API e MCP.
- Sem banco, sem segredos, sem serviços externos em tempo de execução (as fontes Google carregam no navegador).
- O dono ainda **não confirmou se tem conta Cloudflare**; o deploy final depende de `npx wrangler login` feito por ele.
- Repositório público no GitHub (`dvwinck/nerdthings`); não fazer commit/push sem o dono pedir.

## Deixado para quem constrói

- Biblioteca do MCP (SDK oficial ou o helper da Cloudflare) e roteamento.
- Como as páginas são geradas (HTML no servidor é o esperado pelo requisito "sem JS"); "Another quote" e "Copy" podem usar JS como melhoria, desde que "Another quote" funcione sem JS.
- Caminho exato da página do MCP e da lista de personagens (`/characters`).
- Estrutura de pastas, ferramenta de teste, nomes internos.
- Forma de normalizar nomes/ids de personagens, respeitando o padrão `spock`, `jean-luc-picard`.
