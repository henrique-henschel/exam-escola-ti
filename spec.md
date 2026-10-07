# Especificação funcional — Zona Azul Digital

## 1. Objetivo

Especificar o comportamento de uma API REST para gerenciamento de bilhetes de estacionamento da Zona Azul Digital.

A API deve permitir:

* abrir bilhetes;
* encerrar bilhetes e calcular a cobrança;
* consultar bilhetes ativos;
* consultar o relatório diário;
* cancelar bilhetes;
* consultar o histórico de bilhetes por placa.

A implementação deve respeitar integralmente o contrato definido para a prova e os parâmetros da variante atribuída ao aluno.

## 2. Parâmetros da variante

A implementação deve utilizar os seguintes parâmetros:

| Parâmetro              |  Valor |
| ---------------------- | -----: |
| `TARIFA_HORA_CENTAVOS` |  `550` |
| `FRACAO_MINUTOS`       |   `15` |
| `TETO_DIARIO_CENTAVOS` | `8000` |
| `TOLERANCIA_MINUTOS`   |    `0` |
| `PORTA_SERVICO`        | `8004` |

Valores monetários devem ser tratados como inteiros em centavos, sem utilização de ponto flutuante.

## 3. Regras gerais

### 3.1 Validação de entrada

A validação do formato dos dados deve ocorrer antes da aplicação das regras de negócio.

Quando uma requisição contém simultaneamente um dado inválido e uma situação que poderia produzir conflito de negócio, o erro de validação (`422`) deve prevalecer sobre o conflito (`409`).

### 3.2 Identificação de placa

Uma placa válida deve possuir exatamente 7 caracteres alfanuméricos em maiúsculo.

A mesma regra deve ser utilizada em todas as operações que recebem placa.

### 3.3 Estados do bilhete

Um bilhete pode possuir os estados:

* `aberto`;
* `encerrado`;
* `cancelado`.

As transições permitidas são:

* criação → `aberto`;
* `aberto` → `encerrado`;
* `aberto` → `cancelado`.

Bilhetes encerrados ou cancelados não devem retornar ao estado `aberto`.

### 3.4 Unicidade de bilhete aberto

Uma placa pode possuir no máximo um bilhete com estado `aberto` simultaneamente.

Após o encerramento ou cancelamento do bilhete aberto, uma nova abertura para a mesma placa deve ser permitida.

### 3.5 Ordenação

As consultas de bilhetes que exigem ordenação devem apresentar os registros mais novos primeiro.

Isso se aplica à listagem de bilhetes ativos e ao histórico de bilhetes por placa.

---

## 4. UC1 — Abertura de bilhete

### 4.1 Endpoint

`POST /bilhetes`

### 4.2 Entrada

O corpo deve conter:

```json
{
  "placa": "ABC1D23"
}
```

O campo `placa` é obrigatório e deve obedecer à regra de validação de placa.

O campo `entrada` é opcional.

Quando `entrada` não for fornecida, deve ser utilizada a data e hora atual.

Quando `entrada` for fornecida, deve estar no formato ISO-8601 e possuir informação de timezone.

### 4.3 Sucesso

Quando os dados forem válidos e não houver outro bilhete aberto para a placa, a operação deve retornar HTTP `201`.

A resposta deve conter:

* `id`;
* `placa`;
* `entrada`;
* `status`, com valor `aberto`.

A entrada retornada deve representar um instante com offset `-03:00`.

### 4.4 Erros

| Situação                              |  HTTP | Erro                |
| ------------------------------------- | ----: | ------------------- |
| Placa ausente ou inválida             | `422` | `placa_invalida`    |
| Entrada inválida                      | `422` | `entrada_invalida`  |
| Já existe bilhete aberto para a placa | `409` | `bilhete_em_aberto` |

### 4.5 Critérios de aceitação

