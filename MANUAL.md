# 🚓 ps-dispatch - Manual de Funcionalidades

Sistema de despacho avançado para QBCore com suporte integrado a MDT e alertas personalizáveis.

**Versão:** 2.1.7 | **Framework:** QBCore | **Licença:** MIT

---

## 🎯 O que o ps-dispatch faz

O ps-dispatch é um sistema de chamadas de emergência para servidores QBCore. Ele gerencia alertas personalizáveis com prioridades, integração de mapa em tempo real com blips, alertas sonoros e roteamento específico por emprego (polícia, EMS, etc.). Integra-se com o ps-mdt para fluxo de trabalho policial completo.

---

## ⚙️ Como funciona

O sistema de despacho opera através de:
- **Cliente:** Interface NUI para visualização de chamadas, blips no mapa, alertas sonoros
- **Servidor:** Gerenciamento de chamadas, validação de empregos, roteamento de alertas
- **Integração:** PolyZone para zonas de alerta, ps-mdt para gerenciamento policial

Quando um alerta é disparado (por crime, acidente, etc.), ele aparece na interface dos jogadores com o emprego correspondente.

---

## 🔧 Configuração

Edite `shared/config.lua`:

```lua
Config = {
    -- Tipos de alerta
    AlertTypes = {
        ['carjacking'] = {
            message = 'Roubo de Veículo',
            code = '10-35',
            icon = 'fas fa-car-burst',
            priority = 2,
            sprite = 225,           -- Blip sprite
            color = 1,              -- Blip color
            scale = 0.8,            -- Blip scale
            sound = 'alert_1',      -- Som de alerta
            jobs = {'police'}       -- Empregos que recebem
        },
        ['shooting'] = {
            message = 'Disparos',
            code = '10-71',
            icon = 'fas fa-gun',
            priority = 1,
            sprite = 304,
            color = 1,
            scale = 1.0,
            sound = 'alert_2',
            jobs = {'police', 'ems'}
        }
    },
    
    -- Configurações de blip
    BlipSettings = {
        sprite = 161,
        color = 1,
        scale = 0.7,
        flash = true
    },
    
    -- Efeitos sonoros
    SoundEffects = {
        enable = true,
        volume = 0.5
    },
    
    -- Roteamento de empregos
    JobRouting = {
        ['police'] = {dispatch = true, blips = true},
        ['ems'] = {dispatch = true, blips = false},
        ['fire'] = {dispatch = false, blips = true}
    }
}
```

### Copiar Sons para interact-sound
Copie os arquivos de som de `ps-dispatch` para `interact-sound/client/html/sounds` para alertas sonoros funcionarem.

---

## 📤 Exports

### Exports de Alertas Predefinidos (Cliente)
```lua
-- 30+ alertas predefinidos disponíveis
exports['ps-dispatch']:CarJacking(vehicle)
exports['ps-dispatch']:StoreRobbery(camId)
exports['ps-dispatch']:OfficerDown()
exports['ps-dispatch']:Shooting()
exports['ps-dispatch']:VehicleTheft(vehicle)
exports['ps-dispatch']:Fight()
exports['ps-dispatch']:DrugSale()
exports['ps-dispatch']:HouseRobbery()
exports['ps-dispatch']:BankRobbery()
exports['ps-dispatch']:AtmRobbery()
-- ... e muitos outros
```

### Alerta Personalizado (Cliente)
```lua
exports['ps-dispatch']:CustomAlert({
    message = "Atividade Suspeita",
    code = "10-35",
    icon = "fas fa-exclamation-triangle",
    priority = 2,
    coords = coords,                -- vector3 ou tabela {x, y, z}
    jobs = {'police', 'ems'},        -- Empregos que recebem
    blip = {
        sprite = 161,
        color = 1,
        scale = 0.8,
        flash = true
    },
    sound = 'alert_1',
    timeout = 30000                  -- Tempo até o blip desaparecer
})
```

### Disparar Alerta via Servidor
```lua
TriggerClientEvent('ps-dispatch:client:triggerAlert', -1, {
    message = 'Crime reportado',
    code = '10-91',
    jobs = {'police'}
})
```

---

## 📡 Eventos

### Eventos do Cliente
| Evento | Descrição | Parâmetros |
|--------|-----------|-------------|
| `ps-dispatch:client:triggerAlert` | Dispara alerta personalizado | `alertData` (table) |
| `ps-dispatch:client:displayCall` | Exibe chamada recebida | `callData` (table) |
| `ps-dispatch:client:addBlip` | Adiciona blip no mapa | `coords`, `blipData` |
| `ps-dispatch:client:playSound` | Toca som de alerta | `soundName` (string) |

### Eventos do Servidor
| Evento | Descrição | Parâmetros |
|--------|-----------|-------------|
| `ps-dispatch:server:notify` | Envia alerta | `data` (table) |
| `ps-dispatch:server:addCall` | Adiciona chamada | `callData` (table) |
| `ps-dispatch:server:removeCall` | Remove chamada | `callId` (int) |

---

## 🎮 Comandos

