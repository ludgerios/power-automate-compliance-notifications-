# Power Automate Compliance Notifications

Projeto de laboratório desenvolvido no **Microsoft Power Automate** para acompanhar datas de testes de controles, enviar notificações pelo Microsoft Teams e registrar os envios em uma base de log.

A solução é composta por dois fluxos:

- **Fluxo pai:** envia notificações individuais aos responsáveis pelos controles.
- **Fluxo filho:** enviará uma notificação consolidada aos gerentes quando houver controles atrasados em `-1`, `-5` ou `-7` dias.

> O repositório utiliza somente dados fictícios e conexões de laboratório.

---

## Arquitetura da solução

```text
Fluxo pai
Notificação individual ao responsável
        ↓
Finaliza o processamento dos controles
        ↓
Executa o fluxo filho
        ↓
Fluxo filho
Consolida os controles atrasados por gerente
        ↓
Envia um card para cada gerente
```

Atualmente, este README documenta principalmente o **fluxo pai**. O fluxo filho será adicionado ao repositório após a conclusão da adaptação para o ambiente de laboratório.

---

# Fluxo pai: notificação ao responsável

## Objetivo

Ler os controles cadastrados em uma planilha Excel, identificar testes próximos do vencimento ou atrasados e enviar um **Adaptive Card individual** ao responsável pelo controle.

Após o envio, o fluxo registra os dados da notificação no log.

## Fonte de dados de laboratório

- **Arquivo:** `Compliance_Control_Monitoring_Lab.xlsx`
- **Tabela configurada no Excel:** `Tabela1`
- **Armazenamento:** OneDrive for Business de laboratório

### Colunas utilizadas

| Coluna | Finalidade |
|---|---|
| `ControlID` | Código único do controle |
| `ControlOwner` | Nome do responsável |
| `OwnerEmail` | E-mail do responsável |
| `ManagerName` | Nome do gerente |
| `ManagerEmail` | E-mail do gerente |
| `ControlDescription` | Descrição do controle |
| `ControlStatus` | Situação operacional do controle |
| `NextTestDate` | Data do próximo teste em número serial do Excel |
| `TestStatus` | Situação do teste |
| `LastTestDate` | Data do último teste |

---

## Regras do fluxo pai

Um registro somente continua no fluxo quando atende às seguintes regras:

```text
ControlStatus = Work
E
TestStatus = ON TIME ou LATE
E
NextTestDate válida
E
DiasParaVencer = 7, 5, 1, -1, -5 ou -7
```

### Validação do ControlStatus

A comparação normaliza letras maiúsculas, minúsculas e espaços:

```text
toUpper(trim(string(item()?['ControlStatus']))) = WORK
```

São aceitos valores como:

```text
Work
WORK
work
```

São ignorados valores como:

```text
Not Work
Disabled
Implementation Required
```

### Validação do TestStatus

O fluxo pai aceita:

```text
ON TIME
LATE
```

Registros com `CONCLUDED` ou campo vazio são ignorados.

### Validação da data

A coluna `NextTestDate` deve conter um número serial válido do Excel:

```text
isInt(string(item()?['NextTestDate']))
```

Essa validação impede que valores vazios, textos ou `#N/A` sejam enviados ao cálculo.

### Cálculo dos dias

```text
DiasParaVencer = NextTestDate - data atual
```

Expressão utilizada:

```text
sub(
  int(outputs('Obter_Data_para_Cálculo')),
  int(
    div(
      sub(
        ticks(utcNow()),
        ticks('1899-12-30')
      ),
      864000000000
    )
  )
)
```

### Marcos de notificação

| Resultado | Comportamento |
|---:|---|
| `7` | Notifica faltando 7 dias |
| `5` | Notifica faltando 5 dias |
| `1` | Notifica faltando 1 dia |
| `-1` | Notifica com 1 dia de atraso |
| `-5` | Notifica com 5 dias de atraso |
| `-7` | Notifica com 7 dias de atraso |

Qualquer outro valor é ignorado.

### Destinatário

O fluxo pai envia somente para o responsável:

```text
OwnerEmail
```

O gerente não é notificado pelo fluxo pai.

---

## Funcionamento do fluxo pai

### 1. Inicialização e leitura da tabela

O fluxo:

1. é iniciado manualmente;
2. define o ambiente de execução;
3. define o destinatário utilizado nos testes;
4. lista as linhas da tabela `Tabela1`;
5. percorre cada registro retornado pelo Excel.

![Inicialização do fluxo responsável](fotos%20da%20estrutura%20do%20fluxo/inicializacao-fluxo-responsavel.png)

### 2. Validação da data e cálculo

Para cada registro, o fluxo:

1. valida `TestStatus` e `ControlStatus`;
2. valida se `NextTestDate` é um número inteiro;
3. captura a data do próximo teste;
4. prepara o valor para o cálculo;
5. calcula `DiasParaVencer`.

Registros inválidos seguem pelo ramo falso e são ignorados sem interromper a execução.

