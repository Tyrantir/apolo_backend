# CLAUDE.md

Backend do sistema de chat de suporte multiempresa (SaaS). Arquitetura completa, decisões e roadmap: `chat-suporte-arquitetura.md`. Plano de chamadas (fase 2, WebRTC/SIP/WhatsApp Calling): `chat-suporte-frontend-plano.md` §11 (ainda não existe neste repo).

## Comunicação
- Português. Mínimo de palavras. Objetivo.
- Sem introdução, resumo repetido ou elogio. Resposta direta, detalhe só se necessário.
- Dúvida real → perguntar. Decisão óbvia → decidir e informar em uma linha.

## O que é
Monólito modular Java 21+ / Spring Boot. Multi-tenancy por **schema PostgreSQL** (`tenant_<slug>` + `public` para dados da plataforma). Tempo real via WebSocket/STOMP (widget, painel do atendente, painel admin). Redis para presença/locks/cache; RabbitMQ (`StompBrokerRelay`) quando houver mais de uma instância; S3/MinIO para anexos.

Módulos (pacote com fronteira clara, comunicação por serviço e evento de domínio — ver §5 e §14 da arquitetura):
`tenancy`, `identity`, `channel`, `contact`, `triage`, `routing`, `conversation`, `agent`, `schedule`, `sla`, `feedback`, `reporting`, `attachment`, `shared`.

Fase 2 soma **chamadas** (voz/vídeo pelo widget, telefone via SIP, WhatsApp Calling, URA): reaproveita o motor de `triage` e as filas de `routing`, servidor de mídia (ex.: LiveKit) fica fora deste backend. Não é MVP — não desenhar `routing`/`triage` acoplado a chat sem necessidade, mas evitar decisões que fechem a porta (ex.: `conversation` genérica o bastante para representar uma chamada, não só mensagens).

Projeto ainda não scaffolded (sem `pom.xml`/código). Antes de codar qualquer módulo, confirmar que a decisão correspondente em `chat-suporte-arquitetura.md` §2 e §16 (pontos em aberto) já foi tomada.

## Comandos
```bash
mvn clean install
mvn test
mvn spring-boot:run
```
(válidos a partir do momento em que o projeto Maven existir — ver roadmap Fase 1, §15)

## Regras críticas
- **Nome do schema do tenant**: `tenant_` + slug validado por regex (`[a-z0-9_]`, até 50 chars) **antes de qualquer SQL**. Nunca concatenar texto do usuário em `SET search_path`.
- **`MultiTenantConnectionProvider.releaseConnection`** sempre faz `SET search_path TO public` antes de devolver a conexão ao pool — é o maior risco do modelo multi-tenant (vazamento de dados entre empresas).
- **`TenantContext` (ThreadLocal/ScopedValue) limpo no `finally`** de toda requisição, mensagem STOMP e job agendado.
- **Código assíncrono não herda o tenant** (`@Async`, `CompletableFuture`, listeners) — propagar explicitamente, ex. via `TaskDecorator`.
- Queries nativas / `JdbcTemplate` não passam pelo Hibernate: usar `DataSource` decorado com o mesmo `search_path`, ou só acessar o banco via JPA.
- **Migrações Flyway** (`db/public`, `db/tenant`) idempotentes e compatíveis com a versão anterior (expand/contract). Falha num tenant marca `MIGRACAO_PENDENTE` e não trava os demais.
- **Distribuição de conversa**: nunca duas instâncias atribuem a mesma conversa ou excedem a capacidade do atendente — `SELECT ... FOR UPDATE SKIP LOCKED` na fila, lock Redis ou `UPDATE` condicional na reserva do atendente.
- **Mensagens são persistidas antes de publicar** no WebSocket; reconexão sincroniza pelo último `id` recebido.
- **`ChannelInterceptor` valida tenant e permissão no `SUBSCRIBE`** de cada tópico STOMP — nunca confiar no destino enviado pelo cliente.
- **Nunca commitar segredo**: `segredo_assinatura` de canal, credenciais de banco/Redis/S3 e chaves JWT só em config local (`application-local.yml`, `.env`), gitignored.

## Onde mexer
| Tarefa | Módulo |
|---|---|
| Onboarding de tenant, resolução de schema, migração | `tenancy` |
| Usuários, papéis, JWT | `identity` |
| Widget, embed assinado, WhatsApp (fase 2) | `channel` |
| Cadastro do cliente final | `contact` |
| Editor/motor de fluxo de triagem | `triage` |
| Filas, estratégias de distribuição, capacidade | `routing` |
| Conversa, mensagem, transferência, encerramento | `conversation` |
| Presença, status, pausa do atendente | `agent` |
| Horário de atendimento, feriados | `schedule` |
| Política de SLA, prazos, alertas | `sla` |
| CSAT/NPS | `feedback` |
| TME, TMA, abandono, relatórios | `reporting` |
| Upload/download via URL assinada | `attachment` |

## Código
- Seguir o padrão do módulo vizinho. KISS, YAGNI.
- Respeitar a fronteira dos módulos (comunicação por serviço/evento, não acesso direto a repositório de outro módulo) — Spring Modulith ajuda a verificar.
- Comentar o porquê, não o quê.
- Build verde antes de commit.

## Git (Claude decide e executa)
- Claude cria branch, commita, faz push e abre PR por conta própria. **Nunca perguntar se pode commitar** — commitar ao concluir cada mudança lógica e informar o hash em uma linha.
- Nunca commitar direto na `main`. Branch: `feat/`, `fix/`, `refactor/`, `chore/`, `docs/`, `ci/` + kebab-case curto.
- Conventional Commits em português (`feat(routing): ...`). Commits pequenos, um por mudança lógica.
- PR via `gh pr create` contra `main`: título = conventional commit; corpo = o quê, por quê, como testar.
- Branch curta (1–3 dias), sempre a partir de `main` atualizada. Um assunto por PR.
- PR abre com a label `claude-merge` → fluxo automático: `Code Review` + `Build` → `claude-fix` aplica ajustes (máx. 2 rodadas) → `Claude Merge` faz squash e apaga a branch.
- `Claude Merge` exige: Build e Code Review verdes no head, nenhuma thread aberta, sem conflito. Tocou caminho de `.github/sensiveis.txt` (config local, migrações, `tenancy/`, `identity/`, `.github/`) → exige aprovação humana.
- Sem branch protection (plano free do GitHub): a regra é o fluxo acima. Nunca push direto na `main`.
- Merge manual, force-push e exclusão de branch remota fora do workflow: só com pedido explícito.
