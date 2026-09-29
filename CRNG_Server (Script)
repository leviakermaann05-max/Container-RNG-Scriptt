
--// DEXTER'S PREMIUM C-RNG V4
--// SERVER SCRIPT
--// Place in ServerScriptService

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local CONFIG = {
    CollectDistance = 12,
    SellDistance = 20,
    CollectInterval = 0.5,
    SellInterval = 2,
    PotionInterval = 10,

    DefaultSpeed = 16,
    MinSpeed = 8,
    MaxSpeed = 40,

    Potions = {
        LuckPotion = 100,
        SpeedPotion = 150,
    }
}

--// CREATE REMOTES
local remoteFolder = ReplicatedStorage:FindFirstChild("CNRG_Remotes")
if not remoteFolder then
    remoteFolder = Instance.new("Folder")
    remoteFolder.Name = "CNRG_Remotes"
    remoteFolder.Parent = ReplicatedStorage
end

local function getRemote(name, className)
    local remote = remoteFolder:FindFirstChild(name)
    if not remote then
        remote = Instance.new(className)
        remote.Name = name
        remote.Parent = remoteFolder
    end
    return remote
end

local SetAutomation = getRemote("SetAutomation", "RemoteEvent")
local SetSpeed = getRemote("SetSpeed", "RemoteEvent")
local GetStatus = getRemote("GetStatus", "RemoteFunction")
local Notify = getRemote("Notify", "RemoteEvent")

--// GAME FOLDERS
local function getOrCreateFolder(name)
    local folder = workspace:FindFirstChild(name)
    if not folder then
        folder = Instance.new("Folder")
        folder.Name = name
        folder.Parent = workspace
    end
    return folder
end

local LootFolder = getOrCreateFolder("Loot")
local ContainersFolder = getOrCreateFolder("Containers")
local BuyersFolder = getOrCreateFolder("NPCBuyers")

--// PLAYER DATA
local playerData = {}

local function getData(player)
    return playerData[player]
end

local function getCoins(player)
    local stats = player:FindFirstChild("leaderstats")
    return stats and stats:FindFirstChild("Coins")
end

local function getInventory(player)
    local inventory = player:FindFirstChild("Inventory")
    if not inventory then
        inventory = Instance.new("Folder")
        inventory.Name = "Inventory"
        inventory.Parent = player
    end
    return inventory
end

local function setupPlayer(player)
    if playerData[player] then
        return
    end

    local leaderstats = player:FindFirstChild("leaderstats")
    if not leaderstats then
        leaderstats = Instance.new("Folder")
        leaderstats.Name = "leaderstats"
        leaderstats.Parent = player
    end

    local coins = leaderstats:FindFirstChild("Coins")
    if not coins then
        coins = Instance.new("IntValue")
        coins.Name = "Coins"
        coins.Value = 0
        coins.Parent = leaderstats
    end

    getInventory(player)

    playerData[player] = {
        Automation = {
            Collect = false,
            Sell = false,
            Potions = false,
        },
        Speed = CONFIG.DefaultSpeed,
        LastCollect = 0,
        LastSell = 0,
        LastPotion = 0,
    }

    player.CharacterAdded:Connect(function(character)
        local humanoid = character:WaitForChild("Humanoid", 10)
        local data = getData(player)

        if humanoid and data then
            humanoid.WalkSpeed = data.Speed
        end
    end)
end

Players.PlayerAdded:Connect(setupPlayer)

Players.PlayerRemoving:Connect(function(player)
    playerData[player] = nil
end)

for _, player in ipairs(Players:GetPlayers()) do
    setupPlayer(player)
end

--// HELPERS
local function getRoot(player)
    local character = player.Character
    return character and character:FindFirstChild("HumanoidRootPart")
end

local function distanceFromPlayer(player, part)
    local root = getRoot(player)
    if not root or not part then
        return math.huge
    end
    return (root.Position - part.Position).Magnitude
end

local function getPart(object)
    if object:IsA("BasePart") then
        return object
    end
    if object:IsA("Model") then
        return object.PrimaryPart
            or object:FindFirstChildWhichIsA("BasePart", true)
    end
    return nil
end

local function findNearestLoot(player)
    local root = getRoot(player)
    if not root then
        return nil
    end

    local nearest = nil
    local nearestDistance = CONFIG.CollectDistance

    for _, object in ipairs(LootFolder:GetChildren()) do
        local part = getPart(object)
        if part then
            local distance = (root.Position - part.Position).Magnitude
            if distance <= nearestDistance then
                nearest = object
                nearestDistance = distance
            end
        end
    end

    return nearest
end

