# Migração macOS → Windows 11 + 2x Kinect v1 (TouchDesigner)

Guia pra deixar o Windows 11 pronto pra rodar o projeto com **dois sensores Kinect v1**
(Xbox 360 / "Kinect for Windows" original) no TouchDesigner. Escrito em 2026-09-27,
pesquisado na hora — o Kinect v1 SDK é discontinuado desde ~2015 e o Windows 11 não é
oficialmente suportado, então isso envolve alguns contornos conhecidos.

## 1. Hardware — antes de instalar qualquer software

- **Cada Kinect precisa da fonte de alimentação AC própria** (a caixinha externa que
  vem com o sensor "Kinect for Windows" / o adaptador do Xbox 360 avulso). USB sozinho
  **não é suficiente** — o LED verde acende só com USB, mas o sensor não funciona
  direito sem a energia AC também.
- **Cada Kinect tem que estar num controlador USB FÍSICO diferente**, não só numa
  porta física diferente. Várias portas do mesmo notebook/PC costumam compartilhar o
  mesmo controlador USB por trás — se os dois Kinects caírem no mesmo controlador, um
  deles (ou os dois) não funciona direito. Checar no **Gerenciador de Dispositivos
  (Device Manager) → Controladoras de Barramento Serial Universal**, olhar a árvore de
  qual porta pertence a qual controlador (ou usar USBView / USB Tree Viewer da
  Microsoft pra ver isso visualmente).
- **Preferir portas USB 2.0**, não USB 3.0. O Kinect v1 é instável/desconecta em
  portas USB 3.0 em várias máquinas. Se o PC só tiver USB 3.0, um hub USB 2.0 dedicado
  por Kinect costuma ajudar (mas ainda respeitando a regra do controlador — testar).
- Se o notebook/PC não tiver controladores USB suficientes fisicamente separados, uma
  placa PCIe USB 2.0 (com chipset próprio, tipo NEC/Renesas) resolve — cada placa PCIe
  extra normalmente aparece como um controlador novo.

## 2. Software — instalação do SDK/Runtime

O TouchDesigner precisa de UM dos dois (o Runtime é mais leve e suficiente se só for
usar o Kinect dentro do TD, sem compilar nada):

- **Kinect for Windows Runtime v1.8** (mais simples, recomendado se só for usar no TD):
  https://www.microsoft.com/en-us/download/details.aspx?id=40277
- **Kinect for Windows SDK v1.8** (inclui o runtime + ferramentas de dev, ex. o
  Kinect Studio e o verificador de configuração):
  https://www.microsoft.com/en-us/download/details.aspx?id=40278
- (Opcional, só se for programar direto contra a SDK) Developer Toolkit v1.8:
  https://www.microsoft.com/en-us/download/details.aspx?id=40276

**Passos**:
1. **Desconectar os dois Kinects antes de instalar** (sensor desplugado da USB).
2. Se já existir qualquer driver de Kinect antigo instalado (de outra versão do SDK,
   ou de outro software tipo Driver4VR/OpenNI), desinstalar antes pra não conflitar.
3. Rodar o instalador (`KinectRuntime-v1.8-Setup.exe` ou o SDK completo) como
   Administrador.
4. Reiniciar o Windows.
5. **Só depois de instalar e reiniciar**, conectar o primeiro Kinect (energia AC +
   USB), esperar o Windows reconhecer, depois conectar o segundo.

## 3. Problemas conhecidos no Windows 11 (e como contornar)

O SDK v1.8 foi feito pra Windows 7/8/8.1 — não é suportado oficialmente no Windows 11.
Os drivers (`kinectcamera.sys`, etc.) não têm assinatura compatível com a política de
driver signing mais recente do Windows 11 em algumas builds/máquinas. Sintomas comuns:
sensor aparece "desabilitado" ou com erro no Gerenciador de Dispositivos, popup dizendo
que `kinectcamera.sys` não carregou, ou TD não lista nenhum sensor.

Ordem de tentativa (do mais simples pro mais invasivo):

1. **Rodar `sfc /scannow` e `DISM /Online /Cleanup-Image /RestoreHealth`** (prompt de
   comando como Admin) antes de mais nada — tem relato de gente que resolveu só com
   isso (corrupção de sistema mascarando o driver como incompatível).
