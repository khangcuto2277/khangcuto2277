-- Load Rayfield UI Library
local success, Rayfield = pcall(function()
    return loadstring(game:HttpGet('https://sirius.menu/rayfield'))()
end)

if not success or not Rayfield then
    warn("Failed to load Rayfield UI Library!")
    return
end

-- Create the main window
local Window = Rayfield:CreateWindow({
    Name = "J97 hub",
    Icon = 0, -- Icon in Topbar. 0 để không dùng icon.
    LoadingTitle = "Wait",
    LoadingSubtitle = "by Khang",
    Theme = "Ocean", -- Tham khảo các theme tại: https://docs.sirius.menu/rayfield/configuration/themes
    DisableRayfieldPrompts = false,
    DisableBuildWarnings = false,
    ConfigurationSaving = {
        Enabled = true,
        FolderName = nil,
        FileName = "J97 hub"
    },
    Discord = {
        Enabled = true,
        Invite = "Chờ",
        RememberJoins = true
    },
    KeySystem = fasle,
    KeySettings = {
        Title = "Key system",
        Subtitle = "J97 hub",
        Note = "Key: https://discord.gg/ENNkqRJW._",
        FileName = "Key",
        SaveKey = false,
        GrabKeyFromSite = false,
        Key = {"1"}
    }
})

print("Window created successfully!")

------------------------------------------------------------
-- Phần Aimbot Cũ đã có (Ví dụ như Aimbot tự động quay camera theo NPC gần nhất)
------------------------------------------------------------
local RunService = game:GetService("RunService")
local Cam = workspace.CurrentCamera
local Player = game:GetService("Players").LocalPlayer

local validNPCs = {}
local raycastParams = RaycastParams.new()
raycastParams.FilterType = Enum.RaycastFilterType.Blacklist

local function isNPC(obj)
    return obj:IsA("Model") 
        and obj:FindFirstChild("Humanoid")
        and obj.Humanoid.Health > 0
        and obj:FindFirstChild("Head")
        and obj:FindFirstChild("HumanoidRootPart")
        and not game:GetService("Players"):GetPlayerFromCharacter(obj)
end

local function updateNPCs()
    local tempTable = {}
    for _, obj in ipairs(workspace:GetDescendants()) do
        if isNPC(obj) then
            tempTable[obj] = true
        end
    end
    for i = #validNPCs, 1, -1 do
        if not tempTable[validNPCs[i]] then
            table.remove(validNPCs, i)
        end
    end
    for obj in pairs(tempTable) do
        if not table.find(validNPCs, obj) then
            table.insert(validNPCs, obj)
        end
    end
end

local function handleDescendant(descendant)
    if isNPC(descendant) then
        table.insert(validNPCs, descendant)
        local humanoid = descendant:WaitForChild("Humanoid")
        humanoid.Destroying:Connect(function()
            for i = #validNPCs, 1, -1 do
                if validNPCs[i] == descendant then
                    table.remove(validNPCs, i)
                    break
                end
            end
        end)
    end
end

workspace.DescendantAdded:Connect(handleDescendant)

local function predictPos(target)
    local rootPart = target:FindFirstChild("HumanoidRootPart")
    local head = target:FindFirstChild("Head")
    if not rootPart or not head then
        return head and head.Position or rootPart and rootPart.Position
    end
    local velocity = rootPart.Velocity
    local predictionTime = 0.02
    local basePosition = rootPart.Position + velocity * predictionTime
    local headOffset = head.Position - rootPart.Position
    return basePosition + headOffset
end

local function getTarget()
    local nearest = nil
    local minDistance = math.huge
    local viewportCenter = Cam.ViewportSize / 2
    raycastParams.FilterDescendantsInstances = {Player.Character}
    for _, npc in ipairs(validNPCs) do
        local predictedPos = predictPos(npc)
        local screenPos, visible = Cam:WorldToViewportPoint(predictedPos)
        if visible and screenPos.Z > 0 then
            local ray = workspace:Raycast(
                Cam.CFrame.Position,
                (predictedPos - Cam.CFrame.Position).Unit * 1000,
                raycastParams
            )
            if ray and ray.Instance:IsDescendantOf(npc) then
                local distance = (Vector2.new(screenPos.X, screenPos.Y) - viewportCenter).Magnitude
                if distance < minDistance then
                    minDistance = distance
                    nearest = npc
                end
            end
        end
    end
    return nearest
