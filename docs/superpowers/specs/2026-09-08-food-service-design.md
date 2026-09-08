# Food Service — Especificação de Design

## Escopo

Microsserviço de planejamento de refeições e lista de compras para a plataforma LifeOS.
Responsável pelo banco de pratos/receitas do usuário, planejamento semanal de cardápio,
geração automática de lista de compras categorizada por seção do supermercado e informação
nutricional básica opcional.

- **Identidade:** `groupId=dev.iury.lifeos`, `artifactId=food-service`, pacote raiz
  `dev.iury.lifeos.food`, porta HTTP `8083`.
- **Stack:** Java 21, Quarkus 3.37.3, Hibernate ORM with Panache, PostgreSQL, Flyway,
  SmallRye Reactive Messaging Kafka, Quarkus REST Jackson, Hibernate Validator.
- **Banco isolado:** `food_db`; migrations via Flyway são a única forma de alterar o schema.

## Filosofia

Inspirado em Mealime (zero-waste, bucket planning, checklist efêmera de despensa), Paprika
(receitas como acervo pessoal simples) e AnyList (queue/staging + categorização de
supermercado). O objetivo é **reduzir a carga mental da pergunta "o que vamos comer?"** com o
menor atrito possível.

**Princípio Anti-Overengineering:**

- NÃO manter inventário persistente de despensa (abandono em ~4 dias)
- NÃO exigir pesagem em gramas (unidades caseiras: xícara, colher, unidade)
- NÃO forçar calendário rígido de refeições (espiral de culpa)
- NÃO rastrear micronutrientes (ferro, potássio, vitaminas)
- NÃO ter modo de receita com passos detalhados no MVP
- NÃO incluir hidratação — domínio separado (habit-service ou integração com health apps)

## Modelo de Domínio

### Entidades

#### Ingredient (Catálogo Canônico)

Ingrediente padronizado do sistema ou personalizado do usuário. O seed conterá ~200
ingredientes mais comuns da culinária brasileira para evitar digitação repetida.

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | UUID | PK, auto-generated |
| user_id | UUID | nullable (null = ingrediente do sistema) |
| name | VARCHAR(150) | NOT NULL |
| category | VARCHAR(30) | NOT NULL |
| default_unit | VARCHAR(20) | NOT NULL, default UNIT |

`UNIQUE(user_id, name)` — permite coexistir "Arroz" (sistema) e "Arroz integral" (usuário).

#### Dish (Prato / Receita Favorita)

Prato do acervo pessoal do usuário. Modelo intencionalmente simples — o menor prato válido
é `{ name: "Miojo" }`.

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | UUID | PK, auto-generated |
| user_id | UUID | NOT NULL |
| name | VARCHAR(200) | NOT NULL |
| servings | INTEGER | NOT NULL, default 2 |
| prep_time_minutes | INTEGER | nullable |
| cook_time_minutes | INTEGER | nullable |
| notes | TEXT | nullable (instruções livres, dicas) |
| is_favorite | BOOLEAN | NOT NULL, default false |
| deleted_at | TIMESTAMP | nullable, soft-delete |
| created_at | TIMESTAMP | NOT NULL, UTC |
| updated_at | TIMESTAMP | NOT NULL, UTC |

#### DishTag

Etiqueta para categorização de pratos (Rápido, Vegetariano, Low-carb, Comfort Food, etc.).

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | UUID | PK, auto-generated |
| user_id | UUID | NOT NULL |
| name | VARCHAR(50) | NOT NULL |

`UNIQUE(user_id, name)`.

#### DishDishTag (Join Table)

| Campo | Tipo | Restrições |
|-------|------|------------|
| dish_id | UUID | FK → Dish, PK |
| tag_id | UUID | FK → DishTag, PK |

#### DishIngredient

Ingrediente de um prato com quantidade e unidade.

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | UUID | PK, auto-generated |
| dish_id | UUID | FK → Dish, NOT NULL |
| ingredient_id | UUID | FK → Ingredient, NOT NULL |
| quantity | NUMERIC(10,2) | NOT NULL |
| unit | VARCHAR(20) | NOT NULL |
| preparation_note | VARCHAR(100) | nullable (ex: "picado", "em cubos") |
| sort_order | INTEGER | NOT NULL, default 0 |

#### NutritionInfo (Opcional — Nice-to-Have)

