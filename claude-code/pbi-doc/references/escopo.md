# Escopo · /pbi-doc

Define **o que vai em cada arquivo da documentação**. Esse documento é a "fonte da verdade" sobre quais campos, em qual ordem, com qual nível de detalhe — pra cada arquivo gerado.

---

## `00-overview.md` — Sumário do modelo

**Propósito**: leitor abre, entende o modelo em 30 segundos.

**Conteúdo**:
1. **Cabeçalho**
   - Nome do projeto (`.pbip`)
   - Data/hora geração
   - Tagline 1-linha do propósito (inferido a partir das tabelas: "modelo de vendas com análise temporal YoY", "modelo financeiro com DRE consolidado", etc.)

2. **Métricas-resumo**
   - N tabelas reais (excluindo auto-date)
   - N medidas
   - N relacionamentos
   - N colunas totais
   - Tamanho do .pbip (estimado pela soma dos .tmdl)

3. **Inventário de tabelas** (1 linha por tabela)
   - Tabela | Tipo (Fato / Dimensão / Medidas / Aux) | N colunas | N medidas hospedadas | Source resumido

4. **Fontes de dados** (parsing das partições M)
   - Lista única de fontes (Excel, SQL, Web, Sharepoint, etc.)
   - Path/conexão resumida
   - **Sinalizar paths pessoais** (Google Drive, OneDrive, C:\Users\) com aviso visual

5. **Configurações relevantes do modelo**
   - Auto Date/Time (on/off)
   - Culture (pt-BR/en-US/...)
   - Compatibility level
   - Outras flags importantes

---

## `01-tabelas.md` — Catálogo de tabelas

**Propósito**: pra cada tabela do modelo, descreve papel + colunas tipadas.

**Conteúdo (por tabela)**:

```markdown
## {Nome da tabela}

> {Tagline em 1 linha — papel da tabela no modelo, granularidade}
> Tipo: {Fato | Dimensão | Tabela de medidas | Auxiliar}
> Origem: {fonte resumida — ex: Vendas.xlsx (PlanilhaVendas) via Excel.Workbook}

### Descrição
{1-2 parágrafos. Se a tabela tem `description:` declarado no TMDL, usar. Senão, inferir do nome + colunas + uso em medidas.}

### Granularidade
{1 linha = ?  — ex: "1 linha por item de NFe", "1 linha por dia"}

### Colunas

| Coluna | Tipo | Papel | Notas |
|---|---|---|---|
| `cdProduto` | int64 | Chave estrangeira → FotoProduto | — |
| `Data` | dateTime | Data da venda | — |
| ... |

**Papel** = um de: Chave primária / Chave estrangeira / Atributo / Métrica / Calculada
**Notas** = sinalizar coisas relevantes: oculta, calculada, com formato especial, etc.

### Medidas hospedadas (se for "tabela de medidas")
- {Lista linkada pras medidas em 02-medidas.md, agrupadas por displayFolder}

### Source M (resumo)
```m
let Fonte = ...
in #"...":
```
{Não copiar o M inteiro se for >15 linhas — resumir os passos relevantes}
```

**Ordem**: tabelas-fato primeiro, depois dimensões, depois auxiliares (parameter tables, measure tables).

---

## `02-medidas.md` — Catálogo de medidas

**Propósito**: pra cada medida, mostra DAX original + explicação PT.

**Estrutura**: agrupar por **displayFolder** (que existe no TMDL). Se medida não tem displayFolder, agrupar em "Sem pasta".

**Parsing seguro de `displayFolder`**: ler somente o valor da própria linha `displayFolder:`. Nunca usar regex `DOTALL`, `Singleline` ou equivalente para propriedades simples (`displayFolder`, `formatString`, `description`, `lineageTag`, `annotation`), porque isso captura linhas seguintes e contamina a pasta com metadados.

```python
def prop_line(block: str, key: str, default: str = "") -> str:
    pattern = r"^\s*" + re.escape(key) + r":[ \t]*([^\r\n]+)"
    match = re.search(pattern, block, re.MULTILINE)
    return match.group(1).strip() if match else default
```

Campos proibidos em título de pasta, sidebar, nome de grupo, texto explicativo e `data-search`: `lineageTag`, `annotation`, `annotation PBI_FormatHint`, `PBI_FormatHint`.

**Conteúdo (por medida)**:

```markdown
### {Nome da medida}

`{tabela host}.{medida}` · {formatString se relevante} · {displayFolder}

**O que faz:**
{1-3 frases em PT explicando o resultado da medida sem entrar em DAX. Foco no business meaning.}

**DAX:**
\`\`\`dax
{DAX original, formatado, sem comentários originais alterados}
\`\`\`

**Como funciona:**
{Explicação técnica em PT da lógica DAX. Se medida usa outras medidas, listar quais. Se usa funções time intelligence, explicar o contexto. Se tem CALCULATE, explicar o filter modifier.}

**Usa:** {lista de outras medidas/colunas referenciadas}
**É usada por:** {lista de medidas que dependem desta — preencher após processar todas}
```

