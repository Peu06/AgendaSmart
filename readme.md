# Documentação de Análise de Requisitos
## Sistema de Agendamento Automático de Ordens de Serviço (ISP)

**Versão:** 1.0 (Documento inicial)
**Stack proposta:** C# (.NET 8), PostgreSQL, EF Core, React + Vite
**Integração externa:** Hubsoft

---

## 1. Visão Geral

O sistema tem como objetivo automatizar o agendamento de ordens de serviço (OS) de uma ISP, distribuindo-as entre técnicos/equipes de campo com base em critérios operacionais (deslocamento, cidade, disponibilidade, SLA, tipo de serviço, janela de atendimento do cliente e expediente da equipe). O agendamento gerado automaticamente **nunca é confirmado sem validação humana** — todo slot sugerido entra em uma fila de aprovação.

O sistema consome dados do **Hubsoft** (ordens de serviço e agenda-base dos técnicos) e complementa essas informações com dados que o Hubsoft não possui nativamente (cidade de atuação, matriz de deslocamento entre cidades, tempo estimado por tipo de serviço/localidade).

---

## 2. Objetivos

- Reduzir o tempo gasto no agendamento manual de OS.
- Otimizar rotas dos técnicos, minimizando deslocamento ocioso.
- Respeitar SLA contratual de cada tipo de serviço.
- Garantir que nenhuma equipe seja agendada além do seu expediente.
- Dar visibilidade e controle humano final sobre cada agendamento gerado.
- Permitir configuração flexível de disponibilidade, cidades e tempos de deslocamento.

---

## 3. Atores do Sistema

| Ator | Descrição |
|---|---|
| **Administrador/Coordenador de Agendamento** | Configura cidades, matriz de deslocamento, tipos de serviço, e aprova/ajusta/rejeita os agendamentos sugeridos. |
| **Motor de Agendamento (Scheduler Engine)** | Serviço automático (batch/on-demand) que calcula os melhores slots possíveis para cada OS pendente. |
| **Módulo de IA (opcional)** | Camada auxiliar chamada em casos de empate, ambiguidade ou múltiplos cenários viáveis, para sugerir e justificar a melhor opção. |
| **Integração Hubsoft** | Job que sincroniza ordens de serviço e agenda-base dos técnicos. |
| **Técnico/Equipe** | Não interage diretamente com o sistema nesta fase (info gerenciada pelo Coordenador), mas é a entidade central do agendamento. |

---

## 4. Escopo

### 4.1 Dentro do escopo (MVP)
- Cadastro/sincronização de técnicos e equipes.
- Cadastro de cidades e matriz de deslocamento entre cidades.
- Cadastro de disponibilidade dos técnicos por dia/horário (com suporte a importação do Hubsoft + ajuste manual).
- Cadastro de tipos de serviço com tempo estimado padrão e SLA.
- Override de tempo estimado por combinação tipo de serviço + cidade/região.
- Importação de ordens de serviço pendentes do Hubsoft.
- Motor de agendamento automático (heurística de score).
- Tela de fila de aprovação (aprovar / ajustar manualmente / rejeitar e reprocessar).
- Tela de gestão de disponibilidade dos técnicos (calendário por técnico).
- Log de decisões de agendamento (auditoria).

### 4.2 Fora do escopo (fases futuras)
- Roteirização com mapas em tempo real (GPS/trânsito ao vivo).
- Reagendamento automático por atraso em campo (só o inicial é tratado no MVP).

---

## 5. Glossário

| Termo | Definição |
|---|---|
| **OS** | Ordem de Serviço — atividade a ser executada por um técnico em campo. |
| **Cidade de Origem** | Cidade onde a equipe/técnico reside ou de onde inicia o deslocamento diário. |
| **Cidade de Atuação/Trabalho** | Cidade onde a equipe efetivamente presta atendimento em um dado dia (pode ser igual ou diferente da origem). |
| **Slot** | Intervalo de data/hora candidato para alocação de uma OS a um técnico. |
| **SLA** | Prazo máximo (data/hora limite) em que a OS precisa ser atendida. |
| **Score de Agendamento** | Pontuação calculada pelo motor para ranquear os melhores slots/técnicos candidatos a uma OS. |
| **Janela do Cliente** | Intervalo de datas/horários em que o cliente está disponível para receber o técnico. |