![Validação da data do fluxo responsável](fotos%20da%20estrutura%20do%20fluxo/validacao-data-fluxo-responsavel.png)

### 3. Preparação, envio e log

Quando `DiasParaVencer` corresponde a `7`, `5`, `1`, `-1`, `-5` ou `-7`, o fluxo:

1. captura `ControlOwner`;
2. monta a descrição do controle;
3. formata `NextTestDate` como `dd/MM/yyyy`;
4. prepara a quantidade de dias;
5. define o destinatário com `OwnerEmail`;
6. envia o Adaptive Card pelo Microsoft Teams;
7. registra a notificação no log.

![Envio da notificação do fluxo responsável](fotos%20da%20estrutura%20do%20fluxo/envio-notificacao-fluxo-responsavel.png)

### Estrutura completa

![Estrutura completa do fluxo pai](fotos%20da%20estrutura%20do%20fluxo/estrutura-completa.png)

---

## Adaptive Card do responsável

O card apresenta:

- identidade visual do projeto Ludgeriios;
- nome do responsável;
- identificação e descrição do controle;
- data do próximo teste;
- quantidade de dias;
- status do teste;
- botão para abrir a página demonstrativa do projeto.

O template está disponível em:

```text
adaptive-cards/responsible-notification.json
```

A identidade visual está disponível em:

```text
card-icons/ludgeriios-logo.png
```

---

## Log de notificações

O fluxo registra os envios em uma tabela de log com os seguintes campos:

| Campo | Conteúdo |
|---|---|
| `SentAt` | Data e hora do envio |
| `ControlID` | Código do controle |
| `TestDate` | Data prevista do teste |
| `Days` | Dias restantes ou em atraso |
| `Recipient` | Destinatário utilizado |
| `Status` | Resultado do envio |
| `NotificationType` | Tipo da notificação |

Para o fluxo pai:

```text
NotificationType = Responsible
```

---

# Fluxo filho: notificação ao gerente

## Objetivo

O fluxo filho será responsável por consolidar os controles atrasados e enviar **um único card para cada gerente**.

Cada gerente deverá receber somente os controles relacionados ao próprio `ManagerEmail`.

## Regras do fluxo filho

```text
ControlStatus = Work
E
TestStatus = LATE
E
ManagerEmail preenchido
E
NextTestDate válida
E
DiasParaVencer = -1, -5 ou -7
```

## Conteúdo esperado no card

```text
Nº Controle | Responsável | Data do teste | Dias em atraso
```

## Comportamento esperado

- Um card por gerente em cada execução.
- Vários controles podem aparecer no mesmo card.
- Controles de gerentes diferentes não podem ser misturados.
- O envio deve ocorrer dentro do loop de gerentes.
- O envio não deve ocorrer dentro do loop que analisa os controles.

## Estado atual

O fluxo filho será enviado ao repositório depois da conclusão da adaptação para a base de laboratório.

Quando o fluxo filho for incluído, este README será atualizado com:

- estrutura do fluxo;
- regras e expressões;
- template do card consolidado;
- prints do processamento;
- configuração da integração entre pai e filho.

---

## Integração entre os fluxos

Os dois fluxos devem estar na mesma solução do Power Automate.

A chamada do fluxo filho deve ficar depois do loop do fluxo pai:

```text
Processar todos os controles no fluxo pai
        ↓
Finalizar o loop
        ↓
Executar o fluxo filho uma única vez
```

A chamada não deve ficar dentro do loop de controles, pois isso executaria o fluxo filho várias vezes.

---

## Estrutura do repositório

```text
power-automate-compliance-notifications/
├── README.md
├── adaptive-cards/
│   ├── README.md
│   ├── responsible-notification.json
│   └── manager-summary.json
├── card-icons/
│   └── ludgeriios-logo.png
├── fotos da estrutura do fluxo/
│   ├── inicializacao-fluxo-responsavel.png
│   ├── validacao-data-fluxo-responsavel.png
│   ├── envio-notificacao-fluxo-responsavel.png
│   └── estrutura-completa.png
├── sample-data/
│   └── Compliance_Control_Monitoring_Lab.xlsx
└── solution/
```

---

## Como executar em laboratório

1. Importe o fluxo pai como um novo fluxo no Power Automate.
2. Configure conexões próprias para Excel Online e Microsoft Teams.
3. Aponte a ação de leitura para `Compliance_Control_Monitoring_Lab.xlsx`.
4. Selecione a tabela `Tabela1`.
5. Mantenha um destinatário autorizado e fixo durante os testes.
6. Execute o fluxo manualmente.
7. Confira os cards recebidos e os registros no log.
8. Conecte o fluxo filho somente depois de validar o envio consolidado aos gerentes.

---

## Segurança

Este projeto deve utilizar somente dados fictícios.

Antes de tornar o repositório público, confirme que não existem:

- nomes ou e-mails reais;
- nomes de empresas;
- URLs internas;
- nomes de sites corporativos;
- contas de serviço;
- IDs de conexões, arquivos ou ambientes;
- históricos reais de notificações.
