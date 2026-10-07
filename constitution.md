# Constituição do Projeto — Zona Azul Digital

## 1. Objetivo

Estabelecer regras operacionais para garantir que a implementação da Zona Azul Digital permaneça fiel ao contrato da API, às regras de negócio e aos critérios de qualidade definidos para o projeto.

## 2. Regras

### Regra 1 — O contrato é a fonte de verdade

A implementação deve obedecer ao `contrato.json` e às regras funcionais especificadas em `spec.md`.

Exemplos ilustrativos ou comportamentos não definidos explicitamente no contrato não devem substituir uma regra contratual.

### Regra 2 — Dinheiro deve ser representado em centavos inteiros

Valores monetários devem ser representados exclusivamente como números inteiros em centavos.

Não deve haver dependência de ponto flutuante para representar ou comparar valores monetários.

### Regra 3 — Validação precede regra de negócio

Uma entrada estruturalmente inválida deve ser rejeitada antes da avaliação de conflitos de estado ou outras regras de negócio.

Por exemplo, uma placa inválida deve produzir `422 placa_invalida` mesmo que já exista um bilhete aberto associado a uma tentativa semanticamente equivalente.

### Regra 4 — Estados do bilhete devem ser consistentes

Um bilhete aberto pode ser encerrado ou cancelado.

Depois de encerrado ou cancelado, o bilhete não pode retornar ao estado aberto nem sofrer uma segunda transição final.

### Regra 5 — Regras devem ser determinísticas

Para os mesmos dados de entrada e os mesmos parâmetros da variante, a API deve produzir o mesmo resultado.

Regras de cobrança, arredondamento, tolerância, teto diário e cálculo de relatórios devem ser determinísticas e explicitamente documentadas quando definidas pelo contrato ou por material oficial.

Quando existir uma ambiguidade não resolvida pelo contrato, ela deve ser registrada como decisão técnica, sem ser apresentada como se fosse uma regra contratual.

### Regra 6 — Especificação não é implementação

Os documentos deste projeto devem descrever requisitos, comportamento esperado, decisões técnicas, testes e tarefas.

A implementação concreta deve ser gerada posteriormente a partir dessas especificações, sem transformar os arquivos `.md` em código-fonte da solução.

## 3. Critério de conformidade

Uma implementação será considerada conforme quando respeitar as regras desta constituição, o contrato da API e os critérios de aceitação definidos em `spec.md`.
