# Casos de teste — Zona Azul Digital

## 1. Objetivo

Este documento define casos de teste para verificar as regras funcionais da API da Zona Azul Digital.

Os casos priorizam:

* validações de entrada;
* transições de estado;
* regras de cobrança;
* limites;
* ordenação;
* consistência dos relatórios;
* situações de conflito;
* comportamento quando não existem registros.

Os valores utilizados devem respeitar os parâmetros da variante:

| Parâmetro              |  Valor |
| ---------------------- | -----: |
| `TARIFA_HORA_CENTAVOS` |  `550` |
| `FRACAO_MINUTOS`       |   `15` |
| `TETO_DIARIO_CENTAVOS` | `8000` |
| `TOLERANCIA_MINUTOS`   |    `0` |

---

## 2. UC1 — Abertura de bilhete

### T01 — Criar bilhete com placa válida

**Dado:** não existe bilhete aberto para `ABC1D23`.

**Quando:** realizar `POST /bilhetes` com placa válida.

**Então:**

* HTTP `201`;
* bilhete criado;
* status `aberto`;
* placa igual à informada;
* resposta contém `id` e `entrada`.

### T02 — Placa com menos de 7 caracteres

**Dado:** placa `ABC123`.

**Quando:** realizar a abertura.

**Então:**

* HTTP `422`;
* erro `placa_invalida`;
* nenhum bilhete deve ser criado.

### T03 — Placa com mais de 7 caracteres

**Dado:** placa `ABC12345`.

**Quando:** realizar a abertura.

**Então:**

* HTTP `422`;
* erro `placa_invalida`;
* nenhum bilhete deve ser criado.

### T04 — Placa com caractere inválido

**Dado:** placa `ABC-123`.

**Quando:** realizar a abertura.

**Então:**

* HTTP `422`;
* erro `placa_invalida`.

### T05 — Placa contendo minúsculas

**Dado:** placa `ABC1d23`.

**Quando:** realizar a abertura.

**Então:**

* HTTP `422`;
* erro `placa_invalida`.

### T06 — Placa ausente

**Dado:** corpo sem o campo `placa`.

**Quando:** realizar a abertura.

**Então:**

* HTTP `422`;
* erro `placa_invalida`.

### T07 — Entrada opcional omitida

**Dado:** placa válida e nenhum valor para `entrada`.

**Quando:** realizar a abertura.

**Então:**

* HTTP `201`;
* `entrada` deve ser preenchida pelo instante atual;
* status `aberto`.

### T08 — Entrada válida informada

**Dado:** placa válida e entrada em ISO-8601 com timezone.

**Quando:** realizar a abertura.

**Então:**

* HTTP `201`;
* a entrada do bilhete deve corresponder ao valor informado;
* status `aberto`.

### T09 — Entrada sem timezone

**Dado:** entrada sem informação de timezone.

**Quando:** realizar a abertura.

**Então:**

* HTTP `422`;
* erro `entrada_invalida`;
* nenhum bilhete deve ser criado.

### T10 — Entrada em formato inválido

**Dado:** entrada que não representa uma data/hora ISO-8601 válida.

**Quando:** realizar a abertura.

**Então:**

* HTTP `422`;
* erro `entrada_invalida`.

---

## 3. UC2 — Encerramento e cobrança

### T11 — Encerrar bilhete aberto

**Dado:** existe um bilhete aberto.

**Quando:** realizar `POST /bilhetes/{id}/encerramento`.

**Então:**

* HTTP `200`;
* status do bilhete passa a ser encerrado;
* `saida` é registrada;
* `minutos` é calculado;
* `valor_centavos` é retornado como inteiro.

### T12 — Encerrar bilhete inexistente

**Dado:** identificador que não existe.

**Quando:** realizar o encerramento.

**Então:**

* HTTP `404`;
* erro `bilhete_nao_encontrado`.

### T13 — Encerrar bilhete já encerrado

**Dado:** bilhete com estado `encerrado`.

**Quando:** tentar encerrá-lo novamente.

**Então:**

* HTTP `409`;
* erro `bilhete_ja_encerrado`.

### T14 — Duração exatamente igual a uma fração

**Dado:** duração de exatamente 15 minutos.

**Quando:** encerrar o bilhete.

**Então:**

* a duração deve corresponder a uma única fração;
* a cobrança deve considerar uma fração.

### T15 — Um minuto além da fração

**Dado:** duração de 16 minutos.

