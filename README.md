# Power Automate Compliance Notifications

Automação de laboratório desenvolvida no **Microsoft Power Automate** para monitorar datas de testes de controles, enviar notificações pelo Microsoft Teams e registrar os envios em uma tabela de log.

A solução é composta por dois fluxos:

- **Fluxo pai:** envia um Adaptive Card individual ao responsável.
- **Fluxo filho:** consolida controles atrasados e envia um Adaptive Card para cada gerente.

> O repositório utiliza dados fictícios. As conexões do Excel Online e Microsoft Teams devem ser configuradas pelo usuário durante a importação.

## Arquitetura

```text
Fluxo pai
Notificação individual ao responsável
        ↓
Finaliza o processamento da base
        ↓
Executa o fluxo filho uma única vez
        ↓
Fluxo filho
Agrupa controles atrasados por gerente
        ↓
Envia um card consolidado por gerente
```

---

# Fluxo pai

## LAB-COMPLIANCE-NOTIFICACAO-RESPONSAVEL

O fluxo pai consulta a base de laboratório, valida os controles elegíveis e envia notificações individuais aos responsáveis.

## Base de laboratório

- **Arquivo:** `Compliance_Control_Monitoring_Lab.xlsx`
- **Tabela de controles:** `Tabela1`
- **Tabela de log:** `Tabela2`
- **Armazenamento:** OneDrive for Business

### Colunas utilizadas

| Coluna | Utilização |
|---|---|
| `ControlID` | Código do controle |
| `ControlOwner` | Nome do responsável |
| `OwnerEmail` | E-mail demonstrativo do responsável |
| `ControlDescription` | Descrição do controle |
| `ControlStatus` | Situação operacional do controle |
| `NextTestDate` | Data do próximo teste em número serial do Excel |
| `TestStatus` | Situação do teste |
| `LastTestDate` | Data do último teste |
| `ManagerName` | Nome demonstrativo do gerente |
| `ManagerEmail` | E-mail demonstrativo do gerente, utilizado pelo fluxo filho |

## Regras de processamento

O controle somente avança quando todas as condições abaixo são atendidas:

```text
ControlStatus = Work
E
TestStatus = ON TIME ou LATE
E
NextTestDate contém um número serial válido
E
DiasParaVencer = 7, 5, 1, -1, -5 ou -7
```

A validação de `ControlStatus` normaliza letras e espaços:

```text
toUpper(trim(string(item()?['ControlStatus']))) = WORK
```

A validação aceita `Work`, `WORK` e `work`, mas rejeita `Not Work`, `Disabled` e `Implementation Required`.

A data é validada antes do cálculo:

```text
isInt(string(item()?['NextTestDate']))
```

Valores vazios, textos e `#N/A` são ignorados sem interromper a execução.

## Cálculo dos dias

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

Outros resultados não geram notificação.

## Destinatário de teste

O gatilho manual solicita o campo:

```text
DestinatarioTeste
```

O valor é armazenado em `varDestinatario` e utilizado no envio pelo Teams e no registro do log. O endereço informado deve representar um usuário válido no mesmo tenant da conexão do Microsoft Teams.

## Funcionamento técnico

### 1. Inicialização e leitura da base

O fluxo recebe o destinatário de teste, inicializa as variáveis, lê a `Tabela1` e percorre cada controle retornado pelo Excel.

<p align="center">
  <img src="fotos%20da%20estrutura%20do%20fluxo/inicializacao-fluxo-responsavel.png" alt="Inicialização e leitura da base" width="900">
</p>

<p align="center"><em>Inicialização das variáveis e leitura da tabela de controles.</em></p>

### 2. Validação e cálculo

Para cada controle, o fluxo valida `ControlStatus`, `TestStatus` e `NextTestDate`. Em seguida, calcula `DiasParaVencer`.

<p align="center">
  <img src="fotos%20da%20estrutura%20do%20fluxo/validacao-data-fluxo-responsavel.png" alt="Validação da data e cálculo dos dias" width="900">
</p>

<p align="center"><em>Validação da elegibilidade, tratamento da data serial e cálculo dos dias.</em></p>

### 3. Preparação, envio e log

Quando o resultado corresponde a um marco de notificação, o fluxo prepara os dados, envia o Adaptive Card e registra o envio na `Tabela2`.

<p align="center">
  <img src="fotos%20da%20estrutura%20do%20fluxo/envio-notificacao-fluxo-responsavel.png" alt="Preparação, envio e registro no log" width="900">
</p>

<p align="center"><em>Validação da janela, envio pelo Teams e registro da notificação.</em></p>

### 4. Estrutura completa

<p align="center">
  <img src="fotos%20da%20estrutura%20do%20fluxo/estrutura-completa.png" alt="Estrutura completa do fluxo pai" width="900">
</p>

<p align="center"><em>Visão completa do fluxo pai.</em></p>

## Adaptive Card

O card apresenta:

- identidade visual do projeto;
- nome do responsável;
- código e descrição do controle;
- data do próximo teste;
- status do teste;
- dias restantes ou dias em atraso;
- botão para acessar o projeto.

<p align="center">
  <img src="https://github.com/ludgerios/power-automate-compliance-notifications-/blob/main/card-icons/card.png">
</p>

<p align="center"><em>Exemplo do Adaptive Card enviado pelo Microsoft Teams.</em></p>