end

local function aim(targetPosition)
    local currentCF = Cam.CFrame
    local targetDirection = (targetPosition - currentCF.Position).Unit
    local smoothFactor = 0.581
    local newLookVector = currentCF.LookVector:Lerp(targetDirection, smoothFactor)
    Cam.CFrame = CFrame.new(currentCF.Position, currentCF.Position + newLookVector)
end

local heartbeat = RunService.Heartbeat
local lastUpdate = 0
local UPDATE_INTERVAL = 0.4

local aimbotEnabled = false

heartbeat:Connect(function(dt)
    lastUpdate = lastUpdate + dt
    if lastUpdate >= UPDATE_INTERVAL then
        updateNPCs()
        lastUpdate = 0
    end
    if aimbotEnabled then
        local target = getTarget()
        if target then
            local predictedPosition = predictPos(target)
            aim(predictedPosition)
        end
    end
end)

workspace.DescendantRemoving:Connect(function(descendant)
    if isNPC(descendant) then
        for i = #validNPCs, 1, -1 do
            if validNPCs[i] == descendant then
                table.remove(validNPCs, i)
                break
            end
        end
    end
end)

-- Tạo Tab Aimbot
local AimbotTab = Window:CreateTab("Aimbot", 4483362458)

-- Toggle Aimbot cũ
local aimbotToggle = AimbotTab:CreateToggle({
    Name = "Enable Aimbot",
    CurrentValue = false,
    Flag = "AimbotToggle",
    Callback = function(Value)
        aimbotEnabled = Value
    end
})

------------------------------------------------------------
-- Phần NPC Lock Aimbot (Aimbot new) tích hợp từ script "dead rails wall hack aimbot"
------------------------------------------------------------
local Players = game:GetService("Players")
local player = Players.LocalPlayer
local runService = game:GetService("RunService")
local StarterGui = game:GetService("StarterGui")
local camera = workspace.CurrentCamera

-- Hàm tiện ích: lấy Humanoid của nhân vật người chơi
local function getPlayerHumanoid()
    if player.Character and player.Character:FindFirstChild("Humanoid") then
        return player.Character:FindFirstChild("Humanoid")
    end
    return nil
end

-- Hàm tiện ích: gửi thông báo
local function sendNotification(title, text, duration)
    StarterGui:SetCore("SendNotification", {
        Title = title,
        Text = text,
        Duration = duration or 3
    })
end

-- Hàm tìm NPC gần nhất (không xét các đối tượng là nhân vật người chơi)
local function getClosestNPC()
    local closestNPC = nil
    local closestDistance = math.huge
    if not (player.Character and player.Character:FindFirstChild("HumanoidRootPart")) then
        return nil
    end
    for _, object in ipairs(workspace:GetDescendants()) do
        if object:IsA("Model") then
            local humanoid = object:FindFirstChild("Humanoid")
            local hrp = object:FindFirstChild("HumanoidRootPart") or object.PrimaryPart
            if humanoid and hrp and humanoid.Health > 0 and object.Name ~= "Horse" then
                local isPlayer = false
                for _, pl in ipairs(Players:GetPlayers()) do
                    if pl.Character == object then
                        isPlayer = true
                        break
                    end
                end
                if not isPlayer then
                    local distance = (hrp.Position - player.Character.HumanoidRootPart.Position).Magnitude
                    if distance < closestDistance then
                        closestDistance = distance
                        closestNPC = object
                    end
                end
            end
        end
    end
    return closestNPC
end

-- Biến toàn cục để giữ connection của RenderStepped
local newAimbotConnection

-- Hàm khởi chạy NPC Lock Aimbot (Aimbot new)
local function startNewAimbot()
    sendNotification("Code by GioBolqvi", "on Roblox", 3)
    newAimbotConnection = runService.RenderStepped:Connect(function()
        local npc = getClosestNPC()
        if npc and npc:FindFirstChild("Humanoid") then
            local npcHumanoid = npc:FindFirstChild("Humanoid")
            if npcHumanoid.Health > 0 then
                camera.CameraSubject = npcHumanoid
            else
                sendNotification("Killed NPC", npc.Name, 0.4)
                if getPlayerHumanoid() then
                    camera.CameraSubject = getPlayerHumanoid()
                end
            end
        else
            if getPlayerHumanoid() then
                camera.CameraSubject = getPlayerHumanoid()
            end
        end
    end)