| Comando | Descrição | Permissão |
|---------|-----------|------------|
| `/dispatch` | Abre o menu de despacho | Polícia/EMS |
| `/toggleDispatch` | Alterna UI de despacho | Polícia/EMS |

---

## 🔗 Integrações

### Dependências
- **qb-core** (obrigatório) - Framework principal
- **ox_lib** (obrigatório) - UI e locale
- **ps-mdt** (opcional) - Integração com MDT policial
- **PolyZone** (obrigatório) - Detecção de zonas

### Integração com PolyZone
```lua
-- Criar zona de alerta
exports['PolyZone']:CreateCircle(vec2(25.0, -1345.0), 50.0, {
    name = 'zona_banco',
    debugPoly = false
})

-- Disparar alerta ao entrar na zona
exports['ps-dispatch']:CustomAlert({
    message = "Invasão de Banco",
    code = "10-99",
    coords = vec3(25.0, -1345.0, 29.5),
    jobs = {'police'}
})
```

### Integração com ps-mdt
Quando ps-mdt está instalado, as chamadas de despacho aparecem automaticamente no MDT dos oficiais.

---

## 💡 Casos de Uso

### Alerta de Roubo de Veículo (Cliente)
```lua
RegisterNetEvent('veiculos:roubado', function(vehicle)
    exports['ps-dispatch']:CarJacking(vehicle)
end)
```

### Alerta de Tiro (Cliente)
```lua
AddEventHandler('cef:onShotFired', function(weapon)
    if IsPlayerWantedLevelGreater(PlayerId(), 0) then
        exports['ps-dispatch']:Shooting()
    end
end)
```

### Sistema de Loja Sendo Roubada (Servidor)
```lua
RegisterServerEvent('loja:iniciarRoubo', function(storeId)
    local src = source
    local coords = GetEntityCoords(GetPlayerPed(src))
    
    TriggerClientEvent('ps-dispatch:client:triggerAlert', -1, {
        message = 'Loja sendo roubada',
        code = '10-31',
        coords = coords,
        jobs = {'police'},
        blip = {sprite = 52, color = 1, scale = 1.0}
    })
end)
```

### Alerta de Oficial Abatido (Cliente)
```lua
RegisterNetEvent('police:officerDown', function()
    exports['ps-dispatch']:OfficerDown()
end)
```

### Verificar se Jogador está em Serviço (Servidor)
```lua
local function isOnDuty(source)
    local Player = QBCore.Functions.GetPlayer(source)
    return Player.PlayerData.job.onduty
end

if isOnDuty(source) then
    TriggerClientEvent('ps-dispatch:client:displayCall', source, callData)
end
```

### Alerta com PolyZone (Cliente)
```lua
local zone = exports['PolyZone']:CreateCircle(vec2(200.0, -900.0), 30.0)

zone:onPointInOut(function(isPointInside)
    if isPointInside then
        exports['ps-dispatch']:CustomAlert({
            message = "Zona de Perigo",
            code = "10-80",
            coords = GetEntityCoords(PlayerPedId()),
            jobs = {'police'}
        })
    end
end)
```

---

## 🔊 Alertas Sonoros

Sons personalizados devem ser colocados em `interact-sound/client/html/sounds/`:
- `alert_1.ogg` - Alerta padrão
- `alert_2.ogg` - Alerta urgente
- `alert_3.ogg` - Alerta crítico

---

## 🌐 Localização

Suporta sistema de locale do ox_lib. Defina o idioma no `server.cfg`:
```cfg
setr ox:locale pt-BR
```

### Idiomas Suportados
- EN (Inglês)
- DE (Alemão)
- NL (Holandês)
- CS (Tcheco)
- **PT-BR (Português Brasil)**
- ES (Espanhol)

---

## ⚠️ Solução de Problemas

### Alertas não aparecem
- Verifique se o ps-dispatch está startado
- Confirme que o jogador tem o job correto
- Verifique se `jobs` está configurado no alerta

### Blips não aparecem no mapa
- Verifique as configurações de blip em `config.lua`
- Confirme que o jogador tem a UI de despacho aberta
- Verifique se não há conflito com outros sistemas de mapa

### Sons não tocam
- Verifique se os arquivos de som estão em `interact-sound/client/html/sounds`
- Confirme que o `interact-sound` está rodando
- Verifique o volume dos alertas em `config.lua`

### Integração com ps-mdt não funciona
- Verifique se o ps-mdt está startado
- Confirme que ambos usam a mesma versão de compatibilidade
- Olhe os logs do servidor para erros de integração

### Alertas duplicados
- Verifique se o evento não está sendo disparado múltiplas vezes
- Confirme que não há recursos conflitantes disparando alertas
- Use flags para evitar disparos duplicados

### Performance baixa
- Reduza o número de alertas simultâneos
- Aumente o timeout dos blips para remover mais rápido
- Evite disparar alertas em loops

---

## 📚 Créditos
- [OK1ez](https://github.com/OK1ez)
- [Project Sloth Team](https://github.com/Project-Sloth)
- [Candrex](https://github.com/CandrexDev)
- [Lenzh](https://github.com/Lenzh)
