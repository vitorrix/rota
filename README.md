# Rota

App de viagens (PWA) — https://rota.barukstore.com.br

HTML + CSS + JS vanilla, Firebase (Auth Google + Firestore com cache offline). Veja `SPEC.md`.

## Regras
- Repositório público: **nada sensível** no código (códigos de reserva, endereços, telefones, e-mails). Tudo isso é digitado no app e fica só no Firestore.
- `firestore.rules` está no `.gitignore`. Copie `firestore.rules.example`, ponha os e-mails reais e faça deploy local.

## Setup
1. Firebase Console: projeto `rota-baruk`, Auth → Google, Firestore em `southamerica-east1`.
2. Auth → Authorized domains: adicionar `rota.barukstore.com.br`.
3. Colar o `firebaseConfig` em `index.html` (seção "Config do Firebase").
4. `cp firestore.rules.example firestore.rules`, editar e-mails, `firebase deploy --only firestore:rules`.
5. GitHub Pages no repo `vitorrix/rota` (o `CNAME` já está aqui) e CNAME `rota` → `vitorrix.github.io` no Registro.br.

## Debug
- `rotaTests()` no console roda os testes dos cálculos.
- `?hoje=2026-11-13` (ou `setHoje('2026-11-13')`) simula a data para testar antes/durante/depois da viagem.
