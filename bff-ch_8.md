# BFF `bff-ch` — Lista de Ajustes

Documento único. Substitui os anteriores.

**O que é:** lista numerada, na ordem de execução. Cada item diz o que está errado, em qual arquivo, o que acontece na prática e como corrigir. Nenhum item depende de você ter lido outro.

**Contexto do sistema:**

```
App cliente → ALB público → BFF (este projeto, ECS)
                                 ↓ ALB interno
                            Serviço de consentimento (ECS) → Banco
```

O BFF não acessa banco. Todas as operações de consentimento são chamadas HTTP para o outro ECS.

---

## Se você só fizer duas coisas

**Item 1** — hoje, quando o serviço de consentimento está fora, seu BFF responde **sucesso**. No GET devolve "não consentiu"; no POST diz que criou sem ter criado.

**Item 2** — o retry do POST pode criar consentimento duplicado no banco deles.

---

# Parte 1 — Corrigir agora

## 1. Erro `nil` quando o circuit breaker abre

**Arquivo:** `internal/infra/consent_http_adapter.go` — nos **três** métodos (GET, POST, DELETE)

**O que está errado**

```go
result, err := a.breaker.Execute(func() (interface{}, error) {
    lastErr = err          // ⚠️ só é setado se a closure EXECUTAR
    return nil, err
})
// ...
return domain.ConsentStatus{}, lastErr
```

Quando o breaker está **aberto**, o `Execute` retorna erro **sem executar a closure**. O `lastErr` continua `nil`. O método retorna erro nulo com struct vazia.

**O que acontece na prática**

| Rota | Resposta ao usuário | Realidade |
|---|---|---|
| GET | `200` com corpo vazio | O cliente lê "não consentiu" durante uma indisponibilidade |
| POST | Sucesso | O consentimento **não foi criado** |
| DELETE | Sucesso | O consentimento **continua ativo** |

No POST e no DELETE isso deixa de ser erro técnico e vira divergência entre o que foi dito ao usuário e o estado real do consentimento.

**Correção**

```go
for attempt := 0; attempt < maxAttempts; attempt++ {
    result, err := a.breaker.Execute(func() (any, error) { /* ... */ })
    if err == nil {
        return result.(domain.ConsentStatus), nil
    }
    lastErr = err

    if errors.Is(err, gobreaker.ErrOpenState) || errors.Is(err, gobreaker.ErrTooManyRequests) {
        // erro do item 3; o %w preserva a causa para o log
        return domain.ConsentStatus{}, fmt.Errorf("breaker aberto: %w", apperrors.ErrUpstreamUnavailable)
    }
    if !isRetryable(err) {
        break
    }
}

if lastErr == nil {
    lastErr = errors.New("falha desconhecida no serviço de consentimento")
}
return domain.ConsentStatus{}, lastErr
```

Regra: nunca use uma variável capturada por closure como erro de retorno. Use o valor retornado.

---

## 2. Retry no POST pode duplicar o consentimento

**Arquivo:** `internal/infra/consent_http_adapter.go` — método de criação

**O que está errado**

Se o método de criação reusa o mesmo loop de retry do GET, você tem um gerador de duplicatas. A premissa errada é *"deu timeout, então não aconteceu"*.

```
1. BFF envia POST
2. Serviço RECEBE, PROCESSA e GRAVA no banco          ✅
3. A resposta demora 10,1s (GC, rede, lock no banco)
4. Seu timeout de 10s dispara
5. Retry envia o MESMO POST
6. Serviço grava um SEGUNDO consentimento              ❌
```

Com 3 tentativas, uma ação do usuário pode virar 3 registros com timestamps diferentes.

**O que acontece na prática**

O banco do outro ECS fica com registros duplicados do mesmo funcional. A pergunta "quando ele consentiu?" passa a ter três respostas. E se o DELETE remove "o consentimento", ele pode remover só um — o usuário revoga e continua consentido.

**Correção — escolha o nível possível**

**Nível 1 (o certo):** chave de idempotência. Gerada **uma vez, fora do loop**, e reenviada igual em todas as tentativas.

```go
idemKey := uuid.NewString()   // gerada ANTES do loop de retry
req.Header.Set("Idempotency-Key", idemKey)
```

O serviço deles deduplica: a segunda chamada com a mesma chave devolve o resultado da primeira em vez de gravar de novo. Depende de eles suportarem — verifique no repositório deles (checklist no fim).

Bônus: o `http.Transport` do Go trata um POST com o header `Idempotency-Key` como retentável, então ele passa a recuperar sozinho as falhas de conexão reusada — a causa provável do erro esporádico no POST que não tem relação com carga.

**Nível 2:** retentar só erro que comprova que a requisição não saiu.

