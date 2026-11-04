# Taya v3 — Assistente Financeira e de Gestão (Clínica)
Agentize AI · Versão 3.0 · Novembro/2026

Status: ESPECIFICAÇÃO. Nenhum JSON gerado ainda. Decisões de 03/11 incorporadas (instância `taya`, apelido, data de compensação, catálogo fora do Caixa).

---

## 1. Escopo

Taya é a assistente profissional da clínica (proprietária e sócio operacional), via WhatsApp, no n8n, com ferramentas Google.

| Fase | Entrega |
|---|---|
| 1 | Volta ao ar sem estoque: credenciais novas, Contas a Pagar atual, agenda, briefing |
| 2 | Contas a Pagar completo: incluir, editar, cancelar, visão por mês |
| 3 | Áudio: transcrição antes do agente |
| 4 | Caixa + Clientes + Regras + fechamento das 21h |
| 5 | Emissor de NFS-e alimentado pelo Caixa (fora deste documento) |

Fora de escopo agora: estoque, finanças pessoais, integração bancária, automação das notas.

## 2. Decisões já tomadas

- Persona: Taya em tudo (webhook `taya-webhook`, memória `taya:{remoteJid}`).
- Instância Evolution própria da Taya e grupo novo.
- Modelo: DeepSeek V4 Flash. Memória: Redis, TTL 36h.
- Confirmação obrigatória antes de gravar: a Taya repete o lançamento e só grava após "Confirmado".
- "Excluir" nunca apaga linha: vira Status = Cancelado.
- Totais e somas saem de fórmula ou nó de código, nunca da IA.
- Caixa registra a venda inteira, com o valor que a proprietária informar (o que entra na conta). Sem detalhar parcelas nem taxa.
- Formas de pagamento iniciais: Crédito, Débito, Pix, Dinheiro. Regra de NF configurável (aba Regras); hoje só Crédito exige.
- Fechamento do dia às 21h. NFs pendentes não entram no fechamento, mas a Taya responde quando perguntada.

## 3. Planilhas (dois arquivos)

Motivo da separação: a Teka (agente de agendamento) conversa com pacientes e só deve enxergar a Base Clínica. O financeiro fica isolado.

### 3.1 Base Clínica (compartilhada com a Teka)

**clientes** (base já recebida, 87 registros). Todas as colunas atuais foram mantidas, porque a Teka já depende delas. Foram acrescentadas ao final duas colunas: **Apelido** (a proprietária preenche; a Taya busca por Nome e Apelido) e **Origem** (NF / Taya / Teka). UF normalizada para maiúsculas. A aba `revisao_clientes` lista o que revisar (telefone, e-mail, apelido dos primeiros nomes repetidos) e pode ser apagada depois.

**procedimentos**: catálogo atual (PR001..PR009). Pertence ao agente de atendimento (Teka). A Taya não usa o catálogo no Caixa por enquanto; a descrição do lançamento é texto livre e o valor é sempre o informado pela proprietária.

### 3.2 Financeiro Taya (privado)

**Contas a Pagar**: ID (CP-AAMMDD-XXXX), Fornecedor/Descrição, Categoria, Valor, Vencimento, Mês (AAAA-MM, gravado pelo n8n), Status (Pendente/Pago/Atrasado/Cancelado), Data Pagamento, Valor Pago, Recorrente?, Observação, Registrado por, Criado em.

**Caixa**: ID (CX-AAMMDD-XXXX), Data, Tipo (Entrada/Saída), Descrição, Cliente (nome como dito), ID Cliente, Valor, Forma de pagamento, Conta/Origem, Data compensação (gravada pelo n8n: Data + dias da aba Regras), Exige NF (gravado pelo n8n a partir da aba Regras), Status NF (Pendente / Emitida / Cadastro pendente / Não se aplica), Nº NF, Mês, Registrado por, Criado em, Status (Ativo/Cancelado).

**Contas** (origens de saída): Conta bancária, Cartão, Dinheiro. Lista a fechar com a proprietária.

**Regras**: Forma de pagamento → Exige NF (Sim/Não) e Dias p/ compensar. Editável sem mexer em fluxo.

**Resumo Mensal**: totais por mês, por fórmula, somando por data (vencimento nas contas, data do lançamento no caixa). Colunas derivadas (Mês, Exige NF, Data compensação) NÃO usam fórmula na planilha: fórmulas em linhas vazias fazem o append do Sheets gravar abaixo delas. Quem preenche é o sub-workflow.

### 3.3 Organização dos dados (visão de três agentes)

Dois arquivos, separados por domínio (e não uma planilha por tabela). Clientes é a única entidade compartilhada.

| Agente | Papel | Lê | Escreve |
|---|---|---|---|
| Teka | atendimento WhatsApp, agenda, preços | Base Clínica | Base Clínica (clientes novos, lembretes) |
| Taya | assistente financeira e de gestão | Financeiro + clientes | Financeiro (e cadastro de cliente, quando necessário) |
| Emissor de NFS-e (futuro) | emite notas a partir do Caixa | Caixa + clientes | Status NF e Nº NF no Caixa |

