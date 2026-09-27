# SALA_IMERSIVA_MAGIC_SHOW — contexto para o Claude

Projeto TouchDesigner de uma instalação imersiva (sala com projeção, câmera captando
silhuetas de pessoas, conteúdos reagindo a essas silhuetas). Build TD: 2025.33230 —
**abrir sempre com esse build** (binFolder específico, nunca "qualquer 2025").

Arquivo canônico: `SALA_IMERSIVA_MAGIC_SHOW.toe` (o `.N.toe` mais alto na raiz reflete
o último save; versões antigas ficam em `Backup/`).

**Atualizado em 2026-09-21** (sessão longa, reconstruiu boa parte de `CONTEUDO_4` do
zero e reorganizou a Perform Window). Se este resumo divergir do que você vê ao vivo no
TD, **confie no TD** — o patch pode ter sido mexido diretamente na UI depois deste save.

## Como retomar o trabalho

1. `td_list_instances` pra confirmar que o TD abriu e o MCP conectou.
2. `td_get_hints('bootstrap')` — LER INTEIRO antes de mexer em qualquer coisa.
3. Ler este arquivo inteiro antes de mexer em `CONTEUDO_4` ou na Perform Window (`/project1`).
4. **NUNCA pulsar `recreateall` de um `replicatorCOMP`, nem `initializepulse`/`startpulse`
   repetidos de um `feedbackPOP`/`particlePOP`, via script em sequência rápida** — isso
   derrubou o TD (crash completo, perdendo trabalho não salvo) pelo menos 2x nesta sessão.
   Uma pulsação isolada é OK; várias em sequência rápida (ex: debug loop) não.
5. Salvar o projeto (`project.save()`) antes de qualquer mudança estrutural arriscada
   (replicator, feedback loop, particlePOP) — é o único jeito confiável de ter um ponto
   de restauração, já que crashes descartam tudo não salvo.

## Visão geral da Perform Window (`/project1`)

Layout em 4 quadrantes, todo em `hmode='anchors'`/`vmode='anchors'` (percentual, escala
sozinho com o tamanho real da tela — não mexer em pixels fixos aqui):

- **`preview1`** (topo-direita, 0.5-1.0 x 0.5-1.0) — visualização 3D da sala física
  (paredes/piso/teto com geo+material, câmera `cameraViewport`) — é uma pré-vis do
  espaço físico, não o conteúdo do show.
- **`container1`** (meio-direita, 0.5-1.0 x 0.25-0.5) — mostra o **conteúdo final**
  (`contentSwitch1` → `conteudo` → `container1`). Cuidado: `container1` também
  ALIMENTA `preview1` como textura (dupla função: painel visível E processamento de
  sinal) — não isolar um do outro sem checar o outro lado.
- **`input`** (baixo-direita, 0.5-1.0 x 0.0-0.25) — mostra a **silhueta** (o vídeo
  processado que vira máscara pros conteúdos). Internamente compõe dois vídeos
  "magnific_the_two_white_silhouettes_*.mp4" que ficam sempre rodando (nunca é 100%
  preto mesmo sem ninguém na sala — isso já causou confusão, ver Gotchas).
- **`botoes`** (topo-esquerda, 0.0-0.5 x 0.5-1.0) — contém `parameter1`
  (parscope="Contentselect Timerminutes Silhuetas") e `fps_footer`, ambos fixados perto
  do topo do quadrante via `vmode='anchors'` com `topanchor=bottomanchor=1.0` + offsets.
- Quadrante baixo-esquerda (0.0-0.5 x 0.0-0.25... na verdade 0.0-0.5 x 0.0-0.5) — vazio
  de propósito, a pedido do usuário.

`select1.par.top = silhuets` (a fonte bruta da silhueta, em `/project1`).
`level1.par.opacity` liga/desliga a SOBREPOSIÇÃO da silhueta no `conteudo` — **não afeta
a fonte** (`silhuets`/`select3`), são coisas diferentes (já causou confusão).

## Visão geral do que existe em CONTEUDO_4

`CONTEUDO_4` foi **reconstruído do zero** nesta sessão (a versão anterior, `flores` com
Kit A/B, feedback+GLSL, foi apagada pelo usuário no meio do processo). Estrutura atual:

- **`grid_50`** (gridPOP) — grid principal de pontos, paramétrico: `rows=50`,
  `cols` e `sizex` calculados por expressão a partir da proporção de `select2`
  (resolução real do projetor, ex. 5094x1200 → aspecto 4.245). `pos` (nullPOP) é o
  passthrough usado como fonte de posição pelo resto do sistema.
