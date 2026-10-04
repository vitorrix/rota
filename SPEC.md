# ROTA — App de viagens do Vitor e da Thais

> Especificação para implementação no Claude Code.
> URL final: **https://rota.barukstore.com.br**
> Repositório: **vitorrix/rota** (GitHub Pages)
> Primeira viagem: **Porto de Galinhas, 11 a 16/11/2026**

---

## 0. Regras de ouro (ler antes de qualquer código)

1. **O repositório é público.** NUNCA colocar no código, nos commits ou em arquivos versionados: códigos de reserva, endereço da hospedagem, telefones pessoais, e-mails dos usuários, números de cartão. Esses dados são digitados **dentro do app** e ficam só no Firestore.
2. **`firestore.rules` contém os e-mails autorizados → vai no `.gitignore`.** O deploy das regras é feito localmente via Firebase CLI. No repo fica só um `firestore.rules.example` com e-mails fictícios.
3. **Dinheiro sempre em centavos (inteiro).** Nunca float. Formatação em BRL só na exibição (`Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' })`).
4. **Funciona offline.** O app precisa abrir e lançar gastos sem sinal; sincroniza quando a conexão voltar.
5. **Mobile-first.** O uso principal é no iPhone, com uma mão, na praia. Desktop é secundário.
6. **Multi-viagem desde o início.** Porto de Galinhas é a primeira viagem, não a única. Nada de "Porto de Galinhas" hardcoded na lógica.
7. **Fuso:** America/Recife (UTC−3, sem horário de verão — igual a São Paulo). Datas sem hora gravadas como string `YYYY-MM-DD`; datas com hora como Timestamp.

---

## 1. Stack

- **Front:** HTML + CSS + JavaScript vanilla (mesma receita da Secretina). Firebase JS SDK modular v10+ via CDN (`https://www.gstatic.com/firebasejs/...`).
- **Arquivos:** `index.html` (app inteiro), `manifest.json`, `sw.js`, `icons/`, `CNAME`, `firestore.rules.example`, `README.md`.
- **Backend:** Firebase (projeto **novo e separado** da Secretina — sugestão de ID: `rota-baruk`). Plano Spark é suficiente. Sem Firebase Storage na fase 1.
- **Auth:** Google Sign-In (Firebase Auth). Exigir `email_verified`.
- **Banco:** Firestore com cache offline persistente:
  ```js
  initializeFirestore(app, { localCache: persistentLocalCache({ tabManager: persistentMultipleTabManager() }) })
  ```
- **PWA:** manifest + service worker cacheando o app shell (HTML, ícones, SDK). Instalável na tela inicial do iPhone (`apple-touch-icon`, `apple-mobile-web-app-capable`, `theme-color`).
- **Hospedagem:** GitHub Pages com `CNAME` = `rota.barukstore.com.br`.

---

## 2. Identidade visual

- Base escura no padrão Baruk/Secretina: fundo `#080808`, superfícies `#121212` / `#1A1A1A`, texto `#F2F2F2` / `#9A9A9A`.
- **Cor de destaque própria do Rota:** turquesa-mar `#19C3D6` (primária) e coral `#FF7A59` (alertas suaves/ênfase).
- Status: verde `#00C880` (confirmado/verificado), amarelo `#F5B700` (pendente/não verificado), vermelho `#FF4D4F` (problema/suspeito).
- Modo claro também disponível (toggle em Configurações, padrão = sistema).
- Fonte do sistema (`-apple-system, BlinkMacSystemFont, "SF Pro", system-ui, sans-serif`).
- Navegação inferior fixa com 5 abas: **Painel · Gastos · Reservas · Contatos · Mais** (Mais = Roteiro, Checklist, Diário, Configurações). Respeitar `env(safe-area-inset-bottom)`.
- Botão flutuante **"+ Gasto"** visível no Painel e em Gastos.
- Ícone do app: onda estilizada ou bússola, em turquesa sobre fundo escuro.

---

## 3. Estrutura do Firestore