```go
func isSafeToRetryWrite(err error) bool {
    if errors.Is(err, syscall.ECONNREFUSED) { return true }   // ninguém escutando
    var dnsErr *net.DNSError
    if errors.As(err, &dnsErr)              { return true }   // nem resolveu o host
    return false                                              // timeout: NUNCA retentar
}
```

Erro de conexão é seguro. Timeout de resposta nunca é.

**Nível 3:** zero retry na escrita. `maxRetries = 1` e devolva erro honesto — o `ErrIndeterminateResult` do item 3, que vira `502` com mensagem orientando o app a consultar o status antes de reenviar.

**De qualquer forma, separe a config por operação.** Um `maxRetries` único para leitura e escrita é o desenho que produz esse bug.

```go
type Config struct {
    ReadTimeout     time.Duration  // 3s
    WriteTimeout    time.Duration  // 8s
    ReadMaxRetries  int            // 2
    WriteMaxRetries int            // 1  (2 só com Idempotency-Key)
}
```

---

## 3. Tratamento de erros genérico e reutilizável

**Arquivos novos:** `internal/apperrors/errors.go`, `internal/infra/http_errors.go`, `internal/handler/errors.go`
**Arquivos alterados:** os handlers e o adapter de consentimento

Não reorganizar as demais pastas. Adicionar somente o necessário para este tratamento.

### 3.1 Problema atual

```go
if err != nil {
    WriteError(c, http.StatusInternalServerError, err.Error())
    return
}
```

Presente em `GetConfiguracoes`, `GetConsentimentos`, `PostConsentimentos` e `DeleteConsentimentos`.

**Todos os erros viram `500`.** O serviço dependente fora do ar não é culpa do BFF — é `503`/`504`. Com tudo colapsado em 500, o alarme de 5xx dispara para o seu time quando o problema é de outro squad, e o dashboard não distingue "meu BFF quebrou" de "a dependência caiu".

**Detalhes internos vazam.** `err.Error()` pode revelar URL interna, hostname do outro ECS, detalhes de rede, mensagens de biblioteca e o conteúdo técnico da resposta recebida. O erro técnico completo deve ir para o log; o cliente recebe mensagem segura e estável.

### 3.2 Erros genéricos da aplicação

```go
// internal/apperrors/errors.go
package apperrors

import "errors"

var (
    ErrInvalidData         = errors.New("dados inválidos")
    ErrResourceNotFound    = errors.New("recurso não encontrado")
    ErrResourceConflict    = errors.New("recurso já existe")
    ErrUpstreamTimeout     = errors.New("timeout no serviço dependente")
    ErrUpstreamUnavailable = errors.New("serviço dependente indisponível")
    ErrInvalidUpstreamData = errors.New("resposta inválida do serviço dependente")
    ErrIndeterminateResult = errors.New("resultado da operação indeterminado")
    ErrIntegrationAuth     = errors.New("credencial de integração recusada")
)
```

Os nomes podem ser adaptados ao padrão do projeto, mas devem continuar genéricos.

**Não criar erros acoplados a um domínio:**

```go
ErrConsentNotFound
ErrConsentServiceTimeout
ErrConsentServiceUnavailable
```

A informação de que a falha ocorreu numa operação de consentimento entra como **contexto no wrapping**, não no nome do erro.

**Pacote separado do `domain`, de propósito.** `ErrUpstreamTimeout` não é vocabulário de negócio — o domínio não deveria saber que existem "serviços dependentes". Por isso `internal/apperrors` e não `internal/domain/errors.go`.

**Quando criar um erro novo:** só quando ele mudar o status HTTP, o código de resposta, a ação esperada do cliente, o comportamento de retry ou de circuit breaker, ou exigir tratamento próprio via `errors.Is`. Mensagem diferente **não** justifica sentinel novo.

### 3.3 Wrapping e `errors.Is`

Adapters e use cases preservam a categoria com `%w` e acrescentam contexto técnico:

```go
return fmt.Errorf("consultar consentimento do funcional %s: %w", funcional, apperrors.ErrUpstreamUnavailable)
```

A identificação usa `errors.Is`, que atravessa quantos níveis de wrap existirem:

```go
if errors.Is(err, apperrors.ErrUpstreamUnavailable) { /* ... */ }
```

**Nunca comparar mensagens:**

```go
if err.Error() == "serviço dependente indisponível" {   // não fazer
```

### 3.4 Falha do usuário não é a mesma coisa que falha entre serviços

Este ponto evita um bug de produção concreto.

**Autenticação do usuário** — header `Authorization` ausente, JWT inválido ou expirado, `iss`/`aud` errados, claim obrigatória faltando, usuário sem permissão. Resolve-se no middleware ou no handler e resulta em `401` ou `403` ao cliente. Não passa por `apperrors`.

