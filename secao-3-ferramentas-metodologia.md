## 3. Ferramentas e Metodologia

- **Extração:** manual
- **Tratamento e análise:** Power Query, DAX, Power BI
- **Visualização:** Power BI

### 3.1 Visão geral do pipeline (Power Query)

| # | Query | Entrada | Saída | Propósito |
|---|---|---|---|---|
| 1 | DO24OPEN | CSV 88 col. | 17 col. | Óbitos 2024, limpeza inicial |
| 2 | Mortalidade_Geral_2025 | CSV 88 col. | 17 col. | Óbitos 2025, limpeza inicial |
| 3 | 2024&2025 (Mortalidade_Consolidada) | Queries 1+2 | 19 col. | União, tratamento de nulos, classificação de evitabilidade, filtro de idade |
| 4 | CID-10-CATEGORIAS | CSV 7 col. | 2 col. | Dimensão de categorias CID-10 |
| 5 | CID-10-SUBCATEGORIAS | CSV 9 col. | 2 col. | Dimensão de subcategorias CID-10 (query original) |
| 5b | CID-10-SUBCATEGORIAS (2) | Query 5 + tabela manual | 2 col. | Cópia combinada — em uso efetivo nas relações e medidas |
| — | CID-10 Não Identificado | Manual | 2 col. | Códigos CID-10 não reconhecidos pelas tabelas oficiais |
| 6 | DTB_MUNICIPIOS | XLS IBGE | 6 col. | Dimensão de municípios |
| 7 | CNES (BigQuery export) | CSV 5 col. | 5 col. | Dimensão de estabelecimentos |
| — | fnClassificaEvitabilidade | causa_basica, idade | texto | Função custom de classificação (Malta et al., 2007/2011) |

*Observação: a query de ocupações (CBO2002) foi descontinuada e removida do modelo.*

---

### 3.2 Detalhamento das queries

#### 1. DO24OPEN.csv — óbitos 2024

**Carga:** CSV (`;`, 88 colunas, encoding 1252, sem qualificação de texto).

**Tipagem inicial:** numéricas → `Int64`; alfanuméricas → `text`.

**Limpeza:** remove 71 colunas fora do escopo (dados administrativos, mãe/gestação, investigação) → mantém 17 (incluindo `contador` e `ESC2010`, que só são renomeadas/tratadas na etapa seguinte).

**Renomeação (português/snake_case):** ex. `DTOBITO`→`data_obito`, `RACACOR`→`etnia`, `CAUSABAS`→`causa_basica`, `ASSISTMED`→`assistencia_medica`, `TPPOS`→`obito_investigado`.

**Ajuste final:** reconverte colunas renomeadas para `text`, preservando zeros à esquerda.

---

#### 2. Mortalidade_Geral_2025.csv — óbitos 2025

Segue o mesmo padrão da query 1, com diferenças pontuais:

**Tipagem inicial:** `EXAME`, `CIRURGIA`, `TPRESGINFO`, `MAT_CLAS`, `COVID_CLAS` entram como `text` (em vez de `Int64` como em 2024); `TP_ALTERA` entra como `Int64` (em vez de `text`).

**Limpeza e renomeação:** remove as mesmas 71 colunas e mantém as mesmas 17; renomeação idêntica à query 1.

**Ajuste final:** reconverte para `text`, incluindo explicitamente `causa_basica` (em 2024 essa coluna já estava como `text`, mas não passava por essa etapa).

---

#### 3. 2024&2025 — união e tratamento avançado

**União:** combina as tabelas de 2024 e 2025 (`Table.Combine`).

**Tratamento de nulos:** substitui nulos por `"Não informado"` em `hora_obito`, `dt_nasc`, `etnia`, `estado_civil`, `ocupacao`, `estabelecimento`, `assistencia_medica`, `necropsia` e `obito_investigado`.

