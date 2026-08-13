# Efficon

SaaS multi-tenant de gestão de obras para construtoras e incorporadoras. Cobre a obra ponta a ponta, da incorporação ao pós-obra: orçamento, planejamento, suprimentos, execução em campo, qualidade, financeiro, entrega da unidade e atendimento em garantia.

**Produção:** https://sistema.efficongest.com.br
**Escala atual:** 9 módulos, 49 submódulos, 116 tabelas, 28 organizações.
**Documentação de produto:** o PRD completo vive fora do repositório, no vault do Obsidian do gestor (`PRD Efficon (mestre).md` mais uma nota por módulo). A versão condensada está no Knowledge do projeto Lovable.

---

## Stack

- **Front:** React 18, TypeScript, Vite 5, Tailwind 3, shadcn/ui sobre Radix, React Router 6, TanStack Query 5, Recharts, Sonner.
- **Back:** Supabase (Postgres com RLS, Auth, Storage, Edge Functions em Deno).
- **Mobile:** Capacitor 7 (Android e iOS), casca WebView do app de campo.
- **Testes:** Vitest 3 com Testing Library e jsdom; Playwright para e2e.
- **Documentos:** jsPDF com autotable, html2canvas, pdfjs-dist, xlsx.
- **Construção:** projeto Lovable. As rodadas de código são despachadas pelo CTO e auditadas pelo diff.

## Como rodar

```bash
npm install
npm run dev          # servidor de desenvolvimento (Vite)
npm run build        # build de produção
npm run lint         # ESLint
npm test             # suíte completa (Vitest)
npm run test:watch   # suíte em modo observação
npm run test:e2e     # ponta a ponta (Playwright, tests/e2e/run.mjs)
```

App de campo:

```bash
npm run build-apk    # Android debug
npm run build-aab    # Android release (bundle)
```

Migração de base legada (rito próprio, ver `scripts/migration/legacy-to-supabase/`):

```bash
npm run migration:preflight
npm run migration:dry-run
npm run migration:execute
```

## Estrutura

```
src/
  pages/              telas, uma por rota (Medicoes, ContratosServico, GestaoDefeitos, ...)
  components/
    auth/             AuthShell (casca de login e senha), RequireRoutePerm, RequireModule
    ui/               design system da casa sobre shadcn
    <dominio>/        componentes por domínio (medicoes, contratos, acompanhamento, usuarios, ...)
  lib/
    permissions.ts    motor de permissões (useHasPerm, checkPerm)
    permissoes/       catalogo.ts, perfis-modulo.ts, gestao-usuarios.ts
    erp/              regras de domínio (medicao-ciencia, medicao-status, medicao-fluxo, fluxo-projetado, ...)
    pos-obra/         catálogo do SAC, garantia do portal, disciplina-catalogo
    acompanhamento/   matriz, rastro da célula, unidades personalizadas
    relatorios/       cabecalho-filtros.ts (fonte única do cabeçalho)
  contexts/           OrgContext (organização ativa, papel, permissões), UserContext
  test/               suíte principal e guards por rodada
supabase/functions/
  _shared/            código compartilhado entre edges (medicao-efeitos, permissao-usuario, e-mail)
  auth-email-hook/    personaliza os e-mails de conta (webhook do Auth)
  convite-unidade-publico/   portal do cliente, sem sessão Supabase
  contrato-ciencia-publico/  ciência de contrato e de medição por link
  admin-*/            hub interno da equipe Efficon
scripts/migration/    migração da base legada
tests/e2e/            ponta a ponta
.lovable/memory/      decisões de arquitetura por tema (auth, financeiro, features, backlog)
```

## Multi-tenant e segurança

A organização é o tenant, e o isolamento é garantido no BANCO, não na tela.

- Toda tabela carrega `organization_id` direto na linha, com RLS por organização.
- Rota direta sem permissão devolve **zero linhas e zero dado no payload de rede**. Menu escondido não é proteção.
- Ser membro não dá acesso a tudo: a policy exige membro ativo E o vínculo específico (owner, designado ou permissão).
- Sem "Visualização global", o usuário só enxerga as obras marcadas para ele (`obras_acl_restrict`).
- `is_super_admin(auth.uid())` é a primeira condição das policies.
- Acesso externo (proprietário de unidade, síndico, fornecedor) é **sempre por token**, nunca por conta Supabase, e toda escrita passa por edge function.

## Permissões

Chave canônica `grupo.sub.acao`, gravada em `organization_members.permissions` (jsonb). Catálogo em `src/lib/permissoes/catalogo.ts`.

