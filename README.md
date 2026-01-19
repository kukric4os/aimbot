-- =============================================
-- SIMPLE AIMBOT + MENU GUI (Glory Edition Style)
-- Tecla T = abre/fecha o menu
-- Botão Direito do Mouse = Ativa o aimbot (enquanto segurar)
-- =============================================

local Players          = game:GetService("Players")
local RunService       = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService     = game:GetService("TweenService")
local CoreGui          = game:GetService("CoreGui")
local Camera           = workspace.CurrentCamera

local LocalPlayer = Players.LocalPlayer

-- =============================================
-- CONFIGURAÇÕES DO AIMBOT
-- =============================================
local AimbotSettings = {
    Enabled       = false,
    TeamCheck     = true,
    VisibleCheck  = true,
    FOV           = 180,
    Smoothness    = 0.15,      -- 0.1 = bem rápido   |   0.3 = bem suave
    TargetPart    = "Head",    -- Head / HumanoidRootPart / UpperTorso ...
    ToggleKey     = Enum.UserInputType.MouseButton2,   -- Botão direito
}

-- =============================================
-- FOV CIRCLE (visual bonito)
-- =============================================
local fovCircle = Drawing.new("Circle")
fovCircle.Thickness   = 2
fovCircle.NumSides    = 100
fovCircle.Radius      = AimbotSettings.FOV
fovCircle.Color       = Color3.fromRGB(255, 0, 255)
fovCircle.Transparency = 0.9
fovCircle.Filled      = false
fovCircle.Visible     = true

RunService.RenderStepped:Connect(function()
    fovCircle.Position = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)
end)

-- =============================================
-- Encontra o alvo mais próximo
-- =============================================
local function GetClosestPlayer()
    local closestPlayer = nil
    local closestDist = AimbotSettings.FOV

    for _, player in ipairs(Players:GetPlayers()) do
        if player == LocalPlayer then continue end
        if not player.Character then continue end
        
        if AimbotSettings.TeamCheck and player.Team == LocalPlayer.Team then continue end
        
        local character = player.Character
        local targetPart = character:FindFirstChild(AimbotSettings.TargetPart) or character:FindFirstChild("HumanoidRootPart")
        if not targetPart then continue end
        
        -- Visible check (raycast)
        if AimbotSettings.VisibleCheck then
            local rayOrigin = Camera.CFrame.Position
            local rayDir = (targetPart.Position - rayOrigin).Unit * 5000
            local rayParams = RaycastParams.new()
            rayParams.FilterDescendantsInstances = {LocalPlayer.Character}
            rayParams.FilterType = Enum.RaycastFilterType.Blacklist
            
            local result = workspace:Raycast(rayOrigin, rayDir, rayParams)
            if result and not result.Instance:IsDescendantOf(character) then
                continue
            end
        end
        
        local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
        if not onScreen then continue end
        
        local dist = (Vector2.new(screenPos.X, screenPos.Y) - Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y/2)).Magnitude
        
        if dist < closestDist then
            closestDist = dist
            closestPlayer = player
        end
    end
    
    return closestPlayer
end

-- =============================================
-- Lógica do aimbot (mira suave)
-- =============================================
local aiming = false

UserInputService.InputBegan:Connect(function(input)
    if input.UserInputType == AimbotSettings.ToggleKey then
        aiming = true
    end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == AimbotSettings.ToggleKey then
        aiming = false
    end
end)

RunService.RenderStepped:Connect(function()
    if not AimbotSettings.Enabled or not aiming then
        return
    end

    local target = GetClosestPlayer()
    if target and target.Character then
        local part = target.Character:FindFirstChild(AimbotSettings.TargetPart) or target.Character:FindFirstChild("HumanoidRootPart")
        if part then
            local targetPos = part.Position
            local direction = (targetPos - Camera.CFrame.Position).Unit
            local newCFrame = CFrame.lookAt(Camera.CFrame.Position, Camera.CFrame.Position + direction)
            
            Camera.CFrame = Camera.CFrame:Lerp(newCFrame, AimbotSettings.Smoothness)
        end
    end
end)

-- =============================================
--             MENU GUI (tecla T)
-- =============================================

if CoreGui:FindFirstChild("AimbotMenu") then
    CoreGui.AimbotMenu:Destroy()
end

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AimbotMenu"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = CoreGui

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 260, 0, 340)
MainFrame.Position = UDim2.new(0.5, -130, 0.5, -170)
MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
MainFrame.BorderSizePixel = 0
MainFrame.BackgroundTransparency = 1
MainFrame.Visible = false
MainFrame.Parent = ScreenGui

