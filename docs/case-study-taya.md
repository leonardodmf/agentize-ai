# Case study — Taya: assistente virtual de gestão via WhatsApp

Agentize AI · Autor: Leonardo Fagundes

> Status honesto: a v1 funcionou em uso real por duas pessoas. A infraestrutura foi perdida meses depois e a v3 está em reconstrução. Não há métricas medidas; os números de ganho abaixo são estimativas do autor e estão marcados como tal.

---

## 1. Contexto e problema

**Problema.** Uma clínica de estética de pequeno porte operava sem sistema de gestão integrado. Contas a pagar e estoque viviam na memória da proprietária, e a agenda estava em um aplicativo proprietário sem API. O gestor só descobria um vencimento quando o fornecedor cobrava.

**Origem do projeto.** Laboratório do autor para a Agentize AI (automações e agentes de IA para PMEs). A clínica, da parceira do autor, serviu como ambiente real de validação, não como cliente pagante.

**Usuários.** A proprietária da clínica e o sócio operacional (o autor), no mesmo grupo de WhatsApp.

**Restrições**

| Restrição | Detalhe |
|---|---|
| Orçamento | Infraestrutura na faixa de R$ 150/mês (n8n, Evolution API e banco no mesmo plano) |
| Canal | WhatsApp, a única interface que a usuária aceitaria usar |
| Agenda | O app de agenda atual não tem API: integração direta impossível |
| Tempo | Projeto paralelo ao trabalho principal, desenvolvido em incrementos |
| Experiência | n8n e agentes com tool calling foram aprendidos durante o projeto |

## 2. Decisões técnicas (ordem cronológica)

1. **Google Sheets como banco de dados.** Alternativas: PostgreSQL, Airtable. Escolhido porque a usuária já conhece planilha, pode editar à mão, a API é gratuita e o volume de uma PME é pequeno. Limite conhecido: não há transação nem chave estrangeira.
2. **WhatsApp via Evolution API** (já incluída no plano de hospedagem), em vez de APIs pagas por mensagem. Suporta grupos.
3. **Dois workflows separados.** Briefing diário (trigger agendado) e assistente conversacional (webhook). Triggers diferentes, falha isolada, manutenção mais simples.
4. **Google Calendar como agenda.** Alternativas avaliadas: Cal.com (auto-hospedado) e SimplyBook. Calendar venceu por custo zero, API madura e nó nativo no n8n. Ficou aberto avaliar uma agenda própria para o atendimento a pacientes.
5. **Interface em grupo de WhatsApp** com a proprietária e o sócio, em vez de número individual. Os alertas interessam aos dois.
6. **Modelo de linguagem.** Começou com Gemini (incluído no plano). O Gemini 2.5 Flash apresentou erros de tool calling (`Cannot read properties of undefined (reading 'parts')`, `Failed to parse stream`); o 2.0 Flash retornou 429 por cota gratuita zerada. Migrado para **DeepSeek V4 Flash** via API compatível com OpenAI, por preço baixo (centavos de dólar por milhão de tokens, conforme a tabela publicada) e suporte documentado a tool calls. Os modelos de código aberto oferecidos no plano não foram testados.
7. **Memória conversacional.** v1: PostgreSQL (persistente; a memória simples do n8n perde o estado ao reiniciar). v3: **Redis com TTL de 36 h**, porque a janela de contexto expira sozinha e não acumula histórico.
8. **Briefing em série, não em paralelo** (ver obstáculos).
9. **Decisões da v3:** confirmação explícita antes de qualquer gravação; "excluir" vira Status = Cancelado; IDs gerados por código no n8n, não pelo modelo; somas por fórmula ou código, nunca pela IA; dados separados por domínio (base de clientes compartilhada com o agente de atendimento, financeiro isolado).

## 3. Obstáculos e iteração