---

## 6. Regras de Negócio (RN)

### RN01 — Expediente da equipe
Cada técnico possui disponibilidade configurável por **dia específico** (não apenas dia da semana recorrente), contendo:
- Data
- Hora início / hora fim do expediente
- Intervalo de almoço (início/fim, ex.: 1h30)
- Cidade de atuação naquele dia (pode variar dia a dia)

### RN02 — Cidade origem x cidade de atuação
- Se a cidade de atuação do dia for **diferente** da cidade de origem do técnico, a primeira OS do dia só pode iniciar em: `hora_inicio_expediente + tempo_deslocamento(origem → atuação)`.
  - Exemplo: Téc. Felipe mora em Conchal, trabalha em Araras, deslocamento de 20–30 min → expediente às 08:30, primeira OS agendada a partir das **09:00**.
- Se a cidade de atuação for igual à origem, a primeira OS pode iniciar no próprio horário de início do expediente.

### RN03 — Deslocamento entre OS (mesma cidade)
- Tempo fixo de **15 minutos** entre o término de uma OS e o início da próxima, quando ambas estão na mesma cidade.

### RN04 — Deslocamento entre cidades diferentes
- Quando a próxima OS está em cidade diferente da anterior, usa-se o tempo definido na **matriz de deslocamento entre cidades** (cadastro configurável, não fixo em 15 min).

### RN05 — Retorno à cidade de origem no fim do dia
- Se, ao concluir as OS do dia em uma cidade diferente da origem, ainda houver tempo hábil no expediente, o sistema pode considerar o **retorno à cidade de origem** para alocar a última OS do dia lá — desde que o tempo de deslocamento de volta + duração da OS caibam antes do fim do expediente (ver RN06).
- Essa é uma regra de **otimização condicional**, avaliada pelo motor de scoring (não obrigatória).

### RN06 — Não estourar o expediente (horário-limite de início por tipo de serviço)
- Nenhuma OS pode ser agendada de forma que **hora_início + duração_estimada** ultrapasse a hora de fim do expediente.
- Além disso, cada **tipo de serviço** tem seu próprio **horário-limite de início** (não é uma margem fixa genérica, e sim um atributo configurável por tipo de serviço). Exemplos informados pelo negócio, considerando expediente até às 18:00:
  - **Instalação / Mudança de endereço** → horário-limite de início: **16:45**.
  - **Demais tipos de serviço (mais simples/rápidos)** → horário-limite de início: **17:00**.
- Esse valor (`HoraLimiteInicio`) é cadastrado por tipo de serviço (ver seção 9, entidade `TipoServico`) e pode futuramente receber exceção por cidade, se necessário (mesma lógica de `TipoServicoExcecao`).
- O motor de agendamento deve validar: `hora_início_candidata <= HoraLimiteInicio do tipo de serviço`, além de garantir que a OS termine antes do fim do expediente (considerando também o deslocamento de retorno à cidade de origem, quando aplicável — RN05).

### RN07 — Intervalo de almoço
- Nenhuma OS pode ser agendada sobrepondo o intervalo de almoço do técnico. Se o horário calculado invadir o almoço, o sistema desloca o início da OS para depois do término do almoço.

### RN08 — Tempo estimado do serviço
- Cada **tipo de serviço** tem um tempo estimado padrão.
- Pode haver **override** desse tempo por cidade/região ou por descrição/complexidade da OS (tabela de exceções consultada antes de usar o padrão).

