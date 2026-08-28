# SPIKE-01 — Relatório

**Data:** 28/08/2026 · **Timebox:** meio dia · **Commit testado:** `e382739` (probes) · `20d1924` (revalidado após correções de a11y)
**URL pública:** *pendente* — deploy no Vercel ficou com o Lucas (CLI + login).
Todos os testes abaixo rodaram em `http://localhost:8899` (localhost é secure
context, então a API fica disponível sem túnel).

**Ambiente:** macOS 26.2 · Chrome **151.0.7922.175** · sem extensão de agente instalada.

---

## Veredito: **GO CONDICIONAL**

O ciclo mecânico completo — registrar → descobrir → executar → UI reagir → submeter —
foi provado de ponta a ponta contra a implementação real do Chrome, não contra um
mock. O que falta é só a última milha: um agente de linguagem natural chamando as
tools, o que exige o deploy HTTPS público.

Nenhum bloqueio técnico apareceu no caminho. A condição é operacional (deploy +
teste no ChatGPT desktop), não arquitetural.

> Nota sobre o critério 3 do brief: em vez de esperar o agente, dirigi o pipeline
> nativo do browser (`document.modelContext.getTools()` / `.executeTool()`) — a
> mesma superfície que o agente consome. Isso é mais forte que o painel DevTools
> (critério 2) e cobre tudo menos a interpretação em linguagem natural.

### Critérios

| # | Critério | Status |
|---|---|---|
| 1 | Página registra tools sem erro no Chrome 149+ | ✅ 9 tools registradas, zero erro |
| 2 | Painel DevTools lista e executa, UI reage | ⚠️ Não aberto visualmente; equivalente provado via `executeTool()` — UI reagiu |
| 3 | Um caminho de agente real funciona | ⛔ Bloqueado por deploy público |
| 4 | Relatório preenchido | ✅ |

---

## 1. Namespace: a divergência não existe no Chrome

Fonte de confusão real: a doc do Chrome e a spec W3C dizem `document.modelContext`;
o blog da Netlify usa `navigator.modelContext`. No Chrome 151:

```js
document.modelContext === navigator.modelContext   // true — mesmo objeto
Object.getPrototypeOf(document.modelContext).constructor.name  // "ModelContext"
```

Os dois são o **mesmo objeto**; `navigator` é alias. Métodos expostos:

```
ontoolchange · executeTool · getTools · registerTool
```

Não existe `provideContext` (aparece em material antigo — ignorar).

**Para o produto:** usar `document.modelContext`. É o que a spec W3C define no WebIDL
e o que a doc do ChatGPT checa explicitamente (`document.modelContext?.registerTool`).
Fazer feature-detect dos dois só para logar.

**Flag:** o brief presumia `chrome://flags/#enable-webmcp-testing` obrigatória. No
Chrome 151 a API já estava presente **sem tocar em flag nenhuma**. Manter a instrução
no README como fallback para versões 149–150.

## 2. Formato de retorno: os três funcionam, e isso não é boa notícia

Registrei uma tool para cada shape que a documentação disputa:

| Tool | `execute` retorna | O que o `executeTool()` devolveu |
|---|---|---|
| `probe_return_string` | `"PLAIN_STRING_OK"` | `"PLAIN_STRING_OK"` |
| `probe_return_object` | `{shape:"plain_object",ok:true,n:42}` | `'{"shape":"plain_object","ok":true,"n":42}'` |
| `probe_return_mcp_content` | `{content:[{type:"text",text:"..."}]}` | `'{"content":[{"type":"text","text":"MCP_CONTENT_ARRAY_OK"}]}'` |

O retorno é **sempre `typeof === "string"`** — o browser serializa qualquer coisa
que não seja string. Bate com o WebIDL: `Promise<DOMString> executeTool(...)`.

Consequência: o formato MCP `{content:[...]}` da spec **não é desembrulhado** pelo
Chrome. Ele vira um JSON aninhado que o modelo teria que desempacotar sozinho —
custo puro, zero benefício.

> **Recomendação:** retornar **objeto JS puro**, plano e pequeno. É o que a demo
> oficial do Google faz (`{ok:true, confirmationId, ...}`) e o que o sample da
> própria OpenAI faz (`{title: document.title}`).

