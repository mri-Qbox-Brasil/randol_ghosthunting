# randol_ghosthunting — Manual

Minigame de caça-fantasmas inspirado no GTA Online: 5 fantasmas são sorteados a cada restart do servidor e o jogador precisa fotografá-los à noite com a câmera entregue pelo padre do cemitério.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Item `ghostcam`](#item-ghostcam)
4. [Parâmetros fixos](#parâmetros-fixos)
5. [Fluxo do minigame](#fluxo-do-minigame)
6. [Controles da câmera](#controles-da-câmera)
7. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
8. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qb-core`, `es_extended` **ou** `ND_Core` | Sim | Bridge automático em `bridge/`. O bridge ND exige ND_Core 2.0.0+ |
| `ox_lib` | Sim | Callbacks, points, textUI, scaleform, requests de modelo/anim/ptfx |
| `ox_inventory` **ou** `qb-inventory` | Sim | Fornece e torna usável o item `ghostcam`. O bridge detecta qual está ativo |
| `qb-target` | Sim | Interação com o padre usa `exports['qb-target']` |

**Game build**: requer build **2944** ou superior (modelos e sons dos fantasmas vêm do DLC de 2023).

---

## Instalação

1. Copie a pasta `randol_ghosthunting` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure randol_ghosthunting
   ```
3. Cadastre o item `ghostcam` no seu inventário (ver abaixo) e copie `image/ghostcam.png` para a pasta de imagens do inventário.
4. Não há SQL nem persistência — o progresso vive em memória e zera a cada restart.

---

## Item `ghostcam`

Este é o único item do recurso. Ele é entregue automaticamente pelo padre se o jogador ainda não tiver um.

### ox_inventory (`data/items.lua`)

```lua
['ghostcam'] = {
    label = 'Ghost Camera',
    weight = 100,
    stack = true,
    close = true,
    consume = 0,
    description = 'A camera for capturing spookies.',
    server = {
        export = 'randol_ghosthunting.ghostcam',
    },
},
```

### qb-inventory (`qb-core/shared/items.lua`)

```lua
ghostcam = { name = 'ghostcam', label = 'Ghost Camera', weight = 100, type = 'item', image = 'ghostcam.png', unique = false, useable = true, shouldClose = true, combinable = nil, description = 'A camera for capturing spookies.' },
```

---

## Parâmetros fixos

O recurso não tem arquivo de configuração. Os valores abaixo estão no código e só mudam editando os arquivos.

| Parâmetro | Valor | Onde |
|---|---|---|
| Padre (coords) | `-1681.11, -291.01, 50.88`, modelo `cs_priest` | `cl_ghost.lua` |
| Locais candidatos de fantasma | 11 `vec4`, embaralhados a cada start | `sv_ghost.lua` (`ghostLocations`) |
| Fantasmas por rodada | 5 (um por modelo `m23_1_prop_m31_ghost*`) | `sv_ghost.lua` |
| Janela de aparição | das **23h às 2h** do horário do jogo | `cl_ghost.lua` (`nearGhost`) |
| Distância de spawn do fantasma | 50 metros | `cl_ghost.lua` (`lib.points`) |
| Distância máxima para validar a foto | 20 metros | `sv_ghost.lua` (`isNearGhost`) |
| Recompensa por foto | `math.random(2500, 3750)` em dinheiro | `sv_ghost.lua` |

---

## Fluxo do minigame

1. No start do recurso, o servidor embaralha os locais e fixa 5 fantasmas (um por modelo). Os locais são enviados a todos os clientes e ficam valendo até o próximo restart.
2. O jogador fala com o padre no cemitério (opção de target **"Start Hunting"**). O servidor registra a caçada do personagem e entrega um `ghostcam` se ele não tiver.
3. Cada personagem só pode iniciar a caçada **uma vez por restart**.
4. Os fantasmas só aparecem entre **23h e 2h**. Fora dessa janela o objeto é removido mesmo que o jogador esteja perto.
5. Usar o item `ghostcam` abre a câmera. Se o jogador não tiver falado com o padre, o uso é bloqueado com notificação.
6. Tirar a foto com o fantasma na tela valida no servidor (distância máxima de 20 metros e modelo correspondente), paga o valor sorteado e mostra o scaleform "Ghosts Captured x/5". Cada fantasma só conta uma vez.

---

## Controles da câmera

| Controle | Ação |
|---|---|
| Clique esquerdo | Tirar a foto |
| Scroll | Zoom in / zoom out (FOV entre 5 e 80) |
| Backspace | Cancelar e guardar a câmera |

---

## Entrypoints para outros recursos

O recurso registra o export `ghostcam`, consumido pelo `ox_inventory` como handler de uso do item. É ele que abre a câmera no cliente e checa se a caçada foi iniciada.

```lua
-- registrado em bridge/server/<framework>.lua
exports('ghostcam', function(event, item, inventory, slot, data) end)
```

No fluxo `qb-inventory`, o mesmo comportamento é registrado via `QBCore.Functions.CreateUseableItem('ghostcam', ...)`.

---

## Estrutura de arquivos

```
randol_ghosthunting/
├── bridge/
│   ├── client/
│   │   ├── esx.lua        — notificação e estado de login no ESX
│   │   ├── nd.lua         — notificação e estado de login no ND_Core
│   │   └── qb.lua         — notificação, PlayerData e estado de login no QB
│   └── server/
│       ├── esx.lua        — GetPlayer, AddMoney, item ghostcam no ESX
│       ├── nd.lua         — GetPlayer, AddMoney, item ghostcam no ND_Core
│       └── qb.lua         — GetPlayer, AddMoney, item ghostcam (ox_inventory ou qb-inventory)
├── cl_ghost.lua           — padre, spawn dos fantasmas com ptfx, câmera e scaleform
├── sv_ghost.lua           — sorteio dos locais, validação da foto, recompensa
├── image/
│   └── ghostcam.png       — ícone do item para o inventário
└── fxmanifest.lua
```
