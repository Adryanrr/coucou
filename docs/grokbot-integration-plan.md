# Plano de integração — Grok Bot no Coucou

Documento de planejamento. Não implementa a integração.

Escrito em 27 de novembro de 2026 contra o `main` do upstream
[Louis-CFM/coucou](https://github.com/Louis-CFM/coucou) em
`202082ee0204bbc4fb78d8178cea3f0c6fcc34e4` (« Homebrew cask: 0.6.2 »,
24 de novembro de 2026). Os caminhos de código abaixo são os desse
commit, já presentes neste branch porque ele está em cima do merge
do PR de sync (fork `main` ainda não recebeu esse merge). O upstream
substituiu a época antiga do fork por `PillCatalog` + roteamento em
`HookServer`. O Orca foi reencaixado ao lado disso (`agent_orca`,
`OrcaService` → `AgentEventRouter`). Não usar esse socket como modelo
para o Grok Bot: o Grok Bot não documenta um RPC local equivalente.

## Objetivo

Fazer os agentes do Grok Bot aparecerem no notch e na barra de menus
quando pedem autorização, e, se a superfície permitir, conversar com
eles dali. O Grok Bot é o app de desktop com vários Bots (conta
Cursor, app também no iPhone). O Coucou já observa Claude Code,
Codex, Cursor, Gemini CLI, Antigravity, Copilot CLI, Muse, OpenCode,
Amp e Hermes, e leva um cartão de aprovação ao notch e à Lock Screen.

O resultado honesto desta pesquisa: **não há API documentada para
ler ou responder um cartão de aprovação do Grok Bot.** A primeira
entrega possível com superfície oficial é presença do app mais um
deep link que abre o Bot e preenche o compositor, sem enviar e sem
aprovar. Aprovação e chat de verdade ficam bloqueados até existir um
canal estável que devolva a decisão ao Grok Bot.

## Baseline do fork

| | Fork `main` | Upstream `main` |
|---|---|---|
| SHA | `9b4b16fff2311f76f3853ce2014c7a0ae5c3c06b` | `202082ee0204bbc4fb78d8178cea3f0c6fcc34e4` |
| Data | 1 out 2026 | 24 nov 2026 |
| Em relação ao merge-base `835421c` | 3 commits | 1177 commits |

Esse desencontro é o que o PR de sync resolve, sem force-push no
`main`. O merge não era limpo: 9 conflitos (`project.pbxproj`,
`Localizable.xcstrings`, `AppDelegate.swift`, `AppState.swift`,
`HookServer.swift`, `IslandTypes.swift` modify/delete,
`IslandViewContent.swift`, `SettingsView.swift`, `project.yml`).
A resolução ficou com a arquitetura do upstream. As abas
`NSTabViewController` e o `AppLanguage` saíram, porque o hub de
ajustes e o catálogo de 16 línguas (pt-BR inclusive) já existem. O
teto de pills continua 4, e a pill Orca começa desligada. Este
documento está nesse resultado, não no `main` antigo do fork.

## Como o Coucou integra agentes hoje

Contrato público: [`docs/AGENTS.md`](AGENTS.md). Qualquer ferramenta
que escreva JSON terminado em newline no socket Unix
(`~/Library/Application Support/NotchBuddy/nb.sock` no build GitHub;
`~/Library/Containers/fr.louisraille.Coucou/Data/nb.sock` na App
Store) ganha uma pill. O campo `coucou_agent` tem de casar
`^[a-z0-9-]{1,24}$`. O relay é `nb-hook --agent <nome>`.

Ciclo de vida já implementado em `HookServer.processEvent`:

| Evento canônico | Efeito |
|---|---|
| `SessionStart` | cria a pill, estado idle |
| `UserPromptSubmit` | thinking, ticker com o prompt |
| `PreToolUse` / `PostToolUse` | working |
| `Notification` | rate-limit ou pergunta, quando aplicável |
| `Stop` | finished ~5 s; pill declarada e ativa volta a idle, as outras somem |
| `StopFailure` | error |
| `SessionEnd` | igual ao `Stop` quanto à pill |
| `SubagentStart` / `SubagentStop` | linha no ticker |

Pills declaradas vivem em
`NotchBuddy/Sources/CoucouKit/PillCatalog.swift` (`PillCatalog.all`).
ID estável (Keychain, UserDefaults, roteamento). `githubOnly: true`
esconde a pill do build da App Store. No máximo 4 pills além da
principal. Uma pill de catálogo marcada em Settings → Active pills
permanece visível depois da sessão.

Aprovações **não** são genéricas. `processPermissionRequest` só segura
o file descriptor e mostra Allow / Deny para Claude Code, Cursor,
Codex, Copilot CLI, Muse e, no Windows, OpenCode quando o plugin
realmente espera a resposta. Hermes só entra nessa lista com
`coucou_has_transport: true` e o toggle de aprovações, porque
`register_approval_transport` ainda não existe no Hermes 0.15.x. Para
qualquer outro `coucou_agent`, o hook responde na hora
`{"permissionDecision":"ask"}` e o agente pergunta no próprio
terminal. Antigravity é o caso especial de `PreToolUse`: o relay
espera a decisão e, sem clique, devolve `ask`. Regra do projeto:
nunca aprovar sem um clique explícito, e nunca bloquear o agente se
o Coucou não responder.

O cartão que aparece vira `ApprovalInfo` / `ApprovalQueue` e, no
iPhone, `PhoneLink/ApprovalRelay.swift` publica o pedido no CloudKit.
Sem `pendingApproval` real, a Lock Screen não tem o que mostrar.

Abrir o app dono da sessão já existe: `IslandViewContent` usa
`NSWorkspace.shared.openApplication` (no macOS 14+ `activate()` é
ignorado a partir da island). Claude Desktop (`agent_claude-desktop`)
é o precedente mais próximo de « pill sem hook instalado, botão que
abre o app ».

Windows e Linux espelham o catálogo em `windows/src/core/pills.ts` e
os instaladores em `windows/src-tauri/src/agents.rs`. Strings novas
entram em `Localizable.xcstrings` e em `windows/src/i18n/`.

Padrão que este plano copia, por fase:

- Fase 1 — Claude Desktop: pill declarada, sem instalar nada no outro
  app, botão que traz o app para a frente.
- Fase 2 — Hermes `pre_approval_request`: observador, texto no ticker,
  zero decisão.
- Fase 3 — só se aparecer um canal bloqueante no estilo Codex. Até
  lá, o cartão do Grok Bot é « Abrir o Grok Bot » + « Dispensar »,
  nunca Allow / Deny. Dispensar esconde o cartão aqui; o Bot continua
  parado até a resposta dentro do Grok Bot.

## O que o Grok Bot documenta

Fontes oficiais lidas em 27 nov 2026. Nada abaixo foi inventado a
partir do comportamento do Coucou.

- [Deep links](https://docs.x.ai/grok-bot/deep-links) (atualizado em
  19 nov 2026). O app precisa estar instalado e com sessão iniciada.
  - `grokbot://app/v1/prompt?id=<bot-id>&text=<mensagem>` abre o chat
    daquele Bot com `text` no compositor, no máximo 1 000 caracteres.
    O texto só entra se o compositor estiver vazio e **não é
    enviado**. Sem `id`, usa o Bot primário; se não houver primário,
    o link não faz nada. O id é o final do link Share → Copy link.
    `text` vai percent-encoded.
  - `grokbot://app/v1/plugin/add?id=<plugin-id>` e
    `https://x.ai/bot/plugin/<plugin-id>` abrem um plugin do
    marketplace. Conector criado à mão não tem link.
  - A página não documenta rota para um cartão de aprovação, para um
    Bot que não seja o do `id`, nem para enviar a mensagem.
- [Aprovações](https://docs.x.ai/grok-bot/approvals-security-and-privacy).
  O cartão aparece **no chat**. Rótulos: Allow once / Always allow /
  Deny; comando de terminal usa Allow e Skip; regra « ask first » usa
  Allow e Not now. Cartão nascido de uma mensagem de chat espera sem
  timer. Cartão de rotina, mensagem recebida ou outro trabalho de
  fundo expira em 30 minutos. Uma mensagem nova, ou mudar Auto Review,
  fecha os cartões abertos. Aprovar não desfaz o que já rodou.
- Computador local é outra chave, em Settings → Computer → Your
  computer: Ask every time (padrão), Always allow ou Never allow, por
  desktop. A primeira vez, o chat pergunta se o Grok Bot e todos os
  Bots podem rodar comandos nesta máquina (Always allow, Allow once,
  Never, Deny once). Isso não impede o computador na nuvem.
- [Ajustes e notificações](https://docs.x.ai/grok-bot/settings-and-notifications)
  (23 nov 2026). A sidebar distingue Needs attention (pergunta,
  aprovação ou handoff), Unread e Working / typing. Notificações por
  Bot, ligadas pelo próprio Bot no desktop: banner quando ele termina
  ou está esperando. Com o app de desktop em uso no macOS, o aviso
  fica no Mac; se precisar de input, o iPhone do Grok Bot é avisado
  cerca de 3 minutos depois, salvo se o Bot for aberto, silenciado ou
  escondido da sidebar. Não há API para outro processo assinar esses
  avisos.
- [Começar](https://docs.x.ai/grok-bot/get-started) e
  [como funciona](https://docs.x.ai/grok-bot/how-grok-bot-works): o
  trabalho padrão corre num computador na nuvem compartilhado por
  todos os Bots da conta. O laptop pode estar fechado. Vários Bots,
  um primário marcado com estrela.
- Conta é a conta Cursor. Não há modo de privacidade legado. Segredos
  ficam no Vault do Grok Bot, não num arquivo que o Coucou deva ler.

Busca na documentação oficial não encontrou CLI, socket local, hook
no estilo Claude Code, nem endpoint para listar Bots ou responder um
cartão. O binário `grok` / Grok Build é outro produto; não ligar o
Coucou nele.

## Suposições a confirmar no Mac do usuário

Tudo nesta seção vem de engenharia reversa de builds específicos, não
da documentação. Tratar como falso até conferir na versão instalada
(`Support → About`). Não escrever código em cima destes caminhos antes
dessa conferência.

| Afirmação | Onde apareceu | Risco se estiver velha |
|---|---|---|
| Bundle id `com.anysphere.sand`, nome na tela « Grok Bot » | Teardowns dos builds 0.18.0 e 0.47, e o flake Nix que cita os schemes `sand://` e `grokbot://` | `NSWorkspace` não acha o app; a pill nunca mostra « em execução » |
| Scheme legado `sand://` ainda registrado, com rotas `open` e `info` no 0.18 | Mesmo teardown. A página oficial de deep links de nov/2026 só cita `grokbot://` | Usar `sand://` abre nada ou abre outra coisa |
| Dados do daemon em `~/.grokbot/` (`SAND_DATA_ROOT`), incluindo `settings.json` (`localToolPermission`: `ask` / `always` / `never`) e `local-tool-approvals.json` | Teardown 0.18, wiki reconstruída, notas de DFIR | Formato e até o diretório mudam entre versões. O 0.30 já foi descrito com a loja autoritativa dos agentes **no computador remoto**, não neste diretório |
| `~/Library/Application Support/Grok Bot/sand-client-persistence/` guarda réplicas de transcript | Notas de DFIR. Outro texto chama o mesmo diretório de segredos cifrados via safeStorage | Ler o blob pode ser inútil, ou pode vazar conversa para a memória do Coucou |
| Logs em `~/.grokbot/logs/local-exec-daemon.log` **não** trazem comando, id do pedido, exit status nem resultado | Teardown 0.18 e notas de DFIR | Vigiar o log não produz ticker útil |
| Credencial do daemon em `local-exec-daemon-credential.json` e `local-exec-daemon-connection.json` (modo 0600) | Teardown 0.18 | São segredos. O Coucou não abre esses arquivos |
| Aprovações locais exigem um id cunhado pelo app ao clicar Allow, casado por ação+alvo, TTL curto (~10 min no 0.18) | Teardown 0.18 | Escrever `local-tool-approvals.json` à mão seria forjar o clique. Proibido |

Conferência mínima, no Mac, antes da fase 2:

1. `mdls -name kMDItemCFBundleIdentifier -name kMDItemDisplayName` no `.app`, mais `defaults read` do Info.plist para `CFBundleURLTypes`.
2. Versão em Support → About.
3. Com um cartão aberto, listar mtime de `~/.grokbot` e de `~/Library/Application Support/Grok Bot` sem abrir arquivos de credencial. Ver se algum arquivo muda de forma estável e sem conteúdo secreto.
4. Texto exato da notificação de sistema quando um Bot espera (só o que o Notification Center mostra ao usuário).

## O que não fazer

- Não escrever `local-tool-approvals.json`, `settings.json` nem
  qualquer arquivo do Grok Bot para simular Allow, Always allow ou
  Never.
- Não usar Acessibilidade, AppleScript de clique ou captura de tela
  para apertar o cartão. O Coucou não pede permissão de
  Acessibilidade, e um clique sintético fura a regra de aprovação
  explícita.
- Não ler credencial, token, connection JSON, Keychain do
  `safeStorage`, Vault, nem o blob de persistência até estar provado
  que o arquivo é metadado não secreto. Segredos do Coucou continuam
  só no Keychain, e não há motivo para copiar o token Cursor.
- Não chamar `api2.cursor.sh` nem outra API da conta. O Coucou só fala
  com serviços que o usuário configurou, e esta API não está
  documentada como superfície de terceiros.
- Não bloquear o Grok Bot. Ele não é um hook. Se o Coucou não souber
  responder, o cartão permanece no Grok Bot.
- Não publicar na Lock Screen do Coucou um « pedido » reconstruído de
  um banner. O `ApprovalRelay` só republica um `ApprovalInfo` que o
  app local pode de fato resolver. O app iPhone do próprio Grok Bot
  já avisa cerca de 3 minutos depois.
- Não manter timer de polling com a island escondida. Meta do
  projeto: 0 % de CPU nesse estado.

## Fases

### Fase 0 — conferir a suposição no app instalado

Sem código de produto. Resultado: uma nota curta (versão, bundle id,
URL types, se algum arquivo não secreto muda quando um cartão abre).
Essa nota decide se a fase 2 existe.

### Fase 1 — MVP: pill, presença, deep link

Entrega visível, só com API de sistema e o deep link oficial.

- Pill de catálogo `agent_grokbot`, nome « Grok Bot », categoria
  `agent`, `githubOnly: true`, `source: .agent`. Cor proposta
  `#D4D4D8` (neutra, para não colidir com Cursor `#C0C4CC` nem com
  Claude Desktop `#D97757`). Não é cor de marca documentada; trocar
  na implementação se o ícone do app sugerir outra.
- Presença via `NSWorkspace`: notificações de launch, terminate e
  activate, não um timer. Estado idle quando o app está aberto,
  pill oculta ou em repouso quando não está (mesmo critério das pills
  declaradas: se estiver marcada em Active pills, fica em idle em vez
  de sumir).
- Botão que abre `grokbot://app/v1/prompt`. Sem `id`, o link oficial
  usa o Bot primário. Campo opcional em Settings para colar o id
  copiado de Share → Copy link (texto local, não é segredo; não vai
  para o Keychain). `text` opcional, cortado em 1 000 caracteres,
  percent-encoded, só para preencher. O rótulo do botão deixa claro
  que a mensagem não é enviada.
- Fallback se o scheme falhar: `openApplication` pelo bundle id
  confirmado na fase 0. Enquanto o bundle id não estiver confirmado,
  o fallback é o nome do app « Grok Bot », e a falha vira uma linha
  de erro na pill, não um crash.
- Settings → Agents → Grok Bot: explicação de uma frase (« abre o
  app; aprovar continua no Grok Bot »), toggle da pill, campo do id,
  botão Abrir. Sem « Install hooks »: não há o que instalar.
- A pill não mostra cartão de aprovação e não entra em
  `processPermissionRequest`.

Critério de pronto: com o Grok Bot instalado e a pill ligada, abrir e
fechar o app muda a pill; o botão foca o Bot primário (ou o id
colado) com o compositor preenchido e nada é enviado; com a island
escondida o watcher não agenda timer.

### Fase 2 — atividade, só se a fase 0 achar um sinal não secreto

Observador no estilo Hermes. FSEvents no arquivo confirmado, ligado
apenas com a pill ativa. Ticker de uma linha por Bot se o arquivo
tiver nome ou estado (« working », « needs attention ») sem conteúdo
da conversa. Se o único sinal for « o arquivo mudou » sem campo
legível, a fase para: uma pill que pisca a cada escrita não ajuda.

Não interpretar `localToolPermission: always` como « não há
aprovação ». O Auto Review do computador na nuvem é independente
dessa chave.

Se o sinal for claramente « esperando você » e não houver canal de
resposta, a pill ganha badge e o mesmo botão da fase 1. Som no mesmo
nível de um término de sessão, não o som de aprovação, para não
parecer que o Coucou pode decidir.

### Fase 3 — aprovações

Só começa se existir **uma** destas superfícies, confirmada na versão
instalada:

1. Hook ou plugin oficial que bloqueia até receber allow/deny, no
   modelo Codex / Antigravity. Aí sim: `--agent grokbot`, entrada na
   lista de `processPermissionRequest`, Allow e Deny apenas (sem
   Always, como Codex), timeout que devolve a decisão ao Grok Bot em
   vez de engolir o pedido, e `ApprovalRelay` para o iPhone porque a
   decisão volta pelo mesmo file descriptor.
2. Um deep link oficial que abre **aquele** cartão. O botão da pill
   usa esse link. Continua sem Allow / Deny locais.

Sem (1) ou (2), o máximo é o cartão « Abrir o Grok Bot » +
« Dispensar » quando a fase 2 tiver um sinal de espera confiável.
Dispensar não escreve nada no Grok Bot.

Fora de escopo mesmo com (1): Always allow, regras de Auto Review,
Never allow no computador local, e qualquer escrita em
`local-tool-approvals.json`.

### Fase 4 — conversar a partir do notch

O deep link oficial não envia. Chat dentro da island exigiria uma API
de mensagens documentada (enviar e receber em nome do usuário, com o
Bot certo). Essa API não apareceu na documentação. Até aparecer, a
fase 4 é a fase 1: preencher e deixar o usuário enviar no Grok Bot.

Não reutilizar o chat do Coucou (Anthropic / OpenAI / Ollama). Esse
chat fala com outra conta e outro modelo; apresentá-lo como « o Bot »
enganaria.

### Fase 5 — iPhone e Lock Screen

Automática no dia em que a fase 3 caminho (1) existir: o
`ApprovalRelay` já republica qualquer `pendingApproval`. Não criar um
publicador paralelo.

Antes disso, não espelhar. O Grok Bot já manda o aviso ao próprio
iPhone. Dois cartões, um deles sem efeito, é pior do que nenhum.

Windows e Linux ficam para depois do MVP no Mac. O Grok Bot existe
nessas plataformas e o scheme `grokbot://` é o mesmo, mas o Coucou de
lá não tem `NSWorkspace`. Quando for a hora: linha em
`windows/src/core/pills.ts`, watcher em Tauri, strings em
`windows/src/i18n/`. Sem paridade, a pill fica `githubOnly` no Mac e
ausente do catálogo Windows, como outras integrações que nasceram
num lado só.

## Arquivos a mexer na implementação

Nenhum deles muda neste PR.

| Arquivo | Mudança |
|---|---|
| `NotchBuddy/Sources/CoucouKit/PillCatalog.swift` | Entrada `agent_grokbot` |
| `NotchBuddy/Sources/App/GrokBotPresence.swift` (novo) | `NSWorkspace` + montagem do URL `grokbot://` |
| `NotchBuddy/Sources/App/IslandViewContent.swift` | Caso do botão abrir, no mesmo lugar de `agent_claude-desktop` |
| `NotchBuddy/Sources/App/SettingsView.swift` | Seção Agents, sem installer de hook |
| `NotchBuddy/project.yml` | O arquivo novo na target. Não editar o `.xcodeproj` à mão |
| `NotchBuddy/Resources/Localizable.xcstrings` | Rótulos |
| `NotchBuddy/Sources/App/HookServer.swift` | Só na fase 3 caminho (1), na lista que hoje cita codex/copilot/muse/hermes |
| `NotchBuddy/Sources/App/PhoneLink/ApprovalRelay.swift` | Nenhuma mudança de código se o pedido for um `ApprovalInfo` normal |
| `docs/AGENTS.md`, `docs/INTEGRATIONS.md`, `docs/SPEC.md` | Uma seção curta quando a fase 1 existir de verdade |
| `windows/src/core/pills.ts`, `windows/src/i18n/*.json` | Paridade, depois |
| Testes ao lado dos de pill (`tests/PillSessionRegistryTests.swift` ou um `GrokBotLinkTests.swift`) | O construtor de URL: encoding, corte em 1 000, omissão do `id`, recusa de scheme que não seja `grokbot` |

Fora da lista de propósito: `nb-hook`, `~/.claude/settings.json`,
Keychain, e qualquer arquivo sob `~/.grokbot` ou Application Support
do Grok Bot.

## Perguntas em aberto

1. Qual é o bundle id e a versão no Mac em que isso vai ser usado?
2. O Share → Copy link de um Bot termina num id opaco estável? A doc
   diz que o id é o final desse link; falta um exemplo real para o
   campo de Settings não ensinar o formato errado.
3. Quando um cartão está aberto, algum arquivo não secreto muda de
   forma estável? Se não, a fase 2 não existe.
4. O usuário quer a pill para todos os Bots da conta ou só para o
   primário? O deep link sem `id` só alcança o primário. Vários Bots
   exigiriam vários ids colados à mão, ou uma API de lista que não
   está documentada.
5. Vale a pena uma pill que só abre o app, dado que o Grok Bot já
   mostra banner e, minutos depois, notifica o iPhone dele?
6. O código desta integração espera o merge do PR de sync no `main`
   do fork. Até lá, a base deste plano é o branch desse PR, não o
   `main` de 1 out 2026.

## Riscos

- **Superfície instável.** Os teardowns de 0.18 e 0.47 já divergem
  entre si e da doc de novembro de 2026. Código preso a
  `~/.grokbot` quebra no update automático do app.
- **Falsa sensação de controle.** Um cartão Allow / Deny que não
  alcança o Grok Bot faz o usuário achar que negou um comando que
  continua pendente, ou o contrário. Por isso a fase 1 não desenha
  esse cartão.
- **Vazar conversa.** Réplicas de transcript e blobs de persistência,
  se forem legíveis, são o conteúdo dos Bots. O Coucou não tem
  telemetria; mesmo assim, carregar isso na memória da island é um
  vazamento local. A fase 2 só lê o que a conferência marcar como
  metadado.
- **Dois telefones.** Republicar no Coucou enquanto o Grok Bot avisa
  o app dele duplica a Lock Screen.
- **CPU.** `NSWorkspace` por notificação cabe na regra dos 0 %. Um
  poll de diretório não cabe com a island escondida.
- **Base.** Implementar a fase 1 no `main` de 1 out 2026 mexe numa
  árvore que o sync substitui. O lugar certo é o resultado do PR de
  sync, onde `PillCatalog`, `HookServer` e o Orca já convivem.
- **App Store.** Pill `githubOnly` evita levar um scheme de terceiro
  e uma integração sem API estável para a review. O build da App
  Store não ganha a pill na fase 1.

## Ordem sugerida

Fase 0 no Mac real. Se o bundle id e o scheme baterem com o que está
acima, implementar só a fase 1 e parar para o usuário usar uma semana.
Fase 2 apenas com um arquivo confirmado. Fases 3 e 4 esperam
documentação nova, ou um hook oficial, e não um parse do app.
