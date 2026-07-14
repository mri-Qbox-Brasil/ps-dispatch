# ps-dispatch — Manual

Sistema de despacho para polícia e EMS: alertas com blips, sons, lista de chamadas em NUI e mais de 30 exports prontos para outros recursos dispararem ocorrências.

---

## Sumário

1. [Dependências](#dependências)
2. [Instalação](#instalação)
3. [Configuração](#configuração)
4. [Comandos](#comandos)
5. [Teclas](#teclas)
6. [Alertas automáticos](#alertas-automáticos)
7. [Zonas](#zonas)
8. [Blips e sons](#blips-e-sons)
9. [Integrações](#integrações)
10. [Entrypoints para outros recursos](#entrypoints-para-outros-recursos)
11. [Localização](#localização)
12. [Estrutura de arquivos](#estrutura-de-arquivos)

---

## Dependências

| Recurso | Obrigatório | Observação |
|---|---|---|
| `qbx_core` | Sim | Framework base. O recurso usa `exports['qb-core']:GetCoreObject()`, resolvido pelo `provide 'qb-core'` do `qbx_core` |
| `ox_lib` | Sim | Locale, callbacks, keybinds, zonas (`lib.zones`) e comandos (`lib.addCommand`). v3.8.0 ou superior |
| `PolyZone` | Sim | Declarado no `fxmanifest.lua` (`@PolyZone/client.lua`, `CircleZone`, `BoxZone`) |
| `interact-sound` | Não | Necessário apenas para os sons `dispatch`, `panicbutton` e `robberysound`. Sem ele, os alertas que usam esses sons ficam mudos |

---

## Instalação

1. Copie a pasta `ps-dispatch` para `resources/`.
2. Adicione ao `server.cfg`:
   ```
   ensure ps-dispatch
   ```
3. Copie os arquivos de `sounds/` (`dispatch.ogg`, `panicbutton.ogg`, `robberysound.ogg`) para `interact-sound/client/html/sounds/`. Sem isso, os alertas de socorro (`officerdown`, `officerbackup`, `emsdown`) e os de assalto não tocam som.
4. **Conflitos** — não rode junto com outro recurso de dispatch (`qb-dispatch`, `cd_dispatch`, etc.). Os comandos `/911` e `/311` e o evento `ps-dispatch:server:notify` colidem.

---

## Configuração

Arquivo: `shared/config.lua`.

| Campo | Tipo | Obrigatório | Descrição |
|---|---|---|---|
| `Config.ShortCalls` | bool | Sim | Quando `true`, a notificação mostra só o nome do alerta; os detalhes ficam apenas no menu de dispatch |
| `Config.Debug` | bool | Sim | Ativa o debug das zonas e faz jogadores com job type `leo` também gerarem alertas de tiro e de excesso de velocidade |
| `Config.RespondKeybind` | string | Sim | Tecla padrão para marcar waypoint no último chamado (padrão `Y`) |
| `Config.OpenDispatchMenu` | string | Sim | Tecla padrão para abrir o menu de dispatch (padrão `F2`) |
| `Config.AlertTime` | número (s) | Sim | Tempo que o alerta fica na tela quando o alerta não define `alertTime` próprio |
| `Config.MaxCallList` | número | Sim | Máximo de chamados guardados no servidor; ao estourar, o mais antigo é descartado |
| `Config.OnDutyOnly` | bool | Sim | Quando `true`, só quem está em serviço (`job.onduty`) recebe alertas |
| `Config.Jobs` | array de strings | Sim | Job **types** que podem abrir o menu de dispatch (padrão: `leo`, `ems`) |
| `Config.DefaultAlertsDelay` | número (s) | Sim | Cooldown entre alertas automáticos do mesmo tipo, evita spam |
| `Config.DefaultAlerts` | tabela de bool | Sim | Liga/desliga cada alerta automático: `Speeding`, `Shooting`, `Autotheft`, `Melee`, `PlayerDowned`, `Explosion` |
| `Config.MinOffset` | número | Sim | Offset mínimo de deslocamento do blip |
| `Config.MaxOffset` | número | Sim | Offset máximo (em metros) aplicado a blips com `offset = true` |
| `Config.PhoneRequired` | bool | Sim | Quando `true`, exige um item de telefone no inventário para usar `/911` e `/311` |
| `Config.PhoneItems` | array de strings | Sim | Nomes dos itens aceitos como telefone (padrão: `phone`) |
| `Config.EnableHuntingBlip` | bool | Sim | Desenha um blip permanente nas zonas de caça |
| `Config.Locations.HuntingZones` | array | Sim | Esferas onde tiros geram alerta de caça em vez de tiroteio. Campos: `label`, `radius`, `coords` |
| `Config.Locations.NoDispatchZones` | array | Sim | Caixas onde tiros não geram alerta nenhum. Campos: `label`, `coords`, `length`, `width`, `heading`, `minZ`, `maxZ` |
| `Config.WeaponWhitelist` | array de strings | Sim | Armas que não disparam alerta de tiro (granadas, taser, extintor, etc.) |
| `Config.Blips` | tabela | Sim | Aparência do blip e som por `codeName` de alerta — veja [Blips e sons](#blips-e-sons) |
| `Config.Colors` | tabela | Sim | Mapa de índice de cor do GTA para nome legível, usado na descrição do veículo no chamado |

---

## Comandos

Todos registrados via `lib.addCommand`. Não há restrição por ACE — o filtro é o job do jogador.

| Comando | Permissão | Descrição |
|---|---|---|
| `/dispatch` | Job type em `Config.Jobs` | Abre o menu de chamados |
| `/911 [mensagem]` | Qualquer jogador | Envia chamado para a polícia (job type `leo`) |
| `/911a [mensagem]` | Qualquer jogador | Igual ao `/911`, mas anônimo (nome e número ocultos) |
| `/311 [mensagem]` | Qualquer jogador | Envia chamado para o EMS (job type `ems`) |
| `/311a [mensagem]` | Qualquer jogador | Igual ao `/311`, mas anônimo |

As chamadas `/911` e `/311` só passam se a mensagem não estiver vazia, o jogador não estiver algemado (`metadata.ishandcuffed`) e — quando `Config.PhoneRequired` está ligado — houver um item de `Config.PhoneItems` no inventário.

---

## Teclas

Registradas via `lib.addKeybind`; o jogador pode reatribuí-las em Configurações > Teclas do FiveM.

| Nome | Padrão | Ação |
|---|---|---|
| `mri_Q:RespondToDispatch` | `Config.RespondKeybind` (`Y`) | Marca waypoint no último chamado e anexa o jogador como unidade. Só fica ativa enquanto o alerta está na tela |
| `mri_Q:OpenDispatchMenu` | `Config.OpenDispatchMenu` (`F2`) | Abre o menu de chamados |

---

## Alertas automáticos

O `client/eventhandlers.lua` escuta eventos nativos do jogo e dispara alertas sozinho. Cada um respeita a flag correspondente em `Config.DefaultAlerts` e o cooldown de `Config.DefaultAlertsDelay`.

| Flag | Gatilho | Alerta gerado |
|---|---|---|
| `Shooting` | `CEventGunShot` | `VehicleShooting` se estiver em veículo, `Hunting` se estiver em zona de caça, senão `Shooting`. Ignora arma silenciada, arma na whitelist e zona sem dispatch |
| `Melee` | `CEventShockingSeenMeleeAction` | `Fight` |
| `Autotheft` | `CEventPedJackingMyVehicle` | `CarJacking` |
| `Autotheft` | `CEventShockingCarAlarm` | `VehicleTheft` |
| `Explosion` | `CEventExplosionHeard` | `Explosion` |
| `PlayerDowned` | `CEventNetworkEntityDamage` (morte) | `OfficerDown` se o job type for `leo`, `EmsDown` se for `ems`, senão `InjuriedPerson` |
| `Speeding` | Eventos de direção perigosa (`CEventShockingCarChase` e afins) | `SpeedingVehicle`, apenas acima de ~80–100 km/h e como condutor |

Jogadores com job type `leo` não geram alertas de tiro nem de excesso de velocidade, a menos que `Config.Debug` esteja ligado.

---

## Zonas

Definidas em `Config.Locations` e criadas com `lib.zones` na inicialização do recurso.

- **`HuntingZones`** — esferas (`coords` + `radius`). Tiros dentro delas viram alerta `Hunting` em vez de `Shooting`. O blip permanente só aparece se `Config.EnableHuntingBlip = true`.
- **`NoDispatchZones`** — caixas (`coords`, `length`, `width`, `heading`, `minZ`, `maxZ`). Tiros dentro delas não geram alerta algum. As duas Ammunation já vêm configuradas.

---

## Blips e sons

`Config.Blips` é indexado pelo `codeName` do alerta (`shooting`, `carjack`, `bankrobbery`, `officerdown`, …). Cada entrada aceita:

| Campo | Tipo | Descrição |
|---|---|---|
| `radius` | número | Raio do blip circular ao redor da ocorrência (`0` = sem raio) |
| `sprite` | número | Sprite do blip |
| `color` | número | Cor do blip |
| `scale` | número | Escala do blip |
| `length` | número (min) | Quanto tempo o blip leva para sumir (o raio some por fade) |
| `sound` | string | Som tocado. `Lose_1st` usa o `PlaySound` nativo; qualquer outro valor é enviado ao `interact-sound` |
| `sound2` | string | Soundset do som nativo (ex.: `GTAO_FM_Events_Soundset`) |
| `offset` | bool | Quando `true`, o blip é deslocado aleatoriamente em até `Config.MaxOffset` metros |
| `flash` | bool | Faz o blip piscar |

Um `codeName` sem entrada em `Config.Blips` cai no bloco `alert` embutido no próprio alerta — é o caminho usado pelo `CustomAlert`.

---

## Integrações

### interact-sound

Os sons `dispatch`, `panicbutton` e `robberysound` são tocados via `TriggerServerEvent("InteractSound_SV:PlayOnSource", som, 0.25)`. Os arquivos `.ogg` estão em `sounds/` e precisam ser copiados para o `interact-sound`. Alertas cujo som é `Lose_1st` usam o `PlaySound` nativo e não dependem desse recurso.

### Recursos de heist e crime

Qualquer recurso dispara uma ocorrência chamando os exports de alerta no lado cliente. Os alertas de assalto (`FleecaBankRobbery`, `PaletoBankRobbery`, `PacificBankRobbery`, `VangelicoRobbery`, `StoreRobbery`) aceitam um `camId` opcional, exibido no chamado como identificação da câmera de segurança.

---

## Entrypoints para outros recursos

### Alertas prontos (cliente)

Todos coletam sozinhos coordenadas, rua, direção e dados do veículo do jogador que chamou.

```lua
-- Genéricos
exports['ps-dispatch']:Shooting()
exports['ps-dispatch']:VehicleShooting()
exports['ps-dispatch']:SpeedingVehicle()
exports['ps-dispatch']:Fight()
exports['ps-dispatch']:Hunting()
exports['ps-dispatch']:Explosion()
exports['ps-dispatch']:SuspiciousActivity()
exports['ps-dispatch']:DrugDealing()
exports['ps-dispatch']:DrugSale()

-- Veículos
exports['ps-dispatch']:VehicleTheft()          -- usa o veículo atual do jogador
exports['ps-dispatch']:CarJacking(vehicle)     -- entity do veículo
exports['ps-dispatch']:CarBoosting(vehicle)    -- entity do veículo

-- Assaltos (camId é opcional e aparece no chamado)
exports['ps-dispatch']:StoreRobbery(camId)
exports['ps-dispatch']:FleecaBankRobbery(camId)
exports['ps-dispatch']:PaletoBankRobbery(camId)
exports['ps-dispatch']:PacificBankRobbery(camId)
exports['ps-dispatch']:VangelicoRobbery(camId)
exports['ps-dispatch']:HouseRobbery()
exports['ps-dispatch']:YachtHeist()
exports['ps-dispatch']:PrisonBreak()
exports['ps-dispatch']:SignRobbery()
exports['ps-dispatch']:ArtGalleryRobbery()
exports['ps-dispatch']:HumaneRobbery()
exports['ps-dispatch']:TrainRobbery()
exports['ps-dispatch']:VanRobbery()
exports['ps-dispatch']:UndergroundRobbery()
exports['ps-dispatch']:DrugBoatRobbery()
exports['ps-dispatch']:UnionRobbery()

-- Socorro
exports['ps-dispatch']:OfficerDown()
exports['ps-dispatch']:OfficerBackup()
exports['ps-dispatch']:OfficerInDistress()
exports['ps-dispatch']:EmsDown()
exports['ps-dispatch']:InjuriedPerson()
exports['ps-dispatch']:DeceasedPerson()
```

### Alerta customizado (cliente)

```lua
exports['ps-dispatch']:CustomAlert({
    message = 'Atividade suspeita',   -- título do alerta
    dispatchCode = 'meualerta',       -- vira o codeName; use para casar com Config.Blips
    code = '10-80',                   -- código exibido antes do título
    icon = 'fas fa-question',         -- ícone Font Awesome
    priority = 2,                     -- 1 = vermelho, 2 = padrão
    coords = GetEntityCoords(cache.ped),
    job = { 'leo' },                  -- lista de recipients
    gender = true,                    -- inclui o gênero do jogador no chamado
    alertTime = 10,                   -- segundos na tela (nil = Config.AlertTime)
    -- blip usado quando o dispatchCode não existe em Config.Blips
    radius = 0, sprite = 1, color = 1, scale = 0.5, length = 2,
    sound = 'Lose_1st', sound2 = 'GTAO_FM_Events_Soundset',
    offset = false, flash = false,
})
```

### Servidor

```lua
-- Lista de chamados ativos
local calls = exports['ps-dispatch']:GetDispatchCalls()

-- Enviar um alerta já montado a partir do servidor
TriggerEvent('ps-dispatch:server:notify', dispatchData)
```

### Eventos de rede

```lua
-- Cliente -> servidor
TriggerServerEvent('ps-dispatch:server:notify', dispatchData)
TriggerServerEvent('ps-dispatch:server:attach', callId, playerData)  -- anexa unidade ao chamado
TriggerServerEvent('ps-dispatch:server:detach', callId, playerData)  -- desanexa unidade

-- Servidor -> cliente
TriggerClientEvent('ps-dispatch:client:openMenu', src, calls)
TriggerClientEvent('ps-dispatch:client:sendEmergencyMsg', src, mensagem, '911', anonimo)
TriggerClientEvent('ps-dispatch:client:officerdown', src)
TriggerClientEvent('ps-dispatch:client:officerbackup', src)
TriggerClientEvent('ps-dispatch:client:emsdown', src)
```

### Callbacks (`lib.callback`)

```lua
local calls  = lib.callback.await('ps-dispatch:callback:getCalls', false)
local latest = lib.callback.await('ps-dispatch:callback:getLatestDispatch', false)
```

---

## Localização

As strings são traduzidas via `ox_lib` locale. Arquivos em `locales/`:

- `cs.json`, `de.json`, `en.json`, `es.json`, `fr.json`, `nl.json`, `pt-br.json`

O idioma ativo vem da convar no `server.cfg`:

```
setr ox:locale "pt-br"
```

Para adicionar um idioma, crie `locales/<codigo>.json` seguindo a estrutura do `en.json` e reinicie o recurso.

---

## Estrutura de arquivos

```
ps-dispatch/
├── client/
│   ├── main.lua           — NUI, menu, blips, keybinds, zonas e callbacks NUI
│   ├── alerts.lua         — todos os exports de alerta e o CustomAlert
│   ├── eventhandlers.lua  — alertas automáticos a partir de eventos do jogo
│   └── utils.lua          — dados do veículo, rua/zona, gênero, direção, checagem de telefone
├── server/
│   └── main.lua           — lista de chamados, attach/detach de unidades, comandos e callbacks
├── shared/
│   └── config.lua         — todas as opções, blips por alerta e tabela de cores
├── html/                  — build da UI (index.html, index.js, index.css)
├── ui/                    — código-fonte Svelte da UI (não é carregado em runtime)
├── sounds/                — dispatch.ogg, panicbutton.ogg, robberysound.ogg (copiar para o interact-sound)
├── locales/               — cs, de, en, es, fr, nl, pt-br
└── fxmanifest.lua
```
