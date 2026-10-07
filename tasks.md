# Plano de tarefas — Zona Azul Digital

## 1. Objetivo

Decompor a implementação da API da Zona Azul Digital em tarefas independentes e verificáveis, mantendo rastreabilidade entre requisitos, regras de negócio e testes definidos em `spec.md` e `tests.md`.

---

## 2. Preparação da estrutura

### T01 — Definir o modelo de dados do bilhete

* Representar `id`, `placa`, `entrada`, `saida`, `minutos`, `valor_centavos` e `status`.
* Representar explicitamente os estados `aberto`, `encerrado` e `cancelado`.
* Garantir que campos de encerramento não sejam preenchidos em bilhetes ainda abertos.
* Garantir que bilhetes cancelados não possuam cobrança.

**Critério de conclusão:** o modelo deve permitir representar todos os estados e informações exigidos pelo contrato sem ambiguidades.

---

### T02 — Implementar validações de entrada

Implementar as validações para:

* placa;
* entrada;
* data do relatório;
* parâmetros obrigatórios;
* formato ISO-8601;
* timezone.

**Critério de conclusão:** entradas inválidas devem produzir os códigos HTTP e identificadores de erro definidos na especificação.

---

## 3. Ciclo de vida do bilhete

### T03 — Implementar abertura de bilhete

Implementar `POST /bilhetes`.

* Validar a placa.
* Validar `entrada`, quando fornecida.
* Utilizar o instante atual quando `entrada` não for informada.
* Criar bilhete com estado `aberto`.
* Impedir dois bilhetes abertos para a mesma placa.

**Critério de conclusão:** os casos T01–T10 e T48–T53 de `tests.md` devem ser atendidos.

---

### T04 — Implementar encerramento de bilhete

Implementar `POST /bilhetes/{id}/encerramento`.

* Localizar o bilhete.
* Validar se o bilhete pode ser encerrado.
* Registrar saída.
* Calcular duração.
* Aplicar tolerância.
* Calcular frações.
* Aplicar teto diário.
* Registrar `valor_centavos`.
* Alterar estado para `encerrado`.

**Critério de conclusão:** os casos T11–T20 e T54 de `tests.md` devem ser atendidos.

---

### T05 — Implementar cancelamento de bilhete

Implementar `POST /bilhetes/{id}/cancelamento`.

* Localizar o bilhete.
* Permitir cancelamento somente de bilhete aberto.
* Alterar estado para `cancelado`.
* Não gerar cobrança.
* Não preencher saída ou valor.

**Critério de conclusão:** os casos T33–T37, T55 e T60 de `tests.md` devem ser atendidos.

---

## 4. Consultas

### T06 — Implementar consulta de bilhetes ativos

Implementar `GET /bilhetes/ativos`.

* Retornar somente bilhetes `aberto`.
* Ordenar do mais novo para o mais antigo.
* Retornar lista vazia quando não houver ativos.

**Critério de conclusão:** os casos T21–T25 de `tests.md` devem ser atendidos.

---

### T07 — Implementar histórico por placa

Implementar `GET /bilhetes?placa=...`.

* Validar a placa.
* Buscar todos os estados da placa.
* Não retornar registros de outras placas.
* Ordenar do mais novo para o mais antigo.
* Retornar lista vazia quando não houver histórico.

**Critério de conclusão:** os casos T38–T43 de `tests.md` devem ser atendidos.

---

### T08 — Implementar relatório diário

Implementar `GET /relatorios/diario?data=...`.

* Validar a data.
* Considerar somente bilhetes encerrados na data.
* Calcular quantidade.
* Calcular faturamento.
* Calcular tempo médio.
* Aplicar a regra de arredondamento da média.
* Retornar zeros quando não houver encerramentos.

**Critério de conclusão:** os casos T26–T32 e T61 de `tests.md` devem ser atendidos.

---

## 5. Regras de cobrança

### T09 — Implementar cálculo de duração e frações

* Calcular duração em minutos.
* Aplicar `FRACAO_MINUTOS`.
* Arredondar a quantidade de frações para cima.
* Aplicar `TOLERANCIA_MINUTOS`.
* Utilizar somente inteiros em centavos.
* Respeitar `TETO_DIARIO_CENTAVOS`.

**Critério de conclusão:** os casos de cobrança de `tests.md` devem ser atendidos de maneira determinística.

---

### T10 — Resolver e documentar a conversão da tarifa por fração

A variante atual produz:

```text
550 / (60 / 15) = 137,5 centavos
```

A implementação deve possuir uma regra determinística para representar esse resultado como centavos inteiros.

A decisão deve ser documentada e não deve depender de ponto flutuante.

**Critério de conclusão:** a regra adotada deve estar documentada e produzir sempre o mesmo resultado para os mesmos parâmetros.

---

## 6. Integração e consistência

### T11 — Garantir transições de estado

Verificar as transições:

```text
ABERTO → ENCERRADO
ABERTO → CANCELADO
```

e impedir:

```text
ENCERRADO → qualquer estado final diferente
CANCELADO → qualquer estado final diferente
```

**Critério de conclusão:** os casos T54–T56 devem ser atendidos.

---

### T12 — Garantir consistência entre consultas

Verificar que:

* encerramento remove o bilhete da lista de ativos;
* cancelamento remove o bilhete da lista de ativos;
* encerramento permanece no histórico;
* cancelamento permanece no histórico;
* cancelamento não aparece no relatório;
* bilhetes de outras placas não aparecem no histórico consultado.

**Critério de conclusão:** os casos T57–T61 devem ser atendidos.

---

## 7. Tratamento de erros

### T13 — Padronizar respostas de erro

Garantir os códigos:

* `placa_invalida`;
* `entrada_invalida`;
* `data_invalida`;
* `bilhete_nao_encontrado`;
* `bilhete_ja_encerrado`;
* `bilhete_nao_aberto`;
* `bilhete_em_aberto`.

Garantir a precedência de validação estrutural sobre conflitos de negócio.

**Critério de conclusão:** os casos T53, T62 e T63 devem ser atendidos.

---

## 8. Validação final

### T14 — Executar os testes funcionais

Verificar todas as situações descritas em `tests.md`, incluindo:

* fluxos normais;
* casos de borda;
* limites;
* erros;
* transições de estado;
* ordenação;
* relatórios;
* cobrança.

**Critério de conclusão:** todos os casos aplicáveis devem apresentar comportamento compatível com `spec.md` e `contrato.json`.

---

### T15 — Verificar conformidade da API

Validar:

* caminhos dos endpoints;
* métodos HTTP;
* códigos HTTP;
* nomes dos campos;
* formatos de data/hora;
* códigos de erro;
* representação monetária;
* porta de serviço.

**Critério de conclusão:** nenhuma divergência deve permanecer entre a implementação e o contrato.

---

## 9. Checklist de conclusão

* [ ] Modelo de bilhete definido.
* [ ] Validações implementadas.
* [ ] Abertura implementada.
* [ ] Encerramento implementado.
* [ ] Cancelamento implementado.
* [ ] Consulta de ativos implementada.
* [ ] Histórico por placa implementado.
* [ ] Relatório diário implementado.
* [ ] Cobrança por fração implementada.
* [ ] Tolerância implementada.
* [ ] Teto diário implementado.
* [ ] Unicidade de bilhete aberto garantida.
* [ ] Estados e transições validados.
* [ ] Respostas de erro padronizadas.
* [ ] Casos de `tests.md` verificados.
* [ ] Contrato revisado após a implementação.