```
viagens/{viagemId}
  nome: string                    // "Porto de Galinhas 2026"
  destino: string                 // "Porto de Galinhas, PE"
  dataInicio: "2026-11-11"
  dataFim: "2026-11-16"
  orcamentoCentavos: 700000
  membrosEmails: [string]         // e-mails autorizados
  membrosNomes: { [email]: "Vitor" | "Thais" }
  criadoEm, atualizadoEm: Timestamp

viagens/{viagemId}/reservas/{id}
  tipo: "voo" | "hospedagem" | "transfer" | "passeio" | "outro"
  titulo: string
  fornecedor: string              // "Azul via Booking.com", "Airbnb"
  status: "confirmada" | "pendente" | "problema" | "cancelada"
  valorCentavos: number
  statusPagamento: "pago" | "a_pagar" | "nao_cobrado"
  pagoPor: email | null
  inicio: Timestamp | null        // check-in / partida
  fim: Timestamp | null           // checkout / chegada
  detalhes: string                // horários, trechos etc. (texto livre)
  prazoCritico: { quando: Timestamp, descricao: string } | null
  linkOficial: string
  canalAtendimento: string        // "Só pelo app Booking logado → Minhas reservas"
  codigo: string                  // SENSÍVEL — mascarado na UI
  notas: string
  ordem: number

viagens/{viagemId}/gastos/{id}
  valorCentavos: number
  categoria: "comida" | "transfer" | "passeio" | "mercado" | "compras" | "outro"
  descricao: string
  pagoPor: email
  forma: "pix" | "cartao" | "dinheiro"
  tipo: "realizado" | "previsto"
  data: "YYYY-MM-DD"
  criadoPor: email
  criadoEm: Timestamp

viagens/{viagemId}/contatos/{id}
  nome: string
  tipo: "hospedagem" | "transfer" | "jangada" | "buggy" | "restaurante" | "atendimento" | "outro"
  telefone: string                // como digitado
  telefoneNormalizado: string     // só dígitos, com DDI 55 quando nacional
  fonte: string                   // de onde veio o contato
  selo: "verificado" | "nao_verificado" | "suspeito"
  precoCombinadoCentavos: number | null
  alerta: string                  // ex.: "Pagamentos só pelo app Airbnb"
  notas: string

viagens/{viagemId}/roteiro/{data}          // doc id = "2026-11-11"   (FASE 2)
  mareBaixa: [{ hora: "HH:MM", alturaM: number }]
  mareAlta:  [{ hora: "HH:MM", alturaM: number }]
  blocos: [{ hora: "HH:MM", titulo: string, tipo: "trabalho"|"praia"|"passeio"|"refeicao"|"deslocamento"|"outro", notas: string, reservaId?: string }]

viagens/{viagemId}/checklist/{id}           // (FASE 2)
  texto: string
  grupo: "mala" | "antes_de_sair" | "documentos" | "outro"
  feito: boolean
  feitoPor: email | null

viagens/{viagemId}/diario/{id}              // (FASE 2)
  data: "YYYY-MM-DD"
  titulo: string
  texto: string
  linkAlbum: string               // link do álbum compartilhado do iCloud
  ordem: number
  criadoPor: email
```

---

## 4. Regras de segurança (`firestore.rules` — NÃO versionar)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{db}/documents {

    function autorizado() {
      return request.auth != null
        && request.auth.token.email_verified == true
        && request.auth.token.email in ['EMAIL_DO_VITOR', 'EMAIL_DA_THAIS'];
    }

    function membro(viagemId) {
      return autorizado()
        && request.auth.token.email in
           get(/databases/$(db)/documents/viagens/$(viagemId)).data.membrosEmails;
    }

    match /viagens/{viagemId} {
      allow read, update, delete: if autorizado()
        && request.auth.token.email in resource.data.membrosEmails;
      allow create: if autorizado()
        && request.auth.token.email in request.resource.data.membrosEmails;

      match /{sub}/{docId} {
        allow read, write: if membro(viagemId);
      }
    }

    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

Usuário logado que não está na lista vê uma tela "Acesso não autorizado" com botão de sair.

---

## 5. Lógica central

### 5.1 Painel — números

- **Comprometido** = soma de `reservas.valorCentavos` com status ≠ `cancelada`.
- **Gasto** = soma de `gastos` com `tipo = realizado`.
- **Previsto** = soma de `gastos` com `tipo = previsto`.
- **Livre** = orçamento − comprometido − gasto.
- **Dias da viagem** = de `dataInicio` a `dataFim`, inclusive (Porto de Galinhas = 6 dias).