2. No Gerenciador de Dispositivos, achar o dispositivo Kinect com problema → botão
   direito → Propriedades → Driver → **Atualizar Driver → Procurar no meu
   computador → Deixe-me escolher** → selecionar o driver do Kinect manualmente
   (fica em `C:\Windows\System32\DriverStore` ou na pasta de instalação do SDK) e
   forçar a instalação mesmo com aviso de compatibilidade.
3. Se o Windows bloquear por assinatura de driver (unsigned/incompatível), desativar
   a checagem de integridade **temporariamente**, só pra instalar:
   - Prompt como Admin: `bcdedit /set nointegritychecks ON`
   - Reiniciar, instalar/ativar o driver do Kinect
   - Depois reverter: `bcdedit /set nointegritychecks OFF` e reiniciar de novo
     (não deixar isso ligado permanentemente — é um risco de segurança).
   - Alternativa mais "oficial": reiniciar em **Modo de Teste** (`bcdedit /set
     testsigning ON`) só durante a instalação do driver.
4. Se nada disso resolver, considerar rodar o Kinect v1 via **libfreenect** (driver
   open-source, sem SDK da Microsoft) — existe integração com TD por caminhos
   alternativos, mas é mais trabalho de configurar; ficar como plano B.

## 4. Verificar que o Windows está reconhecendo os dois sensores

- Gerenciador de Dispositivos deve mostrar (sem "!" amarelo) três entradas por
  Kinect: **Kinect for Windows Audio Array Control**, **Kinect for Windows Camera**,
  **Kinect for Windows Security Control** — x2 (uma tripla pra cada sensor).
- Se instalou o SDK completo (não só o Runtime), tem o **Kinect for Windows Developer
  Toolkit Browser** com o "Kinect Configuration Verifier" — roda um diagnóstico e
  aponta exatamente o que está errado (driver, firmware, USB, etc.). Vale rodar antes
  de abrir o TouchDesigner.

## 5. Configuração no TouchDesigner

- Instalar o TouchDesigner (build igual ou compatível ao usado no macOS —
  `2025.33230`, mesma versão do projeto atual, pra não ter surpresa de operador
  mudando de comportamento entre versões).
- Cada sensor entra como um **Kinect CHOP** (dados de esqueleto/joint) e/ou um
  **Kinect TOP** (imagem de cor/profundidade) separado — **não dá pra usar dois
  Kinect CHOPs + dois Kinect TOPs ao mesmo tempo sem problema**: há relato confirmado
  no fórum oficial de que com 2x Kinect TOP + 2x Kinect CHOP simultâneos, os CHOPs
  param de receber dado. Se isso acontecer:
  - Testar usar só **1x Kinect TOP por sensor** (sem o CHOP) se só precisar da
    imagem/profundidade, ou só **1x Kinect CHOP por sensor** se só precisar do
    esqueleto — evitar rodar os dois tipos pros dois sensores ao mesmo tempo até
    confirmar que estabilizou.
  - Cada Kinect CHOP/TOP tem um parâmetro **Sensor** (existe só na versão 1, não
    existe no Kinect Azure) — é isso que escolhe QUAL dos dois sensores físicos
    aquele operador está lendo. Sem trocar esse parâmetro em cada um, os dois
    operadores vão ler o mesmo sensor.
  - Testar sensor por sensor, isolado, antes de ligar os dois juntos — assim dá pra
    saber se um problema é de hardware/driver (aparece isolado) ou de
    concorrência entre os dois no TD (só aparece com os dois rodando juntos).

## 6. Pipeline planejado: do Kinect até a silhueta (tudo em TOPs)

Decisão: usar os Kinects como **TOP** (GPU, 2D), não como SOP/point cloud 3D — mais
compatível com o resto do show, que já trabalha inteiramente em TOP/POP (`select3`,
`PIXEL_MAP`, etc.).

**Rota recomendada — Player Index (segmentação pronta pelo SDK)**:
- `Kinect TOP` → parâmetro **Image = "Player Index"**. O Kinect já roda o próprio
  algoritmo de detecção de esqueleto e marca no pixel QUAL pessoa é aquilo (valores em
  incrementos de ~0.1 por jogador/pessoa: ~0.1 = pessoa 1, ~0.2 = pessoa 2, fundo = 0).
  Isso já É a silhueta, sem precisar de threshold manual de profundidade.
- Pra virar uma máscara branco/preto simples (qualquer pessoa = branco, fundo = preto):
  um `thresholdTOP`/`levelTOP` com corte bem baixo (ex: acima de 0.05) já separa
  "tem gente" de "fundo" — ou, se quiser manter a identidade de cada pessoa
  separadamente (pra efeitos por-pessoa no futuro), não limpar, usar o valor cru.