### RN09 — SLA
- Toda OS tem uma data/hora-limite de atendimento (SLA). OS mais próximas de estourar o SLA têm **prioridade** no processamento do motor de agendamento.

### RN10 — Disponibilidade do cliente
- A OS pode conter uma ou mais janelas de datas/horários em que o cliente está disponível. O slot sugerido deve necessariamente estar contido em uma dessas janelas.

### RN11 — Aprovação humana obrigatória
- Nenhum agendamento gerado automaticamente é considerado confirmado. Todo slot sugerido fica com status **"Pendente de Aprovação"** até que um usuário aprove, ajuste ou rejeite.

---

## 7. Requisitos Funcionais (RF)

| ID | Requisito |
|---|---|
| RF01 | O sistema deve importar/sincronizar ordens de serviço pendentes do Hubsoft (via API/integração). |
| RF02 | O sistema deve importar/sincronizar a agenda-base dos técnicos do Hubsoft (dias e horários de trabalho, ausências). |
| RF03 | O sistema deve permitir cadastro manual/complementar de disponibilidade por técnico e por dia (data, hora início, hora fim, almoço, cidade de atuação do dia). |
| RF04 | O sistema deve permitir CRUD de cidades. |
| RF05 | O sistema deve permitir CRUD da matriz de tempo de deslocamento entre pares de cidades. |
| RF06 | O sistema deve permitir CRUD de tipos de serviço, com tempo estimado padrão e SLA padrão. |
| RF07 | O sistema deve permitir cadastro de exceções de tempo estimado por tipo de serviço + cidade/região. |
| RF08 | O sistema deve permitir vincular um técnico a uma cidade de origem. |
| RF09 | O motor de agendamento deve calcular, para cada OS pendente, os melhores técnicos/slots candidatos, aplicando as regras de negócio (seção 6). |
| RF10 | O sistema deve gerar um score/ranking dos slots candidatos e apresentar ao menos a melhor sugestão (idealmente top 3) para cada OS. |
| RF11 | O sistema deve expor uma tela/fila de "Agendamentos Pendentes de Aprovação". |
| RF12 | O usuário deve poder aprovar, editar manualmente (trocar técnico/horário) ou rejeitar (reenviar para reprocessamento) uma sugestão. |
| RF13 | O sistema deve manter um log/auditoria de cada decisão de agendamento (automática ou manual), incluindo o motivo/score. |
| RF14 | O sistema deve expor uma tela de calendário por técnico mostrando disponibilidade e OS já agendadas. |
| RF15 | O sistema deve permitir consultar/gerenciar a disponibilidade dos técnicos (aba de disponibilidade: técnico, cidade, dias, horários, almoço). |
| RF16 | (Opcional/Fase 2) O sistema deve poder consultar um serviço de IA externo para desempatar/sugerir quando houver múltiplos cenários viáveis com scores próximos, retornando também uma justificativa textual. |
| RF17 | O sistema deve permitir reprocessar o agendamento de uma OS específica sob demanda. |
| RF18 | O sistema deve notificar (painel/e-mail) o coordenador quando houver OS perto de estourar o SLA sem agendamento aprovado. |
| RF19 | O importador de ordens deve considerar **somente** OS com status "Aguardando Agendamento" no Hubsoft; ordens com agendamento já definido devem ser ignoradas. |
| RF20 | A camada de integração com o Hubsoft não deve implementar nenhuma operação de edição/cancelamento sobre agendamentos já existentes — apenas leitura de "Aguardando Agendamento" e, futuramente, criação de novos agendamentos aprovados. |

---

## 8. Requisitos Não Funcionais (RNF)