**Extração de campos derivados:**
- `ano_obito` / `ano_nascimento`: últimos 4 dígitos de `data_obito` / `dt_nasc`
- `mes_obito` / `mes_nascimento`: dígitos 3–4 de `data_obito` / `dt_nasc`, com padding
- `idade_formatada`: 2 últimos dígitos de `idade` (convertido para `Int64`)
- `tipo_idade`: 1º dígito de `idade`, indicando a unidade — convertido para `Int64` (1/2/3 = dias/meses/horas, 4 = anos, 5 = anos +100)
- `hora_obito_formatada`: formata `HH:mm` a partir de `hora_obito`, com padding e tratamento de erro residual

**Escolaridade (etapa transitória):** `ESC2010` é renomeada para `escolaridade`, nulos substituídos por `"Não informado"` — mas a coluna é **descartada na limpeza final** (ver abaixo). Escolaridade foi descontinuada como variável de análise.

**Classificação de evitabilidade:** aplica `fnClassificaEvitabilidade(causa_basica, idade)`, gerando a coluna `Evitabilidade`.

**Filtro de idade:** remove óbitos com `idade_formatada > 74` **apenas quando `tipo_idade = 4`** (idade em anos completos). Óbitos registrados em dias, meses ou horas (`tipo_idade` 1/2/3) não são afetados por esse filtro.

**Limpeza final:** remove `hora_obito`, `obito_investigado`, `hora_obito_formatada`, `necropsia`, `estado_civil`, `escolaridade` — já sem uso após as transformações.

---

#### 4. CID-10-CATEGORIAS.csv — dimensão de categorias

**Carga:** CSV (`;`, 7 colunas, encoding 1252, sem qualificação de texto).

**Tipagem e limpeza:** tipa todas as colunas como `text`, promove cabeçalho, remove `CLASSIF`, `REFER`, `EXCLUIDOS`, `DESCRABREV` e a coluna sem nome.

**Renomeação:** `CAT`→`categoria`, `DESCRICAO`→`descricao`.

**Saída:** `categoria`, `descricao`.

---

#### 5. CID-10-SUBCATEGORIAS.csv — dimensão de subcategorias

**Carga:** CSV (`;`, 9 colunas, encoding 1252, sem qualificação de texto).

**Tipagem e limpeza:** tipa todas as colunas como `text`, promove cabeçalho, remove `CLASSIF`, `RESTRSEXO`, `CAUSAOBITO`, `REFER`, `EXCLUIDOS`, `DESCRABREV` e a coluna sem nome.

**Renomeação:** `SUBCAT`→`subcategoria`, `DESCRICAO`→`descricao`.

**Saída:** `subcategoria`, `descricao`.

> **Nota:** esta é a query original, ainda presente no modelo. O modelo, porém, usa em suas relações e medidas uma **cópia combinada** — `CID-10-SUBCATEGORIAS (2)` — que une esta tabela à tabela manual `CID-10 Não Identificado` (ver abaixo). Ver seção 3.3 para detalhes.

---

#### CID-10 Não Identificado — tabela manual complementar

**Origem:** criada manualmente no Power BI, sem uso de Power Query/M.

**Motivo:** alguns códigos CID-10 presentes nos óbitos registrados não foram reconhecidos automaticamente pelas tabelas oficiais `CID-10-CATEGORIAS`/`CID-10-SUBCATEGORIAS`. Em vez de inserir esses códigos diretamente nas tabelas originais (o que misturaria dado oficial com dado suplementado manualmente), optou-se por uma tabela separada, mais fácil de auditar e manter.

**Colunas:** `CID-10` (código), `Descrição` (texto livre, preenchido manualmente).

**Exemplos de códigos cobertos:** A090, A099, K358, N185, O142, O960, O969 — entre outros não capturados pelas listas oficiais.

**Uso no modelo:** esta tabela é combinada (`Append`/`Union`) com uma cópia de `CID-10-SUBCATEGORIAS`, formando a query `CID-10-SUBCATEGORIAS (2)` — que é a tabela efetivamente usada nas relações do modelo e nas medidas DAX (ex.: o `ALL('CID-10-SUBCATEGORIAS (2)')` na medida `1. Proporcao evitável (Brasil)`, seção 3.5).

---

#### 6. RELATORIO_DTB_BRASIL_2024_MUNICIPIOS.xls — dimensão de municípios