Informação nutricional básica por porção do prato. Todos os campos nutricionais são anuláveis
— o usuário pode informar só calorias, ou só proteína, ou nada.

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | UUID | PK, auto-generated |
| dish_id | UUID | FK → Dish, UNIQUE, NOT NULL |
| calories_per_serving | INTEGER | nullable |
| protein_grams | NUMERIC(6,1) | nullable |
| carbs_grams | NUMERIC(6,1) | nullable |
| fat_grams | NUMERIC(6,1) | nullable |

#### MealPlan (Plano Semanal)

Cabeçalho do plano de refeições da semana.

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | UUID | PK, auto-generated |
| user_id | UUID | NOT NULL |
| week_start_date | DATE | NOT NULL (deve ser segunda-feira) |
| notes | TEXT | nullable |
| status | VARCHAR(20) | NOT NULL, default DRAFT |
| created_at | TIMESTAMP | NOT NULL, UTC |
| updated_at | TIMESTAMP | NOT NULL, UTC |

`UNIQUE(user_id, week_start_date)`.

#### MealPlanEntry (Slot de Refeição)

Uma refeição planejada dentro do plano semanal. Pode ser referência a um Dish **OU** uma nota
rápida (ex: "Delivery", "Sobras", "Comer fora") — nunca ambos.

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | UUID | PK, auto-generated |
| meal_plan_id | UUID | FK → MealPlan, NOT NULL |
| day_of_week | VARCHAR(9) | nullable (null = bucket, sem dia definido) |
| slot_type | VARCHAR(20) | NOT NULL |
| dish_id | UUID | FK → Dish, nullable |
| quick_note | VARCHAR(200) | nullable |
| servings_override | INTEGER | nullable (sobrecreve `dish.servings`) |
| sort_order | INTEGER | NOT NULL, default 0 |

`CHECK: (dish_id IS NOT NULL AND quick_note IS NULL) OR (dish_id IS NULL AND quick_note IS NOT NULL)`.

#### ShoppingList (Lista de Compras)

Lista gerada a partir do plano semanal ou criada manualmente.

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | UUID | PK, auto-generated |
| user_id | UUID | NOT NULL |
| meal_plan_id | UUID | FK → MealPlan, nullable |
| title | VARCHAR(100) | NOT NULL, default "Lista da semana" |
| status | VARCHAR(20) | NOT NULL, default ACTIVE |
| total_estimated_cost | NUMERIC(10,2) | nullable |
| created_at | TIMESTAMP | NOT NULL, UTC |

#### ShoppingListItem (Item da Lista)

Item individual na lista de compras, gerado automaticamente ou adicionado manualmente.

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | UUID | PK, auto-generated |
| shopping_list_id | UUID | FK → ShoppingList, NOT NULL |
| ingredient_id | UUID | FK → Ingredient, nullable (null para itens manuais) |
| name | VARCHAR(150) | NOT NULL (denormalized ou manual) |
| quantity | NUMERIC(10,2) | nullable |
| unit | VARCHAR(20) | nullable |
| category | VARCHAR(30) | NOT NULL (denormalized para ordenação por seção) |
| is_checked | BOOLEAN | NOT NULL, default false |
| already_have | BOOLEAN | NOT NULL, default false |
| is_manually_added | BOOLEAN | NOT NULL, default false |
| sort_order | INTEGER | NOT NULL, default 0 |

#### Outbox

Tabela de outbox para publicação confiável de eventos Kafka.

| Campo | Tipo | Restrições |
|-------|------|------------|
| id | UUID | PK, auto-generated |
| aggregate_type | VARCHAR(50) | NOT NULL |
| aggregate_id | UUID | NOT NULL |
| event_type | VARCHAR(50) | NOT NULL |
| payload | JSONB | NOT NULL |
| created_at | TIMESTAMP | NOT NULL, UTC |
| published_at | TIMESTAMP | nullable |

### Enumerações

| Enum | Valores | Display (pt-BR) |
|------|---------|------------------|
| **MealSlot** | `BREAKFAST`, `LUNCH`, `DINNER`, `SNACK` | Café da manhã, Almoço, Jantar, Lanche |
| **PlanStatus** | `DRAFT`, `ACTIVE`, `COMPLETED` | Rascunho, Ativo, Concluído |
| **ListStatus** | `ACTIVE`, `COMPLETED` | Ativa, Concluída |
| **IngredientCategory** | `PRODUCE`, `MEAT_SEAFOOD`, `DAIRY`, `BAKERY`, `PANTRY`, `FROZEN`, `SPICES_OILS`, `BEVERAGES`, `CLEANING`, `OTHER` | Hortifrúti, Carnes, Laticínios, Padaria, Mercearia, Congelados, Temperos e Óleos, Bebidas, Limpeza, Outros |
| **MeasurementUnit** | `UNIT`, `GRAMS`, `KG`, `ML`, `LITERS`, `CUP`, `TABLESPOON`, `TEASPOON` | Unidade, Gramas, Kg, mL, Litros, Xícara, Colher de sopa, Colher de chá |