## 3. Argumentos: `executeTool` exige JSON **string**

Aqui a spec W3C está errada em relação ao que o Chrome implementa. A spec declara
`optional object inputObject = {}`. Na prática:

```js
await mc.executeTool(tool, { topic: 'support' })              // ❌ UnknownError: Failed to parse input arguments
await mc.executeTool(tool, JSON.stringify({topic:'support'})) // ✅
```

Só afeta quem chama `executeTool` na mão (nosso painel de debug/testes).
Dentro de `execute`, os argumentos chegam como **objeto** normalmente.

## 4. Erros: a mensagem se perde — este é o achado mais importante

`probe_throws` lança `new Error("PROBE_INTENTIONAL_FAILURE")`. O que o chamador
recebe:

```
name:    "UnknownError"
message: "Tool was executed but the invocation failed.
          For example, the script function threw an error"
```

A string `PROBE_INTENTIONAL_FAILURE` **não aparece em lugar nenhum**. Uma exceção
vira um erro genérico e o agente não tem como saber o que corrigir.

> **Recomendação (regra dura para o produto):** `execute` **nunca lança**. Todo erro
> esperado é valor de retorno: `{ ok:false, errors:[...] }`. Só assim o agente lê o
> problema e tenta de novo. `submit_form` já segue isso — devolveu
> `{"ok":false,"errors":[...]}` com campo obrigatório vazio, e o agente conseguiria
> agir sobre a lista.

## 5. Chrome **não valida** o `inputSchema` — a página tem que validar

Testado com `fill_form`, cujo schema declara `enum` e `additionalProperties:false`:

| Entrada | Esperado se houvesse validação | O que aconteceu |
|---|---|---|
| `{topic:"NOT_IN_ENUM"}` | rejeitar antes do `execute` | chegou no `execute` |
| `{bogusField:"x"}` | rejeitar (`additionalProperties:false`) | chegou no `execute` |
| `{subscribe:"yes-a-string"}` | rejeitar (esperava boolean) | chegou, virou `true` por coerção |
| `'{not json'` | rejeitar | ✅ único caso rejeitado pelo browser |

O `inputSchema` é **documentação para o modelo**, não contrato executável. Só JSON
malformado é barrado.

