# Fluxo Pai: Notificação ao Responsável

Este diretório contém o pacote importável do fluxo responsável por analisar os controles da base de laboratório e enviar notificações individuais pelo Microsoft Teams.

## Nome do fluxo

`LAB-COMPLIANCE-NOTIFICACAO-RESPONSAVEL`

## Funcionamento

O fluxo executa as seguintes etapas:

1. Recebe manualmente um destinatário de teste;
2. Lê os controles da tabela `Tabela1`;
3. Processa cada registro individualmente;
4. Valida o status do controle e do teste;
5. Valida a data do próximo teste;
6. Calcula os dias restantes ou em atraso;
7. Envia o Adaptive Card;
8. Registra o envio na tabela de log `Tabela2`.

## Regras de processamento

O controle somente é processado quando:

```text
ControlStatus = Work
E
TestStatus = ON TIME ou LATE
E
NextTestDate contém uma data válida