end

-- Hàm dừng NPC Lock Aimbot (Aimbot new)
local function stopNewAimbot()
    if newAimbotConnection then
        newAimbotConnection:Disconnect()
        newAimbotConnection = nil
    end
    if getPlayerHumanoid() then
        camera.CameraSubject = getPlayerHumanoid()
    end
end

-- Toggle "Aimbot new" tích hợp NPC Lock Aimbot
local aimbotNewToggle = AimbotTab:CreateToggle({
    Name = "Aimbot new",
    CurrentValue = false,
    Flag = "AimbotNewToggle",
    Callback = function(Value)
        if Value then
            startNewAimbot()
        else
            stopNewAimbot()
        end
    end
})

------------------------------------------------------------
-- Các Tab và tính năng khác (ví dụ: Brings, ESP, Report Bugs)
------------------------------------------------------------
local Tab = Window:CreateTab("Brings", 4483362458) 

local function GetItemNames()
    local items = {}
    local runtimeItems = workspace:FindFirstChild("RuntimeItems")
    if runtimeItems then
        for _, item in ipairs(runtimeItems:GetDescendants()) do
            if item:IsA("Model") then
                table.insert(items, item.Name)
            end
        end
    else
        warn("RuntimeItems folder not found!")
    end
    return items
end

local Dropdown = Tab:CreateDropdown({
    Name = "Choose item",
    Options = GetItemNames(), 
    CurrentOption = "Select an item",
    MultipleOptions = false,
    Flag = "ItemDropdown", 
    Callback = function(selectedItem)
        if type(selectedItem) == "table" then
            selectedItem = selectedItem[1] 
        end
    end,
})

local RefreshButton = Tab:CreateButton({
    Name = "Refresh Items",
    Callback = function()
        Dropdown:Refresh(GetItemNames())
    end,
})

local collectButton = Tab:CreateButton({
    Name = "Collect Selected Item",
    Callback = function()
        local selectedItemName = Dropdown.CurrentOption
        if type(selectedItemName) == "table" then
            selectedItemName = selectedItemName[1] 
        end

        if selectedItemName == "Select an item" then
            warn("No item selected!")
            return
        end

        local runtimeItems = workspace:FindFirstChild("RuntimeItems")
        if not runtimeItems then
            warn("RuntimeItems folder not found!")
            return
        end

        local selectedItem
        for _, item in ipairs(runtimeItems:GetDescendants()) do
            if item:IsA("Model") and item.Name == selectedItemName then
                selectedItem = item
                break
            end
        end

        if not selectedItem then
            warn("Item not found in RuntimeItems:", selectedItemName)
            return
        end

        local Players = game:GetService("Players")
        local LocalPlayer = Players.LocalPlayer
        if not LocalPlayer then
            warn("LocalPlayer not found!")
            return
        end

        local Character = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
        local HumanoidRootPart = Character:WaitForChild("HumanoidRootPart")

        if not selectedItem.PrimaryPart then
            warn(selectedItem.Name .. " has no PrimaryPart and cannot be moved.")
            return
        end

        selectedItem:SetPrimaryPartCFrame(HumanoidRootPart.CFrame + Vector3.new(0, 1, 0))
        print("Collected:", selectedItem.Name)
    end,
})

local collectAllButton = Tab:CreateButton({
    Name = "Collect All Items",
    Callback = function()
        local runtimeItems = workspace:FindFirstChild("RuntimeItems")
        if not runtimeItems then
            warn("RuntimeItems folder not found!")
            return
        end

        local ps = game:GetService("Players").LocalPlayer
        local ch = ps.Character or ps.CharacterAdded:Wait()
        local HumanoidRootPart = ch:WaitForChild("HumanoidRootPart")

        for _, item in ipairs(runtimeItems:GetDescendants()) do
            if item:IsA("Model") then
                if item.PrimaryPart then
                    local offset = HumanoidRootPart.CFrame.LookVector * 5
                    item:SetPrimaryPartCFrame(HumanoidRootPart.CFrame + offset)
                else
                    warn(item.Name .. " has no PrimaryPart.")
                end
            end
        end 
    end,
})