| ID | Requisito |
|---|---|
| RNF01 | API desenvolvida em C# / .NET 8, arquitetura em camadas (Clean Architecture / DDD leve). |
| RNF02 | Banco de dados PostgreSQL, acesso via Entity Framework Core. |
| RNF03 | Processos de sincronização e o motor de agendamento devem rodar como jobs em background (ex.: Hangfire ou Quartz.NET), não bloqueando requisições da API. |
| RNF04 | O motor de agendamento deve ser capaz de reprocessar um volume diário de OS em tempo hábil (definir SLA técnico, ex.: < 1 min para lote de N ordens). |
| RNF05 | Toda decisão automática deve ser auditável (quem, quando, por quê). |
| RNF06 | O sistema deve ser resiliente a falhas/indisponibilidade temporária da API do Hubsoft (retry, fila). |
| RNF07 | Autenticação/autorização por perfil de usuário (ex.: JWT, roles: Admin, Coordenador, Visualizador). |
| RNF08 | Logs estruturados (ex.: Serilog) e monitoramento de erros. |
| RNF09 | Configurações de regras de negócio (margem de segurança, tempo padrão entre OS, etc.) devem ser parametrizáveis, não *hardcoded*. |

---

## 9. Modelo de Domínio (Entidades Principais)

```
Cidade
 - Id
 - Nome
 - Estado

MatrizDeslocamento
 - Id
 - CidadeOrigemId (FK Cidade)
 - CidadeDestinoId (FK Cidade)
 - TempoMedioMinutos

Tecnico (Equipe)
 - Id
 - Nome
 - HubsoftId
 - CidadeOrigemId (FK Cidade)
 - Ativo

DisponibilidadeTecnico
 - Id
 - TecnicoId (FK Tecnico)
 - Data
 - HoraInicio
 - HoraFim
 - AlmocoInicio
 - AlmocoFim
 - CidadeAtuacaoId (FK Cidade)
 - Origem (Hubsoft | Manual)

TipoServico
 - Id
 - Nome
 - TempoEstimadoPadraoMinutos
 - SlaPadraoHoras
 - HoraLimiteInicio (ex.: 16:45 para Instalação/Mudança de Endereço, 17:00 para os demais)

TipoServicoExcecao
 - Id
 - TipoServicoId (FK TipoServico)
 - CidadeId (FK Cidade, nullable)
 - TempoEstimadoMinutos
 - HoraLimiteInicio (nullable — sobrescreve o padrão do tipo de serviço para essa cidade, se necessário)
 - Observacao

OrdemServico
 - Id
 - HubsoftId
 - ClienteNome
 - Endereco
 - CidadeId (FK Cidade)
 - TipoServicoId (FK TipoServico)
 - DataLimiteSla
 - Status (PendenteAgendamento | SugestaoGerada | AguardandoAprovacao | Agendada | Concluida | Cancelada)
 - JanelaDisponibilidadeCliente (1:N com JanelaCliente)

JanelaClienteDisponibilidade
 - Id
 - OrdemServicoId (FK OrdemServico)
 - DataInicio / DataFim
 - HoraInicio / HoraFim

Agendamento
 - Id
 - OrdemServicoId (FK OrdemServico)
 - TecnicoId (FK Tecnico)
 - Data
 - HoraInicio
 - HoraFim
 - Score
 - OrigemDecisao (Automatico | AutomaticoComIA | Manual)
 - StatusAprovacao (Pendente | Aprovado | Ajustado | Rejeitado)
 - AprovadoPorUsuarioId
 - DataAprovacao
 - Justificativa (texto, especialmente quando IA é usada)

LogDecisaoAgendamento
 - Id
 - AgendamentoId (FK Agendamento)
 - CandidatosAvaliados (JSON: lista de técnicos/slots e scores)
 - DataProcessamento
```

---

## 10. Motor de Agendamento — Proposta de Algoritmo

### 10.1 Abordagem
Para o MVP, recomenda-se uma **heurística de scoring guloso (greedy) com ordenação por prioridade de SLA**, evoluindo depois — se necessário — para uma otimização mais robusta (ex.: adaptação de problema de roteirização — VRP simplificado) ou uso pontual de IA generativa para desempate/justificativa.