**Autenticação entre o BFF e o outro ECS** — o usuário está autenticado corretamente, mas a credencial técnica que o BFF usa foi recusada:

```
Cliente manda token válido → BFF autentica OK → BFF chama o outro ECS
com credencial técnica → outro ECS devolve 401 ou 403
```

**Nunca propagar esse `401`/`403` ao cliente.** O frontend entenderia que o usuário precisa logar de novo, e o usuário entra em loop de re-login por causa de uma API key vencida do BFF.

**E não tratar como `503` também.** Credencial recusada não é indisponibilidade temporária: é configuração quebrada, permanente até alguém agir. Devolver `503` faz o cliente retentar em vão, faz o alarme dizer "downstream fora" quando na verdade a chave venceu, e pode abrir o circuit breaker por um erro que não é do outro serviço.

Use `ErrIntegrationAuth` → **`500`** com código estável próprio (`ERRO_INTEGRACAO`) e **alarme de severidade alta e separado**. Quando isso acontece, 100% das requisições vão falhar até alguém trocar a credencial — é emergência operacional, não degradação.

A resposta ao cliente não deve revelar que uma credencial interna está inválida.

> **Confirmar antes de implementar:** se o modelo real for propagação do token do usuário em vez de credencial técnica, um `401` downstream pode de fato significar token do usuário expirado — e aí `401` ao cliente é o correto. O adapter precisa saber qual modelo usa. Está no checklist do fim.

### 3.5 Tradução de status: padrão compartilhado, exceção no adapter

Status HTTP sozinho nem sempre determina o significado. Um `404` pode ser "consentimento não existe" ou "URL errada / rota não encontrada pelo ALB". Um `400` pode ser payload do cliente ou payload montado errado pelo BFF.

Ao mesmo tempo, replicar o mapeamento inteiro em cada adapter é o que faz cada integração nova copiar o mesmo bloco.

**Regra:** helper compartilhado como **padrão**; o adapter sobrescreve **apenas onde o contrato daquele serviço realmente diverge**.

```go
// internal/infra/http_errors.go
package infra

// MapUpstreamStatus traduz status com semântica HTTP universal.
// O segundo retorno indica se o status foi reconhecido:
//   (nil,  true)  -> sucesso
//   (err,  true)  -> erro reconhecido
//   (nil,  false) -> não reconhecido; o adapter decide
func MapUpstreamStatus(status int, isWrite bool) (error, bool) {
    if status >= 200 && status < 300 {
        return nil, true
    }

    switch status {
    case http.StatusUnauthorized, http.StatusForbidden:
        // assume credencial TÉCNICA do BFF — ver 3.4
        return fmt.Errorf("status %d: %w", status, apperrors.ErrIntegrationAuth), true
    case http.StatusNotFound:
        return fmt.Errorf("status 404: %w", apperrors.ErrResourceNotFound), true
    case http.StatusConflict:
        return fmt.Errorf("status 409: %w", apperrors.ErrResourceConflict), true
    case http.StatusBadRequest, http.StatusUnprocessableEntity:
        return fmt.Errorf("status %d: %w", status, apperrors.ErrInvalidData), true
    case http.StatusServiceUnavailable:
        return fmt.Errorf("status 503: %w", apperrors.ErrUpstreamUnavailable), true
    case http.StatusBadGateway, http.StatusGatewayTimeout:
        if isWrite {
            return fmt.Errorf("status %d em escrita: %w", status, apperrors.ErrIndeterminateResult), true
        }
        return fmt.Errorf("status %d: %w", status, apperrors.ErrUpstreamUnavailable), true
    }

    if status >= 500 {
        return fmt.Errorf("status %d: %w", status, apperrors.ErrUpstreamUnavailable), true
    }
    return nil, false
}
```

> O segundo retorno existe porque `nil` de erro em Go significa "deu certo". Sem o `bool`, "não reconheci este status" e "sucesso" ficariam indistinguíveis — erro fácil de cometer e difícil de achar.

No adapter, as exceções do contrato vêm primeiro e o padrão cobre o resto:

```go
// internal/infra/consent_http_adapter.go
func (a *ConsentHttpAdapter) mapStatus(status int, isWrite bool) error {
    // exceções do contrato DESTE serviço entram aqui, quando existirem e
    // estiverem confirmadas. Ex.: se 409 significar "consentimento já ativo"
    // e não for erro, tratar antes de cair no padrão.

    if err, ok := MapUpstreamStatus(status, isWrite); ok {
        if err != nil {
            return fmt.Errorf("serviço de consentimento: %w", err)
        }
        return nil
    }
    return fmt.Errorf("serviço de consentimento: status inesperado %d", status)
}
```

Uso, com `isWrite` fixo por método (`false` no GET, `true` no POST e no DELETE):