### 5.2 "Pode gastar hoje" (o número mais importante do app)

- **Antes da viagem:** `livre ÷ dias da viagem` → rótulo "Diária planejada".
- **Durante a viagem:**
  - `base = orçamento − comprometido − gastos realizados ANTES de hoje`
  - `diasRestantes = dias de hoje até dataFim, inclusive`
  - `limiteHoje = base ÷ diasRestantes`
  - Mostrar **"Hoje ainda pode: limiteHoje − gastos de hoje"**. Verde se ≥ 0, vermelho se negativo.
- **Depois da viagem:** resumo final (total gasto, por categoria, por pessoa, diferença para o orçamento).

### 5.3 Acerto entre os dois

Somar tudo que cada um pagou (reservas com `statusPagamento = pago` + gastos realizados). Diferença ÷ 2 → "Thais deve R$ X ao Vitor" (ou o contrário). Exibir no Painel e em Gastos.

### 5.4 Prazos críticos

Qualquer reserva com `prazoCritico` nos próximos 7 dias aparece num card de alerta no topo do Painel ("Cancelamento grátis do Airbnb termina em 3 dias"). Reservas com status `pendente` ou `problema` também aparecem no topo, em amarelo/vermelho.

### 5.5 Antigolpe — lista negra de telefones

Constante no código (são números de golpe públicos, podem ficar no repo):

```js
const BLACKLIST_TELEFONES = [
  { numero: '556131420712', motivo: 'Falso atendimento Booking.com encontrado no Google' },
  { numero: '556135501608', motivo: 'Falso atendimento Booking.com encontrado no Google' },
  { numero: '551151183723', motivo: 'Falso atendimento Booking.com encontrado no Google' },
  { numero: '5531976011457', motivo: 'Falso atendimento Booking.com encontrado no Google' },
  { numero: '551151900965', motivo: 'Falso atendimento Booking.com encontrado no Google' },
  { numero: '551142807218', motivo: 'Falso atendimento Booking.com encontrado no Google' },
];
```

- Ao salvar ou editar um contato, normalizar o telefone e comparar. Se bater: bloquear o selo em **suspeito**, mostrar aviso vermelho em tela cheia com o motivo, e exigir confirmação para salvar mesmo assim.
- Tela de Configurações permite adicionar números à lista negra da viagem (salvos num campo `blacklistExtra` no doc da viagem).
- Contatos com selo `nao_verificado` mostram o lembrete fixo: **"Confirme por fonte oficial antes de pagar qualquer coisa."**
- Contatos com selo `suspeito` ficam no fim da lista, em vermelho, e o botão de ligar exige confirmação.

---

## 6. Telas

### 6.1 Login
Logo Rota, botão "Entrar com Google". Depois do login: se tem uma viagem só, abre direto nela; se tem várias, lista para escolher (a mais próxima/atual primeiro).

### 6.2 Painel
De cima para baixo:
1. Alertas (prazos críticos, reservas pendentes/problema).
2. Contagem regressiva ("Faltam 38 dias") / "Dia 3 de 6" durante a viagem.
3. **Card grande "Pode gastar hoje"** (5.2).
4. Barra do orçamento empilhada: comprometido | gasto | livre, com valores.
5. Próximo compromisso (próxima reserva ou bloco do roteiro).
6. Acerto entre os dois (5.3).
7. Maré baixa de hoje (fase 2).

### 6.3 Gastos
- Lista agrupada por dia (mais recente primeiro), com total do dia.
- Filtro por categoria e por pessoa; alternar realizado / previsto.
- Resumo por categoria (barras horizontais simples).
- **Lançamento rápido (bottom sheet):** teclado numérico já aberto no valor → toque na categoria (ícones grandes) → quem pagou (padrão = usuário logado) → salvar. Forma de pagamento, descrição e data opcionais (data padrão = hoje). Meta: lançar em até 3 toques além do valor.
- Deslizar para editar/excluir (excluir pede confirmação).

