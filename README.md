-- GameServer (ServerScriptService)

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local CollectionService = game:GetService("CollectionService")

-- === Remotes ===
local Remotes = ReplicatedStorage:FindFirstChild("Remotes") or Instance.new("Folder", ReplicatedStorage)
Remotes.Name = "Remotes"

local function getOrMakeRemote(name)
    local r = Remotes:FindFirstChild(name)
    if not r then
        r = Instance.new("RemoteEvent")
        r.Name = name
        r.Parent = Remotes
    end
    return r
end

local RE_ToggleAutoBuy   = getOrMakeRemote("ToggleAutoBuy")
local RE_ToggleAutoCollect = getOrMakeRemote("ToggleAutoCollect")
local RE_RequestPurchase = getOrMakeRemote("RequestPurchase")
local RE_CollectNearby   = getOrMakeRemote("CollectNearby")

-- === Configuração dos itens da loja (ordenados por preço) ===
local ShopItems = {
    {name = "Brainrot Pequeno", price = 50,  multiplier = 1},
    {name = "Brainrot Médio",   price = 200, multiplier = 3},
    {name = "Brainrot Épico",   price = 800, multiplier = 10},
    {name = "Brainrot Lendário",price = 2500, multiplier = 40},
}

-- === Leaderstats ===
Players.PlayerAdded:Connect(function(plr)
    -- leaderstats para aparecer na leaderboard
    local ls = Instance.new("Folder")
    ls.Name = "leaderstats"
    ls.Parent = plr

    local coins = Instance.new("IntValue")
    coins.Name = "Coins"
    coins.Value = 0
    coins.Parent = ls

    -- estado de auto features
    local auto = Instance.new("Folder")
    auto.Name = "Auto"
    auto.Parent = plr

    local autoBuy = Instance.new("BoolValue")
    autoBuy.Name = "AutoBuy"
    autoBuy.Value = false
    autoBuy.Parent = auto

    local autoCollect = Instance.new("BoolValue")
    autoCollect.Name = "AutoCollect"
    autoCollect.Value = false
    autoCollect.Parent = auto

    -- inventário simples: melhor item comprado
    local bestItem = Instance.new("StringValue")
    bestItem.Name = "BestBrainrot"
    bestItem.Value = "Nenhum"
    bestItem.Parent = plr
end)

-- === Anti-abuso simples (rate-limit) ===
local lastCall = {}
local function rateLimited(player, key, interval)
    interval = interval or 0.25
    lastCall[player] = lastCall[player] or {}
    local t = os.clock()
    if (lastCall[player][key] or 0) + interval > t then
        return true
    end
    lastCall[player][key] = t
    return false
end

-- === Lógica da loja ===
local function getBestAffordableItem(coins)
    local best = nil
    for _, item in ipairs(ShopItems) do
        if coins >= item.price then
            best = item
        else
            break
        end
    end
    return best
end

local function purchaseBestItem(plr)
    local coins = plr.leaderstats and plr.leaderstats:FindFirstChild("Coins")
    if not coins then return end
    local best = getBestAffordableItem(coins.Value)
    if not best then return end
    coins.Value -= best.price
    -- Atualiza melhor item
    local bestItem = plr:FindFirstChild("BestBrainrot")
    if bestItem then bestItem.Value = best.name end
end

RE_RequestPurchase.OnServerEvent:Connect(function(plr)
    if rateLimited(plr, "buy", 0.5) then return end
    purchaseBestItem(plr)
end)

RE_ToggleAutoBuy.OnServerEvent:Connect(function(plr, state)
    if typeof(state) ~= "boolean" then return end
    local b = plr:FindFirstChild("Auto") and plr.Auto:FindFirstChild("AutoBuy")
    if b then b.Value = state end
end)

RE_ToggleAutoCollect.OnServerEvent:Connect(function(plr, state)
    if typeof(state) ~= "boolean" then return end
    local c = plr:FindFirstChild("Auto") and plr.Auto:FindFirstChild("AutoCollect")
    if c then c.Value = state end
end)