| Problema | Diagnóstico | Solução |
|---|---|---|
| Linha 1 da planilha era um título mesclado | O n8n usava a linha 1 como cabeçalho | Cabeçalhos sempre na linha 1, sem título mesclado |
| Leitura paginada com linhas vazias | A planilha tinha linhas formatadas sem dados | Filtro por campo obrigatório |
| `JSON parameter needs to be valid JSON` e `Bad request` no envio ao WhatsApp | Expressão dentro de JSON bruto quebrava o JSON | Corpo por campos (Using Fields Below) com a expressão no campo |
| Sinal de `=` aparecendo no texto enviado | Expressão colocada em campo de valor fixo | Campo em modo Expression, sem o `=` |
| Tool do Sheets sem uso: `No parameters are set up to be filled by AI` | Nós Tool exigem ao menos um parâmetro marcado como `fromAI` | Marcar um campo como preenchível pela IA |
| Calendar com acesso negado e `403 API has not been used` | Credencial separada da de Sheets; API do Calendar desabilitada no projeto Cloud; app OAuth em modo teste exige usuários de teste | Habilitar a API, cadastrar usuário de teste |
| Briefing disparando à 01:00 em vez de 09:00 | Workflow no fuso America/New_York | Fuso do workflow em America/Sao_Paulo |
| Duas mensagens de briefing e erro `Node ... hasn't been executed` | Leituras em paralelo; o nó de código lia nós que ainda não tinham rodado | Execução em série com Aggregate antes do Code |
| `Invalid Date` nos vencimentos | O Sheets entrega datas como `DD/MM/AA` (ano com 2 dígitos) | Parser com regex para `DD/MM/AA` e `DD/MM/AAAA` |
| Evento das 20h aparecendo como 23:00 | `new Date()` convertia o horário para UTC | Extrair `HH:MM` direto da string ISO com `substring(11,16)` |
| Agente inventava datas ao agendar | O System Message não avalia expressões; o modelo não sabia a data | Injetar a data no User Message por expressão |
| Agente tratava todas as mensagens como do mesmo usuário | Nenhum identificador de remetente no prompt | Passar o nome do remetente (`pushName`) no User Message |
| Agente dizia "já ajustei o evento" sem ter ferramenta de edição | Havia só a tool de criar; o modelo afirmou uma ação que não existia | Criar tools de atualizar e apagar; na v3, resposta baseada no retorno da ferramenta |
| Contas pagas aparecendo como atrasadas no briefing | **Em aberto:** causa não confirmada | Investigar com dados reais na reconstrução |
| Perda total da infraestrutura hospedada | Falta de backup externo dos workflows | Reconstrução a partir da documentação; JSONs passam a ser versionados |

## 4. Arquitetura

### v1 (como foi construída)

```
Evolution API (WhatsApp)
   -> Webhook n8n
   -> Code: filtra fromMe, grupo autorizado e mensagem vazia
   -> AI Agent (LLM) + memória Postgres
        tools: contas a pagar (leitura), estoque (leitura),
               agenda (ler, criar, atualizar, apagar)
   -> HTTP Request: resposta ao grupo
   -> Respond to Webhook

Schedule (09h)
   -> Sheets (contas) -> Aggregate -> Sheets (estoque) -> Aggregate
   -> Calendar (agenda do dia) -> Aggregate
   -> Code: monta a mensagem -> HTTP Request: envia ao grupo
```

### v3 (em construção)

```
WhatsApp -> Evolution -> Webhook n8n
   -> Filtro -> Switch texto/áudio (áudio -> transcrição)
   -> Agente (DeepSeek + Redis 36h)
        contas a pagar: listar / registrar / editar / cancelar
        agenda: listar / criar / editar / apagar
        caixa e clientes (sub-workflows compartilhados)
   -> Resposta no WhatsApp
```

Detalhes em [`taya/arquitetura-v3.md`](../taya/arquitetura-v3.md).

### Componentes

| Componente | Tecnologia |
|---|---|
| Orquestração | n8n |
| WhatsApp | Evolution API |
| Modelo | DeepSeek V4 Flash (antes Gemini) |
| Memória | PostgreSQL (v1), Redis (v3) |
| Dados | Google Sheets |
| Agenda | Google Calendar |

## 5. Resultado

- v1 em uso real por duas pessoas: briefing diário com contas atrasadas, vencimentos da semana, alertas de estoque e agenda; conversa em linguagem natural para consultar contas e estoque e criar, editar e apagar eventos.
- Efeito qualitativo: o acompanhamento financeiro passou de reativo (descobrir quando o fornecedor cobra) para proativo (aviso antes do vencimento).
- **Não há métricas medidas.** Estimativa do autor: eliminação de alguns minutos diários de consulta à planilha. Nenhum dado de tempo foi coletado.
- Hoje: v3 em especificação e planilhas modelo prontas; reconstrução dos fluxos em andamento.

## 6. Aprendizados

**O que faria diferente**

1. Versionar os JSONs dos workflows no Git desde o primeiro dia. A perda da infraestrutura mostrou o custo de depender de uma única plataforma.
2. Travar o modelo de linguagem antes de construir. Boa parte do debug veio de trocar de modelo no meio e de diferenças de tool calling entre eles.
3. Testar em um grupo de homologação, não no grupo real dos usuários.
4. Registrar cada decisão de arquitetura no momento em que ela é tomada.

**Lições técnicas**

- Modelos afirmam ações que não executaram. A resposta ao usuário deve se basear no retorno da ferramenta, e operações que gravam dinheiro ou dados exigem confirmação explícita.
- Aritmética financeira não é papel do modelo: totais saem de fórmula ou código.
- O System Message não avalia expressões do n8n; contexto dinâmico (data, remetente) vai no User Message.
- Planilha como banco funciona para PMEs, mas colunas com fórmula em linhas vazias fazem o append gravar abaixo delas; colunas derivadas devem ser gravadas pelo fluxo.
- Datas chegam em formatos diferentes de cada fonte; o parsing precisa ser defensivo.
- Credenciais, chaves e IDs reais nunca entram em JSON ou documento versionado.

**Lição de produto.** O usuário não queria um sistema de gestão, queria deixar de pensar em gestão. Em vez de um painel para consultar, o agente avisa e executa no canal que a pessoa já usa.