**Ordem dentro de cada displayFolder**: alfabética.

**Tom**: explicação em PT deve ser **didática mas não condescendente**. Pra analista que sabe DAX, mas pode não conhecer o modelo específico.

**Validação obrigatória**: depois de gerar `02-medidas.md` e `index.html`, confirmar que as pastas de medidas refletem apenas `displayFolder`. Para o modelo "Relatório de Comissões", as pastas esperadas são `0. Geral`, `1. FGA`, `2. Ass. Administrativo` e `3. Sucesso do Cliente`.

---

## `03-relacionamentos.md` — Mapa de relacionamentos

**Propósito**: visualizar quem se relaciona com quem, com que cardinalidade.

**Conteúdo**:

### Diagrama ASCII (texto)
```
                    ┌─────────────┐
                    │ dCalendario │
                    └──────┬──────┘
                           │ 1:N
                           ▼
       ┌──────────────┐  N  ┌─────────┐  N  ┌──────────────┐
       │ FotoVendedor │◄────│ fVendas │────►│ FotoProduto  │
       └──────────────┘     └─────────┘     └──────────────┘
                                    (bi-direcional ⚠)
```

(Se modelo tem >10 tabelas, simplificar pra showing só fato + dims principais.)

### Tabela detalhada de relacionamentos

| # | From | To | Cardinalidade | Direção | Ativo | Notas |
|---|---|---|---|---|---|---|
| 1 | fVendas.cdProduto | FotoProduto.'Cod Produto' | N:1 | **Bothdirections** | ✓ | Bi-direcional |
| 2 | fVendas.Data | dCalendario.Data | N:1 | Single | ✓ | — |

**Notas** = sinalizar bi-direcionais, inativos, M:M, etc.

### Análise rápida (1 parágrafo)
{Descrição em PT: "Modelo segue star schema com dCalendario como dimensão de tempo central. Apenas 1 fato (fVendas), 3 dimensões (Calendario, Produto, Vendedor). Único relacionamento bi-direcional é entre fVendas e FotoProduto — pode ser revisitado." Sem opinar (essa é função do review), só descrever.}

---

## `04-dependencias.md` — Grafo de dependências

**Propósito**: mostrar quem depende de quem entre as medidas. Ajuda no impacto de mudanças.

**Conteúdo**:

### Árvore por medida-raiz

Identificar **medidas-raiz** (que não são usadas por nenhuma outra) e mostrar a árvore descendente.

```
% Faturamento YoY
└─ Faturamento
│  └─ fVendas[QtdItens]
│  └─ fVendas[PrecoUnitario]
└─ Referência Faturamento LY
   └─ Faturamento (já mapeada acima)
   └─ dCalendario[Data]
```

### Lista reverse (impacto)

Pra cada medida **base** (usada por outras), listar quem depende.

```markdown
### Faturamento (base)

**Usada por:**
- Margem Bruta
- % Faturamento YoY
- Referência Faturamento LY
- Medida Selecionada

**Implicação:** mudar `Faturamento` afeta 4 outras medidas. Cuidado em refator.
```

### Tabelas mais referenciadas

Top 5 tabelas mais usadas em medidas (sinaliza onde mora a "carne" do modelo).

---

## Regras transversais

**Tom**: PT-BR direto, sem jargão desnecessário, com personalidade DWAY (provocativo quando faz sentido, mas em doc é mais sóbrio que em /pbi-modelo-review).

**Acentuação**: SEMPRE com todos os acentos (regra inviolável CLAUDE.md).

**Nunca usar `?` como substituto de acento**: textos gerados em PT-BR precisam sair em UTF-8 real. Corrigir qualquer ocorrência suspeita em textos de medidas, `O que faz`, `Como funciona`, `É usada por`, títulos, labels, navegação, markdowns e HTML final. Palavras sensíveis: `relatório`, `expressão`, `Referências`, `proporção`, `divisão`, `clínica`, `período`, `variação`, `dinâmico`, `premiação`, `confirmação`, `bonificação`, `aplicável`, `numérica`, `intermediária`, `explícitos`, `função`, `alteração`, `É usada por`. Separadores corretos: `·` e `—`, nunca `?`.

**Excluir auto-date**: tabelas `LocalDateTable_*` e `DateTableTemplate_*` **não entram** em nenhum dos 5 arquivos. Se modelo tem essas tabelas, mencionar **só** no overview ("o modelo tem Auto Date/Time ligado, gerando 2 tabelas-fantasma ocultas — para auditar isso, rode `/pbi-modelo-review`").

