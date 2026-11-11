# Teka v3 — Agente de Atendimento e Agendamento (Clínica)
Agentize AI · Versão 3.0 · Novembro/2026

Status: ESPECIFICAÇÃO. Nenhum JSON gerado ainda. Reconstrução a partir da v2 (workflow principal + 6 sub-workflows + planilha), aproveitando modelagem, regras de negócio e lições, e corrigindo os defeitos listados na seção 10.

---

## 1. Escopo

Teka é a recepcionista virtual da clínica no WhatsApp: atende pacientes, informa procedimentos e valores, cadastra, agenda, altera, cancela, lembra e confirma presença, faz a triagem de intercorrências e chama a proprietária quando precisa.

| Fase | Entrega |
|---|---|
| 1 | Núcleo: atende, identifica e cadastra cliente, informa preços, triagem pelo FAQ, escala para a proprietária, consulta horários livres |
| 2 | Áudio: transcrição antes do agente (sub-workflow compartilhado com a Taya). Vem cedo porque as clientes usam muito áudio |
| 3 | Agenda: criar, listar, alterar e cancelar agendamentos |
| 4 | Lembretes 24h / 6h para todos os agendamentos (inclusive os marcados direto no Calendar), confirmação de presença pela conversa e aviso à proprietária |
| 5 | Produto: template de instalação para outra clínica, checklist, case study |

Decisões de 11/11/2026: áudio sobe para a fase 2; a Teka agenda a qualquer hora, inclusive fora do horário comercial; todo agendamento recebe lembrete, tenha sido marcado pela Teka ou pela proprietária.

Fora de escopo agora: pagamento ou sinal antecipado, mais de um profissional na agenda, mensagens de pós-procedimento automáticas (templates a definir com a proprietária), campanhas e marketing.

## 2. Decisões já tomadas (herdadas da v2)

- Persona: Teka. Webhook `teka-webhook`, instância Evolution própria da Teka, sessão no Redis `teka:{telefone}`, TTL 36h.
- Modelo: DeepSeek V4 Flash, temperatura 0.4. Transcrição: Groq `whisper-large-v3-turbo`.
- Agenda no Google Calendar; dados no Google Sheets (editável pela proprietária sem treinamento).
- Execução sempre em série no n8n; nada de ramos paralelos no webhook.
- Eventos filtrados na origem: Evolution envia só `MESSAGES_UPSERT`.
- Envio para a Evolution com `bodyParameters` (nunca Raw JSON) e `delay` numérico.
- Cadastro progressivo: nome + CPF no primeiro contato; e-mail e endereço só quando o agendamento é confirmado.
- Tom: acolhedor, curto, sem emoji, sem markdown, linguagem neutra de gênero até saber o nome. Não confirma cada passo ("Perfeito!", "Anotado!").
- Triagem pelo FAQ antes de escalar (Escalar = Não / Condicional / Sim).
- `config_clinica` é a fonte única dos parâmetros operacionais.

## 3. Decisões novas da v3

- **Totais, IDs, datas e cálculos saem de código, nunca da IA** (mesma regra da Taya). O agente envia só os campos; o sub-workflow gera o ID, valida e grava.
- **Ferramentas de escrita trabalham por ID** (AG-…, CLI-…), nunca "primeiro agendamento daquele telefone".
- **Nenhuma linha é apagada**: cancelar vira Status = Cancelado.
- **Contexto dinâmico vai no User Message** (data, cliente, agendamentos). O System Message é estático: persona, regras, catálogo e FAQ montados por código num único bloco de texto.
- **Mínimo de dado pessoal no prompt**: o agente vê primeiro nome, se é cadastrada, o que falta no cadastro e os próximos agendamentos. CPF, endereço e e-mail completos não entram no prompt.
- **Horários e regras vêm da planilha**, não do código (`config_clinica` + duração e buffer de `procedimentos`).
- **Datas com fuso explícito**: o modelo informa `AAAA-MM-DD HH:mm` (hora local); o código monta `AAAA-MM-DDTHH:mm:00-03:00`. Nunca depender do fuso do servidor (`new Date().getHours()` em UTC desloca 3h). Workflow configurado em America/Sao_Paulo.
- **Uma linha de cabeçalho por aba** (a v2 tinha duas: rótulo + snake_case, e o n8n lia a segunda como registro).
- **Cliente compartilhada com a Taya** pelos sub-workflows únicos `buscar_cliente` e `cadastrar_cliente`.
- **Nome da clínica, do agente e da proprietária são parâmetros**, para o mesmo workflow servir outra clínica.

