# Plano técnico — Zona Azul Digital

## 1. Objetivo

Este plano descreve as principais decisões técnicas necessárias para implementar a API da Zona Azul Digital a partir do contrato e da especificação funcional.

As decisões devem preservar o contrato da API, os parâmetros da variante e as regras de negócio descritas em `spec.md`.

---

## 2. Modelagem do bilhete

O sistema deve representar um bilhete com, no mínimo, os seguintes conceitos:

* identificador;
* placa;
* entrada;
* saída, quando houver;
* duração em minutos, quando houver encerramento;
* valor em centavos, quando houver cobrança;
* estado.

O estado deve representar explicitamente o ciclo de vida do bilhete:

```text
ABERTO
  ├──→ ENCERRADO
  └──→ CANCELADO
```

### Justificativa

A representação explícita do estado facilita garantir as restrições de negócio:

* somente bilhetes abertos podem ser encerrados;
* somente bilhetes abertos podem ser cancelados;
* bilhetes encerrados ou cancelados não podem voltar a ser abertos;
* bilhetes abertos são os únicos considerados na consulta de ativos.

---

## 3. Validação antes das regras de negócio

A validação estrutural dos dados deve ocorrer antes da aplicação das regras de negócio.

Por exemplo, uma requisição com placa inválida deve produzir `422 placa_invalida`, mesmo que exista uma situação que poderia produzir `409 bilhete_em_aberto`.

### Justificativa

Essa ordem é explicitamente definida pelo contrato e evita que uma entrada estruturalmente inválida seja interpretada como uma operação válida que apenas entrou em conflito com o estado atual.

---

## 4. Representação monetária

Valores monetários devem ser representados exclusivamente como inteiros em centavos.

A API deve utilizar o campo `valor_centavos` para representar os valores monetários.

Não deve ser utilizado ponto flutuante para armazenar ou calcular valores monetários.

### Justificativa

O contrato define que os valores monetários são expressos em centavos inteiros e proíbe a utilização de ponto flutuante para esse cálculo.

Essa decisão também evita problemas de precisão associados à representação de valores monetários.

---

## 5. Cálculo por frações

A duração deve ser calculada em minutos e convertida em quantidade de frações utilizando `FRACAO_MINUTOS`.

Para a variante atual:

```text
TARIFA_HORA_CENTAVOS = 550
FRACAO_MINUTOS = 15
```

Uma hora contém:

```text
60 / 15 = 4 frações
```

O contrato define que a tarifa da fração é obtida a partir da tarifa horária dividida pela quantidade de frações da hora.

Isso resulta em:

```text
550 / 4 = 137,5 centavos
```

### Decisão sobre a representação do resultado

O contrato exige simultaneamente:

1. cálculo da tarifa por fração;
2. valores monetários inteiros em centavos;
3. ausência de ponto flutuante.

Entretanto, os materiais fornecidos não especificam explicitamente qual regra de arredondamento deve ser utilizada quando a tarifa por fração não resultar em um número inteiro de centavos.

Portanto, a implementação não deve escolher silenciosamente uma regra arbitrária.

A regra efetivamente adotada deve ser determinística, documentada e consistente em todos os cálculos. Caso exista orientação adicional do professor ou material oficial durante a prova, essa orientação deve prevalecer.

### Justificativa

Registrar explicitamente a ambiguidade evita que uma decisão de implementação seja apresentada como se fosse uma regra definida pelo contrato.

---

## 6. Arredondamento da duração

A duração deve ser convertida para frações cobradas arredondando para cima.

Com `FRACAO_MINUTOS = 15`:

```text
1–15 minutos  → 1 fração
16–30 minutos → 2 frações
31–45 minutos → 3 frações
46–60 minutos → 4 frações
```

### Justificativa

O contrato determina que a cobrança deve arredondar a duração para cima em relação à fração. Exatamente uma fração deve gerar uma fração cobrada, enquanto um minuto adicional deve avançar para a próxima fração.

---

## 7. Teto diário

A cobrança de um bilhete deve respeitar:

```text
TETO_DIARIO_CENTAVOS = 8000
```

O valor final não deve ultrapassar esse limite.

### Justificativa

O teto é um parâmetro da variante e faz parte da regra de cobrança definida pelo contrato.

A regra deve ser aplicada de maneira determinística antes da apresentação do valor final ao consumidor da API.

---

## 8. Tolerância

A implementação deve utilizar o parâmetro:

```text
TOLERANCIA_MINUTOS = 0
```

A regra geral definida pelo contrato é:

* duração menor ou igual à tolerância → cobrança zero;
* duração superior à tolerância → cobrança desde o primeiro minuto;
* a tolerância não deve ser descontada da duração cobrada.

### Justificativa

A utilização do parâmetro permite que a mesma especificação represente corretamente a variante atual e preserve a regra geral definida pelo contrato.

---

## 9. Unicidade de bilhete aberto

A existência de um bilhete aberto para uma placa deve ser verificada antes da criação de outro bilhete aberto para a mesma placa.

A restrição deve permitir:

```text
placa sem bilhete aberto → criar
placa com bilhete aberto → rejeitar com 409
placa após encerramento   → criar novamente
placa após cancelamento   → criar novamente
```

### Justificativa

Essa abordagem representa diretamente a regra de negócio de no máximo um bilhete aberto por placa.

---

## 10. Consultas e ordenação

As consultas de bilhetes devem possuir ordenação explícita quando definida pelo contrato.

As seguintes operações devem retornar os registros mais novos primeiro:

* `GET /bilhetes/ativos`;
* `GET /bilhetes?placa=...`.

### Justificativa

A ordenação faz parte do comportamento funcional da API e não deve depender da ordem incidental de armazenamento dos registros.

---

## 11. Relatório diário

O relatório diário deve considerar somente bilhetes encerrados na data consultada.

Bilhetes:

* abertos;
* cancelados;

não devem contribuir para o total de bilhetes, faturamento ou média de duração.

A média deve ser calculada sobre os bilhetes encerrados na data e deve respeitar a regra de arredondamento definida pelo contrato.

### Justificativa

Separar explicitamente os estados evita contabilizar operações que ainda não produziram uma cobrança de encerramento ou que foram canceladas.

---

## 12. Datas e timezone

Datas e horários de entrada e saída devem preservar a informação de timezone.

A API deve utilizar o formato ISO-8601 quando representar esses valores.

A variante e o contrato utilizam o timezone `-03:00`.

### Justificativa

Preservar o offset evita ambiguidades na interpretação dos instantes utilizados para calcular duração e determinar a data dos encerramentos no relatório diário.

---

## 13. Erros

Os erros devem utilizar os códigos HTTP e identificadores definidos no contrato:

| Situação                              |  HTTP | Código                   |
| ------------------------------------- | ----: | ------------------------ |
| Placa inválida                        | `422` | `placa_invalida`         |
| Entrada inválida                      | `422` | `entrada_invalida`       |
| Data inválida                         | `422` | `data_invalida`          |
| Bilhete inexistente                   | `404` | `bilhete_nao_encontrado` |
| Bilhete já encerrado                  | `409` | `bilhete_ja_encerrado`   |
| Bilhete não aberto                    | `409` | `bilhete_nao_aberto`     |
| Bilhete aberto existente para a placa | `409` | `bilhete_em_aberto`      |

### Justificativa

Centralizar os códigos de erro reduz inconsistências entre endpoints e mantém o comportamento da implementação alinhado ao contrato.

---

## 14. Separação entre especificação e implementação

Os documentos desta entrega devem descrever comportamento, decisões, critérios e testes, sem implementar a API.

A implementação concreta deve ser produzida posteriormente a partir dos requisitos especificados.

### Justificativa

A Track SDD avalia a capacidade de produzir uma especificação suficientemente clara para orientar a geração da implementação. A especificação deve permanecer independente da tecnologia concreta utilizada para implementar a API.

---

## 15. Estratégia de validação

A implementação deve ser validada em camadas:

1. validação dos dados de entrada;
2. validação das transições de estado;
3. cálculo de duração e cobrança;
4. validação das consultas e ordenações;
5. validação dos relatórios;
6. verificação dos códigos HTTP e erros;
7. execução dos casos de teste especificados em `tests.md`.

### Justificativa

A ordem permite identificar primeiro erros de entrada, depois regras de domínio e finalmente comportamentos derivados das operações realizadas.

---

## 16. Critério de conclusão

A implementação será considerada aderente à especificação quando:

* todos os endpoints definidos estiverem disponíveis;
* os códigos HTTP e erros do contrato forem respeitados;
* as transições de estado forem válidas;
* as regras de cobrança forem aplicadas;
* os valores monetários forem representados em centavos inteiros;
* as consultas respeitarem os critérios de ordenação;
* o relatório considerar somente os encerramentos pertinentes;
* os casos de teste definidos em `tests.md` forem atendidos;
* nenhuma regra definida pelo contrato for substituída por comportamento implícito ou arbitrário.