* Dada uma placa válida sem bilhete aberto, a API deve criar exatamente um bilhete com estado `aberto` e responder `201`.
* Uma placa com quantidade diferente de 7 caracteres deve ser rejeitada com `422` e erro `placa_invalida`.
* Uma placa contendo caracteres não alfanuméricos ou letras minúsculas deve ser rejeitada com `422` e erro `placa_invalida`.
* Uma entrada fornecida sem timezone ou em formato inválido deve ser rejeitada com `422` e erro `entrada_invalida`.
* Uma tentativa de abertura para uma placa que já possui bilhete aberto deve retornar `409` e erro `bilhete_em_aberto`.
* Quando uma requisição possuir formato inválido e também houver conflito de negócio, a resposta deve ser `422`.

---

## 5. UC2 — Encerramento de bilhete

### 5.1 Endpoint

`POST /bilhetes/{id}/encerramento`

### 5.2 Comportamento

O encerramento deve localizar o bilhete pelo `id`.

Um bilhete inexistente deve produzir `404`.

Um bilhete que já esteja encerrado não pode ser encerrado novamente.

Ao encerrar um bilhete aberto, deve ser registrada a saída e calculada a duração e o valor da cobrança.

### 5.3 Resposta de sucesso

A operação deve retornar HTTP `200` e conter:

* `id`;
* `placa`;
* `entrada`;
* `saida`;
* `minutos`;
* `valor_centavos`.

### 5.4 Cálculo da cobrança

A duração deve ser calculada em minutos entre `entrada` e `saida`.

A duração deve ser arredondada para cima em relação à fração definida por `FRACAO_MINUTOS`.

Para `FRACAO_MINUTOS = 15`:

* uma duração de até uma fração corresponde a uma fração cobrada;
* uma duração que ultrapasse uma fração em 1 minuto deve avançar para a próxima fração;
* o cálculo deve utilizar unidades inteiras de centavos.

A tarifa horária da variante é `550` centavos.

O teto diário da variante é `8000` centavos.

O valor final nunca deve ultrapassar o teto diário.

Não deve ser utilizado ponto flutuante para representar valores monetários.

### 5.5 Erros

| Situação             |  HTTP | Erro                     |
| -------------------- | ----: | ------------------------ |
| Bilhete inexistente  | `404` | `bilhete_nao_encontrado` |
| Bilhete já encerrado | `409` | `bilhete_ja_encerrado`   |

### 5.6 Critérios de aceitação

* Um bilhete aberto deve poder ser encerrado e retornar `200`.
* O encerramento deve registrar `saida` e `minutos`.
* Uma duração exatamente igual a uma fração deve cobrar uma fração.
* Uma duração que ultrapasse a fração em um minuto deve ser arredondada para a próxima fração.
* O valor retornado deve ser expresso em `valor_centavos` como inteiro.
* O valor cobrado não deve ultrapassar `TETO_DIARIO_CENTAVOS`.
* Um `id` inexistente deve produzir `404` com `bilhete_nao_encontrado`.
* Um bilhete já encerrado deve produzir `409` com `bilhete_ja_encerrado`.

---

## 6. UC3 — Consulta de bilhetes ativos

### 6.1 Endpoint

`GET /bilhetes/ativos`

### 6.2 Comportamento

A operação deve retornar todos os bilhetes cujo estado atual seja `aberto`.

Os bilhetes devem ser apresentados do mais novo para o mais antigo.

Bilhetes encerrados e cancelados não devem aparecer na resposta.

### 6.3 Resposta

Em caso de sucesso, a operação deve retornar HTTP `200` e uma lista de bilhetes.

Quando não houver bilhetes abertos, deve retornar uma lista vazia.

### 6.4 Critérios de aceitação

* Com nenhum bilhete aberto, a resposta deve ser `200` com lista vazia.
* Um bilhete aberto deve aparecer na resposta.
* Um bilhete encerrado não deve aparecer na resposta.
* Um bilhete cancelado não deve aparecer na resposta.
* Com vários bilhetes abertos, a resposta deve estar ordenada do mais novo para o mais antigo.

---

## 7. UC4 — Relatório diário

### 7.1 Endpoint

`GET /relatorios/diario?data=AAAA-MM-DD`

### 7.2 Entrada

O parâmetro `data` é obrigatório.