**Linkar entre arquivos**: usar links markdown relativos. Ex: em `02-medidas.md`, ao mencionar uma tabela, linkar pra `01-tabelas.md#nome-tabela`.

**Code blocks DAX**: usar fence ` ```dax ` pra sintaxe Markdown highlight (e o HTML aplica syntax highlight via classes `.k`, `.f`, `.s`, `.c`).

**Não inventar números**: se modelo tem 4.2M linhas em fVendas, isso vem do partition info — não inventar tamanho de dados. Se não há info, não citar.

**Não opinar**: a `/pbi-doc` descreve, não julga. Opinião é da `/pbi-modelo-review`.


---

## Placeholders do `templates/relatorio.html`

O template HTML usa estes placeholders `{{...}}` que devem ser substituídos com valores reais derivados dos `.tmdl`. Substituir SOMENTE no HTML — nunca dentro de comentários `<!-- -->` (CSS, JS, comentários ficam intocados).

### Placeholders globais

| Placeholder | Conteúdo |
|---|---|
| `{{PROJECT_NAME}}` | Nome do projeto (ex: `16 - EV16 - Dashboard Vendas`) |
| `{{PROJECT_FILENAME}}` | Nome do arquivo `.pbip` |
| `{{TIMESTAMP}}` | Data de geração (ex: `26 abr 2026`) |
| `{{PROJECT_TAGLINE}}` | 1 frase descrevendo o propósito do modelo (inferido) |
| `{{PROJECT_HERO_SUB}}` | Subtítulo do hero (ex: `EV16 Power BI Week · Aula 01 · gerado em 26 abr 2026`) |
| `{{TABLES_COUNT}}`, `{{MEASURES_COUNT}}`, `{{RELATIONSHIPS_COUNT}}`, `{{COLUMNS_COUNT}}`, `{{SIZE}}` | Métricas inteiras |

### Blocos HTML (gerados pelo Claude com base nos `.tmdl`)

| Placeholder | Conteúdo |
|---|---|
| `{{NAV_TABLES_HTML}}` | Sub-nav de tabelas (`<a>` com badges fato/dim/med/aux) |
| `{{NAV_MEASURES_HTML}}` | Sub-nav de pastas de medidas (`<a>` com counts) |
| `{{INVENTORY_TABLE_ROWS}}` | Linhas `<tr>` da tabela inventário |
| `{{DATA_SOURCES_TEXT}}` | Texto descritivo das fontes |
| `{{WARNINGS_HTML}}` | Callouts.warn pra problemas detectáveis (paths pessoais, etc.) — pode ser vazio |
| `{{CONFIG_LIST_HTML}}` | Items `<li>` da config-list (culture, compatibility, autoDateTime, etc.) |
| `{{TABLES_CARDS_HTML}}` | Todos os `<article class="table-card">` da seção 01 |
| `{{MEASURE_GROUPS_HTML}}` | Todos os `<div class="measure-group">` da seção 02 |
| `{{REL_SVG_HTML}}` | SVG inline do diagrama de relacionamentos (gerar dinâmico) |
| `{{REL_TABLE_ROWS}}` | Linhas `<tr>` da tabela de relacionamentos |
| `{{DEP_TREE_HTML}}` | Árvore de dependências (uma ou mais) |
| `{{DEP_REVERSE_HTML}}` | Cards reverse das medidas-base |
| `{{TOP_TABLES_LIST_HTML}}` | Items `<li>` com tabelas mais referenciadas |

### Regras obrigatorias para blocos `*_HTML`

Os placeholders terminados em `_HTML` devem receber **HTML valido**, nunca Markdown renderizado ou texto solto. Nao usar tabelas Markdown (`| Coluna | Tipo |`), listas sem tags, fences ``` ou pares `nome` + `tipo` concatenados. Todo texto vindo do modelo deve ser escapado para HTML (`&`, `<`, `>`, `"`) antes de entrar em atributos, tabelas ou blocos de codigo.

#### Shape de `{{NAV_TABLES_HTML}}`

Cada item do menu de tabelas deve apontar para o mesmo `id` do card (`tbl-...`) e usar `badge-inline`, igual ao HTML validado. O badge nao pode virar texto cru depois do nome sem `<span>`, senao o menu perde o visual.

```html
<a href="#tbl-dim-pessoas">dim_pessoas <span class="badge-inline dim">Dimensao</span></a>
```

Classes validas para o badge: `fato`, `dim`, `med`, `aux`. Slugs devem ser ASCII, minusculos e estaveis: `dim_pessoas` -> `tbl-dim-pessoas`, `fact_venda_itens` -> `tbl-fact-venda-itens`, `aux_Tipo_Pagamento` -> `tbl-aux-tipo-pagamento`.

#### Shape de `{{NAV_MEASURES_HTML}}`

```html
<a href="#grp-0-geral">0. Geral <span class="count">20</span></a>
```