> **Recomendação:** validar todo argumento dentro do `execute` e devolver o motivo.
> A página já faz isso e o retorno é acionável:
> `{"ok":false,"rejected":["topic: \"NOT_IN_ENUM\" not in [support, sales, other]"]}`.
> Isso também é a base do modelo de segurança — argumentos vindos de agente são
> input não confiável, como o doc [secure-tools](https://developer.chrome.com/docs/ai/webmcp/secure-tools)
> trata.

## 6. `toolchange` e unregister: funcionam

- Tool registrada em **T+10s** disparou `toolchange`; `getTools()` passou de 8 → 9.
- `AbortController.abort()` removeu `probe_throws`: 9 → 8 tools, novo `toolchange`.
- Handle antigo depois do abort: `TypeError: ... not of type 'RegisteredTool'` — falha
  limpa, sem travar a página.

Registro dinâmico de tools é viável. **Aberto:** se o agente do ChatGPT relê a lista
no meio de uma conversa, ou fixa no carregamento da página. Testável só com o agente.

## 7. Latência

Chamadas via `executeTool` no mesmo processo: **1–29 ms** (`fill_form` com highlight
visual: 21–29 ms; leituras: 1–2 ms). Irrelevante — a latência percebida no demo vai
ser inteiramente do modelo, não da ponte WebMCP.

## 8. Imperativo vs declarativo — decisão: **100% imperativo**

No Chrome o form declarativo funcionou melhor do que eu esperava:

- Apareceu como tool distinta (`quickContactTool`) ao lado das 8 imperativas.
- `SubmitEvent.agentInvoked === true` na submissão feita por agente — dá para
  distinguir humano de agente sem gambiarra.
- `SubmitEvent.respondWith()` existe e devolveu retorno custom: `{"ok":true,"team":"sales"}`.
- Eventos `toolactivated` / `toolcancel` e pseudo-classes `:tool-form-active` /
  `:tool-submit-active` existem para feedback visual.

**Mesmo assim, o produto deve ser imperativo — e a razão é externa à qualidade da API:**

> A doc oficial da OpenAI declara a **API declarativa não suportada** no ChatGPT.
> Como o hackathon é da OpenAI e o demo roda no browser do ChatGPT, o declarativo é
> caminho morto para a submissão.

Somando: o declarativo é um form por tool (sem `get_form_state`, sem preenchimento
parcial, sem erro estruturado de volta), e o produto precisa dos três. Ficaria como
progressive enhancement no máximo — **fora do escopo dos 4 dias**.

## 9. Restrição de produto descoberta: **iframes não são vistos**

A doc da OpenAI é explícita: o ChatGPT **não descobre tools dentro de iframes**.

Isso atinge o produto na distribuição. O caminho natural de um form builder é
"cole este `<script>`/`<iframe>` no seu site" — e esse caminho **não funciona** com
agente. O formulário publicado precisa ser uma **página top-level** em URL própria.

Não é bloqueio para o hackathon (o demo é a URL do form), mas muda a narrativa: o
produto vende *link de formulário agent-ready*, não *widget embedável*.

---

## 10. Protocolo ChatGPT — não executado

Os 4 pedidos do Passo 5 dependem do deploy HTTPS público. Ficam como primeiro item
do próximo bloco. Já sabemos, pela doc da OpenAI, o que esperar:

| Superfície | Suporte a site tools |
|---|---|
| ChatGPT desktop — browser interno | ✅ |
| ChatGPT Work · Codex | ✅ |
| **ChatGPT mobile** | ❌ **não suportado** |
| Workspaces Enterprise/Edu | ❌ |

**O item 4 do protocolo (teste por voz) cai** — voz é mobile, e mobile não suporta
site tools. A narrativa de acessibilidade precisa de outro veículo (ditado no
desktop, ou voice mode do desktop se ele passar pelo browser interno — não verificado).

Usuário pode desligar tudo em *Settings → Browser → Permissions → Enable site tools*;
vale checar que está ligado antes de gravar o vídeo.

---

## 11. Pendências

| Item | Situação |
|---|---|
| Deploy HTTPS público (Vercel CLI) | Com o Lucas — desbloqueia o critério 3 |
| Teste no ChatGPT desktop (4 pedidos) | Depois do deploy |
| Painel DevTools → Application → WebMCP | Confirmar visualmente para o vídeo |
| [Model Context Tool Inspector](https://chromewebstore.google.com/detail/gbpdfapgefenggkahomfgkhfehlcenpd) | Não instalado — é o caminho B (agente em Chrome, usa `gemini-3-flash-preview`) |
| Origin trial | Não registrado. Só importa para usuários de Chrome estável sem flag; o demo do hackathon roda no ChatGPT. Baixa prioridade |

---

## 12. Três riscos principais para o build de 4 dias

**1. O caminho do agente ainda é fé, não fato.** Toda a cadeia mecânica está
provada, e a doc da OpenAI confirma `document.modelContext` no browser desktop —
mas ninguém viu um modelo chamar essas tools ainda. Se o ChatGPT ignorar as tools,
descobrir descrições ambíguas ou pedir confirmação a cada chamada, o "wow" do vídeo
morre. *Mitigação: deploy e teste hoje, antes de escrever qualquer feature.*

**2. Zero validação no browser + input de agente = superfície aberta.** O Chrome
entrega `inputSchema` como texto para o modelo e não valida nada. Com formulários
definidos pelo usuário, o `fill_form` do produto é gerado dinamicamente — cada form
publicado precisa validar contra seu próprio schema em runtime, e o modelo de erro
tem que ser retorno, nunca exceção. Escrever isso uma vez, no lugar certo, ou vira
buraco em todo form gerado.

**3. Sem iframe, a distribuição muda.** Forms publicados têm que ser páginas
top-level. Se em algum momento do build aparecer "deixa embutir num site", isso
quebra o produto todo no ChatGPT. Fixar essa decisão agora, na arquitetura.

---

## Anexos

- `screenshots/01-form-filled-by-tools.jpg` — form preenchido inteiramente por
  `fill_form` + `submit_form` via `executeTool()`; banner confirmando WebMCP ativo
  em `document.modelContext`.
- `screenshots/02-call-log-and-declarative.jpg` — painel de log com args, retorno e
  latência por chamada; form declarativo reportando `agentInvoked: true`.