```go
if err := a.mapStatus(resp.StatusCode, false); err != nil {
    return domain.ConsentStatus{}, err
}
```

### 3.6 Erros de transporte

Nem toda falha tem status HTTP. Quando `client.Do` retorna erro, não existe resposta para mapear — e sem isto o `ErrUpstreamTimeout` nunca é produzido por ninguém.

```go
// internal/infra/http_errors.go
func MapTransportError(err error, isWrite bool) error {
    if err == nil {
        return nil
    }

    // cliente desistiu: não é falha do serviço dependente
    if errors.Is(err, context.Canceled) {
        return err
    }

    var netErr net.Error
    isTimeout := errors.Is(err, context.DeadlineExceeded) ||
        (errors.As(err, &netErr) && netErr.Timeout())

    if isTimeout {
        if isWrite {
            // ⚠️ timeout em escrita NÃO é só timeout: a operação pode ter sido concluída
            return fmt.Errorf("timeout em escrita: %w", apperrors.ErrIndeterminateResult)
        }
        return fmt.Errorf("timeout: %w", apperrors.ErrUpstreamTimeout)
    }

    return fmt.Errorf("falha de comunicação: %w", apperrors.ErrUpstreamUnavailable)
}
```

**A distinção do `isWrite` no timeout é a mesma do item 2.** Um timeout durante o POST significa que o consentimento pode ter sido gravado. Classificá-lo como `ErrUpstreamTimeout` faz o retry entender que é seguro tentar de novo — e é assim que nasce registro duplicado.

`context.Canceled` é devolvido sem classificação: o cliente fechou a conexão, não houve falha da integração. Contar isso como erro de downstream polui métrica e pode abrir o circuit breaker sem motivo.

### 3.7 Resposta inválida do serviço dependente

`ErrInvalidUpstreamData` precisa de produtor. É o caso de `2xx` com corpo que não corresponde ao contrato — JSON quebrado, HTML de proxy corporativo, página do WAF, campo obrigatório ausente:

```go
var dto consentStatusResponse
if err := json.NewDecoder(resp.Body).Decode(&dto); err != nil {
    return domain.ConsentStatus{}, fmt.Errorf(
        "decodificar resposta do consentimento: %v: %w", err, apperrors.ErrInvalidUpstreamData)
}
```

Sem isso, um `200` com corpo inválido devolve struct zerada **com sucesso** — o mesmo sintoma do item 1, por outro caminho.

### 3.8 Escrita com resultado indeterminado

```
BFF envia a requisição
    ↓
Outro ECS recebe e EXECUTA a operação
    ↓
A resposta se perde, estoura o timeout, ou volta 502/504 do ALB
    ↓
BFF não sabe se a operação foi concluída
```

Não tratar como indisponibilidade simples. Usar `ErrIndeterminateResult` → `502`, com mensagem que oriente o cliente a **consultar o estado antes de reenviar**:

```json
{
  "erro": {
    "codigo": "RESULTADO_INDETERMINADO",
    "mensagem": "Não foi possível confirmar a operação. Consulte o status antes de tentar novamente.",
    "correlationID": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

**Isto não substitui idempotência.** Se houver retry de `POST`, o outro ECS precisa garantir deduplicação — ver item 2.

### 3.9 Mapeamento central no handler

```go
// internal/handler/errors.go
type mappedError struct {
    Status  int
    Code    string
    Message string
}

func mapApplicationError(err error) mappedError {
    switch {
    case errors.Is(err, apperrors.ErrInvalidData):
        return mappedError{http.StatusUnprocessableEntity, "DADOS_INVALIDOS", "dados inválidos"}
    case errors.Is(err, apperrors.ErrResourceNotFound):
        return mappedError{http.StatusNotFound, "RECURSO_NAO_ENCONTRADO", "recurso não encontrado"}
    case errors.Is(err, apperrors.ErrResourceConflict):
        return mappedError{http.StatusConflict, "RECURSO_JA_EXISTE", "recurso já registrado"}
    case errors.Is(err, apperrors.ErrUpstreamTimeout):
        return mappedError{http.StatusGatewayTimeout, "TEMPO_LIMITE", "tempo limite excedido"}
    case errors.Is(err, apperrors.ErrUpstreamUnavailable):
        return mappedError{http.StatusServiceUnavailable, "SERVICO_INDISPONIVEL", "serviço temporariamente indisponível"}
    case errors.Is(err, apperrors.ErrInvalidUpstreamData):
        return mappedError{http.StatusBadGateway, "RESPOSTA_INVALIDA", "resposta inválida do serviço dependente"}
    case errors.Is(err, apperrors.ErrIndeterminateResult):
        return mappedError{http.StatusBadGateway, "RESULTADO_INDETERMINADO",
            "não foi possível confirmar a operação; consulte o status antes de tentar novamente"}
    case errors.Is(err, apperrors.ErrIntegrationAuth):
        // falha de configuração do BFF — NÃO expor ao cliente que é credencial
        return mappedError{http.StatusInternalServerError, "ERRO_INTEGRACAO", "erro interno"}
    default:
        return mappedError{http.StatusInternalServerError, "ERRO_INTERNO", "erro interno"}
    }
}

