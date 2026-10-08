# Mandato Aberto BR — placar do Congresso Nacional

Versão nacional do [Mandato Aberto](https://www.mandatoaberto.site): os 513 deputados federais e os 81 senadores de todos os estados, com filtro por UF. Presença, proposições (aprovadas, rejeitadas, arquivadas, em andamento), gastos de cota, tamanho do gabinete, custo estimado, direção das propostas e um **score de valor ao cidadão** com metodologia aberta e pesos ajustáveis pelo visitante.

Assembleias estaduais ficam de fora: cada uma publica dados num formato próprio (a versão SP cobre a ALESP).

Não há servidor nem banco de dados. Um job semanal do GitHub Actions coleta os dados oficiais, calcula o score e grava JSON em `public/data/`. A Vercel publica a pasta `public/` a cada commit.

```
config/score.config.json     pesos, critérios, parâmetros e escopo (Brasil)
scripts/build-data.js        orquestra coleta → score → JSON
scripts/collectors/          camara.js · senado.js
scripts/lib/                 classificação de tipo, tema e direção; score
public/                      site (HTML, CSS, JS puro) + data/
.github/workflows/           atualização semanal
tests/                       testes das regras
```

## Rodar localmente

```bash
npm install
npm test
npm run amostra        # gera dados fictícios em public/data (aparece faixa de aviso no site)
npm run dev            # http://localhost:3000
npm run update         # coleta real (demora: baixa arquivos grandes da Câmara)
npm run update -- --casas=senado   # só uma casa; as demais mantêm o último dado
```

Os endereços `mandato-aberto.vercel.app` e `placar-do-mandato-sp.vercel.app` redirecionam para o domínio (ver `vercel.json`). DNS na GoDaddy: `A @ → 216.198.79.1` e `CNAME www → <valor da Vercel>`.

Antes do primeiro deploy, rode `npm run update` (ou dispare o workflow manualmente) para substituir os dados fictícios por dados reais.

## Publicar

1. Crie um repositório no GitHub e suba este projeto.
2. Em **Settings → Actions → General → Workflow permissions**, marque *Read and write permissions* (o job faz commit dos dados).
3. Em **Actions → Atualizar dados (semanal) → Run workflow**, rode a primeira coleta. A primeira demora mais (60 a 120 min); as seguintes usam cache.
4. Na Vercel: *Add New → Project*, importe o repositório. O `vercel.json` já aponta a saída para `public/`. Cada commit do bot gera um deploy novo.

O agendamento roda toda segunda às 06:17 (horário de Brasília). Para mudar, edite o `cron` em `.github/workflows/atualizar-dados.yml`.

## Fontes de dados

| Dado | Câmara | Senado |
|---|---|---|
| Parlamentares | API v2 `/deputados` | `/senador/lista/atual` |
| Presença | `eventos` + `eventosPresencaDeputados` (sessões deliberativas do Plenário), recortado pelo `/historico` de exercício | `/votacao` (votações nominais) |
| Proposições e situação | `proposicoes` + `proposicoesAutores` (arquivos anuais) | `/processo?codigoParlamentarAutor=` |
| Cota | CEAP `camara.leg.br/cotas/Ano-AAAA.csv.zip` | CEAPS `adm.senado.gov.br/.../despesas_ceaps/AAAA` |
| Gabinete | páginas `/deputados/{id}/pessoal-gabinete` e `/verba-gabinete` | relatório `recursos-utilizados` |

### Pendências conhecidas

- **Validação contra dados reais.** Os coletores foram escritos a partir da documentação oficial e testados com dados de exemplo. Na primeira execução real, confira o log do Actions e 3 ou 4 perfis contra as páginas oficiais. Os pontos mais prováveis de ajuste são nomes de campos do Senado (`/processo`, `/votacao`).
- **Raspagem de páginas.** Pessoal e verba de gabinete da Câmara e o relatório do Senado vêm de páginas HTML públicas. Se o layout mudar, o número de assessores volta `null` e o critério sai do score até o ajuste.

## Score

Cada critério vira percentil (0–100) dentro da mesma casa; o score é a média ponderada. Pesos padrão:

| Critério | Peso |
|---|---|
| Assiduidade | 25 |
| Efetividade (aprovadas ÷ propostas, suavizada) | 20 |
| Produção legislativa (substantivas, escala log) | 15 |
| Fiscalização (requerimentos de informação, PFC) | 15 |
| Economia na cota | 15 |
| Enxugamento do gabinete | 10 |

### Adicionar um critério

1. Inclua o critério em `config/score.config.json → componentes` (peso, rótulo, descrição, `maior_melhor` ou `menor_melhor`).
2. Escreva o extrator em `scripts/lib/score.js → extratores`, lendo um campo do registro normalizado.
3. Se o dado ainda não existe, colete-o nos três coletores com o mesmo nome de campo.

O site lê os critérios do JSON, então aparecem sozinhos no ranking, na página de metodologia e nos controles de peso.

### Troca de legislatura

As novas bancadas tomam posse em 1º/02/2027 (Congresso). Nessas datas, atualize `inicio_legislatura` (e `id_legislatura` = 58 na Câmara) em `config/score.config.json`. O histórico semanal antigo continua em `public/data/historico/`.

## Cuidados

- Publique a metodologia junto com o ranking e mantenha os pesos visíveis. O site já faz isso; não esconda.
- Quantidade de propostas não mede qualidade. Por isso honoríficas saem da produção e a efetividade conta mais que o volume.
- Use User-Agent identificável e mantenha a frequência semanal. As fontes são públicas, mas as páginas HTML não são API.
