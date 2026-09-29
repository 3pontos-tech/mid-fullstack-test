# Desafio Full Stack · 3Pontos Tech

Bem-vindo(a) ao processo seletivo da 3Pontos Tech. Este desafio avalia **modelagem de dados**, **decisões técnicas registradas** e domínio da nossa stack, num domínio em que um erro custa dinheiro de alguém.

| Item          | Descrição                              |
|---------------|----------------------------------------|
| **Linguagem** | PHP 8.4                                |
| **Framework** | Laravel 13 + FilamentPHP 5             |
| **Banco**     | PostgreSQL via Docker (já configurado) |
| **Front-end** | Blade, Livewire e Tailwind v4          |
| **Testes**    | Pest 4                                 |

> [!WARNING]
> Pode usar IA. Toda linha entregue é sua, e você vai precisar explicá-la numa conversa técnica. O `MODEL.md` tem uma seção obrigatória sobre o que você **descartou**.

> [!WARNING]
> Não use plugins do Filament além dos que já vêm neste template. Bibliotecas de infraestrutura são permitidas, desde que justificadas no `MODEL.md`. Modelagem, regras, painel e organização do código são seus.

> [!NOTE]
> Este repositório é um ponto de partida, não uma referência. Nada no template está garantido como adequado ao domínio. O que você entregar, inclusive o que já veio pronto, é responsabilidade sua.

---

## O produto

**Passa** é um cartão corporativo pré-pago. A empresa deposita um saldo. Cada funcionário tem um cartão com limite mensal e regras. A **rede do cartão** é a bandeira, como Visa ou Mastercard: ela recebe a compra da maquininha e pergunta ao emissor do cartão, o Passa, se pode aprovar. Depois, informa o que foi de fato cobrado ou cancelado. O financeiro da empresa acompanha tudo em um painel, e cada funcionário acompanha o próprio cartão.

Leia o **Guia da rede** antes de começar. Ele descreve como a rede se comporta, e o seu sistema conversa com ela o tempo todo.

### Vocabulário

Cada termo abaixo tem um único significado neste enunciado.

| Termo | Significado |
|---|---|
| **Compra** | Tudo que a rede informa sob o mesmo `id` de authorization: a authorization e os events que a referenciam. Cada compra cuja authorization chegou tem uma única decisão. |
| **Mês corrente** | O mês calendário atual no fuso da Acme, `America/Sao_Paulo`. |
| **Mês da compra** | O mês calendário, no fuso da Acme, do `occurred_at` da authorization. Todos os valores de uma compra contam nesse mês, qualquer que seja o `occurred_at` dos seus events. |
| **Compra encerrada** | Compra que recebeu uma cancellation, ou cujas captures de `sequence` 1 até a de `final: true` já foram todas recebidas. |
| **Capture que conta** | Uma capture, identificada pelo par `authorization_id` + `sequence`, de uma compra cuja authorization já chegou e não foi recusada por `card_not_found`. |
| **Reserva** | O que uma compra aprovada e não encerrada ainda bloqueia: o valor autorizado menos o já capturado, nunca menos que zero. Compra recusada ou encerrada não reserva nada. |
| **Uso da compra** | O total capturado mais a reserva. |
| **Limite restante** | De um cartão, num mês: o limite mensal menos o uso das compras do cartão cujo mês da compra é aquele. Pode ficar negativo. |
| **Saldo da empresa** | Os depósitos menos as captures que contam. |
| **Saldo disponível da empresa** | O saldo da empresa menos as reservas de todas as compras, de qualquer mês. |
| **Disponível** | De um cartão: o menor valor entre o limite restante do mês corrente e o saldo disponível da empresa, nunca menos que zero. Cartão bloqueado tem disponível zero. |
| **Transaction** | O registro de uma movimentação que altera o limite restante de um cartão, o saldo ou o saldo disponível da empresa. |

### Dados fixos da Acme

Usados pelos cenários de avaliação. Precisam existir no seu seed.