## 4. Planilha: Base Clínica (compartilhada com a Taya)

Arquivo único da clínica. A Teka lê e escreve só aqui; o financeiro da Taya fica em outro arquivo e a Teka não tem acesso.

**clientes** (87 registros vindos das NFs). Colunas atuais mantidas + `Apelido` e `Origem` (NF / Taya / Teka), já definidas na spec da Taya. Mudança da v3: **Telefone normalizado só com dígitos, com DDI** (`5548999999999`). Hoje está como `(48) 99999-9999`, que nunca bate com o número do WhatsApp. Ajuste único na base + o cadastro passa a gravar nesse formato.

**procedimentos**: ID (PR001…), Procedimento, Valor, Duração (min), Buffer (min), Preparo Prévio?, Instrução Pré, Instrução Pós, Descrição Curta, Ativo?. Só `Ativo = sim` vai para o agente.

**faq_intercorrencias**: ID, Palavras-gatilho, Procedimento, Resposta Padrão, Escalar? (Não / Condicional / Sim), Instrução de Triagem.

**agendamentos** (substitui `lembretes_agendamentos`; status do agendamento separado do status de lembrete):

| Coluna | Quem grava |
|---|---|
| ID (AG-AAMMDD-XXXX) | n8n |
| ID Evento GCal | n8n |
| ID Cliente, Nome Cliente, Telefone | n8n (a partir do cadastro) |
| ID Procedimento, Procedimento | n8n |
| Início, Fim (ISO com -03:00) | n8n |
| Status: Agendado / Confirmado / Cancelado / Realizado / Não compareceu | n8n; proprietária pode editar |
| Lembrete 24h em, Lembrete 6h em, Aviso proprietária em, Confirmado em | n8n |
| Origem (Teka / Manual) | n8n |
| Criado em, Atualizado em, Obs | n8n |

**config_clinica** (chave → valor, uma linha de cabeçalho):
`nome_clinica`, `nome_agente`, `nome_proprietaria`, `telefone_proprietaria`, `id_calendario`, horários seg–sex e sábado, `atende_domingo`, `intervalo_slots_min` (30), `antecedencia_minima_horas`, `lembrete_1_horas` (24), `lembrete_2_horas` (6), `aviso_proprietaria_horas` (2), `janela_envio_inicio` / `janela_envio_fim` (horário em que a Teka pode mandar mensagem ativa), `evolution_instancia`.

Fica fora da planilha (e fora do repositório): chaves de API e o ID da própria planilha, que vive num nó `Constantes` no início de cada workflow.

## 5. Fluxos

### 5.1 Teka — Conversacional (webhook)

```
WhatsApp -> Evolution -> Webhook n8n
  -> Responder 200 OK
  -> Filtro (fromMe, grupo, mensagem vazia, tipo não suportado)
  -> Normalizar (telefone, nome, texto, tipo)       [tratar @lid, ver 9.1]
  -> Pausa humana ativa? (Redis teka:pausa:{tel}) -> sim: encerra
  -> Switch: texto ou áudio
       áudio -> sub-workflow transcrever_audio -> texto   (fase 2)
  -> buscar_cliente (por telefone)
  -> Carregar procedimentos, FAQ, config, próximos agendamentos da cliente
  -> Montar Contexto (código)
  -> Agente Teka (DeepSeek + memória Redis 36h)
  -> Enviar resposta (Evolution)
```