### Diagrama de Relacionamentos

```
Ingredient (1) ◄───── (N) DishIngredient (N) ─────► (1) Dish
                                                         │
                                            DishDishTag (N) ─────► (1) DishTag
                                                         │
                                                   (0..1) NutritionInfo
                                                         │
                               MealPlanEntry (N) ────► (1) MealPlan
                                     │
                                     ▼ (gera)
                          ShoppingList (1) ◄───── (N) ShoppingListItem
                                │
                                └── FK → MealPlan (optional)
```

## Regras de Negócio Críticas

### Pratos e Ingredientes

1. **Dish requer apenas `name`.** Todos os outros campos são opcionais. O menor prato válido
   é `{ name: "Miojo" }`.
2. **Quick Note ≠ Dish.** Entradas como "Delivery", "Sobras", "Comer fora" são notas rápidas
   diretas no `MealPlanEntry.quick_note`. Não geram itens na lista de compras.
3. **Ingredientes do sistema** (`user_id = null`) são pré-cadastrados via seed e compartilhados
   entre todos os usuários. Usuários podem criar ingredientes personalizados adicionais.
4. **Deletar ingrediente do sistema é proibido.** Apenas ingredientes do próprio usuário podem
   ser deletados.
5. **Deletar Dish** é soft-delete (`deleted_at`). Remove de futuros planos mas preserva
   histórico.
6. **DishIngredient com Ingredient deletado:** Se um ingrediente do usuário for deletado e
   possuir pratos associados, retorna erro 400. O usuário deve remover a associação primeiro.

### Planejamento Semanal

7. **MealPlan cobre exatamente 1 semana** (segunda a domingo). `week_start_date` deve ser uma
   segunda-feira; caso contrário, retorna erro 400.
8. **Máximo um MealPlan ACTIVE por usuário** a qualquer momento. Ativar um novo plano move o
   anterior para COMPLETED automaticamente.
9. **Entradas são mutuamente exclusivas:** `dish_id` XOR `quick_note`. Violação do CHECK
   retorna erro 400.
10. **Slots fixos:** BREAKFAST, LUNCH, DINNER, SNACK. Sem slots customizáveis no MVP.
11. **Bucket Planning (padrão):** `day_of_week = null` significa "cozinhar durante a semana,
    sem dia fixo". Essa é a experiência padrão que evita a espiral de culpa.
12. **Calendar Planning (opcional):** `day_of_week` pode ser definido para refeições com dia
    específico (ex: churrasco no domingo).
13. **Duplicar plano:** Um MealPlan pode ser copiado integralmente para a semana seguinte,
    criando novos registros de MealPlanEntry. Se já existir plano na semana destino,
    retorna erro 409.

### Lista de Compras — Geração Automática

14. **Algoritmo de geração:** Ao solicitar `POST /meal-plans/{id}/generate-shopping-list`:

    a. Coleta todos os `DishIngredient` das entries com `dish_id IS NOT NULL`.

    b. Aplica multiplicador de porções:
       `quantidade_ajustada = quantity × (servings_override / dish.servings)`.
       Se `servings_override` for null, usa `quantity` diretamente.

    c. Agrupa itens por `ingredient_id`.

    d. Para mesmo `ingredient_id` + mesma `unit`: **soma as quantidades**.

    e. Para mesmo `ingredient_id` + unidades diferentes: **mantém itens separados**
       (sem conversão automática no MVP).

    f. Categoriza cada item pela `category` do ingrediente (`PRODUCE`, `MEAT_SEAFOOD`, etc.).

    g. Ordena por categoria para navegação aisle-by-aisle no supermercado.

15. **Itens manuais** podem ser adicionados livremente (ex: "Detergente", "Papel higiênico").
    Possuem `is_manually_added = true` e requerem `name` e `category`.

### Lista de Compras — Despensa Efêmera