### 10.2 Fluxo geral
1. Ordenar as OS pendentes por **proximidade do SLA** (mais urgente primeiro).
2. Para cada OS, filtrar técnicos candidatos:
   - Atendem a cidade da OS (cidade de atuação no dia = cidade da OS, ou custo de deslocamento aceitável).
   - Têm disponibilidade em algum dia dentro da janela do cliente.
   - Têm o tipo de serviço dentro do seu escopo (se aplicável).
3. Para cada técnico candidato, simular a inserção da OS na agenda do dia (respeitando RN01–RN07: deslocamento, almoço, expediente, retorno à origem).
4. Calcular o **score** do slot candidato considerando, por exemplo:
   - Peso alto: folga até o SLA (quanto mais próximo do limite, maior a urgência/prioridade).
   - Peso médio: menor deslocamento total gerado na rota do técnico naquele dia.
   - Peso médio: menor tempo ocioso resultante na agenda.
   - Peso baixo: preferência por manter o técnico na mesma cidade ao longo do dia (evitar zigue-zague entre cidades).
5. Selecionar o(s) melhor(es) slot(s) (top 1 a 3) por score.
6. Se houver **empate técnico** (scores muito próximos) ou ambiguidade relevante (ex.: dois técnicos igualmente bons, mas com perfis de OS diferentes no dia), acionar o **módulo de IA** passando o contexto estruturado (candidatos, scores, restrições) para obter uma recomendação com justificativa textual.
7. Gravar o(s) resultado(s) como `Agendamento` com status **AguardandoAprovacao**, junto ao log de candidatos avaliados.
8. Notificar a fila de aprovação.

### 10.3 Papel da IA (opcional, RF16)
A IA **não decide sozinha**: ela recebe um payload estruturado (lista de candidatos, restrições, scores já calculados) e é usada para:
- Desempatar cenários muito próximos.
- Gerar uma explicação em linguagem natural do porquê daquela sugestão (facilita a aprovação humana).
- Sinalizar riscos não óbvios (ex.: "esse técnico já está no limite do expediente, considere reagendar o horário de almoço").

A decisão estruturada final continua sendo persistida em campos determinísticos (`TecnicoId`, `Data`, `HoraInicio`), garantindo que o sistema não dependa de uma IA para funcionar.

---

## 11. Fluxo de Aprovação Manual

1. Motor gera sugestão → `Agendamento` com status `Pendente`.
2. Coordenador acessa a tela **"Fila de Aprovação"**, vendo: OS, cliente, cidade, técnico sugerido, data/hora, score, justificativa.
3. Ações possíveis:
   - **Aprovar** → status `Agendado`, confirma na agenda do técnico, (fase 2: envia confirmação ao Hubsoft).
   - **Ajustar manualmente** → coordenador escolhe outro técnico/horário; sistema revalida as regras de negócio (RN01–RN10) antes de permitir salvar; status vira `Ajustado/Aprovado`.
   - **Rejeitar** → OS volta para fila de reprocessamento, podendo excluir o técnico sugerido da próxima rodada.

---

## 12. Tela — Gestão de Disponibilidade de Técnicos (RF15)

Funcionalidade dedicada para o coordenador configurar/ajustar a disponibilidade, complementando o que vem do Hubsoft:

- Seleção do técnico.
- Calendário mensal/semanal com marcação de dias disponíveis.
- Por dia selecionado: hora início, hora fim, intervalo de almoço, cidade de atuação do dia.
- Indicação visual de qual cidade é a "origem" do técnico vs. cidade de atuação daquele dia (quando diferentes).
- Exemplo de cadastro (conforme especificado pelo usuário):
  > Téc. Caio — Mogi Guaçu — disponível dias 10, 11, 12, 14, 15 — 08:30 às 18:00 — almoço de 1h30.
  > Téc. Felipe — origem Conchal, atua em Araras — deslocamento 20–30 min.

---

## 13. Integração com Hubsoft

