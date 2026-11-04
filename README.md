# Agentize AI

Automações e agentes de IA para pequenas e médias empresas, operados pelo WhatsApp.

Este repositório reúne a documentação e os modelos dos projetos da Agentize AI. Os dados de clientes e as credenciais não fazem parte do repositório: cada projeto é documentado com dados anonimizados e placeholders.

## Projetos

### Taya — assistente de gestão (clínica de estética)

Assistente que atua como secretária e assistente financeira: avisa vencimentos, organiza a agenda e registra lançamentos por mensagem, em um grupo de WhatsApp.

- Case study: [`docs/case-study-taya.md`](docs/case-study-taya.md)
- Especificação da v3: [`taya/arquitetura-v3.md`](taya/arquitetura-v3.md)
- Planilha modelo (apenas estrutura, sem dados): [`taya/templates/Financeiro_Taya.xlsx`](taya/templates/Financeiro_Taya.xlsx)

Status: v1 funcionou em uso real; v3 (contas a pagar completo, caixa, áudio) em reconstrução.

### Outros repositórios

- [agente_atendimento_whatsapp](https://github.com/leonardodmf/agente_atendimento_whatsapp): agente de atendimento e agendamento por WhatsApp.
- [Portifolio_IA](https://github.com/leonardodmf/Portifolio_IA): outros projetos de IA.
- [Assitente-conversacional---Taya](https://github.com/leonardodmf/Assitente-conversacional---Taya): versão inicial da Taya.

## Stack

n8n · Evolution API (WhatsApp) · DeepSeek / Gemini · Redis e PostgreSQL · Google Sheets e Calendar

## Estrutura

```
docs/        case studies
taya/        especificação e modelos da Taya
```

## Segurança

Chaves de API, senhas, IDs de planilhas e de grupos, dados pessoais e CPFs nunca são versionados. Os workflows do n8n são publicados apenas com placeholders.

## Autor

Leonardo Fagundes · Agentize AI
