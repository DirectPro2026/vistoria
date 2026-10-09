# Fase 8.4 — Investigação de causas-raiz e correção incremental Sequelize 5 → 6

**Classificação final: AMARELO**

## 1. Estado Git e escopo

- Repositório: `zapproions-tech/zappro`
- Branch isolada: `audit/fase8-sequelize6-migration-20261009`
- SHA inicial conferido: `2dc38204f056ed358e15d78eeef5541fd7e6706b`
- SHA final conferido: `8e4e2af253c017cb0a6e56a857819bdda266d562`
- Commit final: [8e4e2af — teste de persistência de ContactList no PostgreSQL descartável](https://github.com/zapproions-tech/zappro/commit/8e4e2af253c017cb0a6e56a857819bdda266d562)
- Workflow da execução final: [Run 37974452150](https://github.com/zapproions-tech/zappro/actions/runs/37974452150)
- Workflow auxiliar Fase 8.1 do mesmo SHA: [Run 37974452022](https://github.com/zapproions-tech/zappro/actions/runs/37974452022) ainda estava `in_progress` no momento da consolidação; não foi tratado como aprovação nem usado para mascarar o resultado do build.
- `main` permaneceu em `2d31437f6f4b46fd75951be8fbd45d7f87262002`.
- PR #50 permaneceu aberto e não mesclado.
- Nenhum acesso ou alteração à VPS operacional, banco operacional, deploy ou merge foi realizado.

### Commits da fase

- `eb1cdea65a5a41576fb361e07bb17083ca5456d0` — piloto inicial de atributos de criação; a primeira execução revelou problema de posição do decorator e uma asserção de teste inadequada.
- `907af8814b83c6980352cd872402a7c602e97f54` — correção da posição do decorator e do cenário de validação.
- `4725db53af4434e820718ee601d681758efed1d9` — teste ajustado para respeitar o contrato atual de `AppError` (a classe não estende `Error`).
- `8e4e2af253c017cb0a6e56a857819bdda266d562` — teste de persistência em PostgreSQL descartável e etapa correspondente na workflow.

## 2. Diagnóstico representativo confirmado: ContactList

### Arquivos e contrato analisados

- Modelo: `backend/src/models/ContactList.ts`
- Migration: `backend/src/database/migrations/20220315110000-create-ContactLists-table.ts`
- DTO/serviço: `backend/src/services/ContactListService/CreateService.ts`
- Controller: `backend/src/controllers/ContactListController.ts`
- Testes: `backend/src/services/ContactListService/__tests__/CreateService.spec.ts` e `CreateService.integration.spec.ts`

### Evidências

1. O modelo usava `class ContactList extends Model<ContactList>` no candidato Sequelize 6. A classe não declarava atributos de criação separados dos atributos da instância.
2. A migration define `id` como `INTEGER`, `autoIncrement: true`, `primaryKey: true`; `name` é `allowNull: false`; `createdAt` e `updatedAt` são obrigatórios no banco, mas são gerenciados pelo Sequelize. Portanto, o DTO de criação deve poder omitir `id` e timestamps.
3. O serviço aceitava `companyId: number | string`, enquanto o modelo declara `companyId: number`. O controller usa `req.user.companyId`, declarado como `number`, e sobrescreve o valor vindo do body ao montar o argumento do serviço.
4. A migration permite `companyId` nulo, mas o fluxo de criação da aplicação exige um `companyId` numérico autenticado. A migration não foi alterada; a correção não relaxa o requisito do serviço.
5. Foi adicionado `ContactListCreationAttributes` com `id`, `createdAt` e `updatedAt` opcionais e `name`/`companyId` obrigatórios; o modelo passou a usar `Model<ContactList, ContactListCreationAttributes>`. O DTO do serviço foi estreitado para `companyId: number`.
6. No build final, o diagnóstico de `ContactListService/CreateService.ts` desapareceu. A primeira tentativa de implementação introduziu TS1206 por colocar `@Table` sobre o alias de tipo; isso foi corrigido e o erro não aparece no build final.

**Conclusão deste caso:** a tipagem de criação do modelo era insuficiente para a API de criação do Sequelize 6, e o DTO tinha também uma incompatibilidade real de primitivo. A correção combinada está validada para este modelo. Não se generaliza automaticamente a conclusão aos outros modelos.

## 3. Testes e códigos de saída da execução final

Artefato integral: `fase83-candidate-diagnostic-8e4e2af253c017cb0a6e56a857819bdda266d562.zip`.

| Etapa | Resultado |
|---|---|
| Node | `v22.23.3` |
| npm | `10.9.9` |
| Candidato Sequelize | `6.37.8` |
| Candidato sequelize-typescript | `2.1.6` |
| TypeScript | `4.9.5` |
| `npm install --package-lock-only` para o candidato temporário | exit 0 |
| `npm ci --include=optional --foreground-scripts` | exit 0 |
| `npm ls --all --json` | exit 0; stderr vazio |
| Árvore de versões-chave | exit 0 |
| `npm run build` | **exit 2 — ainda falha** |
| Testes focalizados de ContactList | exit 0; 2 suítes, 4 testes aprovados |
| Seeds em PostgreSQL descartável | exit 0 |
| Teste de persistência PostgreSQL | exit 0; 1 teste aprovado |
| Migrations completas (`up`) no PostgreSQL descartável | exit 0 |
| Rollback direcionado de cinco migrations alteradas | exit 0 |
| Integridade de dependências do candidato | PASS |

O teste de integração confirmou que a criação persiste o registro, recebe `id` gerado, recebe `createdAt`/`updatedAt` como `Date` e mantém `companyId = 1`. O registro de teste é removido ao final. O banco é o PostgreSQL descartável da workflow, não um banco operacional.

**Limite importante:** foi validado `up` completo e rollback direcionado de cinco migrations, não um rollback completo de todo o histórico.

## 4. Diagnósticos TypeScript: antes e depois

- Antes do piloto: **74** erros em **42** arquivos.
- Depois do piloto: **73** erros em **41** arquivos.
- Redução: **1 diagnóstico**, removido do caso `ContactListService/CreateService.ts`.
- Build ainda falha; a migração Sequelize 6 não está aprovada.

Códigos no build final:

- `TS2322`: 44
- `TS2345`: 28
- `TS2769`: 1

Não foram contados erros como resolvidos apenas por alteração do contador: o caso ContactList passou por testes unitários e integração de persistência. Os demais 73 diagnósticos continuam abertos ou exigem decisão sobre contrato.

## 5. Triagem adicional e bloqueios identificados

### Announcement — requer decisão sobre o contrato

A investigação dos arquivos `AnnouncementController.ts`, `AnnouncementService/CreateService.ts`, `AnnouncementService/UpdateService.ts`, `models/Announcement.ts` e migrations mostra divergência de tipos: controller/serviços declaram `priority` e `status` como `string`, enquanto modelo e banco definem `priority` como `number`/`INTEGER` e `status` como `boolean`/`BOOLEAN`. Não fiz conversões automáticas. É necessário confirmar os payloads reais do frontend/API e decidir se a normalização pertence ao controller/DTO ou se o contrato público deve mudar.

### Demais erros

O inventário atualizado mantém cada diagnóstico do inventário inicial e registra se ele continua no build final. Os demais erros estão marcados como hipóteses em investigação até que cada modelo, migration, DTO, consumidor e comportamento sejam comparados. Os grupos continuam sendo:

- DTOs versus atributos de criação/atualização;
- `string` versus `number`;
- strings versus datas e timezone;
- coleções e associações;
- atributos de modelo;
- assinatura/sobrecarga ORM.

O descompasso de `CheckIntegrations` — assinatura `Promise<Integrations>` retornando `integrations.dataValues` — continua sendo item de revisão pendente; não foi alterado sem confirmar os consumidores.

## 6. Arquivos alterados nesta fase

1. `.github/workflows/fase83-sequelize-typescript-errors.yml`
2. `backend/src/models/ContactList.ts`
3. `backend/src/services/ContactListService/CreateService.ts`
4. `backend/src/services/ContactListService/__tests__/CreateService.spec.ts`
5. `backend/src/services/ContactListService/__tests__/CreateService.integration.spec.ts`

Não houve alterações em outros modelos/serviços nesta fase. Os testes e a workflow usam o PostgreSQL efêmero de GitHub Actions.

## 7. Integridade de manifests/lockfiles e segurança

Hashes SHA-256 observados antes e depois, registrados pela workflow final:

- `backend/package.json`: `0df4059fdd1c875d28a5ee27290acac83319da41801eebe3e8fd2ee2503cf2a7` — inalterado.
- `backend/package-lock.json`: `d4758bc5aa5c2abba125f538d408da101644334fa9977caf816881e16606c88e` — inalterado.

O `package.json` oficial continua declarando `sequelize@5.22.5` e `sequelize-typescript@1.1.0`. O candidato `sequelize@6.37.8` / `sequelize-typescript@2.1.6` foi preparado em cópia temporária, não nos manifests oficiais. As overrides herdadas `ejs@3.1.10` e `protobufjs@7.6.6` permaneceram intactas porque os arquivos de dependências não foram alterados.

O workflow de diagnóstico terminou com status geral `success`, mas isso **não** aprova o build: `candidate_build_exit=2` e `candidate_typescript_error_lines=73`. A etapa de build captura o código de saída e continua para produzir testes e evidências.

## 8. Decisão e próximo passo

**AMARELO.** Um caso estrutural foi corrigido e validado sem `any`, casts de produção, supressões ou relaxamento de `strict`. Dependências candidatas, migrations e testes do piloto passaram. Porém, o build ainda falha com 73 diagnósticos e há divergências de contrato que impedem correções automáticas seguras.

Próximo passo recomendado: escolher o próximo modelo com dependências pequenas, reproduzir um único erro, comparar modelo/migration/DTO/consumidores e aplicar o padrão somente quando os atributos de criação estiverem demonstrados como causa. Para `Announcement`, decidir primeiro o contrato de `priority`/`status`. Manter a branch isolada; não atualizar manifests/lockfiles oficiais, não fazer merge/deploy e não alterar VPS/banco operacional sem autorização explícita.