|                     |                                                        |
|---------------------|--------------------------------------------------------|
| Depósito inicial    | R$ 10.000,00                                           |
| Fuso horário        | `America/Sao_Paulo`                                    |
| Gestora do painel   | `marina@acme.test` · senha `password` (já vem no seed) |

| Cartão | `card_token` | Login do portador | Limite mensal | Teto por compra | MCC bloqueados | Situação  |
|--------|--------------|-------------------|--------------:|----------------:|----------------|-----------|
| Ana    | `tok_ana`    | `ana@acme.test`   |   R$ 2.000,00 |       R$ 800,00 | 7995           | Ativo     |
| Bruno  | `tok_bruno`  | `bruno@acme.test` |     R$ 500,00 |               — | —              | Ativo     |
| Carla  | `tok_carla`  | `carla@acme.test` |   R$ 1.000,00 |               — | —              | Bloqueado |
| Diego  | `tok_diego`  | `diego@acme.test` |  R$ 50.000,00 |               — | —              | Ativo     |

Senha de todos os portadores: `password`. Esses usuários são criados pelo **seu** seed.

### Visão geral das etapas

| Etapa | O que é | Quem usa |
|---|---|---|
| 1 · Authorization | **API**: a rede pergunta se pode aprovar uma compra | Rede |
| 2 · Events | **API**: a rede informa captures e cancellations | Rede |
| 3 · Transactions e consultas | **Modelo** de transactions + **API** de consulta | Rede e avaliação |
| 4 · Painel | **Filament** em `/admin` | Marina, financeiro da Acme |
| 5 · Área do funcionário | **Front** em Livewire, Blade e Tailwind, fora do Filament | Portador do cartão |

As etapas 1 a 3 não têm interface: são a integração com a rede. As etapas 4 e 5 são a interface e leem o mesmo modelo.

---

## Guia da rede

Não existe nenhum sistema externo neste desafio. A rede é um papel do domínio: nos seus testes, você faz esse papel; na avaliação, os nossos cenários fazem, e se comportam **exatamente** como este guia descreve. Nada chama os seus endpoints por conta própria.

### Assinatura

- Toda requisição da rede, inclusive as consultas, traz `X-Network-Timestamp` (Unix, segundos, o horário daquela entrega) e `X-Network-Signature: sha256=<hex minúsculo do HMAC-SHA256 de "<timestamp>.<corpo bruto>", chave NETWORK_SECRET>`. Requisição sem corpo assina `"<timestamp>."`.
- Assinatura ausente ou inválida, ou timestamp com diferença maior que 5 minutos, para mais ou para menos, em relação ao relógio do Passa: `401`. A assinatura é conferida antes de qualquer outra validação.
- Endpoints `/api/network/*` não usam sessão nem token de usuário. A única autenticação é a assinatura.

### Mensagens

- Toda mensagem tem um `id`, uma string opaca, não vazia, de até 64 caracteres. O `id` identifica a mensagem, e a rede nunca usa o mesmo `id` para conteúdos diferentes.
- Os tipos JSON são estritos: número inteiro é número sem parte fracionária, e string é string. Valores são sempre inteiros em centavos (`*_cents`). `currency` é sempre `BRL`.
- `occurred_at` é o horário do fato na rede, no formato `AAAA-MM-DDTHH:MM:SSZ` (UTC), e nunca é posterior ao envio. O horário em que a mensagem chega ao Passa não tem significado para o negócio.
- Campos que o contrato não descreve para aquele tipo de mensagem são ignorados.
- Mensagem que segue o contrato recebe `2xx`. Mensagem que não segue: `422`.

### Entrega

