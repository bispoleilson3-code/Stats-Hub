-- AutoCollectServer.lua
-- Coloque este script como Script em ServerScriptService.
-- Requisitos:
--  - Em workspace, uma pasta chamada "Brainrots" contendo os modelos/parts a ser coletados.
--  - Um RemoteEvent em ReplicatedStorage chamado "RequestCollectBrainrot".

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")

local BRAINROTS_FOLDER = Workspace:WaitForChild("Brainrots")
local EVENT = ReplicatedStorage:WaitForChild("RequestCollectBrainrot")

local MAX_DISTANCE = 1000       -- distância máxima para ser coletado
local COOLDOWN = 1.5          -- tempo entre pedidos do mesmo jogador (segundos)
local playerCooldowns = {}

-- Função utilitária para achar a posição do objeto (Model ou BasePart)
local function getPositionOf(obj)
    if obj:IsA("BasePart") then
        return obj.Position
    elseif obj:IsA("Model") then
        if obj.PrimaryPart then
            return obj.PrimaryPart.Position
        else
            -- tenta encontrar qualquer BasePart dentro do model
            local part = obj:FindFirstChildWhichIsA("BasePart")
            return part and part.Position
        end
    end
    return nil
end

EVENT.OnServerEvent:Connect(function(player)
    -- Verificações básicas de segurança
    if not player or not player.Character or not player.Character:FindFirstChild("HumanoidRootPart") then
        return
    end

    local now = tick()
    if playerCooldowns[player] and now - playerCooldowns[player] < COOLDOWN then
        return
    end
    playerCooldowns[player] = now

    local hrp = player.Character.HumanoidRootPart
    local closest = nil
    local closestDist = math.huge

    -- Procura o brainrot mais próximo
    for _, item in ipairs(BRAINROTS_FOLDER:GetChildren()) do
        local pos = getPositionOf(item)
        if pos then
            local d = (hrp.Position - pos).Magnitude
            if d < closestDist then
                closestDist = d
                closest = item
            end
        end
    end

    if not closest then
        return
    end

    if closestDist > MAX_DISTANCE then
        return
    end

    -- Checagem para evitar duplo-claim
    if closest:GetAttribute("Claimed") then
        return
    end

    -- Marca como reivindicado
    closest:SetAttribute("Claimed", true)

    -- Log de auditoria (opcional)
    print(("Player %s coletou %s (dist %.2f)"):format(player.Name, closest.Name, closestDist))

    -- Ação de "dar" o item ao jogador.
    -- Exemplo: se for um Tool, enviar para Backpack; caso contrário, clonar e anexar ao personagem.
    if closest:IsA("Tool") then
        closest.Parent = player:WaitForChild("Backpack")
    else
        -- Clone seguro do item para o jogador (evita remover original se quiser reprodução)
        local clone = closest:Clone()
        clone.Parent = player.Character
        -- Opcional: ajustar posição/weld se necessário
        -- Exemplo simples: mover clone para cerca da HumanoidRootPart
        if clone:IsA("BasePart") and player.Character:FindFirstChild("HumanoidRootPart") then
            clone.CFrame = player.Character.HumanoidRootPart.CFrame * CFrame.new(0, 0, -2)
        elseif clone:IsA("Model") then
            if clone.PrimaryPart and player.Character:FindFirstChild("HumanoidRootPart") then
                clone:SetPrimaryPartCFrame(player.Character.HumanoidRootPart.CFrame * CFrame.new(0, 0, -2))
            end
        end
    end

    -- Se quiser remover o objeto do mundo original, descomente:
    -- closest:Destroy()
end)# Stats-Hub