Arquivos relacionados:

```text
adaptive-cards/responsible-notification.json
card-icons/ludgeriios-logo.png
```

## Log de notificações

O envio é registrado na `Tabela2` com os seguintes campos:

| Campo | Conteúdo |
|---|---|
| `Data Envio` | Data e hora do envio |
| `Controle` | Código do controle |
| `Data do Teste` | Data prevista para o teste |
| `Qtd de Dias` | Dias restantes ou em atraso |
| `Email` | Destinatário informado no gatilho |
| `Status` | Resultado do envio |
| `tipo` | `Responsável` |

## Pacote importável

O pacote do fluxo pai está disponível em:

```text
fluxos/fluxo-pai/
```

Após importar o pacote como um novo fluxo:

1. selecione conexões próprias para Excel Online e Microsoft Teams;
2. configure `Compliance_Control_Monitoring_Lab.xlsx`;
3. selecione `Tabela1` na ação de leitura;
4. selecione `Tabela2` na ação de log;
5. informe um usuário válido em `DestinatarioTeste`.

---

# Fluxo filho

## LAB-COMPLIANCE-NOTIFICACAO-GERENTE

O fluxo filho agrupa controles atrasados por `ManagerEmail` e envia um único Adaptive Card consolidado para cada gerente. É chamado uma única vez pelo fluxo pai, após a conclusão do loop de processamento.

## Base de laboratório

- **Arquivo:** `Compliance_Control_Monitoring_Lab.xlsx`
- **Tabela de controles:** `Tabela1`
- **Tabela de log:** `Tabela2`
- **Armazenamento:** OneDrive for Business

### Colunas utilizadas

| Coluna | Utilização |
|---|---|
| `ControlID` | Código do controle |
| `ControlOwner` | Nome do responsável |
| `ControlDescription` | Descrição do controle |
| `ControlStatus` | Situação operacional do controle |
| `NextTestDate` | Data do próximo teste em número serial do Excel |
| `TestStatus` | Situação do teste |
| `ManagerName` | Nome demonstrativo do gerente |
| `ManagerEmail` | E-mail demonstrativo do gerente, destinatário do card consolidado |

## Regras de processamento

O controle somente é incluído na notificação ao gerente quando todas as condições abaixo são atendidas:

```text
ControlStatus = Work
E
TestStatus = LATE
E
ManagerEmail preenchido
E
NextTestDate contém um número serial válido
E
DiasParaVencer = -1, -5 ou -7
```

### Marcos de notificação

| Resultado | Comportamento |
|---:|---|
| `-1` | Inclui o controle no card com 1 dia de atraso |
| `-5` | Inclui o controle no card com 5 dias de atraso |
| `-7` | Inclui o controle no card com 7 dias de atraso |

Outros resultados não geram notificação ao gerente.

## Comportamento de agrupamento

- O fluxo agrupa todos os controles elegíveis pelo campo `ManagerEmail`.
- Cada gerente recebe um único card por execução, contendo apenas os controles associados ao seu e-mail.
- O card consolidado apresenta uma linha por controle no seguinte formato:

```text
Nº Controle | Responsável | Data do teste | Dias em atraso
```

## Funcionamento técnico

> O pacote, o Adaptive Card e as imagens do fluxo filho serão adicionados após a conclusão do desenvolvimento.

## Log de notificações

O envio é registrado na `Tabela2` com os seguintes campos:

| Campo | Conteúdo |
|---|---|
| `Data Envio` | Data e hora do envio |
| `Controle` | Códigos dos controles incluídos no card |
| `Data do Teste` | Datas previstas para os testes |
| `Qtd de Dias` | Dias em atraso por controle |
| `Email` | E-mail do gerente destinatário |
| `Status` | Resultado do envio |
| `tipo` | `Gerente` |

## Pacote importável

O pacote do fluxo filho estará disponível em:

```text
fluxos/fluxo-filho/
```

Após importar o pacote como um novo fluxo:

1. selecione conexões próprias para Excel Online e Microsoft Teams;
2. configure `Compliance_Control_Monitoring_Lab.xlsx`;
3. selecione `Tabela1` na ação de leitura;
4. selecione `Tabela2` na ação de log.

---

# Estrutura do repositório

```text
power-automate-compliance-notifications/
├── README.md
├── adaptive-cards/
│   ├── responsible-notification.json
│   └── manager-notification.json
├── card-icons/
│   └── ludgeriios-logo.png
├── fotos da estrutura do fluxo/
│   ├── inicializacao-fluxo-responsavel.png
│   ├── validacao-data-fluxo-responsavel.png
│   ├── envio-notificacao-fluxo-responsavel.png
│   ├── estrutura-completa.png
│   └── previa-card-responsavel.png
├── fluxos/
│   ├── fluxo-pai/
│   │   ├── README.md
│   │   └── LAB-COMPLIANCE-NOTIFICACAO-RESPONSAVEL.zip
│   └── fluxo-filho/
│       ├── README.md
│       └── LAB-COMPLIANCE-NOTIFICACAO-GERENTE.zip
└── sample-data/
    └── Compliance_Control_Monitoring_Lab.xlsx
```

## Observação sobre importação

O pacote não inclui senhas. Durante a importação, cada usuário deve selecionar as próprias conexões e configurar o arquivo e as tabelas do ambiente de laboratório.
