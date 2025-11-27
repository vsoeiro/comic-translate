# Agents - Migração para WebUI

## Requisitos Funcionais
- Autenticação e usuários: cadastro, login, refresh/expiração, recuperação de senha via token; papéis (admin/user) para uso de credenciais e configurações.
- Gerenciamento de projetos: criar/abrir (equivalente aos .ctpr), listar, excluir, salvar configs por projeto (idiomas, fontes, opções de OCR/tradução/inpainting).
- Upload e ingestão: upload de imagens avulsas e ZIP/CBR/CBZ; validação de tipo/tamanho; associação a projeto.
- Pipeline: disparo assíncrono por projeto ou página (detecção > OCR > tradução > inpainting > renderização); resposta imediata com `job_id`.
- Progresso e logs: status de jobs (fila, em processamento, concluído, erro), porcentagem e etapa; streaming via WebSocket/SSE para atualização em tempo real.
- Visualização e download: viewer com zoom/scroll para original e processada; download da imagem final mantendo versão original/processada.
- Configurações/credenciais: CRUD de credenciais (OpenAI, Azure, Google Vision, DeepL etc.) por usuário/workspace; validação mínima; toggle de provedores por idioma.
- Internacionalização (i18n): strings do front em arquivos de idioma; seleção de idioma herdada do perfil.
- Auditoria: registrar quem disparou jobs e quando, com status final.

## Requisitos Não Funcionais
- Arquitetura cliente-servidor: API HTTP (REST) + eventos (WebSocket/SSE); frontend em React consumindo a API.
- Persistência: SQLAlchemy; SQLite inicial; camada de repositório para troca futura para Supabase/PostgreSQL; migrações com Alembic.
- Segurança: hashing de senha (bcrypt), tokens JWT com refresh, CORS restrito, limite configurável de upload, checagem de mimetype, isolamento de diretórios por usuário/projeto.
- Escalabilidade: tarefas assíncronas (worker/ThreadPool/Celery) para não bloquear requests; filas para jobs pesados; possibilidade de mover armazenamento de arquivos para blob (ex.: Supabase Storage).
- Observabilidade: logs estruturados por request/job; métricas de duração de etapas; IDs de correlação em logs de jobs; registros armazenados localmente.
- UX: feedback em tempo real de progresso/erros; ações idempotentes (reprocessar página não duplica entradas).
- Portabilidade: comando único para subir backend + frontend (uvicorn + build React ou dev server); sem dependências de desktop/Qt.

## Detalhamento Operacional

### Autenticação e sessão
- Fluxo: `POST /auth/signup`, `POST /auth/login` -> retorna `access_token` (curto) e `refresh_token` (longo); `POST /auth/refresh` para renovar; expiração curta (~15 min) e refresh (~7-30 dias).
- Recuperação de senha: `POST /auth/forgot` gera token de reset; `POST /auth/reset` redefine senha com validade curta.
- Papéis: claims no JWT (`role: admin|user`); middlewares/autorizadores por rota (credenciais, configurações avançadas).
- Armazenamento de tokens: refresh token preferencialmente em cookie HttpOnly + Secure + SameSite=Lax; access token em memória (não persiste no storage) para reduzir superfície de ataque.

### Limites e uploads
- Tamanho máximo configurável (`MAX_UPLOAD_MB`); validação de mimetype/extensão (img: png/jpg/webp; pacotes: zip/cbr/cbz).
- Quota opcional por usuário/projeto para evitar abuso; rejeitar arquivos vazios ou com estrutura inválida. Limite fino por usuário ainda não implementado.

### Erros e respostas
- Respostas de erro usam status 4XX/5XX com payload mínimo `{ "error": "<código_curto>", "message": "<descrição_para_humano>" }`, sem stack trace ou detalhe técnico.
- SSE/WS: mensagens de erro seguem mesmo padrão de mensagem curta; reconexão automática pelo cliente.

### Contratos de API (resumo)
- Projetos: `GET/POST/DELETE /projects`, `GET/PUT /projects/{id}`; inclui idiomas/fontes/opções; paginação padrão com `page` e `page_size` (default 50, máx. 200).
- Páginas: `GET/POST/DELETE /projects/{id}/pages`, upload de imagens/ZIP; retorna metadados e links para original/processada; suporta paginação.
- Processamento: `POST /process` com `project_id` e opcional `page_ids` -> `{job_id}` imediato; permite header `Idempotency-Key` para evitar submissões duplicadas.
- Status: `GET /jobs/{job_id}` -> `status` (queued|running|done|error|canceled), `progress` 0-100, `stage` (detect|ocr|translate|inpaint|render), `message`.
- Eventos: WebSocket/SSE em `/events` filtrável por `job_id`/`project_id` emitindo `{job_id, project_id?, status, stage, progress, message, timestamp, correlation_id}`.
- Assets: `GET /assets/{page_id}?version=original|processed` para download/visualização.
- Credenciais: `GET/POST/PUT/DELETE /credentials`; validação mínima (chave não vazia, formato básico).