- A rede espera a resposta de cada entrega por até **2 segundos**. Sem resposta nesse prazo, ou com `5xx`, ela faz imediatamente uma nova entrega da mesma mensagem, sem cancelar a anterior, que pode continuar em processamento.
- Uma resposta pode se perder no caminho. Para a rede, isso é o mesmo que não ter recebido resposta.
- Em falhas internas da rede, entregas da mesma mensagem também podem sair ao mesmo tempo.
- São no máximo 4 entregas por mensagem. Cada entrega tem timestamp e assinatura próprios, e o corpo é idêntico.
- Quando uma mensagem teve mais de uma entrega, a rede usa a primeira resposta `2xx` ou `4xx` que chegar, venha de qualquer uma das entregas, e descarta as demais.
- `4xx` significa que o Passa rejeitou a mensagem, e a rede não a entrega de novo. A rede trata uma authorization rejeitada como recusada, sem enviar cancellation. Um event rejeitado vira uma pendência manual entre a rede e o Passa, resolvida fora do sistema.
- A rede desiste 2 segundos depois da 4ª entrega. Se até lá nenhuma entrega recebeu `2xx` nem `4xx`, ela trata a authorization como recusada e envia depois uma cancellation dela. Um event nessa situação vira pendência manual.
- Resposta que chega depois de a rede desistir é descartada.

### Ordem e volume

- A rede atende muitas maquininhas ao mesmo tempo. Várias mensagens podem chegar simultaneamente, inclusive do mesmo cartão.
- A rede não garante a ordem de chegada. As mensagens de uma compra chegam em qualquer ordem, inclusive events antes da authorization.
- Maquininhas offline enviam a authorization horas ou dias depois do `occurred_at`.
- Em reprocessamentos internos, a rede pode emitir de novo um event já enviado, com um `id` novo e todo o resto do conteúdo idêntico.

### O que a rede garante

- As captures de uma compra têm `sequence` começando em 1, sem lacunas, e a de `final: true` é a última.
- Cada par `authorization_id` + `sequence` corresponde a uma única capture.
- Uma compra tem no máximo uma cancellation, sem contar as reemissões.
- Todo event referencia uma authorization que a rede emitiu, mesmo que ela não tenha chegado ao Passa.

---

## Etapa 1 · Authorization

`POST /api/network/authorizations`

```json
{
  "id": "aut_01M2QTVQD0DZK5CBQ6Y9EF5WNT",
  "card_token": "tok_ana",
  "amount_cents": 12990,
  "currency": "BRL",
  "mcc": "5812",
  "merchant": { "name": "Restaurante Bom Prato", "city": "Porto Alegre", "country": "BR" },
  "occurred_at": "2026-09-17T14:03:22Z"
}
```

| Campo | Contrato |
|---|---|
| `id` | string não vazia, até 64 caracteres |
| `card_token` | string não vazia |
| `amount_cents` | inteiro, de 1 a 100000000 |
| `currency` | `"BRL"` |
| `mcc` | string de 4 dígitos |
| `merchant` | objeto com `name`, `city` e `country` (2 letras maiúsculas), todos string |
| `occurred_at` | `AAAA-MM-DDTHH:MM:SSZ` |

Resposta `200`:

```json
{ "decision": "approved" }
{ "decision": "declined", "reason": "monthly_limit_exceeded" }
```

**Regras**

1. A authorization é recusada pelo **primeiro** motivo da tabela que se aplicar, nesta ordem. `reason` é exatamente o identificador da tabela.

    | Ordem | `reason` | Quando |
    |---|---|---|
    | 1 | `card_not_found` | o `card_token` não existe |
    | 2 | `card_blocked` | o cartão está bloqueado |
    | 3 | `mcc_blocked` | o MCC está bloqueado para o cartão |
    | 4 | `amount_over_purchase_limit` | o `amount_cents` está acima do teto por compra do cartão |
    | 5 | `monthly_limit_exceeded` | o `amount_cents` está acima do limite restante do cartão no mês da compra |
    | 6 | `insufficient_funds` | o `amount_cents` está acima do saldo disponível da empresa |

2. "Acima" é estritamente maior: um valor igual ao teto, ao limite restante ou ao saldo disponível é aprovado.
3. Sem nenhum motivo de recusa, a decisão é `approved`, e a compra passa a reservar o valor autorizado.
4. A decisão é tomada uma única vez, com o estado do momento em que a authorization é processada pela primeira vez. Events da própria compra que chegaram antes dela não entram nessa conta e passam a valer logo depois da decisão.
5. O Passa nunca aprova uma compra acima do que a regra 1 permite.

---

## Etapa 2 · Events