Deve obedecer ao formato `AAAA-MM-DD` e representar uma data válida.

### 7.3 Comportamento

O relatório deve considerar somente bilhetes encerrados na data solicitada.

Bilhetes abertos não devem ser contabilizados.

Bilhetes cancelados não devem ser contabilizados.

### 7.4 Resposta

Em caso de sucesso, deve retornar HTTP `200` contendo:

* `data`;
* `total_bilhetes`;
* `faturamento_centavos`;
* `tempo_medio_minutos`.

O faturamento deve ser representado em centavos inteiros.

A média de tempo deve considerar somente os bilhetes encerrados na data.

Quando o cálculo da média resultar exatamente em `0,5` de minuto, o valor deve ser arredondado para cima.

### 7.5 Erros

| Situação                 |  HTTP | Erro            |
| ------------------------ | ----: | --------------- |
| Data ausente ou inválida | `422` | `data_invalida` |

### 7.6 Critérios de aceitação

* Uma data válida deve produzir resposta `200`.
* Bilhetes encerrados na data consultada devem ser contabilizados.
* Bilhetes abertos não devem ser contabilizados.
* Bilhetes cancelados não devem ser contabilizados.
* Para uma data sem encerramentos, `total_bilhetes` e `faturamento_centavos` devem ser `0`.
* A média deve ser calculada somente sobre os bilhetes encerrados na data.
* Uma data em formato inválido deve produzir `422` com `data_invalida`.

---

## 8. UC5 — Cancelamento de bilhete

### 8.1 Endpoint

`POST /bilhetes/{id}/cancelamento`

### 8.2 Comportamento

Somente um bilhete com estado `aberto` pode ser cancelado.

Após o cancelamento, o estado deve ser `cancelado`.

O cancelamento não deve gerar cobrança.

O bilhete cancelado não deve possuir `saida` nem `valor_centavos`.

### 8.3 Resposta de sucesso

A operação deve retornar HTTP `200`.

A resposta deve identificar o bilhete e apresentar `status: "cancelado"`.

### 8.4 Erros

| Situação                |  HTTP | Erro                     |
| ----------------------- | ----: | ------------------------ |
| Bilhete inexistente     | `404` | `bilhete_nao_encontrado` |
| Bilhete não está aberto | `409` | `bilhete_nao_aberto`     |

### 8.5 Critérios de aceitação

* Um bilhete aberto deve poder ser cancelado com resposta `200`.
* Após o cancelamento, o status deve ser `cancelado`.
* Um bilhete cancelado não deve possuir `saida`.
* Um bilhete cancelado não deve possuir `valor_centavos`.
* Um bilhete cancelado não deve aparecer entre os bilhetes ativos.
* O cancelamento não deve gerar cobrança.
* Um bilhete encerrado não pode ser cancelado e deve produzir `409`.
* Um bilhete inexistente deve produzir `404`.

---

## 9. UC6 — Histórico de bilhetes por placa

### 9.1 Endpoint

`GET /bilhetes?placa=ABC1D23`

### 9.2 Entrada

O parâmetro `placa` é obrigatório e deve obedecer à regra de validação de placa.

### 9.3 Comportamento

A operação deve retornar todos os bilhetes associados à placa informada.

O histórico deve incluir bilhetes independentemente de seu estado:

* `aberto`;
* `encerrado`;
* `cancelado`.

Os resultados devem ser ordenados do mais novo para o mais antigo.

Quando a placa for válida, mas não possuir histórico, deve ser retornada uma lista vazia.

### 9.4 Erros

| Situação                  |  HTTP | Erro             |
| ------------------------- | ----: | ---------------- |
| Placa ausente ou inválida | `422` | `placa_invalida` |

### 9.5 Critérios de aceitação

* Uma placa válida sem histórico deve retornar `200` e lista vazia.
* Uma placa com histórico deve retornar todos os seus bilhetes.
* O histórico deve poder conter bilhetes abertos, encerrados e cancelados.
* Bilhetes de outras placas não podem aparecer.
* Os resultados devem estar ordenados do mais novo para o mais antigo.
* Uma placa inválida deve produzir `422` com `placa_invalida`.

