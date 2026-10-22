# NEXUS CONTROL

Central de gestão e monitoramento de sites para Windows. Monorepo com:

- `apps/desktop` — aplicativo Windows (Electron + React + TypeScript)
- `apps/api` — backend/API (Node + Fastify + PostgreSQL)
- `apps/api/src/monitor` — serviço de monitoramento (Fase 2): mesmo código da API, mas roda como processo separado (`npm run monitor:prod`) para ficar ligado 24/7 na nuvem

Arquitetura e decisões: [docs/ARQUITETURA.md](docs/ARQUITETURA.md) · Hospedagem e custos: [docs/HOSPEDAGEM.md](docs/HOSPEDAGEM.md)

## Status — Fases 1 e 2 concluídas

> **Isto ainda não é um produto pronto para produção.** Faltam alertas por e-mail, relatórios e gestão comercial (Fase 3) e a implantação na nuvem (Fase 4). A lista completa está no final.

## Executar localmente (sem instalar PostgreSQL)

Requisitos: Node.js 20+ (testado em 24).

```bash
npm install
```

1. Configure a API (opcional; o padrão já usa o banco embutido de desenvolvimento):

   ```bash
   copy .env.example apps\api\.env
   ```

2. Crie o administrador (a senha é pedida sem eco, ou use a variável `ADMIN_PASSWORD`):

   ```bash
   npm run create-admin -w @nexus/api -- --email voce@empresa.com --name Voce
   ```

3. Suba a API (aplica as migrações sozinha em desenvolvimento):

   ```bash
   npm run api:dev
   ```

4. Em outro terminal, abra o app:

   ```bash
   npm run desktop:dev
   ```

   Entre com o e-mail/senha criados. O servidor padrão é `http://127.0.0.1:3333`.

### Banco de dados

| Ambiente | `DATABASE_URL` |
|---|---|
| Desenvolvimento/testes | `pglite://./.data/dev` (Postgres embutido em WASM; **recusado em produção**) |
| Produção / PostgreSQL local | `postgres://usuario:senha@host:5432/nexus?sslmode=require` |

Migrações ficam em `apps/api/migrations/*.sql` e são aplicadas em ordem.
Em produção: `npm run build -w @nexus/api` e depois `npm run migrate:prod -w @nexus/api` (a API **não** migra sozinha em produção, salvo `AUTO_MIGRATE=true`).

## Testes

```bash
npm run api:test      # 120 testes: SSRF, checker HTTP (servidor local real), máquina de estados, runner, incidentes, estatísticas, auth, CRUD
npm run typecheck     # API + desktop
```

## Gerar o executável e o instalador Windows

```bash
npm run desktop:dist
```

Saída em `apps/desktop/release/`: instalador NSIS (`NEXUS CONTROL Setup x.y.z.exe`, com atalho no desktop e no menu Iniciar) e versão portátil. O ícone é gerado por `apps/desktop/scripts/make-icon.mjs` (substitua `build/icon.png` por sua arte, 512×512, quando quiser).
`npm run dist` baixa o pacote `winCodeSign` do electron-builder, que contém links simbólicos: no Windows isso exige **Modo Desenvolvedor ligado ou terminal como administrador** (senão falha com "Cannot create symbolic link"). Sem esse privilégio use `npm run dist:local -w @nexus/desktop`, que pula a etapa de edição/assinatura do `.exe` — o instalador e o app funcionam, mas o arquivo `.exe` em si fica com o ícone padrão do Electron. Para a versão de distribuição, gere com o privilégio (ou em CI) para embutir o ícone.
O instalador **não está assinado**: o Windows SmartScreen exibirá aviso até haver um certificado de assinatura de código (necessário também para atualizações automáticas confiáveis).

## Segurança — o que fazer antes de colocar em produção

- Servir a API **somente por HTTPS** (proxy reverso com certificado) e `TRUST_PROXY=true` atrás dele.
- Nunca versionar `.env`; segredos ficam nas variáveis do provedor.
- `ADMIN_PASSWORD` só para criar o primeiro usuário; troque-a depois.
- Fazer backup do PostgreSQL e testar a restauração (procedimento será documentado na Fase 4).

## Nota sobre antivírus (Windows)

Em algumas máquinas o antivírus bloqueia o binário nativo do Rollup (falso positivo). O projeto já usa o substituto WASM oficial (`overrides` no `package.json` raiz), então nada precisa ser desativado.

## Recursos