- **Flores (`geo1`)** — quad (`rectangle1`) instanciado em cada ponto do grid:
  - `mask_lookup`→`mask_to_chop` — amostra `select3` (a máscara) na posição de cada
    ponto (via atributo `Tex`, UV do grid).
  - `mask_lag` (lagCHOP, `lagsamples=True`) — suaviza a máscara por ponto com curva de
    entrada/saída assimétrica (`lag1`=sobe, `lag2`=desce) — dá o crescimento suave.
  - `scale_rand` (patternCHOP, random por ponto) × `mask_lag` = `scale_final` → escala
    de cada flor (`geo1.instancesx/sy`).
  - `rot_speed`/`rot_phase`/`rot_anim`/`rot_final` — rotação contínua por flor, cada
    uma com velocidade e sentido aleatório (`rot_anim.gain.expr = absTime.seconds`,
    dá rotação sem fim, não oscila).
  - `index` (patternCHOP, `wavetype=random`) — índice de textura por ponto, ligado a
    `geo1.instancetexindexop`. Texturas em `textures/tex0..tex7` (cada um é um baseCOMP
    com sua própria cadeia de imagem, saída em `texN/tex`), montado via um
    `replicatorCOMP` (`textures/replicator1`) que o usuário fez manualmente.
  - `flower_count` (constantCHOP) — contagem de pontos ativa, lida por expressão
    (`op('pos').numPoints()`), usada por outros CHOPs que precisam saber "quantas
    flores existem" sem hardcode.
- **Partículas voadoras (`fly_particles/`)** — sistema **remontado igual ao antigo**
  (GLSL + feedbackPOP de verdade, não uma aproximação cíclica em CHOP — essa tentativa
  foi feita e descartada por pedido do usuário, "ficou uma merda"):
  - `fieldFly`→`initFly`→`feedbackFly`→`flySim`→`outFly`, shader em `flySim_shader`
    (textDAT). Máquina de estados por partícula: cada uma tem um `HomeP` fixo (sorteado
    uma vez); fica parada lá até a máscara (`select3`) passar por cima com valor acima
    de `uThreshold`; aí "decola" com velocidade aleatória, gira (`RotFly`), cai
    (gravidade leve), desvanece (`FlyAlpha`) e some quando `FlyAlpha<=0`, voltando a
    ficar parada em `HomeP` esperando a máscara de novo.
  - `FlySize = FlyAlpha * uFlyScale` — o tamanho ENCOLHE junto com o desvanecimento,
    de propósito (pedido do usuário).
  - `geoFly` usa `instance_sopFly` (poptoSOP) pra todos os `instancetop/rop/sop/
    texindexop` — TODOS apontando pro MESMO poptoSOP (padrão do sistema antigo).
  - **Parâmetro exposto**: `fly_particles.par.Flycount` (custom, Int) controla
    `fieldFly.par.cols`. Mudar o valor **redimensiona corretamente** o pool de
    partículas, mas precisa de um "reinit" manual do feedback loop
    (`feedbackFly.par.initializepulse.pulse()` seguido de `.startpulse.pulse()`) — hoje
    existe um `parameterexecuteDAT` (`flycount_reinit`) tentando automatizar isso, mas
    **não confirmei que ele dispara sozinho** (pulses disparados por script MCP são
    pouco confiáveis nesta sessão; testar mudando o parâmetro direto na UI do TD).
  - `geoFly` está registrado em `render1.par.geometry` junto com `geo1` via padrão
    `'* fly_particles/geoFly'` (wildcard `*` pega filhos diretos de CONTEUDO_4, mais
    o caminho explícito pro que está dentro de `fly_particles`).

## Gotchas de TD descobertos/reconfirmados nesta sessão

- **`replicatorCOMP.par.recreateall.pulse()` e `feedbackPOP`/`particlePOP`
  `initializepulse`/`startpulse` pulsados repetidamente via script em sequência
  derrubaram o TD por completo** (crash, perda de tudo não salvo) — pelo menos 2x.
  Uma pulsação isolada funciona; não fazer loop de tentativa-e-erro com pulses.
- **geometryCOMP.instancetop/instancecolorop apontando pra um POP cuja contagem de
  pontos é "dinâmica" (GPU-only)** dá o erro "POP with point count info on GPU can
  only be the main OP". Fix que funcionou: inserir um **`poptoSOP`** entre o POP e o
  `geo.par.instance*op` (força a contagem a existir no CPU) — usar esse SOP pra
  TODOS os `instance*op` do mesmo geo (não misturar POP direto com SOP bridge).
  Pra POPs simples com contagem que muda por thinning/delete, `par.cpureadback=True`
  no op que muda a contagem (ex: `deletePOP`) também resolveu, sem precisar do bridge.