func WriteApplicationError(c *gin.Context, ctx context.Context, err error) {
    m := mapApplicationError(err)

    if m.Status >= http.StatusInternalServerError {
        logger.Error(ctx, "falha ao processar requisição",
            "error", err, "code", m.Code, "status", m.Status)
    } else {
        logger.Warn(ctx, "requisição rejeitada",
            "code", m.Code, "status", m.Status)
    }

    WriteErrorResponse(c, m.Status, m.Code, m.Message)  // ajustar ao componente já existente
}
```

Nome e assinatura de `WriteErrorResponse` devem ser ajustados ao componente de resposta real do projeto. **Não criar um segundo formato de erro** se já existir envelope oficial.

### 3.10 Mapeamento HTTP recomendado

| Categoria | Status | Código estável |
|---|---:|---|
| JSON malformado, campo obrigatório ausente (tratado no handler) | `400` | `REQUISICAO_INVALIDA` |
| Dados sintaticamente válidos, rejeitados pela regra de negócio | `422` | `DADOS_INVALIDOS` |
| Recurso não encontrado | `404` | `RECURSO_NAO_ENCONTRADO` |
| Conflito de recurso | `409` | `RECURSO_JA_EXISTE` |
| Timeout em **leitura** | `504` | `TEMPO_LIMITE` |
| Serviço dependente indisponível | `503` | `SERVICO_INDISPONIVEL` |
| **Circuit breaker aberto** | `503` | `SERVICO_INDISPONIVEL` |
| Resposta inválida do serviço dependente | `502` | `RESPOSTA_INVALIDA` |
| **Resultado indeterminado** (timeout ou 502/504 em escrita) | `502` | `RESULTADO_INDETERMINADO` |
| Credencial de integração recusada | `500` | `ERRO_INTEGRACAO` |
| Erro inesperado | `500` | `ERRO_INTERNO` |

`401` e `403` do **cliente** continuam sob responsabilidade do fluxo de autenticação do BFF e não passam por esta tabela.

Não usar esta tabela para copiar cegamente o status recebido do outro ECS — interpretar o contrato no adapter primeiro (3.5).

### 3.11 Corpo padronizado

Se existir guideline interno de API ou contrato já consumido pelo frontend, **ele deve ser preservado**. Na ausência:

```json
{
  "erro": {
    "codigo": "SERVICO_INDISPONIVEL",
    "mensagem": "serviço temporariamente indisponível",
    "correlationID": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

O frontend decide pelo campo `codigo`, nunca comparando o texto de `mensagem`. Não alterar o envelope público atual sem verificar os consumidores.

### 3.12 Alteração nos handlers

```go
// de
if err != nil {
    WriteError(c, http.StatusInternalServerError, err.Error())
    return
}

// para
if err != nil {
    WriteApplicationError(c, ctx, err)
    return
}
```

Aplicar em `GET`, `POST` e `DELETE /consentimentos` e nos demais handlers que hoje devolvem `err.Error()`, preservando os contratos existentes. Erros detectados antes do use case — JSON malformado, autenticação ausente — continuam tratados no handler ou middleware.

### 3.13 Estratégia de logs

O pacote `internal/logger` **não muda**: ele continua sendo o único jeito de algo virar log. Ele consome o erro; quem o define é `apperrors` e quem decide status e nível é o handler.

| Camada | O que loga | Nível |
|---|---|---|
| Middleware | método, rota, status final, latência | `Info` |
| `handler/errors.go` | falha final da requisição — **ponto único** | `Error` se 5xx, `Warn` se 4xx |
| Use case | evento de negócio | `Info` |
| Adapter | tentativa, duração, status recebido | `Debug` |

Hoje o mesmo erro é registrado no use case, no adapter e no handler (ver "log duplicado" na Parte 4). O adapter **não chama `logger.Error`** — devolve o erro embrulhado.

Regras: nunca registrar token, API key ou secret; incluir `correlationID` e `flowID` quando disponíveis; o `funcional` segue a política interna de dados pessoais e mascaramento.

Se algum use case vier a ser executado por job ou mensageria, o ponto de entrada correspondente registra a falha final.

### 3.14 Testes

**Mapeamento central** — table-driven, cobrindo cada erro de `apperrors` mais um erro desconhecido. Validar status, código, mensagem, e o funcionamento tanto com o erro direto quanto embrulhado uma ou mais vezes com `%w`.

**Adapter** — com `httptest.Server`, conforme o contrato real: `2xx`; `400`/`422`; `401`/`403` sem provocar reautenticação indevida do cliente; `404`; `409`; `502` em leitura e em escrita; `503`; `504` em leitura e em escrita; `2xx` com JSON inválido; timeout do cliente **em leitura e em escrita** (resultados diferentes); erro de conexão; cancelamento de contexto.

**Handlers** — o erro técnico não aparece no corpo; código estável correto; status correto; `correlationID` presente quando o contrato exigir; contrato atual preservado.

### 3.15 Critérios de aceite

1. Erros genéricos, não acoplados a consentimento, em `internal/apperrors`.
2. Identificáveis por `errors.Is` mesmo após múltiplos wraps.
3. Nenhum handler devolve `err.Error()` ao cliente.
4. Erro técnico completo disponível no log.
5. Resposta ao cliente com mensagem segura e código estável.
6. Mapeamento HTTP centralizado no componente de entrada.
7. Adapter responsável por interpretar exceções do contrato do serviço chamado.
8. `401`/`403` downstream não viram falha de autenticação do usuário, e credencial recusada não vira `503`.
9. Timeout e `502`/`504` **em escrita** produzem `ErrIndeterminateResult`, não `ErrUpstreamTimeout` nem `ErrUpstreamUnavailable`.
10. `ErrInvalidUpstreamData` é produzido quando o corpo não corresponde ao contrato.
11. `context.Canceled` não é classificado como falha do serviço dependente.
12. Circuit breaker aberto responde `503`.
13. Logs duplicados eliminados.
14. Contrato público preservado, salvo alteração aprovada.
15. `go test ./...` passa.
16. Estrutura geral do projeto mantida.

### 3.16 Confirmar antes de implementar

- O envelope de erro já existente em `internal/handler/response.go` e eventual guideline interno de API.
- Qual modelo de autenticação o adapter usa (credencial técnica ou token do usuário) — muda o tratamento de `401`/`403` (3.4).
- O significado real de cada status devolvido pelo ECS de consentimento: `404` é mesmo "não encontrado"? `409` existe no contrato? `400`/`422` são erro do cliente ou da integração?
- Quais status podem vir do ALB, gateway ou proxy em vez da aplicação.
- Todos os pontos que hoje retornam `err.Error()` ao cliente.

Não presumir contrato. Onde não houver confirmação, registrar a dúvida no código e tratar como status inesperado.

---

## 4. Type assertion sem checagem causa panic

**Arquivo:** `internal/infra/consent_http_adapter.go` — nos três métodos

**O que está errado**

```go
req.Header.Set("x-itau-correlationID", ctx.Value("correlationID").(string))
```

Se o valor não estiver no contexto, isso entra em **panic**. Acontece em teste unitário, job, warm-up, ou qualquer chamada que não venha do handler HTTP.

**Correção**

```go
func stringFromCtx(ctx context.Context, key ctxKey) string {
    v, _ := ctx.Value(key).(string)   // vazio se ausente
    return v
}
```

---

## 5. DELETE devolve erro quando o consentimento já estava revogado

**Arquivo:** `internal/handler/handler.go` — `DeleteConsentimentos`

**O que está errado**

O serviço deles retorna `404` na segunda chamada de DELETE. Se você propagar isso, o cliente vê erro numa operação que **funcionou**.

Acontece com retry, com conexão instável, ou com o usuário tocando "revogar" duas vezes: ele revoga com sucesso, vê "erro ao revogar", tenta de novo, vê erro de novo — e já estava revogado desde a primeira vez.

**Correção**

```go
err := h.deleteConsentUseCase.Execute(ctx, funcional)

switch {
case err == nil, errors.Is(err, apperrors.ErrResourceNotFound):
    c.Status(http.StatusNoContent)     // 204, sem body — o estado desejado foi alcançado
default:
    WriteApplicationError(c, ctx, err) // do item 3
}
```

`204` com body é violação de protocolo e quebra alguns clientes. Use `c.Status`, não `c.JSON`.

---

# Parte 2 — Fazer na sequência

## 6. Cliente HTTP criado a cada tentativa

**Arquivo:** `internal/infra/consent_http_adapter.go`

```go
client := &http.Client{Timeout: a.timeout}   // dentro da closure, dentro do loop
```

Hoje funciona porque cai no transport padrão, mas é frágil: basta alguém definir um `Transport` inline e você passa a fazer handshake TCP+TLS por requisição. E o transport padrão tem `MaxIdleConnsPerHost: 2` — um BFF fala com poucos hosts e muito volume, exatamente o cenário que esse valor estrangula. Acima de 2 requisições simultâneas, as demais abrem conexão nova toda vez.

Crie o cliente **uma vez** no construtor e guarde no struct:

```go
transport := &http.Transport{
    Proxy:                 http.ProxyFromEnvironment,
    MaxIdleConns:          100,
    MaxIdleConnsPerHost:   100,  // crítico para BFF; o default é 2
    MaxConnsPerHost:       200,
    IdleConnTimeout:       30 * time.Second,  // manter MENOR que o idle timeout do ALB interno
    TLSHandshakeTimeout:   5 * time.Second,
    ExpectContinueTimeout: 1 * time.Second,
    ForceAttemptHTTP2:     true,
}

a.client = &http.Client{Timeout: cfg.Timeout, Transport: transport}
```

O `IdleConnTimeout` precisa ficar abaixo do idle timeout do ALB interno (padrão 60s). Se ficar acima, o ALB fecha a conexão ociosa antes de você e o pool entrega uma conexão morta para a próxima requisição — o que aparece como `unexpected EOF` esporádico, principalmente no POST, já que o Go só retenta sozinho métodos idempotentes.

E drene o corpo antes de fechar, senão a conexão não volta pro pool:

```go
defer func() {
    io.Copy(io.Discard, io.LimitReader(resp.Body, 4<<10))
    resp.Body.Close()
}()
```

---

## 7. Validação do corpo do POST

**Arquivo:** `internal/handler/handler.go` + DTO de entrada

**a) Não faça bind direto num struct de domínio.** Isso deixa o cliente controlar todos os campos, inclusive os que você adicionar no futuro. Use um DTO explícito com só o que ele pode enviar:

```go
type criarConsentimentoRequest struct {
    ConsentimentoEmLiberarDados *bool  `json:"consentimentoEmLiberarDados" binding:"required"`
    Finalidade                  string `json:"finalidade" binding:"required"`
}
```

**b) `*bool` não é preciosismo.** Com `bool`, o Gin não distingue "campo ausente" de `false` — ambos chegam `false`. Num campo de consentimento, isso registra "não autorizado" quando o cliente apenas esqueceu de mandar o campo. Ponteiro + `required` força a diferença.