Slugs de pastas de medidas seguem o mesmo padrao: `0. Geral` -> `grp-0-geral`, `2. Ass. Administrativo` -> `grp-2-ass-administrativo`.

O texto do link deve vir somente do `displayFolder` capturado em uma linha. Não pode conter `lineageTag`, `annotation`, `annotation PBI_FormatHint` ou `PBI_FormatHint`.

#### Shape de cada item em `{{TABLES_CARDS_HTML}}`

Cada tabela precisa ser um `<article class="table-card reveal r-dN">` completo, com `id="tbl-..."` igual ao menu e `data-search` com nome + tipo. Nao gerar apenas titulo + linhas de texto como `pessoa_idstring`; as colunas sempre entram em `<table class="col-table">`.

```html
<article class="table-card reveal r-d2" id="tbl-dim-pessoas" data-search="dim_pessoas Dimensao">
  <div class="table-card-head">
    <div class="table-card-name">dim_pessoas</div>
    <div class="table-card-meta">Dimensao · 12 colunas · 0 medidas · Tabela calculada/manual</div>
  </div>
  <div class="table-card-body">
    <p>dim_pessoas descreve entidades usadas para filtrar e segmentar as fatos.</p>
    <div class="label">Granularidade</div>
    <p>1 linha por entidade de referencia usada como filtro no modelo.</p>
    <div class="label">Colunas</div>
    <table class="col-table">
      <thead><tr><th>Coluna</th><th>Tipo</th><th>Papel</th><th>Notas</th></tr></thead>
      <tbody>
        <tr>
          <td class="col-name">pessoa_id</td>
          <td class="col-type">string</td>
          <td class="col-papel">Chave primaria</td>
          <td class="col-notas">-</td>
        </tr>
      </tbody>
    </table>
    <details class="source-m">
      <summary>Source M resumido</summary>
      <pre class="code">mode: import
source =
  let
    Fonte = ...
  in
    dim_pessoas_Table</pre>
    </details>
  </div>
</article>
```

Se nao houver Source M, omitir o `<details class="source-m">` inteiro. Se houver M, inserir o codigo em `<pre class="code">` com HTML escapado; nunca usar `<code>` sem `<pre>` para o source.

### Padrão de cada `<details class="measure-mini">`

Todas as medidas seguem este shape — medidas-âncora têm classe `.anchor` + atributo `open`:

```html
<div class="measure-group reveal" id="grp-0-geral">
  <h3 class="measure-group-title">0. Geral</h3>
  <p class="measure-group-meta">20 medidas nesta pasta</p>

<details class="measure-mini anchor reveal r-d1" open data-search="$ Total Geral 0. Geral [$ Total a Pagar FGA] + [$ Total a Pagar Ass. Administrativo]">
  <summary>
    <span class="name">{Nome}</span>
    <span class="dax">{DAX-essência em 1 linha}</span>
    <span class="toggle">+</span>
  </summary>
  <div class="mini-body">
    <p class="meta">Tabela: <code>{tabela}</code> · Format: <code>{format}</code></p>
    <p class="desc"><strong>O que faz:</strong> {explicação business}</p>
    <pre class="code">{DAX completo com syntax highlight via spans .k .f .s .c}</pre>
    <div class="label">Como funciona</div>
    <p class="desc">{explicação técnica}</p>
    <div class="label">É usada por</div>
    <p class="measure-deps"><code>{outra-medida-1}</code> · <code>{outra-medida-2}</code></p>
  </div>
</details>
</div>
```

O título `.measure-group-title`, a navegação e o `data-search` devem conter somente o nome limpo da pasta. O `data-search` deve conter nome, pasta e DAX em texto escapado para a busca funcionar, sem metadados TMDL como `lineageTag` ou `annotation`. O DAX completo dentro de `<pre class="code">` tambem deve estar escapado (`<` vira `&lt;`, `>` vira `&gt;`, `&` vira `&amp;`).

### Validação obrigatória do HTML final

Antes de finalizar `index.html`, validar:

```python
from pathlib import Path
import re

html = Path("PBIP/_docs/index.html").read_text(encoding="utf-8")
html_without_pre = re.sub(r"<pre.*?</pre>", "", html, flags=re.S)

assert "{{" not in html
assert "pessoa_idstring" not in html
assert "lineageTag" not in html_without_pre
assert "annotation PBI_FormatHint" not in html_without_pre

required = [
    "gold-grid",
    "section-orb",
    "sidebar-logo",
    "table-card",
    "measure-mini",
    "rel-svg",
]

for token in required:
    assert token in html, token
```

### Excluir tabelas auto-date

`LocalDateTable_*` e `DateTableTemplate_*` são auto-geradas pelo Power BI quando Auto Date/Time está ON — **não fazem parte da doc intencional**. Skipar.
