# Fase 8.3 — Relatório técnico incremental Sequelize 5 → 6

**Decisão: AMARELO — progresso real, mas 74 erros TypeScript continuam e a compatibilidade funcional não está aprovada.**

## 1. Estado e continuidade

- Repositório: `zapproions-tech/zappro`.
- Branch isolada: `audit/fase8-sequelize6-migration-20261009`.
- SHA de referência informado para a Fase 8.2: `cdbcdc8ca9200897bb0666c7f3641d1e5c3e23e6`.
- SHA atual verificado da branch: `2dc38204f056ed358e15d78eeef5541fd7e6706b`.
- Foram analisados os **16 commits intermediários** entre o SHA de referência e o atual; não foi reiniciada a branch nem descartado trabalho anterior.
- `main` continua em `2d31437f6f4b46fd75951be8fbd45d7f87262002`, sem alteração observada nesta fase.
- PR #50 não foi mesclado nem alterado por esta execução.
- Nenhuma VPS ou banco operacional foi acessado/modificado. PostgreSQL usado nos testes foi serviço descartável do GitHub Actions.

A execução mais recente é [GitHub Actions #37947821434](https://github.com/zapproions-tech/zappro/actions/runs/37947821434), SHA `2dc38204f056ed358e15d78eeef5541fd7e6706b`. O job aparece como `success` porque os diagnósticos e testes configurados terminaram; **o build TypeScript continua falhando**, código 2. O status geral da workflow não deve ser interpretado como build aprovado.

## 2. Integridade do candidato e preservação dos arquivos oficiais

Ambiente do runner: Node `v22.23.3`, npm `10.9.9`, `linux/x64`.

| Comando | Código de saída | Resultado |
|---|---:|---|
| `npm install --package-lock-only --ignore-scripts --include=optional --no-audit --no-fund` | 0 | Lockfile temporário Strategy B gerado |
| `npm ci --include=optional --foreground-scripts` | 0 | Instalação candidata aprovada |
| `npm ls --all --json` | 0 | Árvore completa íntegra; stderr vazio |
| `npm ls sequelize sequelize-typescript baileys sharp typescript --all` | 0 | Árvore crítica íntegra |
| `npm run build` | 2 | 74 diagnósticos TypeScript |
| Teste focalizado `ListService.spec.ts` | 0 | 2 testes passaram |
| `node dist/database/runDatabaseTasks.js migrate` | 0 | Todas as migrations aplicadas em PostgreSQL descartável |
| Teste de rollback dirigido | 0 | 5 migrations alteradas validadas em `up/down` dirigido |

Versões candidatas verificadas: `sequelize@6.37.8`, `sequelize-typescript@2.1.6`, `baileys@7.0.0-rc13`, `sharp@0.35.5`, `typescript@4.9.5`.

O `backend/package.json` original manteve SHA-256 `0df4059fdd1c875d28a5ee27290acac83319da41801eebe3e8fd2ee2503cf2a7`. O `backend/package-lock.json` original manteve SHA-256 `d4758bc5aa5c2abba125f538d408da101644334fa9977caf816881e16606c88e` antes e depois. O candidato foi recriado em diretório temporário e os manifests/lockfiles oficiais não foram atualizados.

As correções de segurança herdadas foram preservadas: os commits da Fase 8.3 não alteraram manifests nem lockfiles. Os overrides de `ejs@3.1.10` e `protobufjs@7.6.6` permanecem no manifesto inspecionado.

## 3. Comparação do build

O inventário original da Fase 8.1 registra 107 erros em 53 arquivos. O build reproduzido na cópia temporária Strategy B atual registra **74 erros em 42 arquivos**: queda de 33 diagnósticos e 11 arquivos com erro a menos. Isso é redução de volume, não prova de compatibilidade funcional.

| Código | Baseline candidato combinado | Candidato atual | Diferença |
|---|---:|---:|---:|
| `TS2322` | 65 | 44 | -21 |
| `TS2345` | 31 | 29 | -2 |
| `TS2554` | 7 | 0 | -7 |
| `TS2416` | 2 | 0 | -2 |
| `TS2365` | 1 | 0 | -1 |
| `TS2769` | 1 | 1 | 0 |
| **Total** | **107** | **74** | **-33** |

Comando exato: `npm run build`, executado na cópia candidata depois de instalar com Strategy B, sem `--legacy-peer-deps`. Código de saída registrado: `2`. Log integral em `fase83-candidate-diagnostic/40-candidate-build.log`; linhas de diagnóstico em `fase83-candidate-diagnostic/41-typescript-errors.txt`.

## 4. Classificação original por hipótese

O CSV original classifica as 107 mensagens assim; as categorias são hipóteses herdadas, não causas-raiz já provadas:

- 61: valor passado a `create/update/where/order` não corresponde ao tipo esperado.
- 32: incompatibilidade entre DTO e tipos de criação/atualização `Optional<Model, ...>`.
- 7: assinatura de `QueryInterface`/migração.
- 3: incompatibilidade geral de assinatura/API ORM.
- 2: propriedade declarada na subclasse conflita com `Model` herdado.
- 2: retorno de `count` agrupado usado como número.

## 5. Correções já presentes na branch e evidências

1. **Assinaturas de migrations Sequelize 6:** chamadas `addConstraint` migradas para o formato de opções com `fields` em cinco arquivos de migration. Os sete erros `TS2554` desapareceram. A migration completa executou em PostgreSQL descartável, e as migrations alteradas foram testadas com `up/down` dirigido.
2. **Contagem agrupada de ContactList:** `findAndCountAll` com `group` devolve uma coleção de contagens; o serviço usa `count.length` para expor o número de listas. Foram adicionados dois testes focalizados, ambos aprovados. O filtro por `companyId` foi mantido. Não houve teste de integração específico entre duas empresas nesta execução.
3. **Atributo `dataValues` dos modelos:** overrides incompatíveis foram removidos/ajustados ao longo dos commits. Uma revisão encontrou, contudo, uma inconsistência que precisa ser verificada antes de considerar esse grupo concluído: `CheckIntegrations` declara retorno `Promise<Integrations>`, mas o corpo ainda retorna `integrations.dataValues`. O tipo compila por causa da tipagem permissiva de `dataValues`, mas contrato declarado e valor retornado não coincidem. Não alterei esse retorno sem confirmar os consumidores.
4. **Timestamps de ApiUsages:** quatro atribuições passaram de `moment().format()` para `moment().toDate()`, alinhando o valor com a coluna Date. O build não apresenta mais esses diagnósticos; falta teste de persistência/leitura com timezone explícito.
5. **JSONB de agendas:** `ScheduleEntry` foi extraído para tipo compartilhado e aplicado a Company, Queue e Whatsapp, removendo arrays sem shape útil e `any[]` nesse grupo.
6. **Workflow de diagnóstico:** foram corrigidos a captura dos códigos de saída, a extração do log TypeScript, o tratamento de cancelamento e a serialização dos diagnósticos. As 16 mudanças intermediárias foram revisadas por commit/diff.

## 6. Erros restantes — classificação primária

A classificação abaixo é exclusiva por linha para facilitar a triagem. Ela é heurística e não declara causa-raiz comprovada; algumas mensagens podem envolver mais de uma categoria.

| Grupo primário | Erros | Leitura técnica |
|---|---:|---|
| G1 — DTO de create/update versus atributos inferidos do modelo | 26 | A assinatura de criação/atualização candidata é mais restritiva que DTOs usados por vários serviços. É preciso distinguir correção do DTO de definição correta dos atributos de criação do modelo. |
| G2 — Tipos primitivos string/number | 29 | Rota/formulário/DTO aceita string, enquanto modelo ou associação espera number. Conversão automática em lote pode alterar validação e comportamento de entradas vazias/inválidas. |
| G3 — Valores de data string versus Date | 13 | A entrada usa strings e o modelo declara Date. Conversão exige contrato de formato/timezone, não apenas cast. |
| G4 — Coleções/associações | 4 | Arrays de Wallet/ExtraInfo/ContactCustomField não correspondem ao tipo de persistência/associação esperado. |
| G5 — Outro atributo de modelo | 1 | Tipo de objeto usado em `where` de integração. |
| G6 — Sobrecarga/assinatura de consulta | 1 | `destroy` em ScheduledMessagesEnvio precisa ser comparado à assinatura Sequelize 6. |
| **Total** | **74** | **42 arquivos** |

O inventário atualizado por erro está em `fase83-erros-restantes.csv`, incluindo arquivo, linha, código, mensagem, grupo primário, hipótese e evidência necessária. O CSV original de 107 erros e os logs completos da Fase 8.1 estão preservados no pacote.

## 7. Avaliação de risco e critério de parada

A evidência sustenta que alguns grupos pequenos são corrigíveis: as chamadas de migrations e a contagem agrupada foram corrigidas e testadas. Porém, os erros remanescentes se distribuem por 42 arquivos e incluem pelo menos 29 incompatibilidades string/number, 13 datas e 26 incompatibilidades de payload de criação/atualização. Corrigi-los em massa exigiria casts ou conversões em vários fluxos, sem evidência suficiente de que a semântica original seria preservada. Isso acionaria os critérios de parada da própria fase.

**Não recomendo tentar zerar os 74 erros com casts, `any`, supressões, mudança global de strictness ou alteração ampla de tipos de modelos.** A próxima unidade segura é um modelo/serviço por vez, com tipo de criação explícito e teste comportamental de create/update. Antes de continuar, também deve ser esclarecido o contrato de retorno de `CheckIntegrations`.

## 8. Plano de validação funcional pendente

1. Definir tipos de atributos e de criação para um modelo por vez; iniciar por um par pequeno de serviço/modelo e manter o DTO real, sem `any` ou supressões.
2. Testar criação/atualização e atributos opcionais em PostgreSQL descartável.
3. Adicionar teste explícito de isolamento multiempresa: registros da empresa A não podem ser listados/alterados pela empresa B; manter filtros e permissões existentes.
4. Testar associações e carregamento de relações nos modelos alterados.
5. Testar transação com falha intermediária e confirmar rollback.
6. Executar migrations completas em banco descartável e rollback total em ambiente descartável separado; nesta execução houve `up` completo e rollback dirigido de cinco migrations, não rollback total de todo o histórico.
7. Separar os problemas preexistentes de ESLint e do teste de fuso horário; não alterar a expectativa de timezone sem contrato definido.
8. Só reavaliar a estratégia Sequelize 6 após build sem erros indevidos, testes funcionais, isolamento multiempresa e migrations/rollback aprovados.

## 9. Decisão final

**AMARELO.** A árvore de dependências continua íntegra; o build melhorou de 107 para 74 erros; os testes focados e as migrations/rollbacks dirigidos passaram em PostgreSQL descartável. A migração não está pronta para aprovação funcional. Não houve merge, deploy, alteração de `main`, PR #50, manifest/lockfile oficial ou ambiente operacional.

### SHA-256 de evidências principais

- `00-start-state.txt`: `0dc44d1cfb6f18c971f8c0d05ed484675b93d36e4a5e46aaf4580ac88ac77b60`
- `40-candidate-build.log`: `36c2946a74650be17cd7880a04d8620f26a3ae6ff8ba75c025b28ef58f4af89e`
- `41-typescript-errors.txt`: `b17db733be22df653809734e8b2372dfcd552239d2a7b96d743dafd73670cc93`
- `fase81-typescript-error-inventory.csv`: `e3bf289b5bdc337f4b9ffa0510d2fb14787441985ca296a21b34218fea4cbcde`
- `fase81-locked-candidates-run5.zip`: `94f20f385e36f471cbb3e147303d68a5a1188a4407a78be355505f36514f94a4`
