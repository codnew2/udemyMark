# Plano de Desenvolvimento — CRM Agêntico

## 1. Objetivo

Construir uma memória corporativa inteligente e evolutiva dos grupos econômicos atendidos pelos users, a partir de interações inicialmente provenientes de e-mails do Outlook e transcrições de reuniões do Teams. A arquitetura deverá permitir adicionar futuramente WhatsApp, call reports e outras fontes.

O sistema deverá extrair fatos, identificar e correlacionar assuntos, reconciliar mudanças de informação e manter a situação atual dos clientes com histórico e evidências. Os dados devem permanecer vinculados ao grupo econômico, independentemente de mudanças de users.

## 2. Diretrizes do MVP

- Começar com um serviço de IA no ECS, organizado em etapas explícitas, em vez de vários agentes autônomos.
- Tratar o grupo econômico como associação já fornecida em `dados-brutos`, validando-a e preservando-a.
- Usar fatos estruturados como base para a consolidação da memória.
- Preservar fontes originais, versões e fatos anteriores, mesmo quando forem superados.
- Separar coleta e processamento de IA por mensageria assíncrona.
- Empregar correlação híbrida: identificadores e filtros determinísticos primeiro; busca semântica e LLM quando necessário.
- Não assumir que toda interação exige uma nova versão de resumo.

## 3. Arquitetura proposta

```text
Outlook / Teams / futuros canais
             |
             v
     Ingestão e normalização
             |
             v
     DynamoDB: dados-brutos
             |
             v
        Evento na SQS
             |
             v
      Worker de IA no ECS
             |
             +--> Extração de fatos
             +--> Identificação de assuntos
             +--> Correlação semântica
             +--> Reconciliação de fatos
             +--> Consolidação de situação
             |
             v
       Persistência estruturada
             |
             +--> fatos-cliente
             +--> assuntos-cliente
             +--> versoes-assunto
             |
             v
      Timeline / Memória / Consultas
```

A SQS desacopla a chegada das interações do processamento de IA. A ordem de chegada não equivale necessariamente à ordem dos acontecimentos; o serviço deve lidar com mensagens atrasadas, duplicadas e concorrentes.

## 4. Estruturas de dados

| Estrutura | Responsabilidade |
|---|---|
| `dados-brutos` | Guardar e-mails, transcrições e metadados originais, já associados ao grupo econômico. |
| `fatos-cliente` | Guardar fatos individuais extraídos, suas evidências e relações com outros fatos. |
| `assuntos-cliente` | Manter a situação atual consolidada de cada assunto de negócio. |
| `versoes-assunto` | Preservar versões imutáveis das situações consolidadas e o que mudou em cada atualização. |

### 4.1 Exemplo de `fatos-cliente`

```json
{
  "fatoId": "FATO-002",
  "grupoEconomicoId": "GRUPO-123",
  "assuntoId": "ASSUNTO-001",
  "tipo": "decisao",
  "descricao": "Cliente informou que não será mais necessário desacelerar as obras.",
  "dataReferencia": "2026-11-08T14:30:00Z",
  "tipoDataReferencia": "data_interacao",
  "dataExtracao": "2026-11-08T14:35:00Z",
  "fonte": {
    "interacaoId": "EMAIL-002",
    "canal": "email",
    "trecho": "não será mais necessário desacelerar o ritmo das obras"
  },
  "relacoes": [
    {
      "fatoId": "FATO-001",
      "tipo": "supera"
    }
  ],
  "statusValidacao": "validado"
}
```

**Observações:** IDs e datas são ilustrativos. `FATO-001` representa a intenção anterior de desacelerar. A relação `supera` não apaga o fato antigo. `statusValidacao` deve resultar de verificações reais, não de uma autodeclaração da LLM. Quando a data do fato não é conhecida, a data da interação deve ser identificada como referência, sem afirmar que o acontecimento ocorreu naquele dia.

### 4.2 Exemplo de `assuntos-cliente`

```json
{
  "assuntoId": "ASSUNTO-001",
  "grupoEconomicoId": "GRUPO-123",
  "titulo": "Cronograma das obras",
  "status": "em_andamento",
  "situacaoAtual": "Cliente inicialmente considerou desacelerar as obras, mas posteriormente descartou essa possibilidade e informou a inclusão de um novo projeto.",
  "versaoAtual": 2,
  "dataInicioAssunto": "2026-11-01T09:00:00Z",
  "dataUltimaInteracao": "2026-11-08T14:30:00Z",
  "dataUltimaAtualizacaoIA": "2026-11-08T14:35:00Z",
  "fontesSituacaoAtual": [
    { "interacaoId": "EMAIL-001", "canal": "email" },
    { "interacaoId": "EMAIL-002", "canal": "email" }
  ]
}
```

### 4.3 Exemplo de `versoes-assunto`

```json
{
  "assuntoId": "ASSUNTO-001",
  "versao": 2,
  "grupoEconomicoId": "GRUPO-123",
  "titulo": "Manutenção do ritmo das obras e novo projeto",
  "situacaoConsolidada": "Cliente inicialmente considerou desacelerar as obras, mas posteriormente descartou essa possibilidade e informou a inclusão de um novo projeto.",
  "alteracoes": {
    "informacoesMantidas": ["Planejamento do cronograma das obras"],
    "informacoesSuperadas": ["Intenção de desacelerar as obras"],
    "informacoesNovas": ["Desaceleração descartada", "Inclusão de novo projeto"]
  },
  "fontes": [
    { "interacaoId": "EMAIL-001", "canal": "email" },
    { "interacaoId": "EMAIL-002", "canal": "email" }
  ],
  "novaInteracaoId": "EMAIL-002",
  "versaoAnterior": 1,
  "dataGeracao": "2026-11-08T14:35:00Z"
}
```