### 13.1 Regra crítica — não alterar nada que já existe no Hubsoft
O sistema **nunca escreve, edita ou reagenda** algo que já está definido/confirmado no Hubsoft. O escopo de escrita (quando existir) se limita **exclusivamente** a:
- **Importar (leitura)** ordens de serviço que estão com status **"Aguardando Agendamento"** (ou equivalente) no Hubsoft — apenas essas entram no fluxo do motor de agendamento.
- Ordens que já possuem um agendamento definido no Hubsoft são **ignoradas** pelo importador (não entram na fila, não são candidatas a reprocessamento, e o sistema não possui nenhuma operação de update/delete sobre elas).
- Isso deve virar um requisito explícito de filtro na sincronização (RF01) e uma validação defensiva na camada de integração (nunca chamar endpoints de escrita/edição do Hubsoft sobre uma ordem que não veio como "Aguardando Agendamento").
- **RF19 (novo):** O importador de ordens deve filtrar e trazer somente OS com status "Aguardando Agendamento"; qualquer OS com agendamento já definido no Hubsoft deve ser desconsiderada pelo motor.
- **RF20 (novo):** A camada de integração não deve implementar nenhuma operação de alteração/cancelamento de agendamentos existentes no Hubsoft nesta fase — apenas leitura (e, futuramente, criação de novos agendamentos aprovados, se a API permitir).

### 13.2 Fase atual — protótipo via JSON manual
Como a documentação/API do Hubsoft ainda não está disponível, a integração real fica postergada. Para viabilizar o desenvolvimento e os testes do motor de agendamento desde já, o projeto seguirá com um **protótipo baseado em arquivos/payloads JSON de entrada simulando os dados do Hubsoft**:
- Um "mock" de ordens de serviço (`ordens_aguardando_agendamento.json`) representando o retorno esperado da API de ordens.
- Um "mock" de agenda-base dos técnicos (`agenda_tecnicos.json`) representando o retorno esperado da API de disponibilidade.
- A camada de integração deve ser desenhada com uma **interface/abstração** (ex.: `IHubsoftOrdemServicoProvider`, `IHubsoftAgendaProvider`) para que a implementação "mock via JSON" seja simplesmente substituída, no futuro, por uma implementação real de chamada HTTP à API do Hubsoft — sem impacto no motor de agendamento nem no restante do sistema.
- Isso também facilita testes automatizados do motor de agendamento de forma determinística, sem dependência externa.

### 13.3 Quando a API real estiver disponível
- **Entrada:** ordens de serviço com status "Aguardando Agendamento"; agenda-base/ausências dos técnicos.
- **Frequência sugerida:** sincronização periódica (ex.: a cada 15–30 min) + possibilidade de gatilho manual.
- **Saída (fase 2, se aplicável):** após aprovação, criação do agendamento no Hubsoft para a OS correspondente (apenas criação de um novo agendamento para uma OS que estava "Aguardando Agendamento" — nunca edição de um agendamento pré-existente).
- Deve-se mapear os campos reais da API do Hubsoft (endpoint de ordens e de agenda) para as entidades internas — **recomenda-se validar a documentação da API do Hubsoft** assim que disponível, pois pode haver particularidades (paginação, autenticação, rate limit) que impactam o RNF06.

---

## 14. Arquitetura Proposta (alto nível)

```
┌───────────────────────────────┐
│           API (.NET)          │  ← Controllers / Endpoints REST
├───────────────────────────────┤
│      Camada de Aplicação      │  ← Casos de uso, orquestração
├───────────────────────────────┤
│        Motor de Agendamento   │  ← Regras de negócio + scoring
├───────────────────────────────┤
│         Domínio (DDD)         │  ← Entidades, regras invariantes
├───────────────────────────────┤
│   Infraestrutura / EF Core    │  ← PostgreSQL
├───────────────────────────────┤
│  Integrações (Hubsoft, IA)    │  ← HttpClients dedicados
└───────────────────────────────┘

Jobs em background (Hangfire/Quartz):
 - Sync Hubsoft (ordens + agendas)
 - Execução do motor de agendamento
 - Verificação de SLA próximo do limite
```