- **A chave técnica NUNCA muda.** Mudam rótulo, hierarquia de exibição e gates.
- O catálogo é constante de front: reorganizar é zero migração.
- Bypass total para owner, admin e super_admin.
- **Gate comercial acima do bypass:** módulo em `organizations.disabled_modules` bloqueia todos, inclusive owner. `checkPerm` exige `disabledModules` obrigatório, sem default silencioso.
- Default negado: sem bypass, só passa com a chave explicitamente `true`.
- **Dívida conhecida:** RLS granular por ação não existe. O gate fino é front mais espelho pontual em edge.

## Convenções de dados

| Convenção | Regra |
|---|---|
| Payload | jsonb camelCase na coluna `payload`. Campo novo entra ADITIVO, sem migração. Leitura tolerante |
| Dinheiro | Inteiro em centavos, nunca float |
| Estado derivável | Deriva na leitura, nunca grava (vencido, paga, parcelada, status efetivo da medição) |
| Exclusão | Registro errado é CANCELADO com rastro datado. Sem remoção física |
| Arquivos | Bucket privado com caminho por `organizationId`. Proibido base64 no payload. Download por URL assinada curta |
| Rastro | `created_by` mais datas de estado |
| Ausência | Dado ausente exibe ausência honesta, nunca zero |
| Pontuação | Zero travessão em texto de interface. Use "·" ou vírgula |

## Fontes únicas (não crie uma segunda)

1. **Catálogo do SAC** (`prazos_garantia.payload`) é a fonte única de disciplina de todo o sistema. `categorias_defeito` é fóssil vazio.
2. **Valor vigente do contrato** é o materializado (`contrato.itens`, `valorContrato`), nunca derivado em runtime.
3. **`TIPOS_LOCAL`** (`lib/acompanhamento/locais-arvore.ts`) é o vocabulário único de categoria de local.
4. **`statusMedicaoUI`** é a fonte única de rótulo e cor da medição.
5. **`calcularBaixa`** é o helper único da baixa; **`dinheiroDaBaixa`** é o único conversor do que entra no caixa.
6. **`lib/relatorios/cabecalho-filtros.ts`** é a fonte única do cabeçalho de filtros.

## Regras de negócio que não são óbvias no código

- **Medição:** aprovar internamente não deixa Aprovada, leva a `aguardando_ciencia`. Quem promove é a ciência do fornecedor. A promoção é derivada por leitura (`statusEfetivoMedicao`), sem escrita. Efeitos de dinheiro são idempotentes por `efeitosContratoAplicados`.
- **Contrato:** aditivo só altera o contrato quando fica VIGENTE, e `materializar()` é a única porta de efeitos. `registrarAditivo` recusa contrato com divergência de um centavo ou mais entre `valorContrato` e a soma dos itens.
- **Financeiro:** caixa é dinheiro, o desconto nunca passa pelo caixa. O desconto abate o que falta pagar e o título quita 100%.
- **Pós-obra:** no portal do cliente, a garantia conta pela data de entrega global da obra (`obras.previsao_termino`), nunca pela data da unidade. Sem data global, não calcula nem inventa.
- **Qualidade:** diário enviado não aceita UPDATE. Foto obrigatória na inspeção trava o envio para validação, não a finalização.

## Testes

A suíte é a rede de segurança do projeto e roda inteira em toda rodada de código.

- `src/test/` guarda a suíte principal e os **guards por rodada** (`<sigla>-<rodada>.guards.test.tsx`), que congelam o comportamento entregue.
- Guard sintético não substitui prova de navegador: já houve caso de guard verde com o fluxo real quebrado.
- Instáveis conhecidos: `ntf-p3-caixa-trabalho` e `raio-p4b-raio-arbitrario` caem por timeout sob carga e passam isolados.
- `contrato-schema-canonico` é skip condicional por credencial.

## O que NÃO fazer

- Não crie status novo, lista paralela de disciplina, motor paralelo de gravação nem segunda fórmula para a mesma conta.
- Não embuta migração, índice, extensão ou policy numa rodada de tela: é rodada própria autorizada.
- Não grave base64 em payload nem use URL pública permanente.
- Não altere chave técnica de permissão.
- Não escreva em organização de cliente. A organização de teste é a única gravável.
- Não conserte fora do escopo pedido: reporte e espere.

## Rito de entrega

Cada frente trabalha num draft do Lovable e fecha com o bloco PRONTO PRA MERGE. O gestor coloca na main, o CTO audita task a task pelo diff real com as provas transcritas, e o **publish é sempre clique do gestor**.