Entrada do agente (User Message), montada por código:

```
Hoje é quarta-feira, 11/11/2026, 10:18.
Cliente: Ana (cadastrada, ID CLI-0042; falta e-mail e endereço)
Próximos agendamentos: AG-261114-7F2A, Botox Região, sex 14/11 15:00, Agendado
Mensagem: <texto>
```

O dia da semana vai junto da data para o modelo resolver "sexta que vem" sem erro.

Memória: o mesmo esquema da Taya (Redis, chave por telefone, TTL 36h, janela de 36 mensagens). Decidir na fase 1 entre o nó Redis Chat Memory e o GET/SET manual da v2; o critério é poder **gravar no histórico as mensagens que a Teka envia por conta própria** (lembretes), para a cliente responder "sim" e o agente entender.

### 5.2 Ferramentas (sub-workflows)

Todas chamadas pelo Call n8n Workflow Tool com o **ID fixo do sub-workflow** (a v2 usava `{{$workflow.id}}`, que chama o próprio workflow principal). Cada uma devolve `output` em texto para o agente.

| Ferramenta | Entrada do agente | O que o código faz | Fase |
|---|---|---|---|
| buscar_cliente (compartilhada) | telefone ou nome | normaliza, compara pelos 8 últimos dígitos + DDD (ver 9.1); devolve até 5 candidatos | 1 |
| cadastrar_cliente (compartilhada) | nome, CPF | valida CPF, confere duplicidade por CPF, gera ID, Origem = Teka | 1 |
| atualizar_cliente | ID, e-mail, endereço | atualiza a linha pelo ID (a v2 dava append e duplicava) | 1 |
| verificar_disponibilidade | data, ID do procedimento | lê duração + buffer do catálogo e horários da config; ignora horários passados e a antecedência mínima; devolve até 6 horários | 1 |
| notificar_proprietaria | motivo, resumo | nome e telefone da cliente vêm do contexto, não do modelo; número da proprietária vem da config | 1 |
| criar_agendamento | ID cliente, ID procedimento, data e hora | **revalida a disponibilidade** antes de criar; cria o evento (com o telefone na descrição); grava em agendamentos; devolve o ID AG | 3 |
| listar_agendamentos | (telefone do contexto) | agendamentos futuros não cancelados | 3 |
| alterar_agendamento | ID AG, nova data e hora | revalida, atualiza evento e linha, zera lembretes | 3 |
| cancelar_agendamento | ID AG | remove o evento do Calendar, Status = Cancelado | 3 |
| confirmar_presenca | ID AG | Status = Confirmado, grava Confirmado em | 4 |

Protocolo de agendamento: a Teka mostra os horários → a cliente escolhe → a Teka repete procedimento, dia e hora e pergunta se confirma → só então chama `criar_agendamento` → pede e-mail e endereço se faltarem.

### 5.3 Lembretes (agendado, a cada hora)

O Google Calendar é a fonte dos agendamentos; a aba `agendamentos` guarda o controle (status e lembretes). Assim, o que a proprietária marca direto no Calendar também recebe lembrete.

```
Schedule (1h) -> Ler eventos do Calendar nas próximas 30h
  -> Sincronizar (código): evento sem linha em agendamentos?
       acha a cliente pelo telefone na descrição ou pelo nome no título (buscar_cliente)
         encontrou uma -> cria a linha com Origem = Manual
         não encontrou ou ficou em dúvida -> avisa a proprietária uma única vez
           ("não consegui identificar a cliente do horário de 15h de sexta")
       evento sumiu do Calendar -> Status = Cancelado
  -> Carregar agendamentos com Status Agendado e Início nas próximas 30h
  -> Avaliar (código): qual ação cabe a cada um
       24h antes, Lembrete 24h vazio            -> lembrete_1 (inclui Instrução Pré)
       6h antes, não confirmado, Lembrete 6h vazio -> lembrete_2
       2h antes, não confirmado, Aviso vazio     -> aviso à proprietária
  -> Envia -> grava o horário do envio na linha -> grava a mensagem no histórico Redis da cliente
```