`POST /api/network/events`

```json
{
  "id": "evt_01M2RAP800GH1VARD12M9XDKW1",
  "type": "capture",
  "occurred_at": "2026-09-17T18:40:00Z",
  "authorization_id": "aut_01M2QTVQD0DZK5CBQ6Y9EF5WNT",
  "amount_cents": 12990,
  "currency": "BRL",
  "sequence": 1,
  "final": true
}
```

| Campo | Contrato |
|---|---|
| `id` | string não vazia, até 64 caracteres |
| `type` | `"capture"` ou `"cancellation"` |
| `occurred_at` | `AAAA-MM-DDTHH:MM:SSZ` |
| `authorization_id` | string não vazia |
| `amount_cents` | só em `capture`: inteiro, de 1 a 100000000 |
| `currency` | só em `capture`: `"BRL"` |
| `sequence` | só em `capture`: inteiro, a partir de 1 |
| `final` | só em `capture`: booleano |

Resposta `200` ou `202`, corpo livre.

**Regras**

1. Uma capture informa um valor que a rede **já cobrou** do portador e vai liquidar com o Passa. Ela pode vir em várias partes e com total diferente do autorizado.
2. Toda capture que conta entra no uso da compra, inclusive acima do autorizado, em compra recusada por outro motivo ou depois de uma cancellation.
3. Nos MCC 5812 (restaurantes), 7011 (hotéis) e 7512 (aluguel de carros), é esperado que o total capturado de uma compra chegue a até 20% acima do autorizado, inclusive, calculado em centavos sem arredondamento: `capturado × 100 ≤ autorizado × 120`. Nos demais MCC, espera-se até o valor autorizado.
4. Uma cancellation encerra a compra: a reserva é liberada, e o que já foi capturado continua contando.
5. Events de uma compra cuja authorization ainda não chegou não têm efeito no limite restante, no saldo nem no saldo disponível da empresa. Quando a authorization chega, eles passam a valer. Events de uma compra recusada por `card_not_found` nunca têm esse efeito. Uma compra cuja authorization não chegou não tem decisão nem é divergência: aparece só no painel (Etapa 4, requisito 7).
6. Depois que o Passa recebe todas as mensagens de uma compra, o limite restante, o saldo, o saldo disponível e as divergências são os mesmos, qualquer que tenha sido a ordem de chegada.
7. **Divergência** é uma compra em que o total capturado passa do esperado na regra 3 (numa compra recusada, o autorizado é zero), ou que tem uma capture com `occurred_at` posterior ao da sua cancellation, ou que foi recusada por `card_not_found` e recebeu capture.

---

## Etapa 3 · Transactions e consultas

**Regras**

1. Toda movimentação que altera o limite restante de um cartão, o saldo ou o saldo disponível da empresa é uma **transaction**.
2. Uma transaction que já apareceu num statement nunca some nem muda de `occurred_at`, `type`, `amount_cents` ou `reference`. Statements de meses passados podem ganhar transactions novas.
3. As consultas refletem toda mensagem respondida com `2xx` em até **5 segundos**.

`GET /api/network/cards/{card_token}/available`

```json
{ "available_cents": 187010, "limit_remaining_cents": 187010 }
```

Valores do mês corrente. `card_token` inexistente: `404`.

`GET /api/network/cards/{card_token}/statement?month=2026-09`

```json
{
  "month": "2026-09",
  "limit_cents": 200000,
  "limit_remaining_cents": 187010,
  "transactions": [
    {
      "occurred_at": "2026-09-17T14:03:22Z",
      "type": "<identificador seu>",
      "amount_cents": -12990,
      "reference": "aut_01M2QTVQD0DZK5CBQ6Y9EF5WNT",
      "limit_remaining_after_cents": 187010
    }
  ]
}
```

