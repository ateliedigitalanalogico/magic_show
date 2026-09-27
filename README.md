# Magic Show — Sala Imersiva

Projeto TouchDesigner de uma instalação imersiva: sala com projeção, câmeras
captando silhuetas de pessoas, e conteúdos visuais que reagem em tempo real a
essas silhuetas.

## Como abrir

- Arquivo principal: **`SALA_IMERSIVA_MAGIC_SHOW.<N>.toe`** — sempre o de **maior
  número** na raiz é a versão atual (versões antigas ficam em `Backup/`).
- Build do TouchDesigner: **2025.33230**.
- Antes de mexer em qualquer coisa, ler **`CLAUDE.md`** — tem o estado atual da
  arquitetura do patch, decisões já tomadas e os "gotchas" já descobertos (evita
  redescobrir os mesmos bugs).

## Estrutura da pasta

- `SALA_IMERSIVA_MAGIC_SHOW.<N>.toe` — projeto principal (raiz, sempre o mais novo).
- `CLAUDE.md` — contexto técnico do patch pra retomar o trabalho (arquitetura atual,
  gotchas de TouchDesigner descobertos, pendências).
- `Apoio/` — guias de suporte pra migração/infra:
  - `MIGRACAO_WINDOWS11_KINECT_V1.md` — instalar o SDK do Kinect v1 no Windows 11
    e configurar 2 sensores no TouchDesigner.
  - `SSH_CLAUDE_CODE_WINDOWS.md` — ativar OpenSSH no Windows e rodar o Claude Code
    lá pra controlar o TD remotamente.
- `Tox/` — componentes `.tox` avulsos do projeto (incluindo o `twozero.tox`, o MCP
  usado pra controlar o TD via Claude Code).
- `Backup/` — versões numeradas antigas do `.toe` (histórico de save do próprio
  TouchDesigner).
- `Image/`, `Movie/`, `Audio/`, `Chan/`, `Geo/` — mídia e assets usados pelo patch.

## Sobre o desenvolvimento

Este projeto foi construído em boa parte com o **Claude Code** controlando o
TouchDesigner ao vivo através do plugin **twozero** (MCP). Ver `CLAUDE.md` pra
entender como retomar uma sessão de trabalho assim.