**Carga:** planilha Excel (tabela `DTB_Municípios`).

**Limpeza:** remove linhas de cabeçalho/metadados do relatório IBGE (título, data-base, "UF: 99 - BRASIL"), promove a linha correta a cabeçalho.

**Tipagem:** `UF`, `Município`, `Código Município Completo` → `Int64`; nomes → `text`.

**Função customizada `RemoveAccents`:** remove acentuação caractere a caractere (mapeamento vogais acentuadas/ç/ñ → equivalentes sem acento) via `List.Accumulate`, aplicada a `Nome_UF`, nomes de região intermediária/imediata e `Nome_Município`.

**Limpeza adicional:** remove colunas auxiliares (`Column10`–`Column17`) e colunas de região geográfica intermediária/imediata (fora do escopo).

**Compatibilização de código municipal:** cria `cd_mun_formatado`, truncando `Código Município Completo` (7 dígitos, IBGE) para os 6 primeiros dígitos — compatível com o formato usado no SIM.

**Renomeação final:** `UF`→`cod_uf`, `Nome_UF`→`unidade_federativa`, `Município`→`cod_municipio`, `Código Município Completo`→`cod_municipio_completo`, `Nome_Município`→`nome_municipio`.

**Saída:** `cod_uf`, `unidade_federativa`, `cod_municipio`, `cod_municipio_completo`, `nome_municipio`, `cd_mun_formatado`.

---

#### 7. CNES (BigQuery export) — dimensão de estabelecimentos

**Carga:** CSV exportado do BigQuery (`,`, 5 colunas, encoding UTF-8, sem qualificação de texto).

**Tipagem:** todas as colunas como `text` — `sigla_uf`, `sigla_uf_nome`, `id_estabelecimento_cnes`, `indicador_vinculo_sus`, `tipo_unidade`.

---

#### Função customizada: fnClassificaEvitabilidade

Implementa a classificação de evitabilidade CID-10 seguindo Malta et al., com duas tabelas de referência conforme a faixa etária:

- **`TabelaMenor5`** — critérios de 2007, aplicada a óbitos de 0 a 4 anos.
- **`Tabela5a74`** — critérios atualizados de 2011, aplicada a óbitos de 5 a 74 anos.

**Lógica:**
1. Normaliza o CID (maiúsculas, sem espaços) e a idade (extrai unidade — dias/meses/anos — e valor).
2. Seleciona a tabela de referência conforme a idade em anos.
3. Busca o CID nos intervalos da tabela via função auxiliar `IsInInterval` (compara por prefixo exato ou intervalo numérico dentro da mesma letra).
4. Retorna a classificação encontrada, ou `"Mal-definida"` (CIDs iniciados em R, exceto R95), ou `"Não Evitável - 3. Demais causas"`, ou `"Não Aplicável"` (fora da faixa etária coberta).

---

### 3.3 Tabelas de dimensão categórica (criadas manualmente)

O SIM registra várias variáveis como códigos numéricos (ex.: sexo, raça/cor, local de óbito). Para exibir esses códigos de forma legível nos visuais e nas medidas DAX, foram criadas manualmente no Power BI (sem uso de Power Query/M) tabelas de dimensão que mapeiam cada código à sua descrição:

| Tabela | Descodifica |
|---|---|
| `descricao_sexo` | Código de sexo |
| `descricao_etnia` | Código de raça/cor |
| `descricao_local_obito` | Código de local do óbito |
| `descricao_assistencia_medica` | Código de assistência médica (teve/não teve assistência) |
| `descricao_idade` | Código de idade/unidade de idade |
| `descricao_cnes` | Código de tipo de estabelecimento (CNES) |

Essas tabelas se relacionam com a tabela de fatos `2024&2025` e são usadas tanto em visuais quanto nas medidas DAX de apoio à narrativa (seção 3.5, Grupo 2).

---

### 3.4 Colunas calculadas (DAX)