1. `month` no formato `AAAA-MM`. Sem `month`, vale o mês corrente. Qualquer mês válido responde `200`, mesmo sem transactions. `month` inválido: `422`. `card_token` inexistente: `404`.
2. `transactions` traz as transactions das compras do cartão cujo mês da compra é `month`, em ordem cronológica de `occurred_at`. Em caso de empate, a ordem é sua, desde que seja estável.
3. `occurred_at` e `reference` são os da mensagem de origem da transaction: o `id` da authorization ou do event.
4. `amount_cents` negativo reduz o limite restante, positivo devolve, e zero é permitido.
5. Toda authorization aprovada, toda capture que conta e toda cancellation que liberou reserva aparecem ao menos uma vez como `reference`. Authorizations recusadas não aparecem.
6. **Invariante:** cada `limit_remaining_after_cents` é o anterior somado ao `amount_cents` da linha, e o primeiro parte de `limit_cents`. O último é igual a `limit_remaining_cents`, que é igual ao `limit_remaining_cents` de `/available` quando `month` é o mês corrente. Sem transactions, `limit_remaining_cents` é igual a `limit_cents`.

As duas consultas são assinadas como as demais requisições e são os únicos endpoints de leitura com contrato fixo.

---

## Etapa 4 · Painel

Filament em `/admin`, só para a Marina. O painel não cria endpoints: lê o mesmo modelo das etapas 1 a 3.

**Requisitos**

1. Cartões com o disponível e o limite restante atuais.
2. Statement de cada cartão, de qualquer mês, com o limite restante após cada transaction.
3. História de cada compra: authorization (`decision` e `reason`), captures e cancellation, com `occurred_at`, em ordem cronológica.
4. Statement da empresa: depósitos e captures que contam, com o saldo após cada um, e o saldo disponível atual.
5. Authorizations recusadas, com `reason`.
6. Divergências.
7. Events cuja authorization ainda não chegou.
8. Uma única ação de escrita: **registrar um depósito** para a empresa.
9. Um portador que tente acessar o painel recebe `403`.

---

## Etapa 5 · Área do funcionário

Fora do Filament. Livewire, Blade e Tailwind escritos por você.

**Requisitos**

1. `GET /login`: formulário de e-mail e senha para o portador, com `Auth::attempt`. Sem cadastro, sem recuperação de senha.
2. `GET /my-card`, autenticado: disponível e limite restante do próprio cartão, statement do mês corrente e história de cada compra própria, com `decision`, `reason`, captures e cancellation, em ordem.
3. A tela reflete uma nova authorization ou um novo event em até **5 segundos, sem recarregar a página**. Polling do Livewire basta.
4. Nenhuma URL, parâmetro ou ação Livewire permite a um portador ver dados de outro. A tentativa de acessar um recurso de outro portador devolve `403` ou `404`, e um teste cobre isso.
5. Um usuário sem cartão que acesse `/my-card` recebe `403`.
6. Nenhum componente do Filament nesta área. Layout livre, sem Figma. Estados de vazio e de carregamento contam mais que o acabamento visual.

---

## Como avaliamos

- Antes de cada cenário, rodamos `php artisan migrate:fresh --seed`, com a aplicação no ar via `composer dev` e o `NETWORK_SECRET` do `.env.example`.
- Os cenários fazem o papel da rede e seguem o **Guia da rede** à risca, inclusive os prazos, as entregas e a ordem. A exceção são os cenários que enviam requisições inválidas de propósito para medir `401`, `404` e `422`.
- Os cenários enviam no máximo 20 mensagens simultâneas. Entre mensagens cujo resultado depende uma da outra, esperam 5 segundos, salvo entre entregas da mesma mensagem e mensagens enviadas no intervalo entre elas.
- Medimos três coisas: as respostas que o contrato define; o estado ao final de cada cenário; e, em cada ponto de verificação, o invariante da Etapa 3 e o **Vocabulário**. O estado é medido só pelas consultas da Etapa 3, com pelo menos 5 segundos sem mensagens em trânsito. Divergências e o saldo da empresa são conferidos no painel.
- Entre mensagens simultâneas, qualquer ordem de processamento é aceita: nesses casos, medimos os totais.
- Todo resultado medido decorre deste enunciado. O que o enunciado não define não é medido pelos cenários: é avaliado pela coerência entre o código e o `MODEL.md`.
- Existem cenários não publicados.
- Depois da entrega, há uma conversa técnica sobre o seu código e o seu `MODEL.md`.