**Quando:** encerrar o bilhete.

**Então:**

* a duração deve ser arredondada para duas frações;
* a cobrança deve considerar duas frações.

### T16 — Duração inferior a uma fração

**Dado:** duração positiva inferior a 15 minutos.

**Quando:** encerrar o bilhete.

**Então:**

* a duração deve ser arredondada para uma fração;
* a cobrança deve considerar uma fração, respeitando a regra de tolerância.

### T17 — Exatamente no teto diário

**Dado:** uma cobrança cujo resultado atinge exatamente `8000` centavos.

**Quando:** encerrar o bilhete.

**Então:**

* o valor retornado deve ser `8000`;
* o valor não deve ultrapassar o teto.

### T18 — Cobrança acima do teto

**Dado:** uma duração cuja cobrança calculada seria superior a `8000` centavos.

**Quando:** encerrar o bilhete.

**Então:**

* o valor final deve ser limitado a `8000` centavos.

### T19 — Valor monetário inteiro

**Dado:** qualquer encerramento com cobrança.

**Quando:** verificar a resposta.

**Então:**

* `valor_centavos` deve ser inteiro;
* não deve existir valor monetário em ponto flutuante.

### T20 — Arredondamento de fração não inteira em centavos

**Dado:** os parâmetros da variante produzem `550 / 4 = 137,5` centavos por fração.

**Quando:** for necessário calcular a cobrança.

**Então:**

* a implementação deve possuir uma regra determinística para transformar o cálculo em centavos inteiros;
* a regra utilizada deve ser documentada como decisão técnica;
* não deve depender de representação monetária em ponto flutuante.

---

## 4. UC3 — Bilhetes ativos

### T21 — Nenhum bilhete ativo

**Dado:** não existem bilhetes abertos.

**Quando:** realizar `GET /bilhetes/ativos`.

**Então:**

* HTTP `200`;
* resposta é uma lista vazia.

### T22 — Listar bilhete aberto

**Dado:** existe um bilhete aberto.

**Quando:** consultar os ativos.

**Então:**

* o bilhete aparece na lista;
* status é `aberto`.

### T23 — Bilhete encerrado não aparece

**Dado:** existe um bilhete encerrado.

**Quando:** consultar os ativos.

**Então:**

* o bilhete não aparece.

### T24 — Bilhete cancelado não aparece

**Dado:** existe um bilhete cancelado.

**Quando:** consultar os ativos.

**Então:**

* o bilhete não aparece.

### T25 — Ordenação dos ativos

**Dado:** existem vários bilhetes abertos criados em instantes diferentes.

**Quando:** consultar os ativos.

**Então:**

* o mais novo aparece primeiro;
* os demais aparecem em ordem decrescente de criação.

---

## 5. UC4 — Relatório diário

### T26 — Relatório de data válida

**Dado:** uma data válida.

**Quando:** consultar `/relatorios/diario?data=AAAA-MM-DD`.

**Então:**

* HTTP `200`;
* resposta contém `data`;
* contém `total_bilhetes`;
* contém `faturamento_centavos`;
* contém `tempo_medio_minutos`.

### T27 — Data inválida

**Dado:** data em formato inválido.

**Quando:** consultar o relatório.

**Então:**

* HTTP `422`;
* erro `data_invalida`.

### T28 — Data ausente

**Dado:** nenhuma data informada.

**Quando:** consultar o relatório.

**Então:**

* HTTP `422`;
* erro `data_invalida`.

### T29 — Bilhete aberto não entra no relatório

**Dado:** existe bilhete aberto na data consultada.

**Quando:** gerar o relatório.

**Então:**

* esse bilhete não aumenta `total_bilhetes`;
* não aumenta `faturamento_centavos`;
* não participa da média.

### T30 — Bilhete cancelado não entra no relatório

**Dado:** existe bilhete cancelado na data consultada.

**Quando:** gerar o relatório.

**Então:**

* não deve ser contabilizado;
* não deve gerar faturamento.

### T31 — Data sem encerramentos

**Dado:** nenhum bilhete foi encerrado na data.

**Quando:** gerar o relatório.

**Então:**

* `total_bilhetes = 0`;
* `faturamento_centavos = 0`;
* não deve haver duração de bilhete não encerrado sendo utilizada na média.

### T32 — Média de duração

**Dado:** existem múltiplos bilhetes encerrados na data.

**Quando:** gerar o relatório.