---

## 10. UC7 — Tolerância de cobrança

### 10.1 Regra

A tolerância é definida pelo parâmetro `TOLERANCIA_MINUTOS`.

Para esta variante:

```text
TOLERANCIA_MINUTOS = 0
```

A regra geral é:

* duração menor ou igual à tolerância → cobrança de `0` centavos;
* se a duração ultrapassar a tolerância em 1 minuto, a cobrança deve começar desde o primeiro minuto;
* a tolerância não deve ser subtraída da duração cobrada.

### 10.2 Critérios de aceitação

* Com tolerância igual a `0`, uma duração positiva deve seguir normalmente as regras de cobrança por fração.
* A implementação deve utilizar o valor da variante para determinar a tolerância.
* Quando a tolerância for diferente de zero, uma duração exatamente igual à tolerância deve resultar em cobrança zero.
* Quando a duração exceder a tolerância em um minuto, a cobrança deve considerar a duração desde o primeiro minuto.

---

## 11. UC8 — Restrição de um bilhete aberto por placa

### 11.1 Regra

Uma placa pode possuir no máximo um bilhete com estado `aberto`.

Uma tentativa de abertura enquanto já existe bilhete aberto para a mesma placa deve ser rejeitada.

### 11.2 Erro

A tentativa de criar um segundo bilhete aberto para a mesma placa deve retornar:

```text
HTTP 409
erro: bilhete_em_aberto
```

### 11.3 Reabertura

Depois que o bilhete existente for:

* encerrado; ou
* cancelado;

uma nova abertura para a mesma placa deve ser permitida.

### 11.4 Critérios de aceitação

* A primeira abertura válida para uma placa deve ser aceita.
* A segunda abertura enquanto o primeiro bilhete estiver aberto deve retornar `409`.
* Depois do encerramento do primeiro bilhete, uma nova abertura deve ser aceita.
* Depois do cancelamento do primeiro bilhete, uma nova abertura deve ser aceita.
* A existência de um bilhete aberto para outra placa não deve impedir a abertura para a placa consultada.
* Quando a requisição for inválida, a validação de formato `422` deve prevalecer sobre o conflito `409`.

---

## 12. Regras de erro

Os erros devem utilizar os códigos e identificadores definidos pelo contrato:

| Situação                       |  HTTP | Código                   |
| ------------------------------ | ----: | ------------------------ |
| Placa inválida                 | `422` | `placa_invalida`         |
| Entrada inválida               | `422` | `entrada_invalida`       |
| Data inválida                  | `422` | `data_invalida`          |
| Bilhete inexistente            | `404` | `bilhete_nao_encontrado` |
| Bilhete já encerrado           | `409` | `bilhete_ja_encerrado`   |
| Bilhete não aberto             | `409` | `bilhete_nao_aberto`     |
| Placa já possui bilhete aberto | `409` | `bilhete_em_aberto`      |

A validação de formato deve ocorrer antes das regras de conflito de estado.

## 13. Requisitos não funcionais da implementação

* A API deve expor os endpoints exatamente nos caminhos definidos nesta especificação.
* A porta externa da variante é `8004`.
* A comunicação deve utilizar os formatos JSON definidos pelo contrato.
* Datas e horários devem preservar informação de timezone.
* Valores monetários devem utilizar inteiros em centavos.
* A implementação não deve utilizar ponto flutuante para cálculo monetário.
* As respostas devem respeitar os códigos HTTP definidos.
* O comportamento deve permanecer consistente com os parâmetros fornecidos pela variante.

## 14. Resultado esperado

A implementação gerada a partir desta especificação deve permitir que um consumidor da API execute todo o ciclo de vida de um bilhete:

```text
ABRIR
  ↓
ABERTO
  ├──→ ENCERRAR → ENCERRADO
  │
  └──→ CANCELAR → CANCELADO
```

Deve também permitir consultar os bilhetes ativos, consultar o histórico por placa e produzir o relatório diário dos bilhetes encerrados.