## Cenários publicados

Cada cenário parte do seed. As mensagens chegam uma de cada vez, na ordem listada, todas com `occurred_at` no mês corrente. Os valores estão em reais.

**P1**: cinco compras na Ana, todas no MCC 5812 (129,90 · 45,00 · 300,00 · 80,10 · 15,00), cada uma capturada no valor exato, numa única capture com `final: true`.

Resultado esperado:
- Ana: `available_cents` 143000, `limit_remaining_cents` 143000.
- Diego: `available_cents` 943000, `limit_remaining_cents` 5000000.
- O statement da Ana fecha em 143000.

**P2**: authorization de 800,00 na Ana, MCC 7011. Captures de 300,00 (`sequence` 1), 300,00 (`sequence` 2) e 260,00 (`sequence` 3, `final: true`).

**P3**: authorization de 400,00 no Bruno, MCC 5812, capturada em 480,00 com `final: true`. Em seguida, authorizations de 50,00 e de 20,00 no Bruno, MCC 5812.

Para P2 e P3, escreva no `MODEL.md`, **antes de implementar**: a decisão de cada authorization; o disponível e o limite restante da Ana ou do Bruno e o disponível do Diego depois de cada mensagem; e se a compra fica como divergência.

---

## Fora do escopo

Estorno, fechamento do mês, exportação, comprovante, aprovação de despesa, centro de custo, moeda estrangeira, expiração de reserva por tempo, cadastro e bloqueio de cartões pelo painel, e outras empresas além da Acme. Não implemente. Se a sua modelagem deixar porta aberta para algum deles, anote no `MODEL.md`.

---

## Entrega

1. **`MODEL.md`** (esqueleto no repositório), com as cinco seções dele.
2. Testes Pest cobrindo as etapas 1 a 3, o invariante do statement e o escopo de acesso das etapas 4 e 5.
3. `make check` verde: Pint, Larastan, Rector.
4. Seed com os dados fixos da Acme, executado pelo `php artisan db:seed` padrão.
5. `DEVELOPMENT.md` com as suas anotações. `README.md` seu: como rodar, como testar, suposições.
6. Commits incrementais, **sem squash**, em _Conventional Commits_ em inglês.

---

## Avaliação

| Critério | Peso |
|---|---|
| Modelagem e decisões registradas | 30 |
| Integração com a rede: cenários publicados e não publicados | 30 |
| Arquitetura, segurança e uso da stack | 15 |
| Testes | 10 |
| Painel e área do funcionário | 10 |
| Comunicação: MODEL, README, commits | 5 |

Um modelo completo com implementação parcial vale mais que uma implementação completa com modelo raso. Se o tempo apertar, entregue as etapas na ordem e documente o que ficou de fora.

---

## Como começar

```bash
make env-up        # Postgres, Redis e Mailpit via Docker
composer setup     # dependências, .env, chave, migrations, seed, assets
composer dev       # servidor com vários workers, fila, logs e Vite
```

Painel em `http://127.0.0.1:8000/admin`. A área do funcionário fica em `/login` e `/my-card` depois que você a construir. Testes com `make test`, qualidade com `make check`, o resto em `make help`. Os mesmos alvos existem no `Taskfile.yml`.

---

## Submissão

1. Crie um repositório **privado** a partir deste template ("Use this template", no topo da página).
2. Desenvolva em uma branch `develop`, com commits incrementais.
3. Abra um Pull Request de `develop` para `main` **no seu repositório** e dê acesso de leitura às pessoas indicadas no e-mail do processo.
4. Envie um e-mail para `maria.luiza@3pontos.com` com uma breve apresentação e o link do Pull Request.

> Estimativa: doze a dezesseis horas de trabalho.

---

## Materiais de apoio

- [Filament Docs](https://filament.com/docs) · [Filament Brasil](https://filament.com.br)
- [Laravel Docs](https://laravel.com/docs) · [Pest Docs](https://pestphp.com)
- [Livewire](https://livewire.laravel.com)