| Coluna | Descrição |
|---|---|
| `faixa_etaria` | Classifica cada óbito por faixa etária, a partir de `tipo_idade` e `idade_formatada` (colunas geradas na query `2024&2025`). Avaliada em ordem: `tipo_idade` em {1,2,3} (idade registrada em dias/meses/horas) → "Recém-nascido"; `idade_formatada` ≥ 60 → "Idoso"; ≥ 18 → "Adulto"; ≥ 12 → "Adolescente"; ≥ 0 → "Criança"; caso contrário → "Não classificado". |

```dax
faixa_etaria = 
SWITCH(
    TRUE(),
    '2024&2025'[tipo_idade] IN {1, 2, 3}, "Recém-nascido",
    '2024&2025'[idade_formatada] >= 60, "Idoso",
    '2024&2025'[idade_formatada] >= 18, "Adulto",
    '2024&2025'[idade_formatada] >= 12, "Adolescente",
    '2024&2025'[idade_formatada] >= 0, "Criança",
    "Não classificado"
)
```

---

### 3.5 Medidas DAX

#### Grupo 1 — KPIs gerais (Brasil)

| Medida | Descrição |
|---|---|
| `1. Proporcao evitável (Brasil)` | Proporção de óbitos evitáveis sobre o total nacional. O denominador usa `ALL('CID-10-SUBCATEGORIAS (2)')` propositalmente: quando a pessoa filtra por uma causa (subcategoria CID-10) específica, esse filtro afeta apenas o numerador (óbitos evitáveis), não o total geral usado como base do cálculo. |
| `2. Óbitos no total (Brasil)` | Contagem total de óbitos (`COUNTROWS`). |
| `3. Óbitos evitáveis (Brasil)` | Contagem de óbitos evitáveis (exclui "Não Aplicável", "Não Evitável" e "Mal-definida"). |
| `4. Recém-nascido (óbitos evitáveis)` | Proporção de óbitos evitáveis dentro da faixa "Recém-nascido". |
| `5. Crianças (Óbitos evitáveis)` | Proporção de óbitos evitáveis dentro da faixa "Criança". |
| `6. Adolescentes (Óbitos evitáveis)` | Proporção de óbitos evitáveis dentro da faixa "Adolescente". |
| `7. Adultos (Óbitos evitáveis)` | Proporção de óbitos evitáveis dentro da faixa "Adulto". |
| `8. Idosos (Óbitos evitáveis)` | Proporção de óbitos evitáveis dentro da faixa "Idoso". |
| `9. Taxa de atendimento médico (Brasil)` | Proporção de óbitos com assistência médica (`descricao_assistencia_medica[teve_assistencia] = "Sim"`) sobre o total. |
| `9.1 Óbitos que tiveram atendimento médico` | Contagem absoluta de óbitos com assistência médica. |

*Medidas 4–8 usam a coluna calculada `faixa_etaria` (seção 3.4).*

#### Grupo 2 — Medidas de apoio à narrativa (drill-down macro→micro)

Identificam, em cascata, o estado e o município com maior proporção de óbitos evitáveis, e então a característica predominante dos óbitos evitáveis *dentro* desse município.

| Medida | Descrição |
|---|---|
| `1. UF Maior Proporcao` | UF com maior proporção de óbitos evitáveis (usa medida 1 do Grupo 1 sobre cada UF). |
| `2. Municipio Maior Proporcao` | Dentro da UF acima, o município com maior proporção de óbitos evitáveis. |
| `3. Local Obito Maior Quantidade` | Local de óbito mais frequente no município identificado. |
| `4. Sexo Maior Quantidadade` | Sexo mais frequente no município identificado. |
| `5. Etnia Maior Quantidade` | Raça/cor mais frequente no município identificado. |
| `6. Estabelecimento Maior Quantidade` | Tipo de estabelecimento mais frequente no município identificado. |
| `7. Evitabilidade Maior Quantidade` | Categoria de evitabilidade mais frequente no município identificado. |
| `8. Faixa Etária Maior Quantidade` | Faixa etária mais frequente no município identificado. |
| `9. Atendimento médico Quantidade` | Categoria de assistência médica mais frequente no município identificado. |
| `9.2 Causa Basica Maior Quantidade` | Subcategoria CID-10 mais frequente no município identificado. |