Regras para os três conviverem:
- Caixa guarda apenas o ID do cliente (CLI001). Nada de fórmula entre arquivos.
- Cadastro e busca de clientes são sub-workflows únicos (`cadastrar_cliente`, `buscar_cliente`), chamados por todos os agentes. Isso evita ID duplicado e cliente duplicado: o cadastro confere o CPF antes de criar.
- Formato de ID estável (CLI…, CP-…, CX-…) para facilitar uma migração futura para banco de dados quando o volume pedir.

## 4. Fluxos

### 4.1 Taya — Conversacional (webhook)

```
WhatsApp -> Evolution -> Webhook n8n
  -> Filtro (fromMe, grupo autorizado, vazio)
  -> Switch: texto ou áudio
       áudio -> baixar mídia -> transcrever -> texto
  -> Agente Taya (DeepSeek + Redis 36h)
  -> Responde no WhatsApp -> 200 OK
```

Entrada do agente: `Hoje é DD/MM/AAAA. {pushName}: {mensagem}` (data sempre no User Message; o System Message não avalia expressões).

**Ferramentas de escrita são sub-workflows** (Call n8n Workflow Tool), não nós Sheets soltos. O agente envia só os campos; o sub-workflow gera o ID, valida, calcula Exige NF pelas Regras e grava. Assim o ID nunca depende do modelo.

| Ferramenta | Ação |
|---|---|
| listar_contas_pagar (filtro por mês/status) | leitura |
| registrar_conta_pagar | cria, ID gerado no n8n |
| editar_conta_pagar (por ID) | atualiza campos, inclusive marcar pago |
| cancelar_conta_pagar (por ID) | Status = Cancelado |
| agenda: listar / criar / editar / apagar | Google Calendar |
| buscar_cliente (nome ou apelido) | sub-workflow compartilhado, devolve até 5 candidatos (fase 4) |
| cadastrar_cliente | sub-workflow compartilhado; nome, CPF, apelido; confere CPF antes (fase 4) |
| registrar_caixa | cria lançamento (fase 4) |
| cancelar_caixa (por ID) | Status = Cancelado (fase 4) |
| listar_nf_pendentes | leitura (fase 4) |
| resumo_periodo (dia/mês) | leitura, totais por código (fase 4) |

### 4.2 Protocolo de confirmação

1. A proprietária fala (texto ou áudio).
2. Taya repete o lançamento completo e pergunta "Confirma?".
3. Só com "Confirmado" a Taya chama a ferramenta de escrita.
4. Taya devolve o ID gravado.
Começa com o rascunho na memória da conversa. Se houver erro, evolui para rascunho explícito no Redis.

### 4.3 Busca de clientes

Código no n8n compara o nome falado com Nome e Apelido (sem acento, sem maiúscula, parcial).
- 1 resultado: segue, mostrando o nome completo na confirmação.
- Vários: Taya pergunta qual, listando nome completo e apelido.
- Nenhum: Taya oferece cadastrar (nome, CPF, apelido).
A venda nunca trava: com Exige NF = Não grava sem cadastro; com Exige NF = Sim sem CPF grava como "Cadastro pendente".

### 4.4 Taya — Briefing (agendado)

- 09h: contas atrasadas, vence hoje, vence na semana (seg a dom), agenda do dia. Estoque removido.
- 21h: fechamento do dia: entradas por forma de pagamento, saídas, saldo do dia, valores de crédito a cair amanhã, contas que vencem amanhã.

Mantém as lições anteriores: execução em série com Aggregate; datas DD/MM/AA e DD/MM/AAAA tratadas; horário de eventos por `substring(11,16)`; timezone do workflow em America/Sao_Paulo.

## 5. Áudio (fase 3)

- Evolution entrega o áudio em base64 (webhook_base64) ou via endpoint de download de mídia.
- Transcrição por API de speech-to-text (candidato: Whisper via OpenAI ou Groq). Validar preço e qualidade em PT-BR antes de fixar.
- O agente recebe só o texto transcrito, com a mesma confirmação obrigatória.

## 6. Infraestrutura e credenciais

- n8n, Evolution API e Redis hospedados na Cloudfy. Hosts, portas, chaves e senhas ficam apenas nas credenciais do n8n, nunca neste repositório.
- Credenciais novas: Google (Sheets, Calendar, Drive) com projeto Cloud em produção (evita expirar a cada 7 dias), DeepSeek (OpenAI-compatible), Redis.
- Exportar os JSONs dos workflows para o Git a cada fase.

## 7. Em aberto

- Fontes de despesa da proprietária (cartão, conta bancária, dinheiro): alinhar com ela.
- Outras formas de recebimento além das quatro iniciais.
- Confirmar com a contadora quais formas exigem NF.
- Qualidade da base de clientes: ver seção abaixo.

## 8. Qualidade da base de clientes (diagnóstico)

- 87 clientes; CPF preenchido em todos, sem CPF duplicado e sem nome completo duplicado.
- Sem telefone: 32. Sem e-mail: 39. Telefone fora do padrão: 1.
- UF com grafia inconsistente (ex.: "Sc") e ao menos uma sigla provavelmente errada.
- Primeiros nomes repetidos: 5 (um deles aparece 7 vezes, quatro aparecem 2 vezes). Apelido é necessário só para esses.
- Atualização periódica: reaproveitar o pipeline de extração de NFs para incluir novas clientes, deduplicando por CPF.