### 6.4 Reservas
- Cards ordenados por `inicio`, com faixa colorida de status.
- Card mostra: título, fornecedor, datas/horários, valor, status de pagamento, prazo crítico.
- **Código mascarado** (`HM••••••ZM`): toque longo para revelar por 10 s; botão copiar.
- Botões: abrir link oficial, ver canal de atendimento.
- Formulário completo de criar/editar.

### 6.5 Contatos
- Lista agrupada por tipo, com selo colorido.
- Botões: ligar (`tel:`), WhatsApp (`https://wa.me/<normalizado>`), copiar.
- Campo de alerta sempre visível no card quando preenchido.
- Formulário com validação da lista negra (5.5).

### 6.6 Mais → Configurações
- Dados da viagem (nome, datas, orçamento, membros).
- Criar nova viagem.
- Lista negra extra.
- Tema claro/escuro/sistema.
- Exportar gastos em CSV (fase 2).
- Sair.

---

## 7. Dados iniciais da viagem Porto de Galinhas

Criar um botão **"Criar viagem de exemplo: Porto de Galinhas 2026"** (aparece só quando o usuário não tem nenhuma viagem) que grava os dados abaixo. Campos sensíveis ficam vazios para preencher no app.

**Viagem**
- nome: `Porto de Galinhas 2026` · destino: `Porto de Galinhas, PE`
- dataInicio: `2026-11-11` · dataFim: `2026-11-16`
- orcamentoCentavos: `700000`
- membrosEmails: e-mail de quem está logado (o outro é adicionado em Configurações)

**Reserva 1 — Hospedagem**
- tipo `hospedagem`, título `Airbnb na vila — 50 m da praia`, fornecedor `Airbnb`
- status `confirmada`, statusPagamento `pago`, valorCentavos `177000`
- inicio: 11/11/2026 15:00 · fim: 16/11/2026 12:00
- detalhes: `5 noites × R$ 354. Self check-in com fechadura eletrônica (senha chega pelo app Airbnb). Apartamento inteiro, 1 quarto, piscina no rooftop.`
- prazoCritico: 10/11/2026 15:00 — `Fim do cancelamento gratuito`
- linkOficial: `https://www.airbnb.com.br/trips`
- canalAtendimento: `Mensagens pelo app Airbnb. Atendimento Airbnb 24h pela Central de Ajuda no app.`
- codigo: *(vazio — preencher no app)*
- notas: `Pagamentos SÓ pelo app Airbnb. Pedido de Pix, depósito ou taxa extra = golpe.`

**Reserva 2 — Voo**
- tipo `voo`, título `Azul direto GRU ⇄ REC`, fornecedor `Azul via Booking.com`
- status `problema`, statusPagamento `nao_cobrado`, valorCentavos `196600` (2 × R$ 983)
- inicio: 11/11/2026 10:45 · fim: 16/11/2026 15:50
- detalhes: `Ida 11/11 GRU 10h45 → REC 13h50. Volta 16/11 REC 12h30 → GRU 15h50.`
- linkOficial: `https://www.booking.com`
- canalAtendimento: `SÓ pelo app/site Booking LOGADO → Minhas reservas → Atendimento. Nunca ligar para número achado no Google.`
- notas: `Reserva ficou pendente, sem cobrança. Não comprar de novo antes de saber o status. Se cair, comprar direto no site/app da Azul. Conferir se R$ 983 inclui taxas e bagagem despachada.`

**Contatos**
- `Anfitriã do Airbnb` · tipo `hospedagem` · selo `verificado` · fonte `Página oficial da reserva no app Airbnb` · telefone *(vazio — preencher no app)* · alerta `Número oficial é o do app. Se alguém de outro número disser ser a anfitriã, desconfiar. Pagamentos só pelo app.`
- `Atendimento Booking.com` · tipo `atendimento` · selo `verificado` · telefone *(vazio)* · fonte `—` · alerta `Telefone oficial só aparece logado no app. Todos os números do Google são golpe.`
- `Transfer Recife → Porto` · tipo `transfer` · selo `nao_verificado` · alerta `Usar empresa avaliada ou Uber/99. Desconfiar de abordagem no desembarque.`
- `Jangadas — piscinas naturais` · tipo `jangada` · selo `nao_verificado` · alerta `Contratar só no ponto oficial da associação de jangadeiros, preço tabelado.`