- Limitação a testar: o "Player Index" só marca pixels de pessoas que o Kinect
  conseguiu **rastrear como esqueleto** (tracking ativo) — se alguém ficar parado
  de um jeito estranho ou fora do alcance de tracking (~0.8m–4m, de frente pro
  sensor), pode não aparecer. Testar esse limite fisicamente na sala antes de
  confiar 100% nessa rota.

**Rota alternativa/fallback — threshold de profundidade bruta**:
- `Kinect TOP` → **Image = "Depth"** (0-1, onde 1 = 8.191m do sensor).
- `levelTOP` (ou remap) cortando uma faixa de distância (ex: "qualquer coisa entre
  0.5m e 3m do sensor vira branco, resto vira preto") — não depende do tracking de
  esqueleto, só de distância, então pega qualquer objeto/pessoa nessa faixa (mais
  robusto a esse tipo de falha, mas menos "seletivo" — pode pegar objetos que não
  são pessoas se estiverem na mesma faixa de distância).
- Essa é a rota a usar se a de Player Index se mostrar instável/travando na prática.

**Combinando os 2 sensores**: cada Kinect TOP gera sua própria silhueta (uma por
sensor, cobrindo uma parte da sala/ângulo). Combinar as duas com um `compositeTOP`
ou `maxTOP` (pega o valor mais alto pixel a pixel — bom pra sobreposição de área sem
"cancelar" uma silhueta pela outra) **depois de** alinhar cada uma na posição
correta dentro do frame final (mesma lógica de `transform`/`fit` já usada hoje pra
casar a resolução com `PIXEL_MAP`).

O resultado final dessa cadeia (branco=pessoa, preto=fundo, resolução casada com
`PIXEL_MAP`) é o que substitui a fonte atual de `/project1/silhuets` — todo o resto
do show (flores, chuva, cubos, partículas) já lê daí e não precisa mudar.

## 7. Checklist rápido pro dia da instalação

1. [ ] Windows 11 atualizado (Windows Update em dia) antes de mexer em driver.
2. [ ] `sfc /scannow` + DISM rodados, sem erro.
3. [ ] Kinect Runtime ou SDK v1.8 instalado, PC reiniciado.
4. [ ] Kinect #1 conectado (AC + USB 2.0, controlador A) → aparece limpo no Device
      Manager → testado sozinho no TD (1x Kinect CHOP ou TOP, `Sensor`=0).
5. [ ] Kinect #2 conectado (AC + USB 2.0, controlador B, DIFERENTE do #1) → aparece
      limpo no Device Manager → testado sozinho no TD (`Sensor`=1).
6. [ ] Os dois juntos no TD, checando se os CHOPs continuam recebendo dado dos dois.
7. [ ] Projeto `.toe` aberto, testado ponta a ponta com os dois sensores.

## Fontes

- [Kinect1 | Derivative - TouchDesigner](https://derivative.ca/UserGuide/Kinect1)
- [Kinect CHOP | Derivative - TouchDesigner](https://derivative.ca/UserGuide/Kinect_CHOP)
- [Kinect TOP | Derivative - TouchDesigner](https://derivative.ca/UserGuide/Kinect_TOP)
- [Kinect TOP - TouchDesigner Documentation (parâmetro Image: Color/Depth/Infrared/Player Index)](https://docs.derivative.ca/Kinect_TOP)
- [Solution: Kinect CHOP doesn't work properly (multiple Kinect setup) — TouchDesigner forum](https://forum.derivative.ca/t/solution-kinect-chop-doesnt-work-properly-multiple-kinect-setup/429523)
- [Download Kinect for Windows SDK v1.8 — Microsoft](https://www.microsoft.com/en-us/download/details.aspx?id=40278)
- [Download Kinect for Windows Runtime v1.8 — Microsoft](https://www.microsoft.com/en-us/download/details.aspx?id=40277)
- [Download Kinect for Windows Developer Toolkit v1.8 — Microsoft](https://www.microsoft.com/en-us/download/details.aspx?id=40276)
- [kinect with windows 11 — Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/788416/kinect-with-windows-11)
- [SOLVED Kinect 1414 on Windows 11 Setup Issue — TroikaTronix forum](https://community.troikatronix.com/topic/8747/solved-kinect-1414-on-windows-11-setup-issue)