16. **Checklist efêmera de despensa (Modelo Mealime):** Após gerar a lista, o endpoint
    `POST /shopping-lists/{id}/pantry-check` recebe uma lista de `ingredient_id`s que o
    usuário já possui em casa. Esses itens recebem `already_have = true` e são ocultados da
    view de compras (mas permanecem no registro para auditoria).
17. **NÃO há tabela de despensa persistente.** A marcação "já tenho" é por-lista, por-geração.
    Zero manutenção contínua — resolve o anti-pattern #1 de abandono.
18. **Completar lista:** Marcar todos os itens (onde `already_have = false`) como `is_checked`
    move a lista para `COMPLETED`.
19. **Re-gerar lista:** Gerar nova lista para o mesmo MealPlan substitui a lista anterior se
    estiver ACTIVE. Listas COMPLETED são preservadas como histórico.

### Nutrição (Opcional — Nice-to-Have)

20. **NutritionInfo é por prato, não por ingrediente.** O usuário informa valores por porção
    do prato inteiro.
21. **Todos os campos nutricionais são anuláveis.** Partial tracking é permitido (ex: só
    calorias, sem macros).
22. **Totais diários derivados:** Se entries de um dia possuem NutritionInfo, o sistema calcula
    a soma do dia: `Σ(valor × porções_efetivas)`. Entries sem NutritionInfo contribuem como
    zero.
23. **Sem metas nutricionais no MVP.** Apenas visualização informativa dos totais diários.

## Contratos Kafka

### Produzidos pelo Food Service

#### SHOPPING_LIST_CREATED (tópico: `food.shopping-list.events`)

Emitido atomicamente via Outbox ao gerar uma lista de compras a partir de um MealPlan.

```json
{
  "eventType": "SHOPPING_LIST_CREATED",
  "shoppingListId": "d1e2f3a4-...",
  "userId": "a1b2c3d4-...",
  "weekDate": "2026-09-08",
  "totalEstimatedCost": 250.00,
  "suggestedShoppingDate": "2026-09-13",
  "timestamp": "2026-09-08T10:00:00Z"
}
```

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| eventType | String | Sim | `"SHOPPING_LIST_CREATED"` |
| shoppingListId | UUID | Sim | ID da lista gerada |
| userId | UUID | Sim | Dono da lista |
| weekDate | LocalDate | Sim | Segunda-feira da semana do plano |
| totalEstimatedCost | BigDecimal | Não | Custo estimado (null se sem dados de preço) |
| suggestedShoppingDate | LocalDate | Sim | Sábado da semana (calculado automaticamente) |
| timestamp | Instant | Sim | Quando o evento foi publicado |

### Consumidos pelo Food Service

Nenhum no MVP. O food-service é produtor de eventos, não consumidor.

## Pontos de Integração

| Direção | De → Para | Mecanismo | Status |
|---------|-----------|-----------|--------|
| Food → Task | food → task | Kafka: `food.shopping-list.events` | **MVP** |
| Food → Finance (via Task) | food → task → finance | Kafka chain | **MVP** (usa contratos existentes) |
| Finance → Food | finance → food | REST ou Kafka | **V2** — preço médio de ingredientes para estimativa de custo |
| Food → Task | food → task | Kafka: `food.meal-prep.events` | **V2** — tarefas de preparo (ex: "Descongelar carne") |

### Fluxo Completo: Planejamento → Compras → Finanças

```
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ 1. PLANEJAMENTO (Food Service)                                         │
 │ Usuário monta o cardápio da semana: 5 pratos no bucket + 2 com dia     │
 └─────────────────────────────────┬───────────────────────────────────────┘
                                   │ POST /meal-plans/{id}/generate-shopping-list
                                   ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ 2. LISTA DE COMPRAS (Food Service)                                     │
 │ Ingredientes consolidados, categorizados por seção do supermercado     │
 │ Usuário faz pantry-check: "já tenho sal, óleo, arroz"                  │
 │ → Emite SHOPPING_LIST_CREATED via Outbox                               │
 └─────────────────────────────────┬───────────────────────────────────────┘
                                   │ Kafka: food.shopping-list.events
                                   ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ 3. TAREFA AUTOMÁTICA (Task Service)                                    │
 │ Cria: "🛒 Fazer feira da semana"                                       │
 │ scheduledDate = sábado, expectedAmount = R$250                         │
 └─────────────────────────────────┬───────────────────────────────────────┘
                                   │ Usuário completa a tarefa
                                   │ Kafka: task.completed.events
                                   ▼
 ┌─────────────────────────────────────────────────────────────────────────┐
 │ 4. DESPESA AUTOMÁTICA (Finance Service)                                │
 │ Cria Transaction: EXPENSE, amount = R$250, category = "Alimentação"    │
 │ status = PENDING (até confirmação/reconciliação)                       │
 └─────────────────────────────────────────────────────────────────────────┘
```