Regras:
- **Idempotente**: cada envio grava seu "em"; nada é enviado duas vezes, mesmo com a janela de uma hora se repetindo.
- **Janela de envio**: nada fora de `janela_envio_inicio`–`janela_envio_fim`. Se o horário do lembrete cair fora, envia no último horário permitido antes dele (ex.: atendimento às 9h → lembrete de 6h vai na véspera à noite).
- A resposta da cliente cai no fluxo conversacional; o agente vê o agendamento pendente no contexto e chama `confirmar_presenca` ou `cancelar_agendamento`.
- Textos dos lembretes como modelos na `config_clinica`, com variáveis (`{primeiro_nome}`, `{procedimento}`, `{dia}`, `{hora}`).

### 5.4 Pausa humana

Quando a proprietária responde a cliente manualmente, a Teka para de responder aquela conversa por um tempo (`teka:pausa:{telefone}`, TTL configurável, ex.: 12h). Mensagem `fromMe` enviada pelo celular aciona a pausa; mensagem `fromMe` enviada pela API (a própria Teka) não. Como diferenciar está na seção 9.

## 6. Comportamento do agente (System Message)

Mantido da v2, reescrito como texto estático com parâmetros:
1. Cliente nova: apresentação curta e pedido de nome completo + CPF ("estamos organizando o cadastro da clínica"). Só cadastra com os dois. Depois do cadastro não repete os dados: vai direto ao atendimento.
2. E-mail e endereço só depois de confirmar um agendamento.
3. Valores pela tabela. Desconto: "valor padrão, pode ser parcelado; condição especial, como no valor à vista, só em consulta com a Dra."
4. Pergunta genérica sobre botox: "a partir de R$ 499,90 (uma região); o ideal é uma avaliação gratuita" e oferecer agendar. (Valor lido do catálogo, não fixo no prompt.)
5. Intercorrência: consultar o FAQ; Não → resposta padrão; Condicional → pergunta de triagem; Sim ou triagem positiva → avisa a cliente e chama `notificar_proprietaria`. **Sinal de urgência** (falta de ar, reação que se espalha) → orientar a procurar atendimento de emergência imediatamente, além de notificar.
6. Dúvida clínica densa: não dar aula, encaminhar para a Dra.
7. A Teka atende e agenda 24h, inclusive fora do horário comercial (decidido em 11/11). O horário de funcionamento limita só os horários oferecidos na agenda, não o atendimento.
8. Formato WhatsApp: frases curtas, sem listas, sem markdown, sem emoji. Usar a ferramenta antes de responder.

## 7. Áudio (fase 2)

Sub-workflow `transcrever_audio`, compartilhado com a Taya:
- Baixa a mídia pelo endpoint da Evolution `getBase64FromMediaMessage` (ou recebe via `webhook_base64`).
- Converte para binário e envia ao Groq (`whisper-large-v3-turbo`, idioma pt).
- Devolve só o texto. Falha na transcrição → a Teka pede para a cliente escrever.

## 8. Infraestrutura, segurança e LGPD

- n8n, Evolution e Redis na Cloudfy. Hosts, chaves e senhas só nas credenciais do n8n.
- **A chave da Evolution que estava fixa no nó de download de áudio da v2 deve ser trocada** antes de qualquer coisa ir para o Git.
- Repositório só com placeholders: `ID_PLANILHA`, `ID_CALENDARIO`, `EVOLUTION_URL`, `EVOLUTION_INSTANCIA`. A planilha versionada é só estrutura (sem linhas de cliente).
- Dados de saúde (intercorrências) e CPF são dados sensíveis: não vão para o prompt além do necessário, não vão para logs exportados, não vão para o repositório.
- Google Cloud em produção (evita token expirando a cada 7 dias). Calendar ID lido da config ("By ID").
- Exportar os JSONs dos workflows para o Git a cada fase (o ambiente já foi perdido duas vezes).