local EspTab = Window:CreateTab("ESP", 4483362458)

local ESPHandles = {}
local ESPEnabled = false

local function CreateESP(object, color)
    if not object or not object.PrimaryPart then return end

    local highlight = Instance.new("Highlight")
    highlight.Name = "ESP_Highlight"
    highlight.Adornee = object
    highlight.FillColor = color
    highlight.OutlineColor = color
    highlight.Parent = object

    local billboard = Instance.new("BillboardGui")
    billboard.Name = "ESP_Billboard"
    billboard.Adornee = object.PrimaryPart
    billboard.Size = UDim2.new(0, 200, 0, 50)
    billboard.StudsOffset = Vector3.new(0, 5, 0)
    billboard.AlwaysOnTop = true
    billboard.Parent = object

    local textLabel = Instance.new("TextLabel")
    textLabel.Text = object.Name
    textLabel.Size = UDim2.new(1, 0, 1, 0)
    textLabel.TextColor3 = color
    textLabel.BackgroundTransparency = 1
    textLabel.TextSize = 7
    textLabel.Parent = billboard

    ESPHandles[object] = {Highlight = highlight, Billboard = billboard}
end

local function ClearESP()
    for obj, handles in pairs(ESPHandles) do
        if handles.Highlight then handles.Highlight:Destroy() end
        if handles.Billboard then handles.Billboard:Destroy() end
    end
    ESPHandles = {}
end

local function UpdateESP()
    ClearESP()

    -- ESP cho Items 
    local runtimeItems = workspace:FindFirstChild("RuntimeItems")
    if runtimeItems then
        for _, item in ipairs(runtimeItems:GetDescendants()) do
            if item:IsA("Model") then
                CreateESP(item, Color3.new(1, 0, 0)) 
            end
        end
    end

    -- ESP cho mobs
    local baseplates = workspace:FindFirstChild("Baseplates")
    if baseplates and #baseplates:GetChildren() >= 2 then
        local secondBaseplate = baseplates:GetChildren()[2]
        local centerBaseplate = secondBaseplate and secondBaseplate:FindFirstChild("CenterBaseplate")
        local animals = centerBaseplate and centerBaseplate:FindFirstChild("Animals")
        if animals then
            for _, animal in ipairs(animals:GetDescendants()) do
                if animal:IsA("Model") then
                    CreateESP(animal, Color3.new(1, 0, 1)) -- Tím cho Animals
                end
            end
        end
    end

    local nightEnemies = workspace:FindFirstChild("NightEnemies")
    if nightEnemies then
        for _, enemy in ipairs(nightEnemies:GetDescendants()) do
            if enemy:IsA("Model") then
                CreateESP(enemy, Color3.new(0, 0, 1)) -- Xanh cho Night Enemies
            end
        end
    end

    local destroyedHouse = workspace:FindFirstChild("RandomBuildings") and workspace.RandomBuildings:FindFirstChild("DestroyedHouse")
    local zombiePart = destroyedHouse and destroyedHouse:FindFirstChild("StandaloneZombiePart")
    local zombies = zombiePart and zombiePart:FindFirstChild("Zombies")
    if zombies then
        for _, zombie in ipairs(zombies:GetChildren()) do
            if zombie:IsA("Model") then
                CreateESP(zombie, Color3.new(0, 1, 0)) -- Xanh lá cho Zombies
            end
        end
    end
end

local function AutoUpdateESP()
    while ESPEnabled do
        UpdateESP()
        wait() 
    end
end

local espToggle = EspTab:CreateToggle({
    Name = "ESP Items and Mobs",
    CurrentValue = false,
    Flag = "ESPAllToggle",
    Callback = function(Value)
        ESPEnabled = Value
        if Value then
            UpdateESP()
            coroutine.wrap(AutoUpdateESP)()
        else
            ClearESP()
        end
    end
})

local RTab = Window:CreateTab("Report Bugs", 4483362458) 

local DButton = RTab:CreateButton({
    Name = "Discord server click here to copy",
    Callback = function()
        setclipboard("chờ")
    end,
})