**Implementados (Fases 1 e 2, verificados por testes automatizados e teste ponta a ponta no Electron real com monitor verificando sites reais):**
login/logout com sessões revogáveis · papéis admin/operador/leitor · gestão de usuários (admin) · clientes (ficha completa, CNPJ validado, busca, filtros, arquivar/reativar sem perder histórico) · sites (todos os campos pedidos, vários por empresa, endpoints adicionais, resultado HTTP esperado, intervalo, limites de latência e tentativas configuráveis, pausar/arquivar) · guarda SSRF no cadastro · auditoria · pesquisa global (Ctrl+K) · painel com contagens reais · tema escuro/claro · indicador de conexão · configuração do servidor · iniciar com o Windows · instalador Windows.

**Pendentes:**
- Fase 2 (parcial): falta monitoramento multi-região (o campo região já existe) e extrair o monitor para pacote próprio; alertas por e-mail dependem da Fase 3.
- Fase 3: notificações (e-mail, in-app; Telegram/Discord futuro), relatórios PDF/CSV, gestão comercial (MRR, contratos, pagamentos manuais), filtros por período, visão de auditoria por entidade.
- Fase 4: testes de integração do monitor, implantação na nuvem + HTTPS/domínio, backup/restauração documentados, atualizações automáticas assinadas, múltiplas regiões, expiração/rotação de sessões no desktop, 2FA.
- Notificações dentro do app, atalhos de teclado adicionais e acessibilidade completa ainda não foram revisados.

## Fase 2 — Monitoramento (como funciona)

- **Verificação HTTP/HTTPS real** de cada site (URL principal + endpoints adicionais), no intervalo configurado por site. Mede o tempo até a resposta; **não** mede o carregamento completo da página nem a experiência de todos os usuários.
- **Classificação por verificação:** `ok`, `slow` (acima do limite de atenção), `down` (timeout, DNS, TLS, conexão), `http_error` (status fora do esperado) e `inconclusive` (o monitor não conseguiu medir, ex.: alvo bloqueado). Códigos 4xx/5xx só são falha se não estiverem na lista de resultados esperados do site.
- **Sem falso positivo:** uma falha isolada deixa o site como "Inconclusivo". O incidente só abre após N falhas consecutivas (padrão 2, configurável por site), com rechecagem a cada 30 s. A lentidão acima do limite crítico conta como falha; entre atenção e crítico o site só fica "Lentidão".
- **Incidentes:** abertos automaticamente com início na 1ª falha, motivo técnico, notas e ações. Fecham sozinhos após 2 verificações boas seguidas, com duração calculada. Nunca são apagados. Status: aberto, em investigação, reconhecido, resolvido.
- **Segurança (SSRF):** a cada verificação o domínio é resolvido, todos os IPs são checados (privados/loopback/metadados de nuvem são bloqueados) e a conexão é feita no IP validado. Cada redirecionamento (máx. 3) é revalidado; HTTPS→HTTP é recusado.
- **Estatísticas:** atual, média, mediana, p95 (só com 20+ amostras), mínimo, máximo e disponibilidade em 24 h / 7 d / 30 d. Retenção configurável (`RETENTION_DAYS`).
- **Em desenvolvimento** o monitor roda embutido na API. **Em produção** rode `npm run monitor:prod -w @nexus/api` em um processo/host separado, com `MONITOR_EMBEDDED=false` na API; várias instâncias não verificam o mesmo site duas vezes.

## Como colocar seus projetos no controle

1. **Clientes → + Novo cliente**: preencha ao menos o nome empresarial.
2. Dentro da ficha do cliente, **+ Novo site**: nome do projeto, **URL pública** (https://…) e, se quiser, endpoints extras (ex.: `/api/health`), resultado HTTP esperado, intervalo (mínimo 60 s) e limites de latência.
3. Pronto: o monitor começa a verificar sozinho. Acompanhe em **Visão geral**, **Sites monitorados** (aba Monitoramento/Histórico) e **Incidentes**.

Observações: sites em `localhost`, IPs privados ou domínios internos **são recusados de propósito** (proteção SSRF). Para sites em homologação, use o ambiente "Homologação" com uma URL pública. Para que o monitoramento continue com o seu PC desligado, a API e o monitor precisam estar hospedados na nuvem (Fase 4).

## Versão web (um único arquivo HTML para o GitHub Pages)

```bash
npm run web:build        # gera site/index.html (interface completa, JS e CSS embutidos)
```

O site tem as mesmas telas e configurações do app. Para **salvar e compartilhar** dados ele precisa de uma API hospedada com HTTPS e `CORS_ORIGINS` apontando para o endereço do site. Passo a passo: [docs/PUBLICAR-NO-GITHUB.md](docs/PUBLICAR-NO-GITHUB.md).
