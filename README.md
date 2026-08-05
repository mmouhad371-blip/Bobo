-- ============================================================
-- popcuru - Instant Steal Mobile (TP to Base)
-- Un seul bouton : vole le brainrot + TP à la base en 0.2s
-- Pas de lag, pas de rollback, pas de respawn intempestif
-- Version ultra-optimisée pour mobile
-- ============================================================

-- === CONFIGURATION ===
local Config = {
    StealRange = 60,                    -- Distance max pour détecter une cible
    TpHeightOffset = 2.0,               -- Hauteur de TP à la base
    SpamPromptCount = 8,                -- Nombre de spams du prompt (moins = plus discret)
    PromptDelay = 0.02,                 -- Délai entre chaque spam
    TeleportDuration = 0.06,            -- Durée totale du TP
    SpamCount = 20,                     -- Nombre d'updates CFrame (anti-rollback)
    AntiRollbackDelay = 0.01,           -- Délai entre chaque update CFrame
}

-- === INITIALISATION ===
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local LP = Players.LocalPlayer

local IsStealing = false
local StatusLabel = nil

-- === FONCTIONS UTILITAIRES ===

local function GetRoot(plr)
    if not plr or not plr.Character then return nil end
    return plr.Character:FindFirstChild("HumanoidRootPart")
end

local function GetHumanoid(plr)
    if not plr or not plr.Character then return nil end
    return plr.Character:FindFirstChildOfClass("Humanoid")
end

-- Trouver le joueur le plus proche avec un brainrot
local function GetClosestPlayerWithBrainrot()
    local myRoot = GetRoot(LP)
    if not myRoot then return nil, math.huge end
    
    local closest, closestDist = nil, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LP then
            local root = GetRoot(plr)
            if root then
                local dist = (root.Position - myRoot.Position).Magnitude
                if dist < closestDist then
                    -- Vérifier si le joueur a un brainrot
                    local hasBrainrot = false
                    if plr.Character then
                        for _, child in ipairs(plr.Character:GetChildren()) do
                            if child:IsA("Tool") and (child.Name:lower():find("brain") or child.Name:lower():find("rot")) then
                                hasBrainrot = true
                                break
                            end
                        end
                    end
                    if hasBrainrot then
                        closestDist = dist
                        closest = plr
                    end
                end
            end
        end
    end
    return closest, closestDist
end

-- Désactiver les contraintes (anti-blocage)
local function DisableConstraints(char)
    if not char then return end
    for _, constraint in ipairs(char:GetDescendants()) do
        if constraint:IsA("BasePart") then
            constraint:SetNetworkOwner(nil)
        end
        if constraint:IsA("Attachment") or constraint:IsA("Constraint") then
            pcall(function() constraint:Destroy() end)
        end
    end
end

-- No Clip total
local function SetNoclip(char, enabled)
    if not char then return end
    for _, part in ipairs(char:GetDescendants()) do
        if part:IsA("BasePart") then
            part.CanCollide = not enabled
            part.CanTouch = not enabled
            part.CanQuery = not enabled
            if enabled then
                part:SetNetworkOwner(nil)
            end
        end
    end
end

-- Forcer la synchro réseau
local function ForceNetworkSync(part)
    if not part then return end
    pcall(function()
        part:SetNetworkOwner(nil)
        task.wait(0.005)
        part:SetNetworkOwner(LP)
    end)
end

-- === TROUVER LA BASE DU JOUEUR ===
local function GetBasePosition()
    for _, obj in ipairs(workspace:GetDescendants()) do
        if obj:IsA("BasePart") and (obj.Name:lower():find("base") or obj.Name:lower():find("spawn")) then
            local owner = obj:FindFirstChild("Owner")
            if not owner or owner.Value == LP then
                return obj.Position + Vector3.new(0, Config.TpHeightOffset, 0)
            end
        end
    end
    -- Fallback : spawn
    local spawn = workspace:FindFirstChild("SpawnLocation")
    if spawn then
        return spawn.Position + Vector3.new(0, Config.TpHeightOffset, 0)
    end
    return nil
end