**c) Erro de bind:** `400` para JSON malformado, `422` para JSON válido com regra violada. Não devolva o `err.Error()` do validator — ele expõe nomes de campos internos.

**d) Limite de tamanho.** Sem isso, um corpo de 500 MB consome a memória da task:

```go
c.Request.Body = http.MaxBytesReader(c.Writer, c.Request.Body, 64<<10)  // 64 KB
```

---

## 8. Status codes das rotas de escrita

**Arquivo:** `internal/handler/handler.go`

| Operação | Resultado | Status | Body |
|---|---|---|---|
| POST | Criado | `201 Created` | Representação criada |
| POST | Já existia | `200` ou `409` | Decidir e documentar |
| POST | JSON malformado | `400` | Erro padronizado |
| POST | Regra violada | `422` | Erro com o campo |
| DELETE | Removido ou já não existia | `204 No Content` | **vazio** |
| Qualquer | Downstream fora | `503` | Erro padronizado |
| Qualquer | Resultado indeterminado | `502` | Orientar a consultar antes de reenviar |

---

# Parte 3 — Quando for mexer na estrutura

## 9. O domínio serve de DTO para todo mundo

**Arquivos:** `internal/domain/models.go` + adapters

Hoje `domain.ConsentStatus` é três coisas ao mesmo tempo: o JSON que o outro ECS devolve, o modelo de domínio, e o JSON que o seu cliente recebe.

