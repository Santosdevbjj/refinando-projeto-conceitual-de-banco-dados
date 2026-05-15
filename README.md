# Modelagem Relacional para E-Commerce — Arquitetura de Dados Pronta para Produção

<div align="center">

[![Status](https://img.shields.io/badge/Status-Produção--Ready-00c853?style=for-the-badge)]()
[![Modelo](https://img.shields.io/badge/Modelo-EER%20Relacional-0066cc?style=for-the-badge)]()
[![Bootcamp](https://img.shields.io/badge/DIO-Heineken%20AI%20%2B%20Dados-e8b800?style=for-the-badge&logo=databricks&logoColor=white)](https://www.dio.me)

[![Portfólio](https://img.shields.io/badge/Portfólio-Sérgio_Santos-111827?style=for-the-badge&logo=githubpages&logoColor=00eaff)](https://portfoliosantossergio.vercel.app)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Sérgio_Santos-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/santossergioluiz)

</div>

---

## 1. Problema de Negócio

Plataformas de e-commerce em crescimento enfrentam um problema recorrente de dados: à medida que o volume de pedidos escala, sistemas construídos sem uma modelagem relacional sólida começam a falhar em rastreabilidade, consistência fiscal e controle de estoque.

**O impacto é direto e mensurável:**
- Impossibilidade de distinguir clientes Pessoa Física de Pessoa Jurídica para fins tributários e de compliance
- Falta de rastreabilidade entre pedido, item e origem do estoque — impossibilitando auditorias
- Ausência de controle de estado do pedido (criado → pago → despachado → entregue), gerando inconsistências financeiras
- Dados de pagamento misturados com dados de pedido, dificultando a conciliação

Este projeto entrega a **fundação de dados** que elimina esses gargalos: um modelo EER (Enhanced Entity-Relationship) completo, com integridade referencial, especialização de entidades e rastreabilidade em todo o ciclo de vida da venda.

---

## 2. Contexto

O modelo foi desenvolvido no contexto de uma operação de e-commerce de médio porte, onde diferentes times precisam de visibilidade sobre segmentos distintos da operação:

| Time | Necessidade | Entidade Central |
|---|---|---|
| Operações | Rastrear de onde cada item foi retirado | `itemPedido` → `Estoque` |
| Financeiro | Conciliar pagamentos por cliente e pedido | `Pagamento` → `Cliente` + `Pedido` |
| Logística | Monitorar status de entrega com código de rastreio | `Entrega` → `Pedido` |
| Fiscal | Separar regras de PF e PJ | `ClientePF` / `ClientePJ` |
| Compras | Associar produto ao fornecedor correto | `Produto` → `Fornecedor` |

A base histórica de dados existe, mas sem modelo formal — o que impede análises confiáveis e a construção de pipelines de dados sobre ela.

---

## 3. Premissas da Modelagem

Para garantir decisões de design coerentes, foram adotadas as seguintes premissas:

- **Especialização disjunta** para `Cliente`: cada registro é **exclusivamente** PF ou PJ — nunca ambos. Isso garante regras fiscais e de marketing diferenciadas sem ambiguidade
- **Rastreabilidade completa de estoque**: cada `itemPedido` referencia não apenas o produto, mas o registro de estoque específico de onde o item foi retirado (`idEstoque`), permitindo rastrear lote e localização
- **Flag de cancelamento** (`canceladoFlag`) em `Pedido` preserva o histórico — pedidos nunca são deletados, apenas marcados
- **Campos de data em todas as entidades** (`dataFor`, `dataProd`, `dataEsto`, `dataCli`, `dataPag`, `dataEntr`) garantem auditoria temporal completa
- Valores ausentes em atributos opcionais (ex: detalhes de pagamento) são admitidos via nullable; campos críticos de negócio (FK, status, valores) são tratados como NOT NULL

---

## 4. Estratégia da Solução

O projeto seguiu uma abordagem estruturada de modelagem de dados em três camadas:

### 4.1 Levantamento do Domínio de Negócio
Mapeamento das entidades de negócio e seus relacionamentos antes de qualquer decisão técnica: quem compra, o que é vendido, de onde vem, como é pago, como é entregue.

### 4.2 Modelagem Conceitual (EER)
Definição das entidades, atributos, relacionamentos e cardinalidades. Decisões-chave:
- Uso de **especialização** para `Cliente` (herança com discriminador)
- Tabela de junção `itemPedido` para resolver a relação M:N entre `Pedido` e `Produto`, enriquecida com `quantidade`, `precoVenda` e `idEstoque`
- Separação entre `Entrega` e `Pagamento` como entidades independentes — cada uma com ciclo de vida e status próprios

### 4.3 Validação da Integridade Referencial
Todas as chaves estrangeiras foram validadas quanto à consistência:
- `itemPedido.idEstoque` → `Estoque.idEstoque`
- `Estoque.idFornecedor` → `Fornecedor.idFornecedor`
- `Pagamento.idCliente` → `Cliente.idCliente` (e não apenas `idPedido`)

Essa última decisão garante conciliação financeira independente do pedido.

---

## 5. Diagrama EER

<div align="center">

![Diagrama EER — E-Commerce](https://github.com/user-attachments/assets/4d5cf73f-fee3-43b4-8937-4fccc41adbae)

*Diagrama completo com todas as entidades, atributos, PKs, FKs e relacionamentos*

</div>

---

## 6. Insights Arquiteturais

A modelagem revelou decisões que impactam diretamente a operação:

**Rastreabilidade de Estoque por Item**
A maioria dos modelos de e-commerce liga `itemPedido` apenas ao `Produto`. Aqui, a referência direta ao `Estoque` permite saber **de qual depósito e lote** o item foi retirado — essencial para operações multi-warehouse e recalls de produto.

**Pagamento Desacoplado do Pedido**
`Pagamento` referencia tanto `idPedido` quanto `idCliente` diretamente. Isso viabiliza **múltiplos pagamentos por pedido** (ex: pedido parcelado em dois cartões) sem alterar a estrutura do pedido.

**Especialização Disjunta de Cliente**
O padrão `Cliente` → `ClientePF` / `ClientePJ` evita a armadilha comum de armazenar CPF e CNPJ na mesma tabela com campos nulos. Isso garante **integridade semântica** e facilita compliance (LGPD, tributação).

**Flag de Cancelamento vs. Exclusão**
`canceladoFlag` em `Pedido` implementa **soft delete**: pedidos cancelados são preservados para auditoria, análise de churn e relatórios financeiros — sem quebrar FKs dependentes em `Pagamento` e `Entrega`.

---

## 7. Resultados

O modelo entrega:

| Entregável | Valor de Negócio |
|---|---|
| Rastreabilidade pedido → estoque → fornecedor | Auditoria e gestão de inventário multi-origem |
| Especialização PF/PJ | Compliance fiscal e segmentação de cliente |
| Entidade `Entrega` independente | Integração com APIs de rastreio logístico |
| Entidade `Pagamento` independente | Suporte a múltiplos meios e conciliação financeira |
| Campos de data em todas as entidades | Time-series analytics e auditoria temporal |
| Soft delete via `canceladoFlag` | Preservação de histórico sem quebra de integridade |

Este modelo está pronto para servir como camada de persistência de uma API REST, como fonte para um pipeline de dados (ETL/ELT) ou como base para um Data Warehouse com modelagem dimensional.

---

## 8. Decisões Técnicas

> *"Por que esse design e não outro?"*

**Por que EER e não modelagem diretamente no ORM?**
Começar pelo modelo conceitual — antes de qualquer código — evita que decisões de implementação (ex: limitações do ORM) contaminem o design do domínio. O EER garante que a arquitetura de dados reflita o negócio, não a ferramenta.

**Por que `itemPedido` e não um array em `Pedido`?**
Armazenar itens como estrutura aninhada (comum em bancos NoSQL) compromete a capacidade de query analítica. A tabela de junção normalizada permite agregações por produto, por fornecedor e por estoque sem joins complexos ou desaninhamento.

**Por que separar `Entrega` de `Pedido`?**
Um pedido pode gerar múltiplas entregas (entrega parcial, reenvio por extravio). Manter `Entrega` como entidade independente com `statusEntrega` e `codigoRastreio` suporta esse cenário sem reprocessar o pedido.

---

## 9. Como Reproduzir

O diagrama foi produzido com ferramenta de modelagem EER. Para inspecionar ou evoluir o modelo:

```bash
# Ferramentas recomendadas para abrir/editar o modelo
MySQL Workbench          # modelagem EER nativa, exporta DDL SQL
dbdiagram.io             # colaborativo, suporta importação de SQL
draw.io (diagrams.net)   # exporta PNG/SVG para documentação
```

Para gerar o DDL SQL a partir do diagrama no MySQL Workbench:
```
Database → Forward Engineer → gera CREATE TABLE com PKs, FKs e constraints
```

---

## 10. Aprendizados

**O que foi mais desafiador:**
Definir a cardinalidade correta entre `itemPedido`, `Produto` e `Estoque`. A tentação inicial foi ligar o item diretamente ao produto e ignorar o estoque — o que eliminaria a rastreabilidade de lote/localização.

**O que faria diferente em um sistema real:**
Adicionaria uma entidade `StatusPedido` separada com histórico de transições de estado (log de eventos), em vez de um único campo `status` em `Pedido`. Isso permite auditoria completa da jornada do pedido.

**Principal aprendizado:**
Modelagem de dados não é exercício técnico — é tradução do negócio para estrutura. Cada FK mal planejada cria uma limitação operacional real meses depois, quando o volume de dados torna a refatoração custosa.

---

## 11. Próximos Passos

- [ ] Gerar DDL SQL completo com constraints, índices e comentários
- [ ] Implementar camada de persistência com Spring Boot + JPA (mapeamento das entidades)
- [ ] Criar pipeline de ingestão de dados para um Data Warehouse (modelo Star Schema com fatos de `Pedido` e dimensões de `Cliente`, `Produto`, `Fornecedor`)
- [ ] Adicionar entidade `StatusPedidoLog` para histórico de transições de estado
- [ ] Avaliar migração do modelo relacional para estratégia híbrida (PostgreSQL + cache Redis para consultas de catálogo)

---

<div align="center">

**Sérgio Luiz dos Santos** — Senior Data Engineer & Cloud Architect

[![Portfólio](https://img.shields.io/badge/Portfólio-Sérgio_Santos-111827?style=for-the-badge&logo=githubpages&logoColor=00eaff)](https://portfoliosantossergio.vercel.app)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Sérgio_Santos-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/santossergioluiz)

</div>