**Checklist (fase 2)**
- mala: protetor solar FPS alto · repelente · óculos de sol · chapéu/boné · roupa de banho · sapatilha para recifes · capa estanque para celular
- antes_de_sair: carregador do MacBook · carregador do iPhone · powerbank · fazer check-in online · baixar cartões de embarque · conferir tábua de marés
- documentos: RG ou CNH dos dois

**Diário (fase 2)** — entradas iniciais
1. `A grande triagem` — 12 destinos do Nordeste e as eliminações até sobrar um.
2. `As datas mudaram` — de 9–15 para 11–16 de novembro.
3. `Bateu o martelo: Porto de Galinhas` — vila a pé, sinal bom, zero rolê entre praias.
4. `Caçada à passagem` — Azul por R$ 983, abaixo do Google.
5. `Primeiro perrengue` — reserva pendente na Booking e a enxurrada de telefones-golpe.
6. `Casa garantida` — Airbnb na vila por R$ 1.770.

---

## 8. Fases

### Fase 1 — prazo: 1 semana
Login Google + autorização · multi-viagem · Painel completo (sem maré) · Gastos com lançamento rápido · Reservas · Contatos com antigolpe · Configurações · offline · PWA instalável · deploy em rota.barukstore.com.br · botão de dados iniciais (reservas + contatos).

**Critérios de aceite da fase 1**
- [ ] Vitor e Thais entram com Google e veem a mesma viagem em tempo real.
- [ ] Um terceiro e-mail logado vê "Acesso não autorizado" e não lê nada (testar no emulador ou com conta de teste).
- [ ] Com o iPhone em modo avião, dá para abrir o app e lançar um gasto; ao voltar a conexão, aparece no celular do outro.
- [ ] "Pode gastar hoje" bate com cálculo manual nos três estados (antes, durante, depois) — testar mudando a data do sistema ou com função `hoje()` sobrescrevível em modo debug.
- [ ] Salvar um contato com um número da lista negra dispara o aviso vermelho.
- [ ] Código de reserva aparece mascarado e só revela com toque longo.
- [ ] Nenhum dado sensível no repositório (`git grep` por código, endereço, e-mails).
- [ ] App instalado na tela inicial do iPhone abre em tela cheia.

### Fase 2 — antes de 11/11
Roteiro dia a dia com maré (cadastro manual das marés — os dados virão da tábua oficial da Marinha) · maré de hoje no Painel · Checklist compartilhado · Diário com link do álbum do iCloud · exportar Diário como texto (Markdown) · exportar gastos em CSV compatível com a Secretina (colunas: data, descrição, categoria, valor, forma, pagoPor).

---

## 9. Setup (passos manuais do Vitor)

1. Criar projeto no Firebase Console (`rota-baruk`), ativar Auth → Google e Firestore (região `southamerica-east1`).
2. Em Auth → Settings → Authorized domains, adicionar `rota.barukstore.com.br`.
3. Copiar o `firebaseConfig` para o `index.html` (a config web do Firebase pode ficar pública; quem protege é a regra).
4. Criar `firestore.rules` local com os dois e-mails reais e fazer `firebase deploy --only firestore:rules`.
5. Criar repo `vitorrix/rota`, ativar GitHub Pages, arquivo `CNAME` com `rota.barukstore.com.br`.
6. No Registro.br, criar registro CNAME `rota` → `vitorrix.github.io`. Ativar "Enforce HTTPS" no GitHub Pages depois que o DNS propagar.

---

## 10. Instruções para o Claude Code

- Implementar a **Fase 1 inteira** antes de começar a Fase 2.
- Antes de cada commit, verificar que nenhum dado sensível entrou (regra de ouro nº 1).
- Código organizado em seções comentadas dentro do `index.html` (estado, Firebase, cálculos, renderização por tela, utilitários).
- Funções de cálculo (5.1 a 5.3) puras e isoladas, com um bloco de testes simples rodável no console (`rotaTests()`).
- Textos da interface em português do Brasil.
- Ao terminar cada fase, listar o que foi feito, o que ficou de fora e o que precisa de ação manual.
