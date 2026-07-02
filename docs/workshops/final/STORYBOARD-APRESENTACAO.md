# Storyboard da Apresentação — A Grande Final (F5/F6)

> **Para o Claude for PowerPoint:** gere a apresentação **no mesmo layout do template das Oitavas/Quartas**. **12 slides, máximo 13.** Mantenha o estilo visual: capa cheia, slide de "Stack da fase" (só o que é NOVO), slides de tecnologia no formato *O que é? · Como funciona (com diagrama) · Principais recursos (4 itens) · ▸ Nesta etapa*, slides de conceito-chave, arquitetura e encerramento celebrativo.
> **Rodapé fixo em todos os slides:** `COPA DO MUNDO AZURE 2026` · `A GRANDE FINAL · F5/F6 — VOZ &amp; VISÃO`
> **Paleta/tipografia:** as mesmas do template. Cores de acento sugeridas: **verde-azulado** (F5 / voz / read-only) + **roxo** (F6 / visão / observabilidade) + **vermelho** só para o realce de segurança (X-Gateway-Key).
> **ENXUTO (diretriz do owner):** cobrir **só as tecnologias novas** da Final — **MCP · RAG · Managed Identity (+ Key Vault como direção) · observabilidade (Flow Visualizer/SignalR)**. **NÃO** reexplicar compra async, gateway, identidade ou Container Apps (já vistos em Oitavas/Quartas).
> **Não** incluir blocos de código longos (isso fica no runbook `final-portal-guide.md`); o foco é **explicar as tecnologias e os conceitos-chave**.
> **Rastreabilidade (Art. IV):** todo número/nome/nó bate com o guia e o código real. Fontes: ADE-008 (re-arquitetura sem n8n), ADE-009 (X-Gateway-Key), `final-portal-guide.md`, `SPEAKER-NOTES.md`. **Não invente** APIs, tools nem nós.

---

## Slide 1 — CAPA
- **Etiqueta:** A GRANDE FINAL · COPA DO MUNDO AZURE
- **Título:** **Voz &amp; Visão**
- **Subtítulo:** Chatbot **MCP + RAG** (só sentidos) + **Flow Visualizer** (5 nós ao vivo)
- **Linha de apoio:** A última aula: a aplicação ganha **voz** para responder o estado real da Copa e uma tela onde a arquitetura **se acende** — com segurança **por construção**.
- **Faixa de jornada:** `Oitavas (F1)` → `Quartas (F2/F3)` → **`Final (F5/F6)`**

---

## Slide 2 — RECAP DA JORNADA + "só o que é NOVO"
- **Título:** De onde viemos — e o que a Final ADICIONA
- **Tabela (recap):**
  | Lab | Você construiu |
  |---|---|
  | **Oitavas (F1)** | a **compra** assíncrona (Function → Service Bus → Consumer → SQL) |
  | **Quartas (F2/F3)** | o **gateway YARP** + a **identidade** (CIAM cliente / admin workforce) |
  | **A Final (F5/F6)** | **voz** (chatbot que lê) + **visão** (observabilidade animada) |
- **Destaque:** A Final **ADICIONA** — retro-compatibilidade **dura**: nada das fases anteriores deixa de funcionar.
- **As 4 tecnologias novas de hoje (só estas):** **MCP** · **RAG (tool-use)** · **Managed Identity** · **observabilidade ao vivo (SignalR)**.

---

## Slide 3 — STACK DA FASE · "As tecnologias NOVAS que vamos usar"
Lista (1 linha cada, com ícone) — **só o que é novo**:
- **MCP (Model Context Protocol)** — dá "ferramentas" ao LLM (`tools/list` / `tools/call`); server .NET, ingress **interno**.
- **RAG por tool-use** — o chatbot **recupera** o fato real (via tool) antes de responder; não inventa.
- **Google Gemini (`gemini-2.5-flash`)** — o LLM que decide **qual** tool chamar (function calling `AUTO`); chave **server-side**.
- **Managed Identity** — o FlowEvents lê a telemetria **sem segredo** (role `Log Analytics Reader`).
- **Azure SignalR (Free/Default)** — empurra os eventos ao browser em tempo real (WebSocket) → Flow Visualizer.

---

## Slide 4 — TECNOLOGIA 1 DE 4 · MCP (Model Context Protocol)
- **O que é?** Um **protocolo padrão** para expor "ferramentas" que um LLM descobre e chama em runtime — o "USB-C das integrações de IA".
- **Como funciona (diagrama):** `Chatbot` → `tools/list` (descobre) → `tools/call` (executa, JSON-RPC) → **`McpServer`** (interno, atrás do gateway) → `SELECT` no SQL.
- **Principais recursos (4):**
  - **`tools/list`** — o LLM descobre as ferramentas disponíveis em runtime.
  - **`tools/call` (JSON-RPC 2.0)** — a chamada tipada de uma ferramenta.
  - **McpServer .NET (SDK oficial)** — Container App de **ingress interno**; o browser nunca o chama direto.
  - **7 tools read-only** — todas `[McpServerTool(ReadOnly = true)]`; `SELECT` parametrizado (Dapper).
