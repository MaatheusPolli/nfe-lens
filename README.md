# NFe Lens

Parser e visualizador de **NF-e** (modelo 55) e **NFC-e** (modelo 65) â€” o padrÃ£o de nota fiscal eletrÃ´nica definido pela SEFAZ/Receita Federal â€” escrito em TypeScript, com uma interface web em React para transformar o XML da nota em algo legÃ­vel, pesquisÃ¡vel e exportÃ¡vel, direto no navegador.

> Projeto de portfÃ³lio. Ver [LICENÃ‡A](./LICENSE) antes de reutilizar qualquer parte do cÃ³digo.

**CÃ³digo:** [github.com/MaatheusPolli/nfe-lens](https://github.com/MaatheusPolli/nfe-lens) Â· **Demo:** [maatheuspolli.github.io/nfe-lens](https://maatheuspolli.github.io/nfe-lens/)

---

## Sobre o projeto

Todo desenvolvedor que jÃ¡ trabalhou com fiscal, ERP ou integraÃ§Ã£o de e-commerce no Brasil conhece essa cena: abrir o XML de uma NF-e num editor de texto para achar um valor. A especificaÃ§Ã£o da SEFAZ organiza os dados numa Ã¡rvore XML profunda, cheia de siglas (`ICMS00`, `vBC`, `pICMS`, `CSOSN`, `CFOP`), com mais de vinte grupos de tributaÃ§Ã£o diferentes dependendo do regime da empresa â€” e nÃ£o existia uma ferramenta simples para transformar isso em algo legÃ­vel sem depender de um sistema fiscal completo.

O NFe Lens resolve isso como um **monorepo pnpm** com dois pacotes:

| Pacote | O que Ã© | Depende de |
|---|---|---|
| [`packages/core`](./packages/core) (`@nfe-lens/core`) | Biblioteca TypeScript pura: lÃª o XML, valida a estrutura, mapeia para um modelo de domÃ­nio tipado e oferece exportaÃ§Ã£o para JSON/CSV. Sem nenhuma dependÃªncia de UI â€” pode ser usada em Node, em um bundler de front-end, ou em qualquer runtime JS. | `fast-xml-parser` |
| [`apps/web`](./apps/web) (`@nfe-lens/web`) | AplicaÃ§Ã£o React que consome o `@nfe-lens/core` e oferece uma interface de arrastar-e-soltar, com abas de visualizaÃ§Ã£o, busca e exportaÃ§Ã£o. | `@nfe-lens/core`, React, Tailwind, TanStack Table |

Fluxo de uso tÃ­pico: o usuÃ¡rio arrasta um ou mais arquivos `.xml` para a aplicaÃ§Ã£o web â†’ o `@nfe-lens/core` faz o parse e a validaÃ§Ã£o de cada um, 100% no navegador â†’ a interface exibe o resultado em abas (Resumo, Itens, Tributos, Transporte, CobranÃ§a, XML bruto) â†’ o usuÃ¡rio pode exportar o resultado em JSON ou CSV.

> Documento de planejamento original (escopo, decisÃµes de arquitetura e roadmap): [planejamento.md](./planejamento.md).

---

## O que o parser (`@nfe-lens/core`) faz

### Entrada aceita
- XML de `NFe` "pura" (apenas o documento fiscal) **ou** `nfeProc` (o documento fiscal envolto pelo protocolo de autorizaÃ§Ã£o da SEFAZ, `protNFe`) â€” o parser detecta automaticamente qual dos dois formatos foi recebido.
- Modelo 55 (NF-e) e modelo 65 (NFC-e).

### O que Ã© extraÃ­do e tipado
- **IdentificaÃ§Ã£o** (`ide`): nÃºmero, sÃ©rie, data de emissÃ£o, modelo, ambiente (produÃ§Ã£o/homologaÃ§Ã£o), natureza da operaÃ§Ã£o, finalidade, indicador de presenÃ§a, etc.
- **Emitente e destinatÃ¡rio** (`emit`/`dest`): CNPJ ou CPF, razÃ£o social, endereÃ§o.
- **Itens** (`det[]`): produto (cÃ³digo, descriÃ§Ã£o, NCM, CFOP, quantidade, valores) e todos os tributos por item.
- **Tributos**, modelados como *unions discriminadas* do TypeScript (o compilador garante que cada grupo tributÃ¡rio sÃ³ possa ter os campos que realmente existem para aquele grupo):
  - **ICMS**: grupos `ICMS00`, `10`, `20`, `30`, `40`/`41`/`50`, `51`, `60`, `70`, `90` (regime normal) e `ICMSSN101`, `102`/`103`/`300`/`400`, `201`, `202`/`203`, `500`, `900` (Simples Nacional).
  - **PIS** e **COFINS**: por alÃ­quota, por quantidade, nÃ£o tributado ou outras operaÃ§Ãµes â€” cobrindo os CSTs oficiais.
  - **IPI**: tributado e nÃ£o tributado.
- **Totais** (`total/ICMSTot`): produtos, ICMS, ICMS-ST, IPI, PIS, COFINS, frete, seguro, desconto, valor da nota.
- **Transporte**, **cobranÃ§a/duplicatas**, **pagamentos** e **protocolo de autorizaÃ§Ã£o** (quando presente).

### ValidaÃ§Ãµes
- **Chave de acesso**: formato (44 dÃ­gitos) e dÃ­gito verificador (mÃ³dulo 11) â€” chave invÃ¡lida Ã© reportada como erro, o documento nÃ£o Ã© considerado vÃ¡lido.
- **ConferÃªncia de totais**: soma dos itens comparada aos totais informados na nota, com tolerÃ¢ncia de arredondamento (2 centavos); divergÃªncias viram avisos (`warnings`), nÃ£o erros â€” a nota continua sendo exibida.
- Todos os campos obrigatÃ³rios do XML sÃ£o checados; ausÃªncia ou formato invÃ¡lido gera um erro com cÃ³digo estÃ¡vel (ex.: `IDE_MODELO_NAO_SUPORTADO`, `PRODUTO_NCM_AUSENTE`) e uma mensagem legÃ­vel.

### Filosofia de tratamento de erros: `Result`, nunca exceÃ§Ã£o
```ts
type Result<T> =
  | { ok: true; value: T; warnings: NfeParseWarning[] }
  | { ok: false; errors: NfeParseError[]; warnings: NfeParseWarning[] };
```
`parseNfe(xml)` **nunca lanÃ§a exceÃ§Ã£o** por causa de um XML malformado, incompleto ou malicioso â€” todo problema Ã© acumulado como erro ou aviso e devolvido no `Result`. Isso torna o comportamento previsÃ­vel tanto para quem usa a biblioteca em um script quanto para a UI, que nunca precisa de um `try/catch` para lidar com XML "ruim": ela simplesmente lÃª `result.ok` e mostra os erros de forma amigÃ¡vel.

---

## A aplicaÃ§Ã£o web (`@nfe-lens/web`)

- **Drag-and-drop em lote** (RF08): Ã© possÃ­vel soltar vÃ¡rios XMLs de uma vez; cada um Ã© processado independentemente e listado com um indicador verde (sucesso) ou vermelho (erro), sem que um XML invÃ¡lido impeÃ§a a visualizaÃ§Ã£o dos demais.
- **Abas**: Resumo (identificaÃ§Ã£o, emitente/destinatÃ¡rio, totais, avisos), Itens (tabela com busca por cÃ³digo/descriÃ§Ã£o/NCM), Tributos (detalhamento de ICMS/IPI/PIS/COFINS por item), Transporte, CobranÃ§a (fatura, duplicatas, pagamentos) e XML bruto.
- **ExportaÃ§Ã£o** para JSON (documento completo) e CSV (itens, separador `;`, compatÃ­vel com Excel em pt-BR).
- **Processamento 100% local**: nÃ£o hÃ¡ nenhuma chamada de rede alÃ©m de carregar os prÃ³prios assets da aplicaÃ§Ã£o. O XML nunca sai do navegador do usuÃ¡rio â€” nÃ£o existe backend.

---

## Arquitetura e stack tÃ©cnica

```
nfe-lens/
â”œâ”€â”€ packages/core/                  @nfe-lens/core
â”‚   â”œâ”€â”€ src/
â”‚   â”‚   â”œâ”€â”€ domain/                 tipos, enums fiscais, Result<T,E>
â”‚   â”‚   â”œâ”€â”€ parser/                 leitor XML genÃ©rico + parseNfe()
â”‚   â”‚   â”œâ”€â”€ mappers/                XML â†’ domÃ­nio, um mapper por grupo/entidade
â”‚   â”‚   â”œâ”€â”€ validators/             chave de acesso, conferÃªncia de totais
â”‚   â”‚   â”œâ”€â”€ exporters/              toJson, toCsv
â”‚   â”‚   â””â”€â”€ utils/                  conversÃ£o numÃ©rica segura (toDecimal)
â”‚   â””â”€â”€ tests/                      Vitest + fixtures XML anonimizadas
â””â”€â”€ apps/web/                       @nfe-lens/web
    â””â”€â”€ src/
        â”œâ”€â”€ components/             DropZone, tabelas, painÃ©is, exportaÃ§Ã£o
        â”œâ”€â”€ hooks/                  useNfeParser, useNfeList
        â””â”€â”€ lib/                    formataÃ§Ã£o (moeda, CNPJ/CPF, datas)
```

| Camada | Tecnologia | Por quÃª |
|---|---|---|
| Linguagem | TypeScript 5, `strict` habilitado | modelagem de tipos fiscais complexos com seguranÃ§a |
| XML | `fast-xml-parser` v5 | rÃ¡pido, sem DOM, roda em qualquer ambiente JS |
| Build da lib | `tsup` | gera ESM + CJS + `.d.ts` a partir de um Ãºnico `src/index.ts` |
| Testes | Vitest + Testing Library | fixtures reais anonimizadas, snapshots, testes de UI |
| UI | React 18 + Vite + TanStack Table | tabelas com busca sem dependÃªncias pesadas |
| Estilo | Tailwind CSS | consistÃªncia visual rÃ¡pida |
| Monorepo | pnpm workspaces | um Ãºnico lockfile, `workspace:*` entre os pacotes |
| CI | GitHub Actions | build â†’ lint â†’ typecheck â†’ test com cobertura, a cada push/PR |

### Por que `apps/web` precisa do `packages/core` jÃ¡ compilado
`@nfe-lens/web` importa `@nfe-lens/core` pelo pacote publicado internamente via workspace (`main`/`types` apontam para `dist/`), nÃ£o pelo cÃ³digo-fonte. Isso significa que **rodar `apps/web` (dev, build, test ou typecheck) exige que `packages/core` tenha sido buildado antes** â€” por isso o script `build` da raiz roda em ordem topolÃ³gica (`pnpm -r run build` builda `core` antes de `web`) e o workflow de CI builda tudo **antes** de rodar lint/typecheck/test.

---

## Como rodar

```bash
pnpm install                          # na raiz do monorepo

# 1Âª vez (ou apÃ³s alterar o core): compilar o core antes de tocar no web
pnpm --filter @nfe-lens/core build

pnpm dev                              # sobe o app web em http://localhost:5175 (ou a porta livre)
pnpm build                            # builda core + web, em ordem
pnpm test                             # roda os testes de ambos os pacotes
pnpm test:coverage                    # cobertura (sÃ³ onde o script existe â€” o core)
pnpm lint                             # ESLint em ambos
pnpm typecheck                        # tsc --noEmit em ambos
```

---

## Qualidade e testes

- **89 testes automatizados** (80 no `core`, 9 no `web`), cobrindo:
  - Todos os grupos de ICMS/PIS/COFINS/IPI suportados, individualmente.
  - Os trÃªs formatos de entrada: `nfeProc`, `NFe` pura, modelo 55 e modelo 65 (NFC-e).
  - Casos de erro: XML malformado, raiz nÃ£o reconhecida, chave de acesso com dÃ­gito verificador invÃ¡lido, campos obrigatÃ³rios ausentes.
  - RegressÃ£o de um bug real encontrado em revisÃ£o (`tpNF` invÃ¡lido nÃ£o gerava erro e produzia um documento corrompido silenciosamente â€” corrigido).
  - UI: arrastar arquivo invÃ¡lido mostra erro sem travar a aplicaÃ§Ã£o; exportaÃ§Ã£o CSV gera cabeÃ§alho correto; seleÃ§Ã£o da nota Ã© atualizada corretamente ao remover itens da lista.
  - Performance: parse de uma nota com 500 itens em menos de 200 ms (medido com aquecimento de JIT prÃ©vio, para evitar um teste instÃ¡vel).
- **Cobertura de cÃ³digo** no `core`: acima de 85% em statements/lines/functions (configurado como piso obrigatÃ³rio em `vitest.config.ts`; o CI falha se cair abaixo disso).
- Fixtures de teste usam **CNPJ, CPF, nomes e chaves de acesso fictÃ­cios**, gerados especificamente para os testes â€” nenhum dado fiscal real estÃ¡ commitado no repositÃ³rio.

---

## SeguranÃ§a

Como o propÃ³sito da ferramenta Ã© abrir XMLs de **origem desconhecida** (o usuÃ¡rio pode carregar qualquer nota, de qualquer sistema emissor), o projeto trata o XML de entrada como nÃ£o confiÃ¡vel. Pontos verificados:

- **Sem execuÃ§Ã£o de cÃ³digo dinÃ¢mico**: nenhum uso de `eval`, `new Function`, `dangerouslySetInnerHTML` ou `innerHTML` em todo o cÃ³digo-fonte.
- **XXE (XML External Entity)**: `fast-xml-parser` v5 rejeita entidades externas por padrÃ£o (`External entities are not supported`) e nÃ£o expande entidades internas (proteÃ§Ã£o contra ataques do tipo "billion laughs"). O leitor de XML do projeto (`readXml`) alÃ©m disso envolve o parser em `try/catch`, entÃ£o qualquer XML malicioso Ã© tratado como "arquivo invÃ¡lido", nunca propagado como exceÃ§Ã£o nÃ£o tratada.
- **CSV/Formula Injection (CWE-1236)**: como a descriÃ§Ã£o/cÃ³digo do produto vÃªm do XML e podem ser exportados para CSV, um valor malicioso como `=cmd|' /C calc'!A0` poderia virar uma fÃ³rmula executÃ¡vel ao abrir o CSV no Excel/Sheets. O exportador (`toCsv`) neutraliza isso: qualquer cÃ©lula que comece com `=`, `+`, `-`, `@`, tabulaÃ§Ã£o ou retorno de carro Ã© prefixada com `'`, conforme a mitigaÃ§Ã£o recomendada pela OWASP â€” com teste de regressÃ£o dedicado.
- **Sem coleta de dados**: a aplicaÃ§Ã£o web nÃ£o faz nenhuma chamada de rede para processar as notas; tudo roda no navegador do usuÃ¡rio.
- **DependÃªncias auditadas**: `pnpm audit` nÃ£o reporta nenhuma vulnerabilidade conhecida (dependÃªncias de produÃ§Ã£o e de desenvolvimento, incluindo o servidor de desenvolvimento do Vite).
- **Nenhuma credencial, chave de API, senha ou segredo** estÃ¡ presente em nenhum arquivo do repositÃ³rio.

---

## Modelo de domÃ­nio (resumo)

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

Exemplo de uso programÃ¡tico da biblioteca:

```ts
import { parseNfe, toJson, toCsv } from "@nfe-lens/core";

const xml = /* conteÃºdo do arquivo .xml */;
const result = parseNfe(xml);

if (result.ok) {
  console.log(result.value.chaveAcesso);
  console.log(toJson(result.value));
  console.log(toCsv(result.value));
  if (result.warnings.length > 0) console.warn(result.warnings);
} else {
  console.error(result.errors); // nunca lanÃ§a exceÃ§Ã£o
}
```

---

## Status do projeto

| Etapa | Status |
|---|---|
| Setup do monorepo, domÃ­nio, parser, tributos, validadores, exporters | âœ… ConcluÃ­do |
| Interface web (drag-and-drop, abas, exportaÃ§Ã£o, lote) | âœ… ConcluÃ­do |
| PublicaÃ§Ã£o (deploy da UI, publicaÃ§Ã£o da lib) | â³ NÃ£o iniciado |

Escopo explicitamente fora da v1 (ver [planejamento.md](./planejamento.md)): validaÃ§Ã£o contra XSD oficial, assinatura digital/consulta Ã  SEFAZ, geraÃ§Ã£o de DANFE em PDF.

---

## LicenÃ§a

**Todos os direitos reservados.** Este repositÃ³rio Ã© disponibilizado publicamente apenas para fins de leitura e avaliaÃ§Ã£o (ex.: portfÃ³lio). NÃ£o Ã© concedida nenhuma permissÃ£o de cÃ³pia, uso, modificaÃ§Ã£o ou redistribuiÃ§Ã£o deste cÃ³digo, total ou parcial, sem autorizaÃ§Ã£o expressa e por escrito do autor. Veja o arquivo [LICENSE](./LICENSE) para os termos completos.


## Licença

### Uso Pessoal
Este projeto é licenciado sob a **Licença MIT** para uso pessoal, educacional e não comercial.

### Uso Comercial
Se você deseja usar este projeto comercialmente, deve obter uma licença comercial do autor.

**Uso comercial inclui:**
- Incorporar em um produto comercial
- Oferecer como serviço
- Usar em atividades com fins lucrativos
- Implantação empresarial

**Para obter uma licença comercial:**
Email: ma.polli@hotmail.com