### Modelagem e persistência
- Entidades iniciais (SQLAlchemy + repositórios): User, Session/RefreshToken, Credential, Project, Page, Job, JobLog (opcional), ProviderConfig (por idioma/provedor).
- Relacionamentos: User 1-N Projects; Project 1-N Pages/Jobs; Job 1-N JobLogs; Credentials por User ou workspace compartilhado (flag).
- Campos-chave: Job guarda `status`, `stage`, `progress`, `started_at`, `finished_at`, `error`; Page guarda caminhos original/processado e idioma configurado.
- Migrações com Alembic; seeds opcionais para criar usuário admin.
- Armazenamento de arquivos e logs em `/data/<user>/<project>`; não guardar arquivos binários no banco. Logs persistem localmente em `/data/logs`.

### Assíncrono e fila
- Fila padrão: `queue.Queue` + `ThreadPoolExecutor` (1-2 workers) para MVP; persistência de progresso em Job/JobLog. Ponto de extensão para Celery/RQ no futuro.
- Pipeline por página: detect -> ocr -> translate -> inpaint -> render; cada etapa atualiza `stage` e `progress` e publica evento SSE/WS.
- Idempotência: reprocessar página reutiliza registro e sobrescreve caminho processado sem duplicar entradas.
- Confiabilidade: retry automático de cada etapa no máximo 1x adicional (total 2 tentativas) com timeout padrão de 60s por etapa.
- Cancelamento: aceitar solicitação de cancelamento de job; marcar status `canceled`, interromper tarefas pendentes e não re-enfileirar.
- Limite de concorrência e quotas por usuário ainda não aplicados; usar fila global simples.

### Segurança prática
- Hash de senha com bcrypt; JWT com rotas protegidas e CORS limitado a origens configuradas.
- Isolamento de diretórios de trabalho por usuário/projeto; limpeza de uploads temporários.
- Validação de mimetype antes de salvar; limites de tamanho e contagem de páginas em ZIP.
- Armazenamento de chaves/credenciais criptografado ou ofuscado em repouso; masking em logs.
- Preferir TLS/HTTPS e rate limiting básico em rotas sensíveis (auth, upload).

### Observabilidade
- Logs estruturados por request e por job com `correlation_id`, gravados em `/data/logs`.
- Métricas: duração por etapa do pipeline, contagem de erros por provedor, tamanho médio de upload.
- Eventos de auditoria para disparo de jobs e mudanças de credenciais; retenção local em `/data`.

### Frontend (React) – fluxos principais
- Auth: telas de login/signup, reset; guarda tokens e renova via refresh.
- Dashboard: lista projetos, cria/exclui, abre detalhes.
- Upload: seleciona projeto, envia imagens ou ZIP, mostra validação e progresso de upload.
- Viewer: lista páginas, alterna original/processada, zoom/scroll, download.
- Monitor de jobs: tabela/cartões com status, etapa e progresso; streaming via SSE/WS com reconexão.
- Configurações: cadastro de credenciais por provedor/idioma, toggles de provedores ativos, preferências de idioma da interface.

### Deploy e operação local
- Comando alvo: `uv run`/`uvicorn app.main:app --reload` para backend + build React servido como estático atrás de proxy reverso (ou `npm run dev` em dev).
- Variáveis de ambiente sugeridas: `DATABASE_URL` (sqlite:///./app.db), `JWT_SECRET`, `JWT_EXP_MIN`, `JWT_REFRESH_EXP_DAYS`, `ALLOWED_ORIGINS`, `MAX_UPLOAD_MB`, `WORKDIR_BASE`, `ENABLE_INPAINT`, `ENABLE_TRANSLATE`, chaves de provedores (`OPENAI_API_KEY`, etc.). Definir defaults seguros no código.
- Armazenamento de arquivos local em `WORKDIR_BASE/<user>/<project>` com versões original/processada; caminho configurável para futura troca por blob storage. `WORKDIR_BASE` padrão em `/data`.

### Decisões tomadas
- Fila/worker: usar `queue.Queue` + `ThreadPoolExecutor` (1-2 workers) no início; interface de serviço isolada para plugar Celery/RQ/Redis depois.
- SSE/WS: payload `{job_id, project_id?, status, stage, progress, message, timestamp, correlation_id}`; cliente reconecta expondo `Last-Event-ID` e retoma polling via `GET /jobs/{job_id}` em caso de falha.
- Refresh tokens: um token ativo por sessão; refresh gera novo token e invalida o anterior (tabela RefreshToken com `revoked_at`). Logout revoga.
- Confiabilidade do pipeline: retry por etapa em 1 tentativa adicional e timeout de 60s; cancelamento suportado; sem throttling por usuário nesta fase.
- Quotas: simples/desligadas por padrão; apenas `MAX_UPLOAD_MB` global e validação de contagem de páginas no ZIP. Quota fina por usuário/projeto pode ser adicionada depois.
