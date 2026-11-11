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

### Teka — atendimento e agendamento (clínica de estética)

Recepcionista virtual da clínica no WhatsApp: atende pacientes, informa procedimentos e valores, cadastra, agenda, envia lembretes, confirma presença, faz triagem de intercorrências e chama a proprietária quando necessário. Compartilha a Base Clínica (clientes e procedimentos) com a Taya.

- Especificação da v3: [`teka/arquitetura-v3.md`](teka/arquitetura-v3.md)

Status: v2 testada parcialmente; v3 em especificação (núcleo, áudio, agenda, lembretes).

### Outros repositórios

- [agente_atendimento_whatsapp](https://github.com/leonardodmf/agente_atendimento_whatsapp): repositório antigo da Teka (substituído pela pasta `teka/`).
- [Portifolio_IA](https://github.com/leonardodmf/Portifolio_IA): outros projetos de IA.
- [Assitente-conversacional---Taya](https://github.com/leonardodmf/Assitente-conversacional---Taya): versão inicial da Taya.

## Stack

n8n · Evolution API (WhatsApp) · DeepSeek / Gemini · Groq Whisper · Redis e PostgreSQL · Google Sheets e Calendar

## Estrutura

```
docs/        case studies
taya/        especificação e modelos da Taya
teka/        especificação e modelos da Teka
```

## Segurança

Chaves de API, senhas, IDs de planilhas e de grupos, dados pessoais e CPFs nunca são versionados. Os workflows do n8n são publicados apenas com placeholders.

## Autor

Leonardo Fagundes · Agentize AI
