# Playbook de Vendas para Barbeiros

> Documento de passagem — atualizado em 25/11/2026.

## Para que serve

Material de aula sobre venda de produtos dentro da barbearia: mentalidade, diagnóstico na cadeira, atendimento passo a passo, scripts, objeções, exposição, pós-venda, exercícios de sala (role-play com checklist de avaliação do aluno) e calculadora de indicadores. O público do conteúdo são barbeiros (alunos); a página foi feita para ser usada em sala por quem conduz o treinamento. Publicado em 03/11/2026; em 25/11/2026 havia 2 registros de uso no banco, ou seja, uso baixo até agora. Quem exatamente usa e com que frequência não está documentado.

## Onde está

| O quê | Endereço | Hospedagem | Conta | Banco |
|---|---|---|---|---|
| Playbook | https://podium-educacao-salesops.github.io/playbook-vendas-podium/ | GitHub Pages (branch `main`, raiz) | GitHub `podium-educacao-salesops` | — |
| Painel de uso | https://podium-educacao-salesops.github.io/playbook-vendas-podium/?gestao | idem | idem | — |
| Histórico de uso (Worker) | https://playbook-vendas-api.podium-playbook.workers.dev | Cloudflare Workers | Cloudflare `podium-playbook` | D1 `playbook_vendas_db` |

Repositório: https://github.com/podium-educacao-salesops/playbook-vendas-podium (o código do Worker está em `api/`).

Não confundir com outros repositórios de nome parecido na pasta da Daiane (`playbook-5-passos`, `playbook-executivo`): são outros materiais.

## Como funciona

- `index.html` — página inteira (HTML, CSS e JS num arquivo, sem build). Fontes do Google Fonts; nenhuma outra dependência externa além do Worker.
- A página nasceu como Artifact do Claude, que grava dados por `window.claude.use('db')` e `window.claude.use('user')`. Fora do Claude isso não existe, então um **adaptador** no `index.html` (bloco que começa com "Fora do Claude não existe window.claude") imita essas duas funções gravando no Worker.
- Cada navegador recebe um id (`localStorage` `pb-uid`) e a pessoa informa o nome uma vez (`pb-nome`). O registro de cada pessoa (dias de uso, seções vistas, avaliações e indicadores) é gravado inteiro a cada mudança com `PUT /uso/:id`.
- Com `?gestao` no link, a página mostra o painel de histórico de todos, que relê `GET /uso` a cada 60 segundos.

```
index.html -> adaptador window.claude -> PUT /uso/:id, GET /uso -> Worker playbook-vendas-api -> D1 playbook_vendas_db (tabela uso)
```

## Rodar no computador

```bash
cd playbook-vendas-podium
python3 -m http.server 4190
# abrir http://localhost:4190/  e  http://localhost:4190/?gestao
```

Rodando local, a página grava no Worker publicado (o endereço está fixo no adaptador; o CORS aceita localhost). Para usar um Worker local, troque temporariamente a constante `API` no adaptador para `http://localhost:8787`.

## Publicar uma alteração

Push na `main` publica. O GitHub Pages reconstrói em cerca de 1 minuto; não há build.

**Se a página for substituída por uma versão nova exportada de um Artifact**, reinserir o adaptador antes do script "Histórico de uso". Sem ele, a página abre normalmente mas não grava nada e o `?gestao` fica vazio.

O Worker é publicado à parte, com `wrangler deploy` (ver abaixo).

## Variáveis de ambiente

Nenhuma. A página é estática e o Worker não usa secret nem variável, só o binding do banco (`DB`, no `api/wrangler.toml`).

## Rotinas automáticas

Nenhuma. Sem cron nem webhook. (O painel `?gestao` recarrega a lista a cada 60 segundos enquanto está aberto.)

## Regras e cuidados

- **Sem senha**, no mesmo padrão do Simulador SDR e do onboarding. Riscos e opções no documento de passagem do time.
- **Pessoa = navegador.** A mesma pessoa em dois aparelhos aparece duas vezes; limpar os dados do navegador cria uma pessoa nova.
- O Worker está numa conta Cloudflare diferente da do Simulador SDR. Ao rodar `wrangler`, confira em qual conta está logado (`npx wrangler whoami`).

## Worker `playbook-vendas-api` (histórico de uso)

Código em `api/src/index.js`, schema em `api/schema.sql`.

| Item | Valor |
|---|---|
| Endereço | https://playbook-vendas-api.podium-playbook.workers.dev (a raiz responde 404; as rotas estão em `/uso`) |
| Conta Cloudflare | `podium-playbook` (subdomínio `podium-playbook`; a mesma do Worker do onboarding) |
| Banco | D1 `playbook_vendas_db` (id no `api/wrangler.toml`), tabela `uso` (`id`, `nome`, `dados` em JSON, `atualizado`) |

| Rota | O que faz |
|---|---|
| `GET /uso` | Todos os registros, do mais recente para o mais antigo (painel `?gestao`) |
| `GET /uso/:id` | Registro de uma pessoa |
| `PUT /uso/:id` | Grava o registro inteiro `{nome, dados}` (até 256 KB) |

CORS: aceita `https://podium-educacao-salesops.github.io`, o endereço antigo e localhost (constante `ORIGENS`). Se o site mudar de endereço, incluir a origem nova e republicar o Worker antes.

Rodar local e publicar (não há `package.json`; o `npx` baixa o wrangler):

```bash
cd api
npx wrangler d1 execute playbook_vendas_db --local --file=schema.sql
npx wrangler dev                      # http://localhost:8787
npx wrangler login                    # conta podium-playbook
npx wrangler deploy                   # push no GitHub não publica o Worker
```

Depois do deploy, faça commit e push do mesmo código: este Worker não pode voltar a existir só no dashboard da Cloudflare.

## Pendências da passagem

- [x] Repositório transferido para `podium-educacao-salesops` e Worker republicado com a origem nova (25/11/2026).
- [ ] **Antes de mudar o endereço**, incluir a nova origem (`https://<org>.github.io`) em `ORIGENS` no `api/src/index.js` e fazer `wrangler deploy`. Sem isso a página para de gravar e o `?gestao` não carrega.
- [ ] Na mudança de endereço do Pages, o `localStorage` não acompanha: cada pessoa ganha um id novo e precisa digitar o nome de novo; o histórico antigo fica no banco com o id antigo.
- [ ] Passar a conta Cloudflare `podium-playbook` para outra pessoa (convidar membro com acesso a Workers e D1, ou trocar o login). Se o Worker for recriado em outra conta, o endereço muda: trocar a constante `API` no adaptador do `index.html` e migrar o D1 (`npx wrangler d1 export playbook_vendas_db --remote --output=dump.sql`, criar o banco na conta nova, `d1 execute --file=dump.sql`, atualizar o `database_id`).
- [ ] Decidir se o histórico continua sem senha.
- [ ] Confirmar quem usa o playbook hoje e se ele segue em uso (2 registros em 25/11/2026).