- **mathCHOP: a ordem real das operações é `preop → chanop → chopop → postop →
  preoff → gain → postoff`** — ou seja, `postop` roda ANTES de `preoff`/`gain`/
  `postoff`, apesar do nome. Pra fazer `(x - 0.5) * 2` e DEPOIS elevar ao quadrado,
  precisa de DOIS mathCHOPs em sequência (um só pro deslocamento, outro só com
  `postop='square'` no valor já deslocado) — não dá pra fazer num nó só.
- **geometryCOMP novo (POP) não expõe `inputConnectors`/`inputCOMPConnectors` do jeito
  esperado via Python** pra receber a forma-base — o caminho que funcionou foi criar a
  geometria (`gridPOP`, etc.) como FILHO do próprio `geometryCOMP` (destruindo antes o
  `torus1` padrão que já vem lá dentro), não tentar conectar de fora.
- **`select3` / a fonte de silhueta pode nunca ficar 100% preta** mesmo sem ninguém na
  sala, se a fonte incluir vídeos de fallback/placeholder sempre tocando (aqui,
  `magnific_the_two_white_silhouettes_*.mp4` dentro de `/project1/input`). Antes de
  assumir "bug na minha lógica de escala/máscara", checar os valores reais do TOP
  fonte (`numpyArray().min()/.max()/.mean()`) — economiza muito tempo de debug.
- **`level1.par.opacity` (liga/desliga a sobreposição visual da silhueta) é
  completamente separado da fonte real da máscara** (`silhuets`/`select3`) — mexer
  num não afeta o outro.
- **`replicatorCOMP.par.destination` default `=parent()`** cria os replicantes como
  IRMÃOS do replicator (não filhos) — `rep.childCount` sempre dá erro/0, tem que
  olhar `rep.parent().children` depois de rodar.
- **`replicatorCOMP` já vem com um textDAT de callbacks auto-gerado** (nome sugerido
  em `par.callbacks`, ex. `replicator1_callbacks`) — não criar um novo, só popular o
  que já existe (criar um novo dá colisão de nome e um callback duplicado inútil).
  O `onRemoveReplicant` default do TD já vem com `replicant.destroy()` — **não
  chamar `.destroy()` de novo lá dentro** se o `recreateall` já faz isso sozinho
  (chamar duas vezes gera erro "operador já deletado" e pode travar a recriação).
- **`deletePOP.par.invert`**: `'delete'` (default) mantém só os pontos que baterem no
  filtro (`thinstep` etc.); `'keep'` faz o oposto (mantém tudo MENOS o filtro) — nome
  contra-intuitivo, testar com `numPoints()` sempre que usar thinning.
- Layout de painel: usar `hmode='anchors'`/`vmode='anchors'` (fração 0-1 do pai) em vez
  de pixels fixos ou expressões manuais tipo `parent().par.w/2` — é o jeito nativo do
  TD de fazer layout responsivo de verdade, e o manual (`td_get_hints('panel_layout')`)
  documenta isso bem. Não recriar painéis novos "espelhando" conteúdo que já existe
  na network — tudo que está em `/project1` já vai pra Perform Window; o trabalho é só
  reposicionar o que já existe.

## Pendências / próximos passos possíveis

- Confirmar se `flycount_reinit` (parameterexecuteDAT em `fly_particles`) dispara
  sozinho quando `Flycount` muda pela UI do TD (não pelo MCP) — se não disparar,
  investigar um gatilho mais confiável (ex: `chopexecuteDAT` observando algo, ou só
  aceitar que precisa de um pulse manual depois de mudar o valor).
  - Nesse `pending também expor outros parâmetros do sistema Fly (`Flyspeed`,
  `Flydrift`, `Flyfade`, `Flyscale`, `Threshold`, `Invert`) como custom pars em
  `fly_particles`, hoje são valores fixos direto no `flySim` (0.4, 0.3, 0.3, 0.05,
  0.5, 0 respectivamente).
- O sistema de flores (`geo1`) não tem "wiggle" (balanço leve de posição em x/y) —
  foi cogitado mas não implementado nesta sessão; só tem escala+rotação+crescimento
  por máscara.
- Salvar o projeto — várias mudanças estruturais grandes foram feitas nesta sessão,
  confirmar que o save mais recente reflete o estado atual antes de fechar o TD.
