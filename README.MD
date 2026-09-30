# Documentação da PokéAPI v2 para o app Flutter

Base de todas as chamadas: `https://pokeapi.co/api/v2/` Documentação oficial: https://pokeapi.co/docs/v2 Arquivos que acompanham este documento:

|Arquivo|Conteúdo|
|---|---|
|`pokeapi_documentacao.md`|Este documento|
|`pokeapi_models.dart`|Todos os modelos Dart (sem dependências)|
|`pokeapi_client.dart`|Cliente HTTP mínimo com cache em memória (usa `package:http`)|

## Índice

1. [Visão geral e regras de uso](#1-vis%C3%A3o-geral-e-regras-de-uso)
2. [Convenções das respostas](#2-conven%C3%A7%C3%B5es-das-respostas)
3. [Funcionalidades da API](#3-funcionalidades-da-api)
4. [Endpoints](#4-endpoints)
5. [Dicionário de dados: significado de cada campo](#5-dicion%C3%A1rio-de-dados)
6. [Estruturas de dados no Flutter](#6-estruturas-de-dados-no-flutter)
7. [Fluxo por tela e arquitetura sugerida](#7-fluxo-por-tela-e-arquitetura-sugerida)
8. [Pontos de atenção e o que não foi verificado](#8-pontos-de-aten%C3%A7%C3%A3o-e-o-que-n%C3%A3o-foi-verificado)
9. [Fontes](#9-fontes)

---

## 1. Visão geral e regras de uso

- **Somente leitura.** Só existe o método `GET`. Não há escrita.
- **Sem autenticação.** Nenhuma chave de API, token ou cadastro.
- **Formato:** JSON.
- **Sem rate limit**, desde a migração para hospedagem estática em novembro de 2018. Mesmo assim, os mantenedores pedem que você limite a frequência de requisições para reduzir o custo de hospedagem.
- **Política de uso justo (obrigatória):**
    - Faça **cache local** de todo recurso que requisitar.
    - Seja cordial com os demais desenvolvedores.
    - Reporte vulnerabilidades de forma responsável (ver `SECURITY.md` no repositório `PokeAPI/pokeapi`).
    - Quem não cumprir pode ter o **IP banido permanentemente**.
- **Versão:** use sempre `/api/v2/`. A v1 foi descontinuada.
- **GraphQL (beta):** existe uma interface GraphQL com o mesmo conjunto de dados em `https://beta.pokeapi.co/graphql/v1beta`, documentada em https://pokeapi.co/docs/graphql. Este documento cobre só o REST.
- **Wrappers oficiais/comunitários** (Node, Python, Kotlin, Swift, Go, Rust etc.) estão listados na documentação oficial. Para Dart/Flutter o site não lista um wrapper; por isso este pacote traz modelos e um cliente próprios.

## 2. Convenções das respostas

### 2.1 Referências entre recursos

Quase toda resposta aponta para outros recursos por URL em vez de embutir os dados. Existem dois formatos:

```json
{ "name": "electric", "url": "https://pokeapi.co/api/v2/type/13/" }   // NamedAPIResource
{ "url": "https://pokeapi.co/api/v2/evolution-chain/10/" }             // APIResource (sem nome)
```

Para seguir a referência, faça um `GET` na `url`. O **ID** é sempre o último número da URL (`idFromUrl()` no `pokeapi_models.dart`).

### 2.2 Paginação

Chamar um endpoint sem ID/nome devolve uma lista paginada:

```json
{ "count": 248, "next": ".../ability/?limit=20&offset=20", "previous": null,
  "results": [ { "name": "stench", "url": ".../ability/1/" } ] }
```

|Campo|Significado|
|---|---|
|`count`|Total de recursos disponíveis naquele endpoint|
|`next` / `previous`|URL da próxima/anterior página, ou `null`|
|`results`|Itens da página (`name` + `url`, ou só `url` nas listas sem nome)|

- Tamanho padrão da página: **20**. Use `?limit=60` para mudar e `?limit=60&offset=60` para avançar.
- **Listas sem nome** (só `url`, sem `name`): `characteristic`, `contest-effect`, `evolution-chain`, `machine`, `super-contest-effect`. As demais são nomeadas.

### 2.3 Idiomas (localização)

- A API **não traduz a resposta** e não tem parâmetro `?lang=`. Os textos traduzidos vêm em listas (`names`, `descriptions`, `effect_entries`, `flavor_text_entries`, `genera`...), e cada item tem um campo `language`.
- O idioma **Português do Brasil** é `language/13`, com `name: "pt-br"`, `iso639: "pt"`, `iso3166: "br"` (verificado em `GET /language/13`). Filtre por `language.name == "pt-br"`.
- **A cobertura de pt-br é parcial.** Sempre use fallback para `en` (`language/9`). O Dart já faz isso: `names.localized()` tenta `pt-br`, depois `en`.
- O campo `name` principal de um recurso (`pikachu`, `thunderbolt`) é um **identificador fixo em inglês** (slug). Nomes exibíveis ficam em `names`.
- Os nomes exibíveis dos Pokémon ficam em **`pokemon-species`**, não em `pokemon`.

### 2.4 Versões dos jogos

Muito dado muda de um jogo para outro (texto, preço, golpes aprendidos, localização). A API organiza isso em três níveis:

- **`version`**: um jogo específico (Red, Blue, Yellow...).
- **`version-group`**: jogos muito parecidos agrupados (Red/Blue).
- **`generation`**: conjunto de jogos que compartilham os mesmos Pokémon novos.

Campos como `version_group_details`, `version_details`, `flavor_text_entries` e `past_values` repetem a informação **uma vez por versão ou grupo de versões**. Para mostrar um valor só, escolha a última entrada (a mais recente) ou a de um jogo específico.

### 2.5 Imagens

- A API não serve imagens diretamente: `sprites` traz URLs de arquivos do repositório `PokeAPI/sprites` no GitHub.
- **Sprite por ID** sem chamar a API: `https://raw.githubusercontent.com/PokeAPI/sprites/master/sprites/pokemon/{id}.png`
- Sprites de itens: `.../sprites/items/{nome}.png` (ex.: `master-ball.png`).
- Sugestão: use `cached_network_image` no Flutter, que também reduz o tráfego.

### 2.6 Marcação dentro de textos

Textos de efeito podem conter marcações internas, por exemplo `[Catches]{mechanic:catch}`, e quebras (`\n`). Em golpes, o texto pode conter o placeholder `$effect_chance`. O `pokeapi_models.dart` traz `cleanEffect()` e `cleanFlavor()`, e `Move.shortEffect()` já substitui o placeholder.

### 2.7 Valores nulos

`null` é comum e significativo: um golpe sem precisão (`accuracy: null`) nunca erra; `evolution_details` pode ter dezenas de campos `null`; `baby_trigger_item` quase sempre é `null`. Todo campo opcional é `?` nos modelos Dart.

---

## 3. Funcionalidades da API

A documentação oficial organiza a API em 12 grupos. Esta é a descrição do que cada um oferece.

|Grupo|O que permite fazer|
|---|---|
|**Berries**|Consultar as frutas (berries) do jogo: tempo de crescimento, colheita máxima, tamanho, firmeza, sabores e o item correspondente. Inclui as firmezas (`berry-firmness`) e os sabores (`berry-flavor`), que determinam se um Pokémon gosta ou não da fruta conforme sua natureza.|
|**Contests**|Dados de concursos Pokémon: tipos de condição (`contest-type`), efeitos de golpes em concursos (`contest-effect`) e em super concursos (`super-contest-effect`), com pontos de "appeal" e "jam".|
|**Currencies**|Moedas usadas nos jogos (ex.: Poké Dollar), com nomes por idioma.|
|**Encounters**|Como e em que condições Pokémon selvagens aparecem: métodos (`encounter-method`, ex.: andar na grama, pescar, surfar), condições (`encounter-condition`, ex.: enxame, hora do dia) e valores dessas condições (`encounter-condition-value`).|
|**Evolution**|Famílias de evolução (`evolution-chain`), gatilhos (`evolution-trigger`, ex.: subir de nível, troca, usar item) e variáveis usadas em evoluções que dependem de dados internos (`evolution-variable`).|
|**Games**|Estrutura dos jogos: gerações (`generation`), Pokédexes regionais (`pokedex`), jogos (`version`) e grupos de jogos (`version-group`).|
|**Items**|Itens do jogo (`item`), seus atributos (`item-attribute`), categorias (`item-category`), efeito do golpe Fling (`item-fling-effect`) e bolsos da mochila (`item-pocket`). Traz preços por jogo, quem segura o item e as TMs/HMs relacionadas.|
|**Locations**|Mundo dos jogos: regiões (`region`), locais (`location`), áreas dentro dos locais com os Pokémon encontráveis (`location-area`) e áreas do Pal Park (`pal-park-area`).|
|**Machines**|Ligação entre TMs/HMs, o golpe que ensinam e o jogo (`machine`). Varia por versão.|
|**Moves**|Golpes (`move`) com poder, precisão, PP, prioridade, tipo, efeito e metadados de batalha, além de ailments (`move-ailment`), estilos do Battle Palace (`move-battle-style`), categorias (`move-category`), classes de dano (`move-damage-class`), métodos de aprendizado (`move-learn-method`) e alvos (`move-target`).|
|**Pokémon**|O núcleo: dados de batalha do Pokémon (`pokemon`), espécie (`pokemon-species`), formas (`pokemon-form`), habilidades (`ability`), tipos (`type`), stats (`stat`), naturezas (`nature`), características (`characteristic`), grupos de ovo (`egg-group`), gênero (`gender`), taxa de crescimento (`growth-rate`), cor, forma corporal e habitat, stats do Pokéathlon, além de `pokemon/{id}/encounters` (onde encontrar).|
|**Utility**|Suporte: idiomas (`language`) e os tipos comuns usados por todos os outros (nomes, descrições, efeitos, textos).|

---

## 4. Endpoints

Todos: `GET https://pokeapi.co/api/v2/{endpoint}/{id ou nome}/`. Lista: `GET .../{endpoint}/?limit=&offset=`.

Colunas: **ID** = aceita só ID numérico (lista sem nome) ou ID/nome. **App** = uso na versão simples do app (L = tela de lista, D = tela de detalhe, E = tela de extras, O = opcional, — = não usado).

### Pokémon

|Endpoint|ID|Retorna|Modelo Dart|App|
|---|---|---|---|---|
|`pokemon`|id/nome|Dados de batalha: stats, tipos, habilidades, golpes, sprites|`Pokemon`|L, D|
|`pokemon/{id}/encounters`|id/nome|**Lista** de áreas onde o Pokémon aparece|`List<LocationAreaEncounter>`|O|
|`pokemon-species`|id/nome|Nome traduzido, descrição, categoria, captura, gênero, evolução|`PokemonSpecies`|D|
|`pokemon-form`|id/nome|Formas alternativas (mega, regionais etc.)|`PokemonForm`|O|
|`ability`|id/nome|Habilidade: efeito e quem a tem|`Ability`|E|
|`type`|id/nome|Relações de dano entre tipos|`PokeType`|E|
|`stat`|id/nome|Um stat (HP, ataque...) e o que o afeta|`Stat`|O (nomes pt-br dos stats)|
|`nature`|id/nome|Natureza: stat que sobe/desce|`Nature`|—|
|`characteristic`|**só id**|Característica ligada aos IVs|`Characteristic`|—|
|`egg-group`|id/nome|Grupo de ovo e espécies|`EggGroup`|O|
|`gender`|id/nome|Espécies por gênero e taxa|`Gender`|—|
|`growth-rate`|id/nome|Curva de XP por nível|`GrowthRate`|O|
|`pokemon-color`|id/nome|Cor da espécie|`PokemonColor`|O|
|`pokemon-shape`|id/nome|Forma corporal|`PokemonShape`|O|
|`pokemon-habitat`|id/nome|Habitat|`PokemonHabitat`|O|
|`pokeathlon-stat`|id/nome|Stat do Pokéathlon|`PokeathlonStat`|—|

### Evolução

|Endpoint|ID|Retorna|Modelo Dart|App|
|---|---|---|---|---|
|`evolution-chain`|**só id**|Árvore de evolução da família|`EvolutionChain`|D|
|`evolution-trigger`|id/nome|Evento que dispara evolução|`EvolutionTrigger`|—|
|`evolution-variable`|id/nome|Variável usada em expressões de evolução|`EvolutionVariable`|—|

### Golpes e máquinas

|Endpoint|ID|Retorna|Modelo Dart|App|
|---|---|---|---|---|
|`move`|id/nome|Golpe completo|`Move`|E|
|`move-ailment`|id/nome|Condição de status causada|`MoveAilment`|—|
|`move-battle-style`|id/nome|Estilo do Battle Palace|`MoveBattleStyle`|—|
|`move-category`|id/nome|Categoria geral do efeito|`MoveCategory`|—|
|`move-damage-class`|id/nome|physical / special / status|`MoveDamageClass`|O|
|`move-learn-method`|id/nome|Como se aprende (nível, TM, ovo...)|`MoveLearnMethod`|O|
|`move-target`|id/nome|Alvo do golpe|`MoveTarget`|—|
|`machine`|**só id**|TM/HM ↔ golpe ↔ jogo|`Machine`|—|

### Jogos

|Endpoint|ID|Retorna|Modelo Dart|App|
|---|---|---|---|---|
|`generation`|id/nome|Novidades de uma geração|`Generation`|O (filtro)|
|`pokedex`|id/nome|Pokédex regional|`Pokedex`|O|
|`version`|id/nome|Jogo específico|`Version`|—|
|`version-group`|id/nome|Grupo de jogos|`VersionGroup`|—|

### Itens

|Endpoint|ID|Retorna|Modelo Dart|App|
|---|---|---|---|---|
|`item`|id/nome|Item completo|`Item`|—|
|`item-attribute`|id/nome|Atributo (ex.: consumível)|`ItemAttribute`|—|
|`item-category`|id/nome|Categoria e bolso|`ItemCategory`|—|
|`item-fling-effect`|id/nome|Efeito do Fling|`ItemFlingEffect`|—|
|`item-pocket`|id/nome|Bolso da mochila|`ItemPocket`|—|
|`item-price`|id/nome|Preços (aparece na documentação oficial; não consta no índice `/api/v2/`)|`ItemPrice`|—|
|`currency`|id/nome|Moeda|`Currency`|—|

### Locais e encontros

|Endpoint|ID|Retorna|Modelo Dart|App|
|---|---|---|---|---|
|`region`|id/nome|Região|`Region`|—|
|`location`|id/nome|Local (cidade, rota)|`Location`|—|
|`location-area`|id/nome|Área com Pokémon encontráveis|`LocationArea`|O|
|`pal-park-area`|id/nome|Área do Pal Park|`PalParkArea`|—|
|`encounter-method`|id/nome|Método de encontro|`EncounterMethod`|O|
|`encounter-condition`|id/nome|Condição de encontro|`EncounterCondition`|—|
|`encounter-condition-value`|id/nome|Valor da condição|`EncounterConditionValue`|—|

### Berries e concursos

|Endpoint|ID|Retorna|Modelo Dart|App|
|---|---|---|---|---|
|`berry`|id/nome|Fruta|`Berry`|—|
|`berry-firmness`|id/nome|Firmeza|`BerryFirmness`|—|
|`berry-flavor`|id/nome|Sabor|`BerryFlavor`|—|
|`contest-type`|id/nome|Tipo de concurso|`ContestType`|—|
|`contest-effect`|**só id**|Efeito em concurso|`ContestEffect`|—|
|`super-contest-effect`|**só id**|Efeito em super concurso|`SuperContestEffect`|—|

### Utilitários

|Endpoint|ID|Retorna|Modelo Dart|App|
|---|---|---|---|---|
|`language`|id/nome|Idioma (`pt-br` = 13, `en` = 9)|`Language`|O|
|`meta`|—|Consta no índice da API, mas **não está descrito na documentação oficial**; sem modelo|—|—|

---

## 5. Dicionário de dados

Convenções: tipos JSON (`int`, `string`, `bool`, `list`, `obj`); `→ X` indica referência (`NamedAPIResource`/`APIResource`) para o endpoint X; `?` = pode ser `null`.

### 5.1 Tipos comuns (aparecem em todo lugar)

|Tipo|Campos|Significado|
|---|---|---|
|`NamedAPIResource`|`name`, `url`|Referência para outro recurso, com o slug|
|`APIResource`|`url`|Referência sem nome|
|`Name`|`name`, `language`|Nome do recurso em um idioma|
|`Description`|`description`, `language`|Descrição em um idioma|
|`Effect`|`effect`, `language`|Texto do efeito em um idioma|
|`VerboseEffect`|`effect`, `short_effect`, `language`|Efeito completo e resumido|
|`FlavorText`|`flavor_text` (ou `text` em itens), `language`, `version` ou `version_group`|Texto "de Pokédex" por idioma e por jogo|
|`GenerationGameIndex`|`game_index`, `generation`|Índice interno do recurso nos dados do jogo, por geração|
|`VersionGameIndex`|`game_index`, `version`|Idem, por jogo|
|`MachineVersionDetail`|`machine` → machine, `version_group`|TM/HM que ensina o golpe em cada grupo de jogos|
|`Encounter`|`min_level`, `max_level`, `condition_values`, `chance`, `method`|Faixa de nível, condições, chance (em %) e método de um encontro|
|`VersionEncounterDetail`|`version`, `max_chance`, `encounter_details`|Encontros por jogo; `max_chance` é a chance máxima total naquele jogo|

### 5.2 `language`

|Campo|Significado|
|---|---|
|`id`|ID do idioma (9 = en, 13 = pt-br)|
|`name`|Código usado nos filtros (`pt-br`, `en`, `ja`...)|
|`official`|Se o idioma é oficial nos jogos|
|`iso639` / `iso3166`|Código ISO de idioma e de país|
|`names`|Nome do idioma escrito em vários idiomas|

### 5.3 `pokemon`

|Campo|Tipo|Significado|
|---|---|---|
|`id`, `name`|int, string|Identificador e slug|
|`base_experience`|int|XP base ganho ao derrotar este Pokémon|
|`height`|int|Altura em **decímetros** (÷10 = metros)|
|`weight`|int|Peso em **hectogramas** (÷10 = kg)|
|`is_default`|bool|Se é a forma padrão da espécie|
|`order`|int|Ordem de exibição/ordenação|
|`abilities[]`|list|Habilidades: `is_hidden` (habilidade oculta), `slot` (posição), `ability` → ability|
|`forms[]`|list → pokemon-form|Formas deste Pokémon|
|`game_indices[]`|list|Índice interno por jogo (`game_index`, `version`)|
|`held_items[]`|list|Itens que pode segurar no mato: `item` → item e `version_details[]` com `rarity` (chance) e `version`|
|`location_area_encounters`|string|URL para `/pokemon/{id}/encounters`|
|`moves[]`|list|Golpes: `move` → move e `version_group_details[]` com `level_learned_at` (0 se não for por nível), `move_learn_method`, `version_group`|
|`species`|→ pokemon-species|Espécie (nome traduzido, evolução, descrição)|
|`sprites`|obj|URLs de imagens (ver 5.3.1)|
|`cries`|obj|URLs dos gritos (`latest`, `legacy`); _não verificado na doc_|
|`stats[]`|list|`base_stat` (valor base), `effort` (pontos de esforço/EV dados ao derrotar), `stat` → stat|
|`types[]`|list|`slot` (1 = primário, 2 = secundário), `type` → type|
|`past_types[]`, `past_abilities[]`|list|Tipos/habilidades que o Pokémon tinha em gerações anteriores; _não verificado na doc_|

Os **stats** são sempre 6: `hp`, `attack`, `defense`, `special-attack`, `special-defense`, `speed`.

#### 5.3.1 `sprites`

|Campo|Significado|
|---|---|
|`front_default`, `back_default`|Imagem de frente / de costas|
|`front_shiny`, `back_shiny`|Versão shiny (cor alternativa)|
|`front_female`, `back_female`, `front_shiny_female`, `back_shiny_female`|Variações femininas (ficam `null` quando não há diferença)|
|`other['official-artwork'].front_default`|Arte oficial em alta resolução; _não verificado na doc, confirme numa resposta real_|
|`other['home'].front_default`|Imagem do Pokémon HOME; _idem_|

### 5.4 `pokemon-species`

|Campo|Significado|
|---|---|
|`id`, `name`, `order`|Identificador, slug e ordem|
|`names[]`|**Nome exibível por idioma**|
|`flavor_text_entries[]`|Descrição da Pokédex, uma por jogo e idioma|
|`genera[]`|Categoria, ex.: "Seed Pokémon" (`genus` + `language`)|
|`form_descriptions[]`|Descrição das formas|
|`gender_rate`|Chance de ser fêmea em **oitavos** (0 a 8); `-1` = sem gênero|
|`capture_rate`|0 a 255; quanto maior, mais fácil capturar|
|`base_happiness`|Felicidade inicial (0 a 255)|
|`is_baby`, `is_legendary`, `is_mythical`|Classificações especiais|
|`hatch_counter`|Ciclos necessários para o ovo chocar|
|`has_gender_differences`|Se macho e fêmea têm aparência diferente|
|`forms_switchable`|Se pode trocar de forma durante a batalha/jogo|
|`growth_rate`|→ growth-rate|
|`pokedex_numbers[]`|Número em cada Pokédex (`entry_number`, `pokedex`)|
|`egg_groups[]`|→ egg-group (**lista**)|
|`color`, `shape`, `habitat`|→ pokemon-color, pokemon-shape, pokemon-habitat|
|`evolves_from_species`|Espécie anterior na evolução; `null` para a forma base|
|`evolution_chain`|`{url}` → evolution-chain (**é por aqui que se chega à árvore**)|
|`generation`|Geração em que surgiu|
|`pal_park_encounters[]`|Dados do Pal Park (`base_score`, `rate`, `area`)|
|`varieties[]`|Variedades (`is_default`, `pokemon` → pokemon)|

### 5.5 `evolution-chain`

|Campo|Significado|
|---|---|
|`id`|ID da cadeia|
|`baby_trigger_item`|Item que o pai precisa segurar para o ovo vir como bebê; quase sempre `null`|
|`chain`|Nó raiz (`ChainLink`)|

`ChainLink` (recursivo):

|Campo|Significado|
|---|---|
|`is_baby`|Se este nó é um Pokémon bebê (só na raiz)|
|`species`|→ pokemon-species neste ponto da cadeia|
|`evolution_details[]`|**Como evoluir até esta espécie** (pode ter mais de um item, ex.: por jogo/forma)|
|`evolves_to[]`|Lista de nós filhos (mesmo formato)|

`EvolutionDetail` (quase tudo é `null` na prática; só os campos relevantes vêm preenchidos):

|Campo|Significado|
|---|---|
|`trigger`|→ evolution-trigger (level-up, trade, use-item...)|
|`min_level`, `min_happiness`, `min_beauty`, `min_affection`|Nível, felicidade, beleza ou afeto mínimos|
|`item`|Item a usar (ex.: pedra)|
|`held_item`|Item que deve estar segurando|
|`known_move`, `known_move_type`, `used_move`, `min_move_count`|Golpe que deve conhecer / tipo de golpe / golpe usado / quantas vezes usar|
|`location`, `region`|Local ou região exigidos|
|`time_of_day`|`day`, `night`, `dusk`, `full-moon` ou `""`|
|`gender`|ID do gênero exigido|
|`party_species`, `party_type`|Espécie ou tipo exigidos no time|
|`trade_species`|Espécie com a qual precisa trocar|
|`relative_physical_stats`|1 = ataque > defesa, 0 = iguais, -1 = ataque < defesa|
|`needs_multiplayer`, `needs_overworld_rain`, `turn_upside_down`, `near_special_rock`|Condições especiais (link, chuva, virar o 3DS, perto de rocha especial)|
|`min_steps`, `min_damage_taken`|Passos ou dano sofrido mínimos|
|`allowed_natures[]`|Naturezas permitidas|
|`required_pokemon_form`, `evolved_pokemon_form`|Forma exigida / forma resultante|
|`version_group`|Grupo de jogos em que a evolução surgiu|
|`is_default`|Se é a evolução esperada nos jogos principais|
|`condition_expression`|Expressão para evoluções que dependem de dados internos (`expression`, `percentage_chance`, `variables[]`)|

### 5.6 `evolution-trigger` e `evolution-variable`

|Endpoint|Campos|Significado|
|---|---|---|
|`evolution-trigger`|`id`, `name`, `names[]`, `pokemon_species[]`|Evento (level-up, trade...) e espécies que resultam dele|
|`evolution-variable`|`id`, `name`, `symbol`, `data_type`, `source` (`pokemon` ou `player-input`), `version_group`, `names[]`, `descriptions[]`|Parâmetro usado em expressões de evolução (ex.: Encryption Constant, símbolo `EC`)|

### 5.7 `type`

|Campo|Significado|
|---|---|
|`id`, `name`, `names[]`|Identificação e nomes traduzidos|
|`damage_relations`|Relações de dano (abaixo)|
|`pokemon[]`|Pokémon deste tipo (`slot`, `pokemon`)|
|`moves[]`|Golpes deste tipo|
|`generation`|Geração em que o tipo surgiu|
|`move_damage_class`|Classe de dano padrão dos golpes deste tipo (só em gerações antigas)|
|`game_indices[]`|Índices internos por geração|

`damage_relations` (sempre do ponto de vista **deste** tipo):

|Campo|Significado|Multiplicador|
|---|---|---|
|`no_damage_to`|Tipos que **não** sofrem dano de golpes deste tipo|0×|
|`half_damage_to`|Tipos que sofrem metade|0,5×|
|`double_damage_to`|Tipos que sofrem o dobro|2×|
|`no_damage_from`|Tipos cujos golpes **não** ferem este tipo|0×|
|`half_damage_from`|Tipos cujos golpes causam metade|0,5×|
|`double_damage_from`|Tipos cujos golpes causam o dobro (= fraquezas)|2×|

Para Pokémon com dois tipos, **multiplique** os fatores dos dois tipos. O helper `damageTakenMultipliers(List<PokeType>)` faz isso.

### 5.8 `move`

|Campo|Significado|
|---|---|
|`id`, `name`, `names[]`|Identificação e nomes traduzidos|
|`power`|Poder base; `null`/0 se não causa dano direto|
|`accuracy`|% de acerto; `null` = nunca erra|
|`pp`|Quantas vezes pode ser usado|
|`priority`|-8 a 8; define a ordem de execução na batalha|
|`type`|→ type do golpe|
|`damage_class`|→ move-damage-class: `physical`, `special` ou `status`|
|`effect_chance`|% de chance do efeito secundário ocorrer|
|`effect_entries[]`|Efeito (`effect` e `short_effect`) por idioma|
|`effect_changes[]`|Efeitos anteriores do golpe, por grupo de jogos|
|`flavor_text_entries[]`|Descrição por idioma e jogo|
|`past_values[]`|Valores antigos (poder, precisão, PP, tipo) por grupo de jogos|
|`stat_changes[]`|Stats afetados (`stat`, `change`)|
|`target`|→ move-target|
|`generation`|Geração de origem|
|`machines[]`|TMs/HMs que ensinam o golpe|
|`learned_by_pokemon[]`|Pokémon que aprendem o golpe|
|`contest_type`, `contest_effect`, `super_contest_effect`, `contest_combos`|Dados de concursos|
|`meta`|Metadados de batalha (abaixo)|

`meta`:

|Campo|Significado|
|---|---|
|`ailment`|→ move-ailment causado (`none` se nenhum)|
|`category`|→ move-category (ex.: `damage`, `ailment`)|
|`min_hits`, `max_hits`|Número de acertos; `null` = sempre 1|
|`min_turns`, `max_turns`|Duração em turnos; `null` = 1 turno|
|`drain`|% do dano que vira cura (positivo) ou recuo (negativo)|
|`healing`|% do HP máximo recuperado|
|`crit_rate`|Bônus na taxa de crítico|
|`ailment_chance`, `flinch_chance`, `stat_chance`|Chance (%) de causar status, recuar o alvo ou mudar stats|

### 5.9 Tabelas de apoio de golpes e máquinas

|Endpoint|Campos|Significado|
|---|---|---|
|`move-ailment`|`id`, `name`, `moves[]`, `names[]`|Condição de status (paralisia etc.) e golpes que a causam|
|`move-battle-style`|`id`, `name`, `names[]`|Estilo no Battle Palace (attack, defense, support)|
|`move-category`|`id`, `name`, `moves[]`, `descriptions[]`|Grupo amplo de efeitos|
|`move-damage-class`|`id`, `name`, `descriptions[]`, `moves[]`, `names[]`|physical/special/status|
|`move-learn-method`|`id`, `name`, `names[]`, `descriptions[]`, `version_groups[]`|Como aprende (nível, TM, ovo, tutor...)|
|`move-target`|`id`, `name`, `descriptions[]`, `moves[]`, `names[]`|Quem recebe o efeito do golpe|
|`machine`|`id`, `item`, `move`, `version_group`|A TM/HM (item) ensina o golpe em certo grupo de jogos; varia por versão|

### 5.10 `ability`

|Campo|Significado|
|---|---|
|`id`, `name`, `names[]`|Identificação e nomes traduzidos|
|`is_main_series`|Se existe nos jogos principais|
|`generation`|Geração de origem|
|`effect_entries[]`|Efeito (`effect`, `short_effect`) por idioma|
|`effect_changes[]`|Efeitos anteriores por grupo de jogos|
|`flavor_text_entries[]`|Descrição curta por idioma e jogo|
|`pokemon[]`|Quem tem a habilidade: `is_hidden`, `slot`, `pokemon`|

### 5.11 `stat`, `nature`, `characteristic`, `pokeathlon-stat`

|Endpoint|Campo|Significado|
|---|---|---|
|`stat`|`game_index`|Índice interno no jogo|
||`is_battle_only`|`true` para stats só de batalha (precisão, evasão)|
||`affecting_moves`|Golpes que aumentam/diminuem o stat (`increase[]`/`decrease[]` com `change` e `move`)|
||`affecting_natures`|Naturezas que aumentam/diminuem (`increase[]`/`decrease[]`)|
||`characteristics[]`|Características ligadas ao stat|
||`move_damage_class`|Classe de dano associada (ataque → physical)|
||`names[]`|**Nomes traduzidos do stat** (útil para rotular as barras)|
|`nature`|`increased_stat`, `decreased_stat`|Stat que sobe / desce (naturezas neutras: `null`). Nos jogos principais o efeito é de ±10%|
||`likes_flavor`, `hates_flavor`|Sabor de fruta que agrada / desagrada|
||`pokeathlon_stat_changes[]`|Mudanças nos stats do Pokéathlon (`max_change`, `pokeathlon_stat`)|
||`move_battle_style_preferences[]`|Preferência de estilo no Battle Palace (`low_hp_preference`, `high_hp_preference`, `move_battle_style`)|
|`characteristic`|`id`|(lista sem nome, só ID)|
||`gene_modulo`|Resto da divisão do "gene" do Pokémon|
||`possible_values[]`|Valores de IV que levam a essa característica|
||`highest_stat`, `descriptions[]`|Stat mais alto e texto; _não verificado na doc_|
|`pokeathlon-stat`|`affecting_natures`|Naturezas que aumentam/diminuem o stat (`max_change`, `nature`)|

### 5.12 Espécie: grupos de apoio

|Endpoint|Campos|Significado|
|---|---|---|
|`egg-group`|`id`, `name`, `names[]`, `pokemon_species[]`|Grupo de ovo e espécies que o compõem|
|`gender`|`pokemon_species_details[]` (`rate`, `pokemon_species`), `required_for_evolution[]`|Espécies e taxa de ocorrência; espécies que precisam deste gênero para evoluir|
|`growth-rate`|`formula`, `descriptions[]`, `levels[]` (`level`, `experience`), `pokemon_species[]`|Fórmula da curva e XP necessário por nível|
|`pokemon-color`|`id`, `name`, `names[]`, `pokemon_species[]`|Cor usada na Pokédex|
|`pokemon-shape`|`id`, `name`, `awesome_names[]`, `names[]`, `pokemon_species[]`|Forma corporal; `awesome_names` são os nomes "estilosos"|
|`pokemon-habitat`|`id`, `name`, `names[]`, `pokemon_species[]`|Habitat|
|`pokemon-form`|`id`, `name`, `order`, `form_order`, `is_default`, `is_battle_only`, `is_mega`, `form_name`, `pokemon`, `sprites`, `version_group`, `names[]`, `form_names[]`|Forma alternativa (mega, regional, etc.) com sprites próprios|

### 5.13 Jogos

|Endpoint|Campos|Significado|
|---|---|---|
|`generation`|`id`, `name`, `names[]`, `main_region`, `abilities[]`, `moves[]`, `pokemon_species[]`, `types[]`, `version_groups[]`|Tudo que foi introduzido naquela geração|
|`pokedex`|`id`, `name`, `is_main_series`, `descriptions[]`, `names[]`, `pokemon_entries[]` (`entry_number`, `pokemon_species`), `region`, `version_groups[]`|Pokédex regional ou nacional (`region` é `null` na nacional)|
|`version`|`id`, `name`, `names[]`, `version_group`|Jogo específico|
|`version-group`|`id`, `name`, `order`, `generation`, `move_learn_methods[]`, `pokedexes[]`, `regions[]`, `versions[]`|Jogos muito parecidos; `order` ordena quase por data de lançamento|

### 5.14 Itens e moedas

|Endpoint|Campos|Significado|
|---|---|---|
|`item`|`id`, `name`, `names[]`|Identificação e nomes|
||`prices[]`|Preço por grupo de jogos: `currency`, `purchase_price` (`null` se não compra), `sell_price` (`null` se não vende), `version_group`|
||`fling_power`, `fling_effect`|Poder e efeito do golpe Fling|
||`attributes[]`, `category`|Atributos e categoria|
||`effect_entries[]`, `flavor_text_entries[]`|Efeito e descrição por idioma|
||`game_indices[]`|Índices por geração|
||`sprites.default`|URL da imagem|
||`held_by_pokemon[]`|Quem pode segurar no mato (`pokemon`, `version_details[]` com `rarity` e `version`)|
||`baby_trigger_for`|Cadeia de evolução que exige este item para gerar bebê|
||`machines[]`|TMs/HMs relacionadas|
|`item-attribute`|`items[]`, `names[]`, `descriptions[]`|Atributo (ex.: consumível, usável em batalha)|
|`item-category`|`items[]`, `names[]`, `pocket`|Categoria e bolso onde o item entra|
|`item-fling-effect`|`effect_entries[]`, `items[]`|Efeito do Fling e itens que o têm|
|`item-pocket`|`categories[]`, `names[]`|Bolso da mochila|
|`currency`|`id`, `name`, `names[]`|Moeda (ex.: `poke-dollar`)|

### 5.15 Locais e encontros

|Endpoint|Campos|Significado|
|---|---|---|
|`region`|`id`, `name`, `names[]`, `locations[]`, `main_generation`, `pokedexes[]`, `version_groups[]`|Região do mundo|
|`location`|`id`, `name`, `names[]`, `region`, `game_indices[]`, `areas[]`|Cidade, rota etc.|
|`location-area`|`id`, `name`, `game_index`, `location`, `names[]`|Seção de um local (andar de cave, por exemplo)|
||`encounter_method_rates[]`|Métodos de encontro e a chance (`rate`) por jogo|
||`pokemon_encounters[]`|Pokémon encontráveis (`pokemon`, `version_details[]` → `Encounter`)|
|`pal-park-area`|`id`, `name`, `names[]`, `pokemon_encounters[]` (`base_score`, `rate`, `pokemon_species`)|Área do Pal Park|
|`encounter-method`|`id`, `name`, `order`, `names[]`|Método (andar, pescar, surfar); `order` serve para ordenar|
|`encounter-condition`|`id`, `name`, `names[]`, `values[]`|Condição que muda quem aparece (enxame, hora)|
|`encounter-condition-value`|`id`, `name`, `condition`, `names[]`|Estado da condição (`swarm-yes`/`swarm-no`)|

### 5.16 Berries e concursos

|Endpoint|Campos|Significado|
|---|---|---|
|`berry`|`growth_time` (horas por estágio; são 4), `max_harvest`, `natural_gift_power`, `natural_gift_type`, `size` (mm), `smoothness`, `soil_dryness` (maior = seca o solo mais rápido), `firmness`, `flavors[]` (`flavor`, `potency`), `item`|Dados da fruta; o item associado está em `item`|
|`berry-firmness`|`berries[]`, `names[]`|Firmeza (usada em Pokéblocks/Poffins)|
|`berry-flavor`|`berries[]` (`berry`, `potency`), `contest_type`, `names[]`|Sabor; agrada conforme a natureza|
|`contest-type`|`berry_flavor`, `names[]` (`name`, `color`, `language`)|Categoria avaliada nos concursos|
|`contest-effect`|`appeal`, `jam`, `effect_entries[]`, `flavor_text_entries[]`|Corações ganhos pelo usuário (`appeal`) e perdidos pelo oponente (`jam`)|
|`super-contest-effect`|`appeal`, `flavor_text_entries[]`, `moves[]`|Efeito em super concursos|

---

## 6. Estruturas de dados no Flutter

### 6.1 Mapa endpoint → classe Dart

| Endpoint                  | Classe raiz                                                            | Principais classes aninhadas                                                                                                             |
| ------------------------- | ---------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `pokemon`                 | `Pokemon`                                                              | `PokemonAbility`, `PokemonType`, `PokemonStat`, `PokemonMove`, `PokemonMoveVersion`, `PokemonHeldItem`, `PokemonSprites`, `PokemonCries` |
| `pokemon/{id}/encounters` | `List<LocationAreaEncounter>`                                          | `VersionEncounterDetail`, `Encounter`                                                                                                    |
| `pokemon-species`         | `PokemonSpecies`                                                       | `Genus`, `PokemonSpeciesDexEntry`, `PalParkEncounterArea`, `PokemonSpeciesVariety`                                                       |
| `evolution-chain`         | `EvolutionChain`                                                       | `ChainLink`, `EvolutionDetail`, `EvolutionConditionExpression`                                                                           |
| `type`                    | `PokeType`                                                             | `TypeRelations`, `TypePokemon`                                                                                                           |
| `move`                    | `Move`                                                                 | `MoveMetaData`, `MoveStatChange`, `PastMoveStatValues`, `ContestComboSets`, `ContestComboDetail`                                         |
| `ability`                 | `Ability`                                                              | `AbilityEffectChange`, `AbilityPokemon`                                                                                                  |
| `stat`                    | `Stat`                                                                 | `MoveStatAffectSets`, `MoveStatAffect`, `NatureStatAffectSets`                                                                           |
| `nature`                  | `Nature`                                                               | `NatureStatChange`, `MoveBattleStylePreference`                                                                                          |
| demais endpoints          | `Berry`, `Item`, `Generation`, `Region`, `Location`, `LocationArea`... | ver o nome do endpoint na tabela da seção 4                                                                                              |

O nome `PokeType` evita conflito com o `Type` do `dart:core`.

### 6.3 Tipos auxiliares

|Classe / função|Para que serve|
|---|---|
|`NamedApiResource`, `ApiResource`|Referências; têm o getter `id` (extraído da URL)|
|`ResourceList<T>`|Lista paginada (`count`, `next`, `previous`, `results`, `hasNext`)|
|`Name`, `Description`, `Effect`, `VerboseEffect`, `FlavorText`, `Genus`|Textos por idioma; todos implementam `Localized`|
|`list.localized()`|Primeiro item em pt-br, senão en, senão `null`|
|`list.localizedLast()`|Último item em pt-br, senão en (ideal para `flavor_text_entries`)|
|`FlavorText.clean`, `cleanFlavor()`, `cleanEffect()`|Limpam quebras de linha e marcações|
|`idFromUrl(url)`|Extrai o ID do final da URL|
|`pokemonSpriteUrl(id)`|Monta a URL do sprite por ID|
|`damageTakenMultipliers(types)`|Fraquezas/resistências para 1–2 tipos|

### 6.4 Atalhos prontos nos modelos

|Onde|Membro|Retorna|
|---|---|---|
|`Pokemon`|`heightMeters`, `weightKg`|Altura em m, peso em kg|
|`Pokemon`|`baseStat('speed')`, `baseStatTotal`|Stat base e soma dos stats|
|`Pokemon`|`imageUrl`|Arte oficial → sprite → URL por ID|
|`PokemonSpecies`|`displayName()`, `description()`, `genus()`|Nome, descrição e categoria traduzidos|
|`PokemonSpecies`|`femalePercent`, `malePercent`|Proporção de gênero (`null` se sem gênero)|
|`PokeType` / `Ability` / `Move`|`displayName()`|Nome traduzido|
|`Ability` / `Move`|`shortEffect()`|Efeito curto traduzido e limpo|
|`ChainLink`|`flatten()`|Lista plana da árvore de evolução|
|`EvolutionDetail`|`summary`|Texto curto da condição ("Nível 16", "Item: fire-stone")|

### 6.5 Exemplos de uso

**Tela de detalhe (nome, descrição, stats, tipos):**

```dart
final client = PokeApiClient();
final d = await client.pokemonDetail(25); // pokemon + species + evolution chain

final nome = d.species.displayName();          // "Pikachu"
final texto = d.species.description();         // pt-br ou en
final altura = d.pokemon.heightMeters;         // 0.4
final hp = d.pokemon.baseStat('hp');
final tipos = d.pokemon.types.map((t) => t.type.name).toList();
```

**Fraquezas (chamadas aos tipos com cache):**

```dart
final types = await Future.wait(
  d.pokemon.types.map((t) => client.type(t.type.name)),
);
final mult = damageTakenMultipliers(types);
final fraquezas = mult.entries.where((e) => e.value >= 2).toList();
final imunidades = mult.entries.where((e) => e.value == 0).toList();
```

**Árvore de evolução:**

```dart
void walk(ChainLink node, [int depth = 0]) {
  print('${'  ' * depth}${node.species.name}');
  for (final next in node.evolvesTo) {
    print('${'  ' * (depth + 1)}(${next.evolutionDetails.first.summary})');
    walk(next, depth + 1);
  }
}
walk(d.evolution!.chain);
```

**Lista paginada com ID e sprite (sem chamadas extras por item):**

```dart
final page = await client.listPokemon(limit: 30, offset: 0);
for (final r in page.results) {
  final id = r.id;                  // extraído da URL
  final img = pokemonSpriteUrl(id); // sprite pelo repositório
}
// próxima página: page.hasNext -> listPokemon(offset: 30)
```

---

## 7. Fluxo por tela

![[Pasted image 20260930170014.png]]

---

