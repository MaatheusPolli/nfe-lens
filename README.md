# NFe Lens

Parser e visualizador de **NF-e** (modelo 55) e **NFC-e** (modelo 65) — o padrão de nota fiscal eletrônica definido pela SEFAZ/Receita Federal — escrito em TypeScript, com uma interface web em React para transformar o XML da nota em algo legível, pesquisável e exportável, direto no navegador.

> Projeto de portfólio com licença dupla: livre para uso pessoal e educacional (MIT), licença comercial para uso comercial. Ver [Licença](#licença).

**Código:** [github.com/MaatheusPolli/nfe-lens](https://github.com/MaatheusPolli/nfe-lens) · **Demo:** [maatheuspolli.github.io/nfe-lens](https://maatheuspolli.github.io/nfe-lens/)

---

## Sobre o projeto

Todo desenvolvedor que já trabalhou com fiscal, ERP ou integração de e-commerce no Brasil conhece essa cena: abrir o XML de uma NF-e num editor de texto para achar um valor. A especificação da SEFAZ organiza os dados numa árvore XML profunda, cheia de siglas (`ICMS00`, `vBC`, `pICMS`, `CSOSN`, `CFOP`), com mais de vinte grupos de tributação diferentes dependendo do regime da empresa — e não existia uma ferramenta simples para transformar isso em algo legível sem depender de um sistema fiscal completo.

O NFe Lens resolve isso como um **monorepo pnpm** com dois pacotes:

| Pacote | O que é | Depende de |
|---|---|---|
| [`packages/core`](./packages/core) (`@nfe-lens/core`) | Biblioteca TypeScript pura: lê o XML, valida a estrutura, mapeia para um modelo de domínio tipado e oferece exportação para JSON/CSV. Sem nenhuma dependência de UI — pode ser usada em Node, em um bundler de front-end, ou em qualquer runtime JS. | `fast-xml-parser` |
| [`apps/web`](./apps/web) (`@nfe-lens/web`) | Aplicação React que consome o `@nfe-lens/core` e oferece uma interface de arrastar-e-soltar, com abas de visualização, busca e exportação. | `@nfe-lens/core`, React, Tailwind, TanStack Table |

Fluxo de uso típico: o usuário arrasta um ou mais arquivos `.xml` para a aplicação web → o `@nfe-lens/core` faz o parse e a validação de cada um, 100% no navegador → a interface exibe o resultado em abas (Resumo, Itens, Tributos, Transporte, Cobrança, XML bruto) → o usuário pode exportar o resultado em JSON ou CSV.

---

## O que o parser (`@nfe-lens/core`) faz

### Entrada aceita
- XML de `NFe` "pura" (apenas o documento fiscal) **ou** `nfeProc` (o documento fiscal envolto pelo protocolo de autorização da SEFAZ, `protNFe`) — o parser detecta automaticamente qual dos dois formatos foi recebido.
- Modelo 55 (NF-e) e modelo 65 (NFC-e).

### O que é extraído e tipado
- **Identificação** (`ide`): número, série, data de emissão, modelo, ambiente (produção/homologação), natureza da operação, finalidade, indicador de presença, etc.
- **Emitente e destinatário** (`emit`/`dest`): CNPJ ou CPF, razão social, endereço.
- **Itens** (`det[]`): produto (código, descrição, NCM, CFOP, quantidade, valores) e todos os tributos por item.
- **Tributos**, modelados como *unions discriminadas* do TypeScript (o compilador garante que cada grupo tributário só possa ter os campos que realmente existem para aquele grupo):
  - **ICMS**: grupos `ICMS00`, `10`, `20`, `30`, `40`/`41`/`50`, `51`, `60`, `70`, `90` (regime normal) e `ICMSSN101`, `102`/`103`/`300`/`400`, `201`, `202`/`203`, `500`, `900` (Simples Nacional).
  - **PIS** e **COFINS**: por alíquota, por quantidade, não tributado ou outras operações — cobrindo os CSTs oficiais.
  - **IPI**: tributado e não tributado.
- **Totais** (`total/ICMSTot`): produtos, ICMS, ICMS-ST, IPI, PIS, COFINS, frete, seguro, desconto, valor da nota.
- **Transporte**, **cobrança/duplicatas**, **pagamentos** e **protocolo de autorização** (quando presente).

### Validações
- **Chave de acesso**: formato (44 dígitos) e dígito verificador (módulo 11) — chave inválida é reportada como erro, o documento não é considerado válido.
- **Conferência de totais**: soma dos itens comparada aos totais informados na nota, com tolerância de arredondamento (2 centavos); divergências viram avisos (`warnings`), não erros — a nota continua sendo exibida.
- Todos os campos obrigatórios do XML são checados; ausência ou formato inválido gera um erro com código estável (ex.: `IDE_MODELO_NAO_SUPORTADO`, `PRODUTO_NCM_AUSENTE`) e uma mensagem legível.

### Filosofia de tratamento de erros: `Result`, nunca exceção
```ts
type Result<T> =
  | { ok: true; value: T; warnings: NfeParseWarning[] }
  | { ok: false; errors: NfeParseError[]; warnings: NfeParseWarning[] };
```
`parseNfe(xml)` **nunca lança exceção** por causa de um XML malformado, incompleto ou malicioso — todo problema é acumulado como erro ou aviso e devolvido no `Result`. Isso torna o comportamento previsível tanto para quem usa a biblioteca em um script quanto para a UI, que nunca precisa de um `try/catch` para lidar com XML "ruim": ela simplesmente lê `result.ok` e mostra os erros de forma amigável.

---

## A aplicação web (`@nfe-lens/web`)

- **Drag-and-drop em lote** (RF08): é possível soltar vários XMLs de uma vez; cada um é processado independentemente e listado com um indicador verde (sucesso) ou vermelho (erro), sem que um XML inválido impeça a visualização dos demais.
- **Abas**: Resumo (identificação, emitente/destinatário, totais, avisos), Itens (tabela com busca por código/descrição/NCM), Tributos (detalhamento de ICMS/IPI/PIS/COFINS por item), Transporte, Cobrança (fatura, duplicatas, pagamentos) e XML bruto.
- **Exportação** para JSON (documento completo) e CSV (itens, separador `;`, compatível com Excel em pt-BR).
- **Processamento 100% local**: não há nenhuma chamada de rede além de carregar os próprios assets da aplicação. O XML nunca sai do navegador do usuário — não existe backend.

---

## Arquitetura e stack técnica

```
nfe-lens/
├── packages/core/                  @nfe-lens/core
│   ├── src/
│   │   ├── domain/                 tipos, enums fiscais, Result<T,E>
│   │   ├── parser/                 leitor XML genérico + parseNfe()
│   │   ├── mappers/                XML → domínio, um mapper por grupo/entidade
│   │   ├── validators/             chave de acesso, conferência de totais
│   │   ├── exporters/              toJson, toCsv
│   │   └── utils/                  conversão numérica segura (toDecimal)
│   └── tests/                      Vitest + fixtures XML anonimizadas
└── apps/web/                       @nfe-lens/web
    └── src/
        ├── components/             DropZone, tabelas, painéis, exportação
        ├── hooks/                  useNfeParser, useNfeList
        └── lib/                    formatação (moeda, CNPJ/CPF, datas)
```

| Camada | Tecnologia | Por quê |
|---|---|---|
| Linguagem | TypeScript 5, `strict` habilitado | modelagem de tipos fiscais complexos com segurança |
| XML | `fast-xml-parser` v5 | rápido, sem DOM, roda em qualquer ambiente JS |
| Build da lib | `tsup` | gera ESM + CJS + `.d.ts` a partir de um único `src/index.ts` |
| Testes | Vitest + Testing Library | fixtures reais anonimizadas, snapshots, testes de UI |
| UI | React 18 + Vite + TanStack Table | tabelas com busca sem dependências pesadas |
| Estilo | Tailwind CSS | consistência visual rápida |
| Monorepo | pnpm workspaces | um único lockfile, `workspace:*` entre os pacotes |
| CI | GitHub Actions | build → lint → typecheck → test com cobertura, a cada push/PR |

### Por que `apps/web` precisa do `packages/core` já compilado
`@nfe-lens/web` importa `@nfe-lens/core` pelo pacote publicado internamente via workspace (`main`/`types` apontam para `dist/`), não pelo código-fonte. Isso significa que **rodar `apps/web` (dev, build, test ou typecheck) exige que `packages/core` tenha sido buildado antes** — por isso o script `build` da raiz roda em ordem topológica (`pnpm -r run build` builda `core` antes de `web`) e o workflow de CI builda tudo **antes** de rodar lint/typecheck/test.

---

## Como rodar

```bash
pnpm install                          # na raiz do monorepo

# 1ª vez (ou após alterar o core): compilar o core antes de tocar no web
pnpm --filter @nfe-lens/core build

pnpm dev                              # sobe o app web em http://localhost:5175 (ou a porta livre)
pnpm build                            # builda core + web, em ordem
pnpm test                             # roda os testes de ambos os pacotes
pnpm test:coverage                    # cobertura (só onde o script existe — o core)
pnpm lint                             # ESLint em ambos
pnpm typecheck                        # tsc --noEmit em ambos
```

---

## Qualidade e testes

- **89 testes automatizados** (80 no `core`, 9 no `web`), cobrindo:
  - Todos os grupos de ICMS/PIS/COFINS/IPI suportados, individualmente.
  - Os três formatos de entrada: `nfeProc`, `NFe` pura, modelo 55 e modelo 65 (NFC-e).
  - Casos de erro: XML malformado, raiz não reconhecida, chave de acesso com dígito verificador inválido, campos obrigatórios ausentes.
  - Regressão de um bug real encontrado em revisão (`tpNF` inválido não gerava erro e produzia um documento corrompido silenciosamente — corrigido).
  - UI: arrastar arquivo inválido mostra erro sem travar a aplicação; exportação CSV gera cabeçalho correto; seleção da nota é atualizada corretamente ao remover itens da lista.
  - Performance: parse de uma nota com 500 itens em menos de 200 ms (medido com aquecimento de JIT prévio, para evitar um teste instável).
- **Cobertura de código** no `core`: acima de 85% em statements/lines/functions (configurado como piso obrigatório em `vitest.config.ts`; o CI falha se cair abaixo disso).
- Fixtures de teste usam **CNPJ, CPF, nomes e chaves de acesso fictícios**, gerados especificamente para os testes — nenhum dado fiscal real está commitado no repositório.

---

## Segurança

Como o propósito da ferramenta é abrir XMLs de **origem desconhecida** (o usuário pode carregar qualquer nota, de qualquer sistema emissor), o projeto trata o XML de entrada como não confiável. Pontos verificados:

- **Sem execução de código dinâmico**: nenhum uso de `eval`, `new Function`, `dangerouslySetInnerHTML` ou `innerHTML` em todo o código-fonte.
- **XXE (XML External Entity)**: `fast-xml-parser` v5 rejeita entidades externas por padrão (`External entities are not supported`) e não expande entidades internas (proteção contra ataques do tipo "billion laughs"). O leitor de XML do projeto (`readXml`) além disso envolve o parser em `try/catch`, então qualquer XML malicioso é tratado como "arquivo inválido", nunca propagado como exceção não tratada.
- **CSV/Formula Injection (CWE-1236)**: como a descrição/código do produto vêm do XML e podem ser exportados para CSV, um valor malicioso como `=cmd|' /C calc'!A0` poderia virar uma fórmula executável ao abrir o CSV no Excel/Sheets. O exportador (`toCsv`) neutraliza isso: qualquer célula que comece com `=`, `+`, `-`, `@`, tabulação ou retorno de carro é prefixada com `'`, conforme a mitigação recomendada pela OWASP — com teste de regressão dedicado.
- **Sem coleta de dados**: a aplicação web não faz nenhuma chamada de rede para processar as notas; tudo roda no navegador do usuário.
- **Dependências auditadas**: `pnpm audit` não reporta nenhuma vulnerabilidade conhecida (dependências de produção e de desenvolvimento, incluindo o servidor de desenvolvimento do Vite).
- **Nenhuma credencial, chave de API, senha ou segredo** está presente em nenhum arquivo do repositório.

---

## Modelo de domínio (resumo)

```ts
interface NfeDocument {
  chaveAcesso: string;
  versao: string;
  ide: Identificacao;
  emitente: Pessoa;
  destinatario?: Pessoa;
  itens: Item[];
  totais: Totais;
  transporte?: Transporte;
  cobranca?: Cobranca;
  pagamentos: Pagamento[];
  infoAdicional?: string;
  protocolo?: Protocolo;
  avisos: NfeParseWarning[];
}
```

Exemplo de uso programático da biblioteca:

```ts
import { parseNfe, toJson, toCsv } from "@nfe-lens/core";

const xml = /* conteúdo do arquivo .xml */;
const result = parseNfe(xml);

if (result.ok) {
  console.log(result.value.chaveAcesso);
  console.log(toJson(result.value));
  console.log(toCsv(result.value));
  if (result.warnings.length > 0) console.warn(result.warnings);
} else {
  console.error(result.errors); // nunca lança exceção
}
```

---

## Status do projeto

| Etapa | Status |
|---|---|
| Setup do monorepo, domínio, parser, tributos, validadores, exporters | ✅ Concluído |
| Interface web (drag-and-drop, abas, exportação, lote) | ✅ Concluído |
| Publicação (deploy da UI, publicação da lib) | ⏳ Não iniciado |

Escopo explicitamente fora da v1: validação contra XSD oficial, assinatura digital/consulta à SEFAZ, geração de DANFE em PDF.

## Licença

Este projeto usa **licença dupla**. O texto completo está em [LICENSE](LICENSE).

- **Uso pessoal, educacional e não comercial:** livre, nos termos da licença MIT (mantenha o aviso de copyright).
- **Uso comercial:** exige licença comercial. Isso inclui usar em produto ou serviço comercial, vender acesso ou derivados, usar em atividades com fins lucrativos e implantar em empresas.

Contato para licença comercial: ma.polli@hotmail.com