## 9. Em aberto

1. **Identificação do número**: o WhatsApp pode entregar `remoteJid` como `@lid` (sem telefone). Testar o payload real da Evolution antes de construir (lição da v2) e ver se vem o número em outro campo. Validar também o nono dígito: alguns números chegam sem ele, por isso a busca compara DDD + 8 últimos dígitos.
2. **Pausa humana**: confirmar no payload como diferenciar mensagem enviada pelo celular da enviada pela API (campo de origem ou guardar no Redis os IDs das mensagens que a Teka enviou).
3. **Mensagens em sequência**: cliente que manda 3 mensagens seguidas recebe 3 respostas. Avaliar buffer no Redis (aguardar alguns segundos e juntar) depois da fase 3.
4. Política de cancelamento e antecedência mínima para marcar e desmarcar (proposta inicial: marcar com 2h de antecedência; sem restrição para desmarcar).
5. Duração da avaliação gratuita e se ela tem buffer.
6. Textos dos lembretes e das instruções pré e pós com a proprietária.
7. Memória: Redis Chat Memory ou GET/SET manual (ver 5.1).
8. Como a proprietária registra agendamentos manuais no Calendar: pedir que coloque o nome completo da cliente no título (e, se possível, o telefone na descrição) para a sincronização reconhecer.

Resolvidos em 11/11: agenda manual recebe lembrete (seção 5.3); agendamento fora do horário comercial liberado (seção 6).

## 10. O que muda da v2 (diagnóstico)

| Problema na v2 | Efeito | v3 |
|---|---|---|
| Tools com `workflowId = {{$workflow.id}}` | ferramentas chamavam o workflow principal | ID fixo de cada sub-workflow |
| System Message sem `=` com expressões | agente não via cliente, catálogo, FAQ nem histórico | contexto no User Message, System montado por código |
| Sem data atual no prompt | "amanhã", "sexta" sem referência | data + dia da semana na entrada |
| Telefone `(48) 9…` × `5548…`; checagem em `nome` (coluna é "Nome Completo") | cliente cadastrada tratada como nova | telefone normalizado + `buscar_cliente` |
| Ramo de áudio desconectado, nó Whisper vazio | áudio não funcionava | sub-workflow `transcrever_audio` |
| Lembretes lendo aba e colunas que não existem; status nunca atualizado | lembretes nunca saíam (ou repetiam) | aba `agendamentos`, envio idempotente |
| Sem confirmação de presença | pendência da v2 | `confirmar_presenca` + lembrete no histórico |
| Duas linhas de cabeçalho | linha de rótulos lida como registro | uma linha por aba |
| `getHours()` no servidor; horários fixos no código | deslocamento de 3h; config ignorada | fuso explícito; horários da config |
| Alterar e cancelar pelo telefone, `.first()` | podia mexer no agendamento errado | por ID, com `listar_agendamentos` |
| `salvar_cliente` só fazia append | atualizar duplicava a cliente | `cadastrar` confere CPF; `atualizar` por ID |
| Criar sem revalidar horário | dois agendamentos no mesmo horário | revalida antes de criar |
| Chave da Evolution fixa em um nó | exposição da credencial | trocar a chave; só credenciais do n8n |

Aproveitado da v2 sem mudança de essência: modelagem da planilha, catálogo e FAQ, regras de negócio do prompt, padrão de sub-workflow com `output`, cálculo de horários livres (com o fuso corrigido), lições de Redis, Evolution e execução em série.