-- === TÉLÉPORTATION À LA BASE (Anti-Rollback) ===
local function TeleportToBase()
    local char = LP.Character
    if not char then return end
    
    local hrp = char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    
    local humanoid = char:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end

    local basePos = GetBasePosition()
    if not basePos then
        print("[popcuru] ❌ Aucune base trouvée")
        return false
    end

    local targetCF = CFrame.new(basePos)

    -- Désactiver les contraintes
    DisableConstraints(char)

    -- Activer le No Clip
    SetNoclip(char, true)

    -- Spam du CFrame (anti-rollback)
    for i = 1, Config.SpamCount do
        pcall(function()
            hrp.CFrame = targetCF
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            ForceNetworkSync(hrp)
        end)
        task.wait(Config.AntiRollbackDelay)
    end

    -- Reset du Humanoid pour solidifier
    pcall(function()
        humanoid:ChangeState(Enum.HumanoidStateType.Running)
        task.wait(0.005)
        humanoid:ChangeState(Enum.HumanoidStateType.Seated)
        task.wait(0.005)
        humanoid:ChangeState(Enum.HumanoidStateType.Running)
    end)

    -- Dernière application
    hrp.CFrame = targetCF
    hrp.AssemblyLinearVelocity = Vector3.zero
    hrp.AssemblyAngularVelocity = Vector3.zero
    ForceNetworkSync(hrp)

    -- Désactiver le No Clip
    task.wait(0.01)
    SetNoclip(char, false)

    -- Dernière synchro
    task.wait(0.02)
    pcall(function()
        hrp:SetNetworkOwner(nil)
        task.wait(0.01)
        hrp:SetNetworkOwner(LP)
    end)

    return true
end

-- === FONCTION PRINCIPALE : INSTANT STEAL + TP ===
local function InstantStealAndTP()
    if IsStealing then return end
    IsStealing = true

    if StatusLabel then
        StatusLabel.Text = "🔄 Vol en cours..."
        StatusLabel.TextColor3 = Color3.fromRGB(255, 200, 50)
    end

    -- 1. Trouver la cible
    local target, dist = GetClosestPlayerWithBrainrot()
    if not target then
        if StatusLabel then
            StatusLabel.Text = "❌ Aucune cible avec brainrot"
            StatusLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
            task.delay(1.5, function()
                if StatusLabel then StatusLabel.Text = "🔴 En attente..." end
            end)
        end
        IsStealing = false
        return
    end

    if dist > Config.StealRange then
        if StatusLabel then
            StatusLabel.Text = "❌ Cible trop loin (" .. math.floor(dist) .. "m)"
            StatusLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
            task.delay(1.5, function()
                if StatusLabel then StatusLabel.Text = "🔴 En attente..." end
            end)
        end
        IsStealing = false
        return
    end

    -- 2. Récupérer la cible
    local targetChar = target.Character
    local targetRoot = GetRoot(target)
    local myRoot = GetRoot(LP)
    if not targetChar or not targetRoot or not myRoot then
        IsStealing = false
        return
    end

    -- 3. TP devant la cible (zone de détection)
    local look = targetRoot.CFrame.LookVector
    local tpPos = targetRoot.Position + look * 2.5 + Vector3.new(0, 2.0, 0)
    myRoot.CFrame = CFrame.new(tpPos, targetRoot.Position)
    task.wait(0.02)

    -- 4. Spam du prompt pour voler
    local prompts = {}
    for _, desc in ipairs(targetChar:GetDescendants()) do
        if desc:IsA("ProximityPrompt") and desc.Enabled then
            table.insert(prompts, desc)
        end
    end

    if #prompts > 0 then
        for i = 1, Config.SpamPromptCount do
            for _, prompt in ipairs(prompts) do
                pcall(function()
                    prompt:InputHoldBegin()
                    task.wait(0.002)
                    prompt:InputHoldEnd()
                end)
            end
            task.wait(Config.PromptDelay)
        end
    else
        -- Tentative avec un RemoteEvent
        for _, child in ipairs(targetChar:GetDescendants()) do
            if child:IsA("RemoteEvent") and (child.Name:lower():find("steal") or child.Name:lower():find("take")) then
                pcall(function() child:FireServer(target) end)
                break
            end
        end
    end

    -- 5. TP à la base (avec le brainrot)
    task.wait(0.05)
    local tpSuccess = TeleportToBase()

    -- 6. Feedback
    if tpSuccess then
        if StatusLabel then
            StatusLabel.Text = "✅ Vol + TP réussi !"
            StatusLabel.TextColor3 = Color3.fromRGB(100, 255, 100)
        end
        print("[popcuru] ✅ Instant steal + TP base réussi !")
    else
        if StatusLabel then
            StatusLabel.Text = "⚠️ Vol réussi, TP base échoué"
            StatusLabel.TextColor3 = Color3.fromRGB(255, 200, 50)
        end
        print("[popcuru] ⚠️ Vol réussi mais TP base échoué")
    end

    task.delay(2, function()
        if StatusLabel then
            StatusLabel.Text = "🔴 En attente..."
            StatusLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
        end
    end)

    IsStealing = false