Resultado: se o time deles renomear `dataConsulta` → `dataDaConsulta`, **o contrato do seu app quebra** sem uma linha sua mudar. Absorver esse tipo de mudança é literalmente a função de um BFF. E campos internos que eles adicionem vazam direto pro cliente.

Sintoma visível: `DataConsulta` é `string` no seu domínio. Domínio não trabalha com data como string.

**Três structs:**

```go
// domain/models.go — sem tag json, tipos ricos
type ConsentStatus struct {
    Funcional    string
    Autorizado   bool
    ConsultadoEm time.Time
}
```

```go
// infra/consent_dto.go — contrato do OUTRO ECS
type consentStatusResponse struct {
    Funcional     string `json:"funcional"`
    Consentimento bool   `json:"consentimentoEmLiberarDados"`
    DataConsulta  string `json:"dataConsulta"`
}

func (r consentStatusResponse) toDomain() (domain.ConsentStatus, error) {
    consultadoEm, err := time.Parse(time.RFC3339, r.DataConsulta)
    if err != nil {
        return domain.ConsentStatus{}, fmt.Errorf("dataConsulta inválida %q: %w", r.DataConsulta, err)
    }
    return domain.ConsentStatus{r.Funcional, r.Consentimento, consultadoEm}, nil
}
```