- **▸ Nesta etapa:** subir o McpServer e ver `tools/list` retornar **exatamente 7 sentidos**, todos `readOnly: true`.

---

## Slide 5 — TECNOLOGIA 2 DE 4 · RAG (grounding por tool-use)
- **O que é?** **Retrieval-Augmented Generation**: o modelo **recupera** um fato de uma fonte externa **antes** de gerar a resposta — em vez de "lembrar" (e inventar).
- **Como funciona (diagrama):** `"Quando o Brasil joga?"` → **Gemini** escolhe a tool → `consultar_partidas` → **SQL real** → resposta **fundamentada**.
- **Principais recursos (4):**
  - **Function calling (`AUTO`)** — o Gemini decide **qual** das 7 tools chamar.
  - **Grounding** — a resposta vem do **banco**, não do conhecimento paramétrico do modelo.
  - **Chave server-side** — a `GEMINI_API_KEY` fica no **proxy `/llm`**; um guard falha o build se vazar no bundle.
  - **`gemini-2.5-flash`** — o modelo do lab.
- **Destaque honesto (conceito-chave):** aqui RAG **não** é vector store/embeddings — é **grounding via MCP** (`SELECT`, não similaridade). Mesmo princípio ("recuperar antes de gerar"), implementação diferente.
- **▸ Nesta etapa:** fazer ≥3 perguntas e ver, no painel do chatbot, **qual** tool foi chamada a cada uma.

---

## Slide 6 — CONCEITO-CHAVE · A regra de ouro (agora trivial)
- **Título:** Segurança **por construção**, não por roteamento
- **Ideia central:** o McpServer é **só sentidos** — **não existe** ferramenta de escrita para o LLM chamar. A regra "o chatbot nunca escreve no banco" vale **por construção**.
- **Frase-âncora:** *"O que não existe não pode ser chamado."*
- **A demonstração AO VIVO (o clímax do F5):** pedir ao chatbot *"cria um alerta pra mim quando abrir ingresso VIP"* → ele **não tem** a tool.
- **Nuance honesta (não pule):** o LLM pode *dizer em texto* "criei o alerta" — isso é **alucinação de texto**, **não** uma tool call. **Nada** é gravado. A segurança não depende de o LLM "se comportar".
- **A auditoria mais simples que existe:** ler o `tools/list`. Sete verbos, todos de leitura. Fim.

---

## Slide 7 — TECNOLOGIA 3 DE 4 · Managed Identity
- **O que é?** Uma identidade do **Azure AD** gerenciada pela plataforma: o serviço se autentica **sem senha e sem segredo no código**.
- **Como funciona (diagrama):** `FlowEvents` (System-assigned MI) → role **`Log Analytics Reader`** no workspace → `LogsQueryClient` consulta os traces (Kusto).
- **Principais recursos (4):**
  - **System-assigned** — a identidade nasce e morre com o Container App.
  - **RBAC** — recebe o papel `Log Analytics Reader` (mínimo necessário).
  - **Sem credencial** — nenhuma connection string de telemetria a guardar/rotacionar.
  - **Fail-visível** — sem o papel, o `LogsQueryClient` toma **403** e os nós **nunca acendem**.
- **▸ Nesta etapa:** ligar a Managed Identity do `ca-flow` e conceder `Log Analytics Reader` no workspace.

---

## Slide 8 — CONCEITO-CHAVE · Key Vault, o destino de produção
- **Título:** A mesma identidade que lê a telemetria elimina o segredo em claro
- **Hoje, no lab:** segredos (SQL, Gemini, SignalR) são **App Settings / secretref** do Container App.
- **Em produção (EPIC-004 — próximo passo, honestamente NÃO cabeado neste lab):** esses segredos deveriam virar **Key Vault references** resolvidas por **Managed Identity**, e o SQL usar **`Authentication=Active Directory Managed Identity`**.
- **A ponte:** a **mesma** Managed Identity do FlowEvents é a peça que, amanhã, resolve os segredos do Key Vault. Aprender MID hoje é aprender a base do Key Vault de produção.
- **Honestidade arquitetural (destaque):** registrado como débito conhecido em `docs/security/final-security-debt.md`. O lab **ensina a Managed Identity**; o **Key Vault é a direção**, não um passo entregue aqui.

---

## Slide 9 — TECNOLOGIA 4 DE 4 · Observabilidade ao vivo (FlowEvents + SignalR)
- **O que é?** Um serviço de leitura de telemetria (**FlowEvents**) + **Azure SignalR** que empurra eventos ao browser em tempo real (WebSocket) → o **Flow Visualizer**.
- **Como funciona (diagrama):** `traces (correlationId)` → `FlowEvents` lê via **Kusto** → `TraceEventMapper` classifica cada trace num nó → **SignalR** → a rota `/flow` **acende** os nós.
- **Principais recursos (4):**
  - **Trace-driven** — o motor lê **traces correlacionados**; é agnóstico a quem os emitiu.
  - **Azure SignalR (Free_F1)** — **Service Mode `Default`** (⚠️ não Serverless — o FlowHub é hospedado pelo serviço).
  - **CORS restrito** — o WebSocket usa credentials → origin **exato** do front, nunca `*`.
  - **5 nós** — a "bolinha" atravessa a jornada em **&lt; 30s**, pelo mesmo `correlationId`.