A atualização de `assuntos-cliente` e a criação de `versoes-assunto` devem ser coordenadas com controle de concorrência e, quando aplicável, transação no DynamoDB.

## 5. Estratégia de correlação de assuntos

1. Restringir os candidatos ao **mesmo grupo econômico**.
2. Aproveitar relações determinísticas, como ID de thread, identificadores de operação, projeto ou documento.
3. Comparar embeddings do novo fato com representações semânticas dos assuntos candidatos.
4. Utilizar a LLM apenas quando o vínculo permanecer ambíguo ou exigir interpretação contextual.
5. Registrar o motivo da associação, as evidências e o nível de confiança; casos incertos devem ser revisados ou mantidos pendentes.

### Decisão de custo

Para poucos assuntos por grupo econômico, começar com embeddings armazenados junto aos metadados dos assuntos e similaridade calculada no serviço pode ser suficiente. Avaliar `pgvector` ou OpenSearch somente quando volume, latência e medições de custo justificarem. Não presumir que uma infraestrutura vetorial dedicada seja mais barata.

O índice semântico é auxiliar; o DynamoDB permanece como fonte de verdade dos registros de negócio.

## 6. Roadmap de desenvolvimento

**Estimativa preliminar:** 8–12 semanas para um desenvolvedor, considerando a coleta de dados já encaminhada. Não inclui homologações, aprovações de segurança ou atrasos de infraestrutura. As etapas podem se sobrepor.

| Etapa | Duração indicativa | Atividades principais | Entregável |
|---|---|---|---|
| 1. Modelagem e contratos | 1 semana | Definir JSONs, IDs, datas, evidências, status e relacionamentos. | Contratos e exemplos validados. |
| 2. Ingestão assíncrona | 1 semana | Integrar `dados-brutos`, SQS e worker ECS; retries, DLQ e idempotência. | Conteúdo processado com segurança. |
| 3. Extração de fatos | 1–2 semanas | Extrair fatos atômicos, tipos, datas, entidades, evidências e incertezas. | `fatos-cliente` preenchida. |
| 4. Identificação e correlação | 1–2 semanas | Filtrar candidatos, aproveitar threads, buscar semanticamente e validar vínculos. | Fatos vinculados a assuntos. |
| 5. Reconciliação | 1–2 semanas | Detectar confirmação, atualização, contradição, cancelamento e conclusão. | Relações entre fatos preservadas. |
| 6. Memória evolutiva | 1–2 semanas | Consolidar `situacaoAtual`, versionar e preservar fontes. | Situação atual e histórico consultáveis. |
| 7. Timeline e consultas | 1 semana | Consultar por cliente, assunto e período; ordenar fatos relevantes. | Timeline inicial e endpoints. |
| 8. Validação e robustez | 1–2 semanas | Testar falhas, duplicações, mensagens fora de ordem, autorização, custos e observabilidade. | MVP validado. |

## 7. Critérios de aceitação

- [ ] Um e-mail pode gerar vários fatos individuais com evidências verificáveis.
- [ ] Um novo e-mail da mesma thread é associado ao assunto correto.
- [ ] Uma transcrição do Teams pode continuar um assunto iniciado por e-mail.
- [ ] Uma decisão nova pode superar um fato anterior sem apagá-lo.
- [ ] A situação atual é atualizada com base nos fatos reconciliados, preservando contexto histórico relevante.
- [ ] Cada versão do assunto registra data e fontes utilizadas.
- [ ] A timeline pode ser consultada por grupo econômico, assunto e período.
- [ ] O reprocessamento da mesma interação não duplica fatos nem versões.
- [ ] Mensagens concorrentes ou atrasadas não sobrescrevem indevidamente o conhecimento mais recente.
- [ ] O acesso respeita as autorizações do users e as regras do banco.

## 8. Requisitos não funcionais prioritários

- **Confiabilidade:** não perder interações ou fatos durante falhas temporárias.
- **Rastreabilidade e auditabilidade:** vincular afirmações e decisões às fontes e versões.
- **Consistência:** preservar fatos históricos ao atualizar o estado atual.
- **Idempotência:** evitar duplicação em reprocessamentos.
- **Escalabilidade:** suportar aproximadamente 1.800 users e novos canais.
- **Extensibilidade:** manter a lógica central independente da origem da interação.
- **Segurança e privacidade:** criptografia, controle de acesso e proteção dos dados dos clientes.
- **Qualidade da IA:** validar evidências, registrar confiança e tratar casos ambíguos.
- **Disponibilidade e observabilidade:** retries, DLQ, métricas, logs e rastreamento do processamento.
- **Assincronicidade:** não bloquear a coleta enquanto a IA processa.

## 9. Evolução após o MVP

- Gestão completa de pendências: responsáveis, prazos, estados, alertas e regras de conclusão.
- Resumo executivo consolidado de todos os assuntos do grupo econômico.
- Inclusão de WhatsApp, call reports e outros canais por adaptadores.
- Busca vetorial dedicada, se métricas comprovarem necessidade.
- Orquestração multiagente mais sofisticada, somente se trouxer benefício demonstrável.

## 10. Primeira atividade recomendada

Detalhar e validar o contrato de `fatos-cliente` antes de implementar os prompts e o fluxo de correlação. Definir especialmente: identidade e deduplicação dos fatos, vínculo com fontes, semântica das datas, tipos de relação entre fatos, status de validação e regras para atualizações concorrentes.

**Princípio central:** a memória consolidada deve ser derivada de fatos estruturados e reconciliados, e não ser a única fonte de verdade do sistema.