## Seed de Ingredientes (Amostra)

O migration `V2__seed_ingredients.sql` populará ingredientes básicos da culinária brasileira:

| Categoria | Exemplos |
|-----------|----------|
| PRODUCE | Cebola, Alho, Tomate, Batata, Cenoura, Alface, Limão, Banana, Maçã |
| MEAT_SEAFOOD | Peito de frango, Carne moída, Linguiça, Filé de peixe, Ovo |
| DAIRY | Leite, Queijo muçarela, Manteiga, Iogurte, Creme de leite, Requeijão |
| BAKERY | Pão francês, Pão de forma, Bolo pronto |
| PANTRY | Arroz, Feijão, Macarrão, Farinha de trigo, Açúcar, Café, Óleo de soja |
| FROZEN | Legumes congelados, Polpa de fruta, Sorvete |
| SPICES_OILS | Sal, Azeite, Pimenta, Orégano, Cominho, Molho de tomate, Shoyu |
| BEVERAGES | Suco, Refrigerante, Água mineral, Cerveja |
| CLEANING | Detergente, Sabão em pó, Desinfetante, Papel higiênico |

## Superfície REST API

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| **Pratos** | | |
| GET | `/api/v1/dishes` | Listar (filtros: favorite, tag, busca por nome) |
| POST | `/api/v1/dishes` | Criar prato |
| GET | `/api/v1/dishes/{id}` | Detalhe (com ingredientes e nutrition) |
| PUT | `/api/v1/dishes/{id}` | Atualizar |
| DELETE | `/api/v1/dishes/{id}` | Soft-delete |
| POST | `/api/v1/dishes/{id}/favorite` | Toggle favorito |
| **Ingredientes do Prato** | | |
| GET | `/api/v1/dishes/{id}/ingredients` | Listar ingredientes |
| POST | `/api/v1/dishes/{id}/ingredients` | Adicionar ingrediente |
| PUT | `/api/v1/dishes/{dishId}/ingredients/{ingredientId}` | Atualizar quantidade/unidade |
| DELETE | `/api/v1/dishes/{dishId}/ingredients/{ingredientId}` | Remover |
| **Nutrição** | | |
| GET | `/api/v1/dishes/{id}/nutrition` | Obter info nutricional |
| PUT | `/api/v1/dishes/{id}/nutrition` | Definir/atualizar info nutricional |
| **Catálogo de Ingredientes** | | |
| GET | `/api/v1/ingredients` | Listar (sistema + usuário, filtro por categoria) |
| POST | `/api/v1/ingredients` | Criar ingrediente custom |
| PUT | `/api/v1/ingredients/{id}` | Atualizar (somente custom) |
| DELETE | `/api/v1/ingredients/{id}` | Deletar (somente custom, sem pratos vinculados) |
| **Tags de Pratos** | | |
| GET/POST | `/api/v1/dish-tags` | Listar / criar |
| PUT/DELETE | `/api/v1/dish-tags/{id}` | Atualizar / deletar |
| **Planos Semanais** | | |
| GET | `/api/v1/meal-plans` | Listar (filtro por status, data) |
| POST | `/api/v1/meal-plans` | Criar plano (informar `week_start_date`) |
| GET | `/api/v1/meal-plans/{id}` | Detalhe (com entries e totais nutricionais) |
| PUT | `/api/v1/meal-plans/{id}` | Atualizar notas |
| DELETE | `/api/v1/meal-plans/{id}` | Deletar plano DRAFT |
| POST | `/api/v1/meal-plans/{id}/activate` | Ativar plano |
| POST | `/api/v1/meal-plans/{id}/duplicate` | Duplicar para próxima semana |
| **Entries do Plano** | | |
| GET | `/api/v1/meal-plans/{id}/entries` | Listar entries |
| POST | `/api/v1/meal-plans/{id}/entries` | Adicionar entry (dish ou quick note) |
| PUT | `/api/v1/meal-plans/{planId}/entries/{entryId}` | Atualizar entry |
| DELETE | `/api/v1/meal-plans/{planId}/entries/{entryId}` | Remover entry |
| **Listas de Compras** | | |
| GET | `/api/v1/shopping-lists` | Listar |
| POST | `/api/v1/meal-plans/{id}/generate-shopping-list` | Gerar lista do plano |
| GET | `/api/v1/shopping-lists/{id}` | Detalhe (agrupado por categoria) |
| POST | `/api/v1/shopping-lists/{id}/items` | Adicionar item manual |
| PATCH | `/api/v1/shopping-lists/{id}/items/{itemId}` | Toggle checked / already_have |
| DELETE | `/api/v1/shopping-lists/{id}/items/{itemId}` | Remover item manual |
| POST | `/api/v1/shopping-lists/{id}/pantry-check` | Marcar staples como "já tenho" |