- **▸ Nesta etapa:** criar o SignalR (Free/Default) + o FlowEvents, fazer uma compra real e ver `/flow` acender **5 nós**.

---

## Slide 10 — CONCEITO-CHAVE · "Onde foi o n8n?" (lição de simplificação)
- **Título:** Removemos um componente — **não** o substituímos
- **Ideia central:** no desenho original havia um **6º nó** para orquestrar a notificação pós-compra. Ele **não existe mais**: a notificação virou uma etapa **inline** dentro da **Function Consumer** (nó 3).
- **Frase-âncora:** *"É a **Function** que orquestra o pós-compra."* (NUNCA dizer "automação no-code" nem citar orquestração externa como presente — ela **não existe**.)
- **Visual (antes → agora):**
  | | Antes | Agora (Final) |
  |---|---|---|
  | Pós-compra | orquestração externa (Container App + Postgres) | **inline** na Function Consumer |
  | Nós no visualizer | 6 | **5** |
  | Rastreabilidade | trace correlacionado | **trace correlacionado (igual)** |
- **Trade-off honesto:** a notificação fica **invisível** na animação (dobrada no nó 3). Ganhamos simplicidade (menos peças/custo/falhas); a observabilidade dela vive no **log correlacionado**.

---

## Slide 11 — ARQUITETURA · A foto completa da Final
- **Título:** A Grande Final, ponta a ponta
- **Diagrama (reusar o `final-f5-f6-mcp-flow.drawio`):**
  - **F5 (voz):** `Browser` → **`Gateway YARP`** (guardião · `X-Entra-OID` · **`X-Gateway-Key`**) → **`McpServer`** (interno, 7 sentidos) → **`SQL`** (SELECT) · proxy `/llm` → **`Gemini`** (chave server-side).
  - **Compra (5 nós):** `Gateway (0)` → `Function Entry (1)` → `Service Bus (2)` → `Function Consumer (3, notificação inline)` → `SQL (4)`.
  - **F6 (visão):** **`FlowEvents`** (Managed Identity → **`Kusto`** por `correlationId`) → **`Azure SignalR`** → rota **`/flow`**.
- **Legenda:** **Zero** n8n, **zero** PostgreSQL. O gateway é o **nó 0** e o **guardião único**. Retro-compatível com Oitavas/Quartas.

---

## Slide 12 — ENCERRAMENTO DA FINAL (celebrativo)
- **Título:** Você concluiu **A Grande Final** — e o Living Lab inteiro
- **As 4 missões (o que você provou):**
  - **Voz** (F5) — uma IA consulta dados reais **com segurança** (7 sentidos, zero escrita).
  - **Visão** (F6) — observabilidade distribuída: uma compra animada em **5 nós** por `correlationId`.
  - **Blindar** (hardening) — gateway guardião único: **`X-Gateway-Key`** fecha o bypass; chave Gemini nunca no bundle.
  - **Simplificar** (re-arquitetura) — **menos peças** (notificação inline), mesma função, retro-compat.
- **Fala de fechamento:** "Você começou com **uma compra de ingresso** e terminou com um sistema **Azure-native** completo: assíncrono, com gateway, identidade federada, um chatbot que conversa com os dados **sem nunca poder alterá-los**, e uma tela onde a própria arquitetura **se acende**. Isso é uma **Grande Final**. Você construiu tudo, do zero, com as próprias mãos."
- **A lição que se leva:** guardião único · segurança por construção · observabilidade correlacionada · **simplicidade deliberada** (remover &gt; substituir).
- **Quiz de encerramento:** link no guia do aluno (`final-portal-guide.md`).

---

### Notas de geração (para o Claude for PowerPoint)
- Marcar nos slides 4, 5, 7, 9 o selo **"TECNOLOGIA X DE 4"** (canto), como o template das Oitavas/Quartas.
- Diagramas: caixas + setas simples, no estilo "COMO FUNCIONA" (não usar screenshots).
- Os slides 6, 8 e 10 são os **conceitos-chave** — dar destaque visual (cor de acento, frase grande).
- O slide 12 é o **encerramento do workshop inteiro** (não só da fase) — tom celebrativo, herdado do fecho das Quartas/2.6.
- Speaker notes detalhadas (a fala de cada bloco, cronômetro, erros comuns) estão em `SPEAKER-NOTES.md` — não precisam ir nos slides.
- **Pendência conhecida:** o `.pptx` binário é gerado fora do repo (Claude for PowerPoint), a partir deste storyboard — mesmo padrão de Oitavas/Quartas (o `.pptx` fica no repo só quando exportado; aqui entregamos o **storyboard + o deck reveal.js `slides.md`**).