-- Título
local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 45)
Title.BackgroundTransparency = 1
Title.Text = "Glory Aimbot • 2026"
Title.TextColor3 = Color3.fromRGB(0, 255, 140)
Title.TextSize = 24
Title.Font = Enum.Font.SourceSansBold
Title.Parent = MainFrame

-- =============================================
-- Função Toggle
-- =============================================
local function CreateToggle(name, default, callback)
    local current = default
    
    local container = Instance.new("Frame")
    container.Size = UDim2.new(1, -30, 0, 42)
    container.BackgroundTransparency = 1
    container.Parent = MainFrame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.65, 0, 1, 0)
    label.BackgroundTransparency = 1
    label.Text = name
    label.TextColor3 = Color3.fromRGB(220, 220, 220)
    label.TextSize = 17
    label.Font = Enum.Font.SourceSans
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = container

    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0.28, 0, 0.75, 0)
    button.Position = UDim2.new(0.70, 0, 0.125, 0)
    button.BackgroundColor3 = current and Color3.fromRGB(0, 200, 80) or Color3.fromRGB(180, 0, 0)
    button.Text = current and "ON" or "OFF"
    button.TextColor3 = Color3.new(1,1,1)
    button.Font = Enum.Font.SourceSansBold
    button.TextSize = 15
    button.BorderSizePixel = 0
    button.Parent = container

    button.MouseButton1Click:Connect(function()
        current = not current
        button.BackgroundColor3 = current and Color3.fromRGB(0, 200, 80) or Color3.fromRGB(180, 0, 0)
        button.Text = current and "ON" or "OFF"
        callback(current)
    end)

    return container
end

-- Posição dos toggles
local yOffset = 60
local spacing = 45

-- Toggles do aimbot
local toggles = {
    {
        name = "Aimbot Enabled",
        value = AimbotSettings.Enabled,
        callback = function(v) AimbotSettings.Enabled = v end
    },
    {
        name = "Team Check",
        value = AimbotSettings.TeamCheck,
        callback = function(v) AimbotSettings.TeamCheck = v end
    },
    {
        name = "Visible Check",
        value = AimbotSettings.VisibleCheck,
        callback = function(v) AimbotSettings.VisibleCheck = v end
    },
}

for _, t in ipairs(toggles) do
    local tog = CreateToggle(t.name, t.value, t.callback)
    tog.Position = UDim2.new(0, 15, 0, yOffset)
    yOffset = yOffset + spacing
end

-- Botão Fechar
local CloseBtn = Instance.new("TextButton")
CloseBtn.Size = UDim2.new(0, 36, 0, 36)
CloseBtn.Position = UDim2.new(1, -42, 0, 5)
CloseBtn.BackgroundColor3 = Color3.fromRGB(200, 40, 40)
CloseBtn.Text = "X"
CloseBtn.TextColor3 = Color3.new(1,1,1)
CloseBtn.Font = Enum.Font.SourceSansBold
CloseBtn.TextSize = 22
CloseBtn.BorderSizePixel = 0
CloseBtn.Parent = MainFrame

-- =============================================
-- Animação de abrir / fechar
-- =============================================
local openTweenInfo  = TweenInfo.new(0.45, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
local closeTweenInfo = TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.In)

local guiOpen = false

local function OpenMenu()
    MainFrame.Visible = true
    MainFrame.BackgroundTransparency = 1
    MainFrame.Size = UDim2.new(0, 220, 0, 280)
    
    TweenService:Create(MainFrame, openTweenInfo, {
        BackgroundTransparency = 0,
        Size = UDim2.new(0, 260, 0, 340)
    }):Play()
end

local function CloseMenu()
    local tween = TweenService:Create(MainFrame, closeTweenInfo, {
        BackgroundTransparency = 1,
        Size = UDim2.new(0, 220, 0, 280)
    })
    tween:Play()
    tween.Completed:Connect(function()
        MainFrame.Visible = false
    end)
end

CloseBtn.MouseButton1Click:Connect(function()
    guiOpen = false
    CloseMenu()
end)

-- Tecla T abre/fecha
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.T then
        guiOpen = not guiOpen
        if guiOpen then
            OpenMenu()
        else
            CloseMenu()
        end
    end
end)

print("Aimbot com MENU GUI carregado! Pressione T para abrir")