**Então:**

* a média deve considerar somente esses bilhetes;
* quando o resultado possuir exatamente `0,5` minuto, deve ser arredondado para cima.

---

## 6. UC5 — Cancelamento

### T33 — Cancelar bilhete aberto

**Dado:** existe bilhete aberto.

**Quando:** realizar `POST /bilhetes/{id}/cancelamento`.

**Então:**

* HTTP `200`;
* status passa a ser `cancelado`;
* não há cobrança.

### T34 — Cancelar bilhete inexistente

**Dado:** ID inexistente.

**Quando:** realizar o cancelamento.

**Então:**

* HTTP `404`;
* erro `bilhete_nao_encontrado`.

### T35 — Cancelar bilhete encerrado

**Dado:** bilhete encerrado.

**Quando:** tentar cancelá-lo.

**Então:**

* HTTP `409`;
* erro `bilhete_nao_aberto`.

### T36 — Cancelar bilhete já cancelado

**Dado:** bilhete cancelado.

**Quando:** tentar cancelá-lo novamente.

**Então:**

* HTTP `409`;
* erro `bilhete_nao_aberto`.

### T37 — Cancelamento não gera cobrança

**Dado:** bilhete aberto.

**Quando:** cancelá-lo.

**Então:**

* status é `cancelado`;
* não deve possuir `saida`;
* não deve possuir `valor_centavos`;
* não deve contribuir para faturamento.

---

## 7. UC6 — Histórico por placa

### T38 — Placa sem histórico

**Dado:** placa válida sem bilhetes.

**Quando:** realizar `GET /bilhetes?placa=ABC1D23`.

**Então:**

* HTTP `200`;
* resposta é `[]`.

### T39 — Consultar histórico

**Dado:** existem vários bilhetes para a placa.

**Quando:** consultar o histórico.

**Então:**

* todos os bilhetes da placa são retornados;
* os diferentes estados podem aparecer.

### T40 — Histórico não mistura placas

**Dado:** existem bilhetes para `ABC1D23` e `XYZ9K88`.

**Quando:** consultar `ABC1D23`.

**Então:**

* nenhum bilhete de `XYZ9K88` deve aparecer.

### T41 — Ordenação do histórico

**Dado:** existem vários bilhetes da mesma placa.

**Quando:** consultar o histórico.

**Então:**

* o bilhete mais novo aparece primeiro.

### T42 — Placa inválida

**Dado:** placa com formato inválido.

**Quando:** consultar o histórico.

**Então:**

* HTTP `422`;
* erro `placa_invalida`.

### T43 — Placa ausente

**Dado:** parâmetro `placa` ausente.

**Quando:** consultar o histórico.

**Então:**

* HTTP `422`;
* erro `placa_invalida`.

---

## 8. UC7 — Tolerância

### T44 — Duração dentro da tolerância

**Dado:** uma variante com tolerância positiva e duração menor que a tolerância.

**Quando:** encerrar o bilhete.

**Então:**

* `valor_centavos = 0`.

### T45 — Duração exatamente igual à tolerância

**Dado:** uma variante com tolerância positiva e duração exatamente igual à tolerância.

**Quando:** encerrar o bilhete.

**Então:**

* `valor_centavos = 0`.

### T46 — Um minuto além da tolerância

**Dado:** duração igual à tolerância mais um minuto.

**Quando:** encerrar o bilhete.

**Então:**

* a cobrança deve começar desde o primeiro minuto;
* a tolerância não deve ser descontada da duração cobrada.

### T47 — Variante atual sem tolerância

**Dado:** `TOLERANCIA_MINUTOS = 0`.

**Quando:** encerrar um bilhete com duração positiva.

**Então:**

* não existe período gratuito;
* a cobrança segue diretamente as regras de fração.

---

## 9. UC8 — Um bilhete aberto por placa

### T48 — Primeira abertura

**Dado:** placa sem bilhete aberto.

**Quando:** criar um bilhete.

**Então:**

* a abertura é aceita;
* o bilhete fica `aberto`.

### T49 — Segunda abertura simultânea

**Dado:** placa já possui bilhete aberto.

**Quando:** tentar criar outro.

**Então:**

* HTTP `409`;
* erro `bilhete_em_aberto`;
* o segundo bilhete não deve ser criado.

### T50 — Reabrir após encerramento

**Dado:** existe bilhete encerrado para a placa.

**Quando:** criar novo bilhete.

**Então:**