end

-- === INTERFACE MOBILE ===
local function CreateGUI()
    local gui = Instance.new("ScreenGui")
    gui.Name = "popcuru_InstantSteal"
    gui.Parent = LP:WaitForChild("PlayerGui")
    gui.ResetOnSpawn = false

    local frame = Instance.new("Frame")
    frame.Parent = gui
    frame.Size = UDim2.new(0, 340, 0, 150)
    frame.Position = UDim2.new(0.5, -170, 0.82, 0)
    frame.BackgroundColor3 = Color3.fromRGB(10, 10, 22)
    frame.BackgroundTransparency = 0.15
    frame.BorderSizePixel = 0
    frame.Active = true
    frame.Draggable = true

    local corner = Instance.new("UICorner")
    corner.Parent = frame
    corner.CornerRadius = UDim.new(0, 14)

    local stroke = Instance.new("UIStroke")
    stroke.Parent = frame
    stroke.Color = Color3.fromRGB(255, 200, 100)
    stroke.Thickness = 1.5
    stroke.Transparency = 0.3

    local title = Instance.new("TextLabel")
    title.Parent = frame
    title.Size = UDim2.new(1, 0, 0, 28)
    title.Position = UDim2.new(0, 0, 0, 0)
    title.BackgroundTransparency = 1
    title.Text = "⚡ popcuru - Instant Steal"
    title.TextColor3 = Color3.fromRGB(255, 200, 100)
    title.TextScaled = true
    title.Font = Enum.Font.GothamBlack

    local info = Instance.new("TextLabel")
    info.Parent = frame
    info.Size = UDim2.new(1, 0, 0, 20)
    info.Position = UDim2.new(0, 0, 0, 32)
    info.BackgroundTransparency = 1
    info.Text = "🟢 Bouton = Voler + TP à la base"
    info.TextColor3 = Color3.fromRGB(180, 180, 200)
    info.TextScaled = true
    info.Font = Enum.Font.GothamMedium

    local status = Instance.new("TextLabel")
    status.Parent = frame
    status.Size = UDim2.new(1, 0, 0, 24)
    status.Position = UDim2.new(0, 0, 0, 56)
    status.BackgroundTransparency = 1
    status.Text = "🔴 En attente..."
    status.TextColor3 = Color3.fromRGB(255, 100, 100)
    status.TextScaled = true
    status.Font = Enum.Font.GothamBold
    status.Name = "StatusLabel"
    StatusLabel = status

    -- GROS BOUTON UNIQUE
    local stealBtn = Instance.new("TextButton")
    stealBtn.Parent = frame
    stealBtn.Size = UDim2.new(0.8, 0, 0, 50)
    stealBtn.Position = UDim2.new(0.1, 0, 0.85, 0)
    stealBtn.BackgroundColor3 = Color3.fromRGB(255, 150, 50)
    stealBtn.BorderSizePixel = 0
    stealBtn.Text = "🚀 STEAL & TP"
    stealBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    stealBtn.Font = Enum.Font.GothamBold
    stealBtn.TextSize = 20
    stealBtn.AutoButtonColor = true
    local cornerBtn = Instance.new("UICorner")
    cornerBtn.Parent = stealBtn
    cornerBtn.CornerRadius = UDim.new(0, 12)
    local strokeBtn = Instance.new("UIStroke")
    strokeBtn.Parent = stealBtn
    strokeBtn.Color = Color3.fromRGB(255, 200, 100)
    strokeBtn.Thickness = 1.5

    stealBtn.MouseButton1Click:Connect(InstantStealAndTP)

    return gui
end

-- === LANCEMENT ===
print("==========================================")
print("⚡ popcuru - Instant Steal Mobile")
print("   🚀 Un seul bouton : Vol + TP base")
print("   🛡️ Anti-rollback / No clip / Sans lag")
print("==========================================")

CreateGUI()

while task.wait(60) do end