## MVP vs Nice-to-Have

### MVP (Entrega 1)

- Ingredientes: catálogo sistema (seed ~200 itens brasileiros) + CRUD custom do usuário
- Dish CRUD (nome, porções, tempo de preparo, notas, favorito)
- DishIngredient com quantidade e unidade
- DishTag para categorização de pratos
- MealPlan semanal (bucket + calendar)
- MealPlanEntry (4 slots fixos, dish XOR quick note)
- Duplicação de plano para próxima semana
- Geração automática de ShoppingList com consolidação por ingrediente e categorização por
  seção do supermercado
- Checklist efêmera de despensa (already_have por lista, sem inventário persistente)
- Itens manuais na lista de compras
- Kafka: publicação de `SHOPPING_LIST_CREATED` via Outbox
- REST API completa com paginação

### Nice-to-Have (Entrega 2+)

- NutritionInfo por prato (calorias + macros P/C/G)
- Totais nutricionais diários derivados do plano
- Metas nutricionais (target de calorias/macros por dia)
- Sugestão de pratos baseada em ingredientes perecíveis não usados (anti-waste)
- Templates de plano (padrões semanais reutilizáveis)
- Import de receita por URL (web scraping)
- Conversão automática de unidades compatíveis (mL→L, g→kg)
- Preço estimado por ingrediente (integração futura com finance-service)
- Store-specific aisle customization (reordenar categorias por supermercado)
- Cooking mode com step-by-step e screen wake-lock
- Tarefas de preparo automáticas (ex: "Descongelar carne", "Preparar marmitas")
- Barcode scanning para identificação de ingredientes

## Anti-patterns a Evitar

1. **Inventário persistente de despensa:** Pesquisa mostra abandono em ~4 dias quando o
   usuário precisa manter quantidades de tudo que tem na cozinha. A solução correta é a
   checklist efêmera pré-compras do modelo Mealime: "Já tem óleo? Sal? Arroz?" — 5 segundos,
   zero manutenção contínua.

2. **Calendário rígido obrigatório:** Forçar "terça-feira 19h: salmão" cria culpa quando a
   vida real interfere (trabalho atrasou, amigos convidaram para jantar). Bucket planning
   ("5 pratos para a semana") como default; o usuário _pode_ alocar dias se quiser.

3. **Micronutrientes:** Ferro, potássio, vitaminas B12 — complexidade que nenhum usuário comum
   vai manter atualizado. Máximo: calorias + proteína/carboidrato/gordura por porção.

4. **Pesagem obrigatória em gramas:** Exigir que o usuário pese cada ingrediente causa burnout
   em ~14 dias. Permitir unidades caseiras (xícara, colher, unidade inteira).

5. **Receitas com 50 passos:** O campo `notes` (texto livre) é suficiente no MVP. Instruções
   passo-a-passo detalhadas com temporizadores integrados são nice-to-have.

6. **Listas de compras não consolidadas:** "1 cebola", "1 cebola picada" e "2 cebolas médias"
   como 3 itens separados frustra no supermercado. Consolidar por `ingredient_id` canônico +
   mesma unidade.

7. **Waste por desconexão de receitas:** Planos com ingredientes perecíveis usados em apenas
   uma receita desperdiçam comida e dinheiro (meio maço de coentro, 1/4 de repolho).
   Futuramente, sugerir receitas que compartilhem perecíveis.

8. **Hidratação no food-service:** Água não tem calorias, não gera item de compra, e não
   pertence ao domínio de alimentação. Se implementado, pertence a um habit-service ou
   integração direta com Apple Health / Google Health Connect.