* a nova abertura deve ser aceita.

### T51 — Reabrir após cancelamento

**Dado:** existe bilhete cancelado para a placa.

**Quando:** criar novo bilhete.

**Então:**

* a nova abertura deve ser aceita.

### T52 — Placas diferentes

**Dado:** `ABC1D23` possui bilhete aberto.

**Quando:** criar bilhete para `XYZ9K88`.

**Então:**

* a criação para `XYZ9K88` deve ser aceita.

### T53 — Validação antes do conflito

**Dado:** existe bilhete aberto para a placa válida correspondente.

**Quando:** enviar uma requisição com placa em formato inválido.

**Então:**

* a resposta deve ser `422`;
* o erro deve ser de validação;
* o conflito `409` não deve substituir a validação.

---

## 10. Transições de estado

### T54 — Ciclo de encerramento

**Dado:** bilhete aberto.

**Quando:**

1. criar o bilhete;
2. encerrá-lo;
3. tentar encerrá-lo novamente.

**Então:**

* primeira transição: `aberto → encerrado`;
* segunda tentativa: `409 bilhete_ja_encerrado`.

### T55 — Ciclo de cancelamento

**Dado:** bilhete aberto.

**Quando:**

1. criar o bilhete;
2. cancelá-lo;
3. tentar cancelá-lo novamente.

**Então:**

* primeira transição: `aberto → cancelado`;
* segunda tentativa: `409 bilhete_nao_aberto`.

### T56 — Reabertura após finalização

**Dado:** uma placa possui um bilhete que já foi encerrado ou cancelado.

**Quando:** criar outro bilhete para a mesma placa.

**Então:**

* a nova abertura deve ser permitida;
* o bilhete anterior deve permanecer com seu estado final.

---

## 11. Consistência entre operações

### T57 — Cancelamento remove bilhete dos ativos

**Dado:** bilhete aberto.

**Quando:**

1. consultar ativos;
2. cancelar o bilhete;
3. consultar ativos novamente.

**Então:**

* o bilhete aparece antes do cancelamento;
* não aparece depois do cancelamento.

### T58 — Encerramento remove bilhete dos ativos

**Dado:** bilhete aberto.

**Quando:**

1. consultar ativos;
2. encerrar o bilhete;
3. consultar ativos novamente.

**Então:**

* o bilhete aparece antes do encerramento;
* não aparece depois do encerramento.

### T59 — Encerramento gera histórico

**Dado:** bilhete aberto para uma placa.

**Quando:**

1. encerrá-lo;
2. consultar o histórico da placa.

**Então:**

* o bilhete deve permanecer no histórico;
* seu estado deve ser `encerrado`.

### T60 — Cancelamento permanece no histórico

**Dado:** bilhete aberto para uma placa.

**Quando:**

1. cancelá-lo;
2. consultar o histórico da placa.

**Então:**

* o bilhete deve permanecer no histórico;
* seu estado deve ser `cancelado`.

### T61 — Cancelamento não gera relatório

**Dado:** bilhete cancelado em determinada data.

**Quando:** consultar o relatório dessa data.

**Então:**

* o bilhete não deve aumentar o total;
* não deve aumentar o faturamento;
* não deve participar da média.

---

## 12. Precedência de validação

### T62 — Placa inválida e placa já ocupada

**Dado:** existe um bilhete aberto associado à placa válida correspondente.

**Quando:** enviar uma requisição cuja placa fornecida seja inválida.

**Então:**

* deve prevalecer `422 placa_invalida`;
* não deve ser retornado `409 bilhete_em_aberto`.

### T63 — Dados inválidos não alteram estado

**Dado:** uma requisição contém dados inválidos.

**Quando:** enviar a requisição.

**Então:**

* a operação deve ser rejeitada;
* nenhum novo bilhete deve ser criado;
* nenhum bilhete existente deve mudar de estado.

---

## 13. Resumo de cobertura

Os testes cobrem:

* validação de placas;
* validação de datas e horários;
* criação de bilhetes;
* encerramento;
* cancelamento;
* transições de estado;
* unicidade de bilhete aberto;
* cobrança por fração;
* teto diário;
* tolerância;
* valores monetários em centavos;
* consultas de ativos;
* histórico;
* ordenação;
* relatório diário;
* média de duração;
* códigos HTTP;
* códigos de erro;
* precedência entre validação e regra de negócio;
* consistência entre diferentes operações da API.