*Todas filtram por `Evitabilidade` (exclui "Não Aplicável", "Não Evitável" e "Mal-definida") e usam o padrão `ADDCOLUMNS` + `TOPN(1, ..., DESC)` + `MAXX` para extrair o valor da categoria de maior contagem.*

> **Descontinuada:** `9.1 Escolariade Maior Quantidade` — dependia da coluna `escolaridade`, removida da tabela de fatos. Numeração mantida como está no modelo para não quebrar referências em visuais existentes.

#### Grupo 3 — Narrativa dinâmica

`0. Narrativa Grupo Prioritario` — combina as medidas dos Grupos 1 e 2 em um texto formatado (Card visual), descrevendo a causa selecionada, sua proporção nacional, e o perfil predominante (sexo, raça/cor, faixa etária, causa básica, local, estabelecimento, assistência médica e motivo de evitabilidade) do município de maior proporção de óbitos evitáveis dentro do estado de maior proporção.

```dax
0. Narrativa Grupo Prioritario = 
VAR CausaAtual = SELECTEDVALUE('CID-10-SUBCATEGORIAS'[descricao], "Todas as causas evitáveis")
VAR UF = [1. UF Maior Proporcao]
VAR Municipio = [2. Municipio Maior Proporcao]
VAR Sexo = [4. Sexo Maior Quantidadade]
VAR RacaCor = [5. Etnia Maior Quantidade]
VAR LocalObito = [3. Local Obito Maior Quantidade]
VAR Estabelecimento = [6. Estabelecimento Maior Quantidade]
VAR Evitabilidade = [7. Evitabilidade Maior Quantidade]
VAR FaixaEtaria = [8. Faixa Etária Maior Quantidade]
VAR AtendimentoContinuado = [9. Atendimento médico Quantidade]
VAR CausaBasica = [9.2 Causa Basica Maior Quantidade]
VAR ValorProporcao = FORMAT( [1. Proporcao evitável (Brasil)], "0.00%" )

VAR ExibirEstabelecimento = 
    IF(
        LocalObito <> "Hospital" && LocalObito <> "Outros estabelecimentos de saúde",
        "Não se aplica (ocorrência fora de unidade de saúde)", 
        Estabelecimento
    )
VAR Quebra = UNICHAR(10)
VAR QuebraDupla = UNICHAR(10) & UNICHAR(10)
RETURN
    "⚠️ Causa a ser analisada: " & UPPER( CausaAtual ) & Quebra & 
    "📊 Proporção dos óbitos: Esta causa representa " & ValorProporcao & " de todos os óbitos registrados entre 2024 e 2025 (evitáveis e não evitáveis). Se você filtrar por um Estado ou município específico, esse percentual é calculado apenas dentro dessa região. Sem filtro, o percentual considera o Brasil inteiro." & QuebraDupla &
    "💡 Observação: O recorte municipal, o perfil da vítima e o cenário do óbito refletem o contexto estadual (visão do macro para o micro). Primeiro identifica-se o Estado de maior proporção; em seguida, dentro desse Estado, o município de maior proporção. A partir da quantidade de óbitos em cada característica (perfil da vítima e cenário do óbito) desse município, os tópicos abaixo refletem sempre a categoria de maior quantidade em cada uma delas." & QuebraDupla &
    "📍 Foco estadual: " & UF & Quebra &
    "📍 Foco municipal: " & Municipio & QuebraDupla &

    "👤 Perfil principal: " &
    "Sexo " & Sexo & " | Raça/Cor: " & RacaCor & " | Faixa etária: " & FaixaEtaria & QuebraDupla &

    "🏥 Cenário do Óbito:" & Quebra &
    "   • Causa básica do óbito: " & CausaBasica & Quebra &
    "   • Local predominante: " & LocalObito & Quebra &
    "   • Estabelecimento: " & ExibirEstabelecimento & Quebra &
    "   • Atendimento médico: " & AtendimentoContinuado & Quebra &
    "   • Motivo da evitabilidade: " & Evitabilidade
```