```go
// handler/dto.go — contrato PÚBLICO do BFF (seu, versionável)
type consentimentoResponse struct {
    Funcional                   string `json:"funcional"`
    ConsentimentoEmLiberarDados bool   `json:"consentimentoEmLiberarDados"`
    DataConsulta                string `json:"dataConsulta"`
}
```

O contrato público continua **idêntico** ao de hoje — nada quebra pro cliente. A diferença é que passa a ser uma escolha sua, declarada num arquivo.

---

# Parte 4 — Itens menores

Todos reais, nenhum urgente. Um por linha.

| Item | Arquivo | O que fazer |
|---|---|---|
| Campo `dynamodbClient interface{}` não usado | adapter | Remover. Se DynamoDB entrar, é outra porta e outro adapter |
| Erros ignorados: `req, _ :=` e `Decode` sem check | adapter | Verificar. Se o downstream devolver HTML do WAF com `200`, você retorna struct vazia "com sucesso" |
| Retry sem jitter | adapter | Somar jitter ao backoff — sem ele, todas as tasks retentam sincronizadas |
| `time.Sleep` no retry ignora cancelamento | adapter | `select` com `<-ctx.Done()` |
| `gin.Default()` gera log não-estruturado | `main.go` | `gin.New()` + `Recovery()` + middleware de log JSON |
| Log duplicado entre use case e adapter | ambos | Use case loga o evento de negócio (`Info`); adapter loga o técnico (`Debug`) |
| Envelope de resposta inconsistente | `response.go` | Um formato só, inclusive para erros, nas quatro rotas |
| Validação do JWT | `jwt.go` | Confirmar que valida assinatura via JWKS + `exp`/`iss`/`aud`, não só decodifica |
| Retry em dois níveis | conversa com o outro time | Se vocês retentam 3× e eles 2× contra o banco, um pico vira 6× de carga no recurso mais escasso |

---

# Checklist: 15 minutos, sem depender de ninguém

**No CloudWatch — métricas do ALB interno, 30 dias:**

- [ ] `TargetResponseTime` (p99 e **Máximo**) → dimensiona o `WriteTimeout` do item 2
- [ ] `HTTPCode_ELB_5XX_Count` vs `HTTPCode_Target_5XX_Count` → separa erro do balanceador de erro da aplicação deles; é o que o mapeamento do item 3 precisa distinguir
- [ ] `HealthyHostCount` → os vales marcam os deploys deles e explicam os `503` (item 3.10)

**No repositório do serviço de consentimento:**

- [ ] Eles tratam `Idempotency-Key`? → **responde qual nível do item 2 você consegue implementar**
- [ ] O `DELETE` deles é idempotente? O que retorna na segunda chamada? → **confirma o item 5**
- [ ] Qual o timeout deles com o banco? → se for maior que o seu timeout para eles, o item 2 acontece de forma sistemática, não eventual
- [ ] Eles retentam contra o banco? → retry em dois níveis, Parte 4
- [ ] O `404` deles significa mesmo "não encontrado"? `409` existe no contrato? `400`/`422` são erro do cliente ou da integração? → item 3.16
- [ ] Eles registram o `x-itau-correlationID` que você manda? → item 3.13

**No seu código, verificações de 30 segundos cada:**

- [ ] Como o adapter autentica no outro ECS: credencial técnica ou token do usuário? → **decide o tratamento de `401`/`403` do item 3.4**
- [ ] Qual o envelope de erro atual em `internal/handler/response.go`? Existe guideline interno de API? → item 3.11
- [ ] Os métodos POST e DELETE do adapter têm o mesmo `lastErr` capturado por closure? → item 1
- [ ] Eles têm o mesmo loop de retry do GET? → item 2
- [ ] O adapter cria `&http.Client{}` dentro do loop de tentativas? → item 6
- [ ] `ctx.Value("correlationID").(string)` aparece nos três métodos? → item 4
- [ ] O `DELETE` propaga o `404` do downstream para o cliente? → item 5
- [ ] O `POST` faz `ShouldBindJSON` direto num struct de `domain`? → item 7
- [ ] O `POST` responde `200` em vez de `201`? O `DELETE` responde com body? → item 8
- [ ] O handler serializa o struct de domínio direto na resposta (`WriteData(c, 200, consent)`)? → item 9
- [ ] Existe o campo `dynamodbClient interface{}` no adapter? → Parte 4

---

# Resumo da ordem

**Agora (Parte 1):** 1, 2, 3, 4, 5
**Na sequência (Parte 2):** 6, 7, 8
**Quando for mexer na estrutura (Parte 3):** 9
**Conforme aparecer:** Parte 4

Antes de começar, rode o checklist de 15 minutos — ele confirma quais itens são reais no seu código e dá os números que os itens 2 e 6 precisam.