---

## 15. Casos de Uso Principais (resumo)

1. **UC01** — Sincronizar ordens de serviço do Hubsoft.
2. **UC02** — Sincronizar agenda-base dos técnicos do Hubsoft.
3. **UC03** — Cadastrar/editar disponibilidade manual de um técnico.
4. **UC04** — Cadastrar cidade e matriz de deslocamento.
5. **UC05** — Cadastrar tipo de serviço e exceções de tempo estimado.
6. **UC06** — Executar motor de agendamento (automático, em lote ou sob demanda por OS).
7. **UC07** — Consultar fila de aprovação.
8. **UC08** — Aprovar / ajustar / rejeitar agendamento sugerido.
9. **UC09** — Visualizar calendário de um técnico.
10. **UC10** — Consultar log/auditoria de decisões.

---

## 16. Riscos e Pontos de Atenção

| Risco | Impacto | Mitigação |
|---|---|---|
| API do Hubsoft ainda não documentada/disponível | Alto | Desenvolvimento inicia com protótipo via mocks JSON e camada de integração abstraída (interfaces), permitindo trocar por chamadas reais sem impactar o motor de agendamento. |
| Filtro incorreto de status pode trazer ordens já agendadas no Hubsoft para o motor | Alto | Validação explícita (RF19/RF20) e teste automatizado garantindo que somente "Aguardando Agendamento" entra no fluxo; nenhuma operação de escrita/edição sobre agendamentos existentes. |
| Regras de negócio conflitantes em casos extremos (ex.: técnico sem nenhum slot viável) | Médio | OS sem candidato viável deve cair em fila de "Exceção Manual" com alerta ao coordenador. |
| Dependência de IA externa para desempate (custo, latência, indisponibilidade) | Médio | IA é opcional/fase 2; sistema deve funcionar 100% sem ela via heurística de score. |
| Alterações frequentes na agenda dos técnicos após aprovação | Médio | Necessário fluxo de reagendamento/cancelamento (detalhar em fase 2). |
| Volume de OS pendentes gerando gargalo no motor | Médio | Processamento em lote assíncrono, com fila (ex.: RabbitMQ) se volume crescer. |

---

## 17. Próximos Passos Sugeridos

1. Definir a estrutura dos JSONs mock de "Ordens Aguardando Agendamento" e "Agenda de Técnicos" (formato de protótipo, seção 13.2) para já iniciar o desenvolvimento do motor sem depender da API real.
2. Modelar o schema definitivo do PostgreSQL a partir da seção 9 (incluindo `HoraLimiteInicio` por tipo de serviço).
3. Definir o modelo de scoring (pesos) da seção 10 com o time de operações.
4. Prototipar a tela de Fila de Aprovação e a tela de Disponibilidade de Técnicos (wireframes).
5. Definir se a integração com IA (RF16) entra no MVP ou fica para fase 2.
6. Planejar sprints (sugestão): Sprint 1 — cadastros base (cidade, deslocamento, tipos de serviço com hora-limite, técnicos); Sprint 2 — camada de integração abstraída + mocks JSON (RF19/RF20); Sprint 3 — motor de agendamento; Sprint 4 — fila de aprovação + calendário de disponibilidade; Sprint 5 — ajustes finos, IA (se aplicável) e testes de carga.
7. Quando a documentação da API real do Hubsoft estiver disponível, mapear os campos e substituir a implementação mock pela implementação HTTP real (sem alterar o motor de agendamento), reforçando sempre a regra de nunca escrever sobre agendamentos pré-existentes (RN — seção 13.1).

---

*Documento inicial de análise de requisitos — sujeito a revisão conforme validação com o time de operações e confirmação das particularidades da API do Hubsoft.*