--// COLLECTION
local function collectLoot(player)
    local data = getData(player)
    if not data then
        return
    end

    local now = os.clock()
    if now - data.LastCollect < CONFIG.CollectInterval then
        return
    end
    data.LastCollect = now

    local loot = findNearestLoot(player)
    if not loot then
        return
    end

    local value = loot:GetAttribute("Value")
    if typeof(value) ~= "number" or value < 0 then
        return
    end

    local inventory = getInventory(player)
    local item = Instance.new("Folder")
    item.Name = tostring(loot:GetAttribute("ItemName") or loot.Name)
    item:SetAttribute("SellValue", math.floor(value))
    item:SetAttribute("Rarity", tostring(loot:GetAttribute("Rarity") or "Common"))
    item.Parent = inventory

    loot:Destroy()
end

--// SELLING
local function isNearBuyer(player)
    local root = getRoot(player)
    if not root then
        return false
    end

    for _, buyer in ipairs(BuyersFolder:GetChildren()) do
        local part = getPart(buyer)
        if part then
            local distance = (root.Position - part.Position).Magnitude
            if distance <= CONFIG.SellDistance then
                return true
            end
        end
    end

    return false
end

local function sellInventory(player)
    local data = getData(player)
    if not data then
        return
    end

    local now = os.clock()
    if now - data.LastSell < CONFIG.SellInterval then
        return
    end
    data.LastSell = now

    if not isNearBuyer(player) then
        return
    end

    local inventory = getInventory(player)
    local coins = getCoins(player)
    if not coins then
        return
    end

    local earnings = 0

    for _, item in ipairs(inventory:GetChildren()) do
        local value = item:GetAttribute("SellValue")
        if typeof(value) == "number" and value > 0 then
            earnings += math.floor(value)
            item:Destroy()
        end
    end

    if earnings > 0 then
        coins.Value += earnings
        Notify:FireClient(player, "Sold items for " .. earnings .. " coins!")
    end
end

--// POTIONS
local function buyPotion(player, potionName)
    local price = CONFIG.Potions[potionName]
    if not price then
        return false
    end

    local coins = getCoins(player)
    if not coins or coins.Value < price then
        return false
    end

    coins.Value -= price

    local potions = player:FindFirstChild("Potions")
    if not potions then
        potions = Instance.new("Folder")
        potions.Name = "Potions"
        potions.Parent = player
    end

    local count = potions:FindFirstChild(potionName)
    if not count then
        count = Instance.new("IntValue")
        count.Name = potionName
        count.Value = 0
        count.Parent = potions
    end

    count.Value += 1
    Notify:FireClient(player, "Bought " .. potionName .. "!")
    return true
end

local function autoBuyPotion(player)
    local data = getData(player)
    if not data then
        return
    end

    local now = os.clock()
    if now - data.LastPotion < CONFIG.PotionInterval then
        return
    end
    data.LastPotion = now

    -- Example default purchase. Change this to your chosen potion.
    buyPotion(player, "LuckPotion")
end

--// REMOTE VALIDATION
local allowedFeatures = {
    Collect = true,
    Sell = true,
    Potions = true,
}

SetAutomation.OnServerEvent:Connect(function(player, feature, enabled)
    local data = getData(player)
    if not data then
        return
    end

    if typeof(feature) ~= "string"
        or not allowedFeatures[feature]
        or typeof(enabled) ~= "boolean" then
        return
    end

    data.Automation[feature] = enabled
end)

SetSpeed.OnServerEvent:Connect(function(player, requestedSpeed)
    local data = getData(player)
    if not data or typeof(requestedSpeed) ~= "number" then
        return
    end

    local speed = math.clamp(
        math.floor(requestedSpeed),
        CONFIG.MinSpeed,
        CONFIG.MaxSpeed
    )

    data.Speed = speed

    local character = player.Character
    local humanoid = character
        and character:FindFirstChildOfClass("Humanoid")

    if humanoid then
        humanoid.WalkSpeed = speed
    end
end)

GetStatus.OnServerInvoke = function(player)
    local data = getData(player)
    if not data then
        return nil
    end

    return {
        Automation = {
            Collect = data.Automation.Collect,
            Sell = data.Automation.Sell,
            Potions = data.Automation.Potions,
        },
        Speed = data.Speed,
    }
end

--// AUTOMATION LOOP
task.spawn(function()
    while true do
        for _, player in ipairs(Players:GetPlayers()) do
            local data = getData(player)
            if data then
                if data.Automation.Collect then
                    collectLoot(player)
                end

                if data.Automation.Sell then
                    sellInventory(player)
                end

                if data.Automation.Potions then
                    autoBuyPotion(player)
                end
            end
        end

        task.wait(0.2)
    end
end)

print("[CNRG Server] Dexter's Premium C-RNG V4 loaded")