-- === Esteira: geração de itens "Brainrot" ===
local Workspace = game:GetService("Workspace")
local ConveyorStart = Workspace:FindFirstChild("ConveyorStart")
local ConveyorEnd   = Workspace:FindFirstChild("ConveyorEnd")
local CollectorZone = Workspace:FindFirstChild("CollectorZone")

local BrainrotFolder = Workspace:FindFirstChild("Brainrots") or Instance.new("Folder", Workspace)
BrainrotFolder.Name = "Brainrots"

local function spawnBrainrot()
    if not ConveyorStart or not ConveyorEnd then return end
    local part = Instance.new("Part")
    part.Name = "Brainrot"
    part.Size = Vector3.new(1.5,1.5,1.5)
    part.Shape = Enum.PartType.Ball
    part.Material = Enum.Material.Neon
    part.Anchored = false
    part.CanCollide = true
    part.CFrame = ConveyorStart.CFrame + Vector3.new(0, 2, 0)
    part.Parent = BrainrotFolder

    -- Valor do item (moedas quando coletado)
    part:SetAttribute("Value", math.random(5,15))

    -- movimento simples em direção ao ConveyorEnd
    local dir = (ConveyorEnd.Position - ConveyorStart.Position).Unit
    local bv = Instance.new("BodyVelocity")
    bv.Velocity = dir * 12
    bv.MaxForce = Vector3.new(1e5,1e5,1e5)
    bv.Parent = part

    -- limpeza automática ao final
    game:GetService("Debris"):AddItem(part, 25)
end

-- Spawner
task.spawn(function()
    while task.wait(2.0) do
        spawnBrainrot()
    end
end)

-- === Coleta (servidor valida) ===
local function collectNearbyForPlayer(plr, radius)
    radius = radius or 12
    local char = plr.Character
    if not (char and char.PrimaryPart) then return end
    local origin = char.PrimaryPart.Position

    local coins = plr.leaderstats and plr.leaderstats:FindFirstChild("Coins")
    if not coins then return end

    local collected = 0
    for _, item in ipairs(BrainrotFolder:GetChildren()) do
        if item:IsA("BasePart") and item.Name == "Brainrot" then
            local dist = (item.Position - origin).Magnitude
            if dist <= radius then
                local value = item:GetAttribute("Value") or 5
                collected += value
                item:Destroy()
            end
        end
    end
    if collected > 0 then
        coins.Value += collected
    end
end

RE_CollectNearby.OnServerEvent:Connect(function(plr)
    if rateLimited(plr, "collect", 0.25) then return end
    collectNearbyForPlayer(plr, 12)
end)

-- Coletor baseado em toque (para quando itens passam pela zona)
if CollectorZone and CollectorZone:IsA("BasePart") then
    CollectorZone.Touched:Connect(function(hit)
        if hit and hit.Parent and hit.Name == "Brainrot" then
            -- Procura jogador mais próximo para creditar
            local nearestPlayer, nearestDist = nil, 25
            for _, plr in ipairs(Players:GetPlayers()) do
                local char = plr.Character
                if char and char.PrimaryPart then
                    local d = (char.PrimaryPart.Position - hit.Position).Magnitude
                    if d < nearestDist then
                        nearestDist = d
                        nearestPlayer = plr
                    end
                end
            end
            if nearestPlayer then
                local value = hit:GetAttribute("Value") or 5
                local coins = nearestPlayer.leaderstats and nearestPlayer.leaderstats:FindFirstChild("Coins")
                if coins then coins.Value += value end
            end
            hit:Destroy()
        end
    end)
end

-- === Loop de Auto-Features (servidor observa flags e age de forma justa) ===
task.spawn(function()
    while task.wait(1.0) do
        for _, plr in ipairs(Players:GetPlayers()) do
            local autoFolder = plr:FindFirstChild("Auto")
            if autoFolder then
                local doCollect = autoFolder:FindFirstChild("AutoCollect")
                if doCollect and doCollect.Value then
                    collectNearbyForPlayer(plr, 12)
                end
                local doBuy = autoFolder:FindFirstChild("AutoBuy")
                if doBuy and doBuy.Value then
                    purchaseBestItem(plr)
                end
            end
        end
    end
end)
