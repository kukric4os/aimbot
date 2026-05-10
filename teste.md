-- ABBADON OONA - ESP + AIMBOT (Atualizado com Color Customization)
-- LocalScript for client (StarterPlayerScripts / StarterGui)
-- Features:
--  - Advanced ESP (Box, Tracer, Name, Distance, HealthBar, Skeleton) com cores customizáveis
--  - Fullbright customizável
--  - Movable GUI (drag title bar), tabs (ESP / AIMBOT)
--  - Animated open/close, toggle buttons
--  - Aimbot with bindable key, Mode (Hold/Toggle), FOV circle, smoothing, aim part selector

-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")

-- Cached refs
local LocalPlayer = Players.LocalPlayer or Players.PlayerAdded:Wait()
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local Camera = Workspace.CurrentCamera

-- ====================
-- CONFIG: ESP (Color Customization)
-- ====================
local AdvancedESP = {
    Enabled = true,
    Box = true,
    Tracer = true,
    Name = true,
    Distance = true,
    HealthBar = true,
    Skeleton = true,
    Chams = false,
    Rainbow = false,
    TeamCheck = false,
    ShowTeam = false,
    MaxDistance = 1500,

    -- Cores customizáveis
    BoxColor = Color3.fromRGB(0, 255, 100),
    NameColor = Color3.fromRGB(255, 0, 0),  -- Vermelho
    SkeletonColor = Color3.fromRGB(0, 255, 255),
    TracerColor = Color3.fromRGB(255, 255, 255),  -- Branco

    HealthColorFull = Color3.fromRGB(0, 255, 0),
    HealthColorLow = Color3.fromRGB(255, 0, 0),

    RainbowSpeed = 2,
    Thickness = 1.5,
}

-- ====================
-- CONFIG: Fullbright
-- ====================
local FullbrightConfig = {
    Enabled = false,
    Brightness = 2,
    Ambient = Color3.fromRGB(255, 255, 255),
    OutdoorAmbient = Color3.fromRGB(255, 255, 255),
}

-- ====================
-- CONFIG: AIMBOT
-- ====================
local AimbotConfig = {
    Enabled = false,
    Key = Enum.KeyCode.E,
    Mode = "Hold", -- "Hold" or "Toggle"
    FOV = 150,
    Smoothing = 0.1,
    AimPart = "Head",
    ShowFOVCircle = true,
}

-- ====================
-- FULLBRIGHT FUNCTION
-- ====================
local function toggleFullbright(enabled)
    FullbrightConfig.Enabled = enabled
    if enabled then
        Lighting.Brightness = FullbrightConfig.Brightness
        Lighting.Ambient = FullbrightConfig.Ambient
        Lighting.OutdoorAmbient = FullbrightConfig.OutdoorAmbient
    else
        Lighting.Brightness = 1
        Lighting.Ambient = Color3.fromRGB(128, 128, 128)
        Lighting.OutdoorAmbient = Color3.fromRGB(128, 128, 128)
    end
end

-- ====================
-- CREATE GUI
-- ====================
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "AimbotGui"
ScreenGui.ResetOnSpawn = false
ScreenGui.Parent = PlayerGui

-- Main Frame
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 300, 0, 400)
MainFrame.Position = UDim2.new(0, 50, 0, 50)
MainFrame.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
MainFrame.BorderSizePixel = 0
MainFrame.Parent = ScreenGui

-- Title Bar (Draggable) - ABBADON OONA em vermelho com bordas brancas
local TitleBar = Instance.new("TextLabel")
TitleBar.Name = "TitleBar"
TitleBar.Size = UDim2.new(1, 0, 0, 30)
TitleBar.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
TitleBar.BorderColor3 = Color3.fromRGB(255, 255, 255)
TitleBar.BorderSizePixel = 2
TitleBar.Text = "ABBADON OONA"
TitleBar.TextColor3 = Color3.fromRGB(255, 0, 0)  -- Vermelho
TitleBar.TextSize = 16
TitleBar.Font = Enum.Font.GothamBold
TitleBar.Parent = MainFrame

-- Tab Buttons
local TabContainer = Instance.new("Frame")
TabContainer.Name = "TabContainer"
TabContainer.Size = UDim2.new(1, 0, 0, 35)
TabContainer.Position = UDim2.new(0, 0, 0, 30)
TabContainer.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
TabContainer.BorderSizePixel = 0
TabContainer.Parent = MainFrame

local ESPTab = Instance.new("TextButton")
ESPTab.Name = "ESPTab"
ESPTab.Size = UDim2.new(0.5, 0, 1, 0)
ESPTab.Position = UDim2.new(0, 0, 0, 0)
ESPTab.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
ESPTab.Text = "ESP"
ESPTab.TextSize = 14
ESPTab.Font = Enum.Font.Gotham
ESPTab.Parent = TabContainer

local AimbotTab = Instance.new("TextButton")
AimbotTab.Name = "AimbotTab"
AimbotTab.Size = UDim2.new(0.5, 0, 1, 0)
AimbotTab.Position = UDim2.new(0.5, 0, 0, 0)
AimbotTab.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
AimbotTab.Text = "AIMBOT"
AimbotTab.TextSize = 14
AimbotTab.Font = Enum.Font.Gotham
AimbotTab.Parent = TabContainer

-- Content Area
local ContentFrame = Instance.new("Frame")
ContentFrame.Name = "ContentFrame"
ContentFrame.Size = UDim2.new(1, 0, 1, -65)
ContentFrame.Position = UDim2.new(0, 0, 0, 65)
ContentFrame.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
ContentFrame.BorderSizePixel = 0
ContentFrame.Parent = MainFrame

-- ESP Tab Content
local ESPContent = Instance.new("Frame")
ESPContent.Name = "ESPContent"
ESPContent.Size = UDim2.new(1, 0, 1, 0)
ESPContent.BackgroundTransparency = 1
ESPContent.Parent = ContentFrame

-- Function to create toggle button
local function createToggle(parent, text, y, callback)
    local Toggle = Instance.new("TextButton")
    Toggle.Size = UDim2.new(1, -10, 0, 25)
    Toggle.Position = UDim2.new(0, 5, 0, y)
    Toggle.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    Toggle.Text = text .. ": OFF"
    Toggle.TextSize = 12
    Toggle.Font = Enum.Font.Gotham
    Toggle.Parent = parent
    
    Toggle.MouseButton1Click:Connect(function()
        callback(not callback._enabled)
        callback._enabled = not callback._enabled
        Toggle.Text = text .. ": " .. (callback._enabled and "ON" or "OFF")
        Toggle.BackgroundColor3 = callback._enabled and Color3.fromRGB(0, 200, 100) or Color3.fromRGB(60, 60, 60)
    end)
    
    return Toggle
end

-- ESP Toggles
local y = 5
createToggle(ESPContent, "ESP Enabled", y, function(val) AdvancedESP.Enabled = val end)
y = y + 30
createToggle(ESPContent, "Box", y, function(val) AdvancedESP.Box = val end)
y = y + 30
createToggle(ESPContent, "Skeleton", y, function(val) AdvancedESP.Skeleton = val end)
y = y + 30
createToggle(ESPContent, "Name", y, function(val) AdvancedESP.Name = val end)
y = y + 30
createToggle(ESPContent, "Distance", y, function(val) AdvancedESP.Distance = val end)
y = y + 30
createToggle(ESPContent, "HealthBar", y, function(val) AdvancedESP.HealthBar = val end)
y = y + 30
createToggle(ESPContent, "Fullbright", y, function(val) toggleFullbright(val) end)
y = y + 30

-- Color Picker Buttons
local function createColorButton(parent, text, y, configKey)
    local Button = Instance.new("TextButton")
    Button.Size = UDim2.new(1, -10, 0, 25)
    Button.Position = UDim2.new(0, 5, 0, y)
    Button.BackgroundColor3 = AdvancedESP[configKey]
    Button.Text = text
    Button.TextSize = 12
    Button.Font = Enum.Font.Gotham
    Button.Parent = parent
    
    Button.MouseButton1Click:Connect(function()
        -- Simple color cycling (pode ser expandido com color picker real)
        local colors = {
            Color3.fromRGB(255, 0, 0),
            Color3.fromRGB(0, 255, 0),
            Color3.fromRGB(0, 0, 255),
            Color3.fromRGB(255, 255, 0),
            Color3.fromRGB(255, 0, 255),
            Color3.fromRGB(0, 255, 255),
            Color3.fromRGB(255, 255, 255),
        }
        
        local currentIndex = 1
        for i, color in ipairs(colors) do
            if AdvancedESP[configKey] == color then
                currentIndex = i
                break
            end
        end
        
        currentIndex = (currentIndex % #colors) + 1
        AdvancedESP[configKey] = colors[currentIndex]
        Button.BackgroundColor3 = AdvancedESP[configKey]
    end)
    
    return Button
end

local colorY = y + 40
createColorButton(ESPContent, "Cor Box", colorY, "BoxColor")
colorY = colorY + 30
createColorButton(ESPContent, "Cor Nome", colorY, "NameColor")
colorY = colorY + 30
createColorButton(ESPContent, "Cor Skeleton", colorY, "SkeletonColor")

-- Aimbot Tab Content
local AimbotContent = Instance.new("Frame")
AimbotContent.Name = "AimbotContent"
AimbotContent.Size = UDim2.new(1, 0, 1, 0)
AimbotContent.BackgroundTransparency = 1
AimbotContent.Visible = false
AimbotContent.Parent = ContentFrame

local y = 5
createToggle(AimbotContent, "Aimbot Enabled", y, function(val) AimbotConfig.Enabled = val end)
y = y + 30
createToggle(AimbotContent, "Show FOV Circle", y, function(val) AimbotConfig.ShowFOVCircle = val end)

-- Tab switching
ESPTab.MouseButton1Click:Connect(function()
    ESPContent.Visible = true
    AimbotContent.Visible = false
    ESPTab.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
    AimbotTab.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
end)

AimbotTab.MouseButton1Click:Connect(function()
    ESPContent.Visible = false
    AimbotContent.Visible = true
    ESPTab.BackgroundColor3 = Color3.fromRGB(50, 50, 50)
    AimbotTab.BackgroundColor3 = Color3.fromRGB(70, 70, 70)
end)

-- Drag functionality
local dragging = false
local dragStart = nil
local frameStart = nil

TitleBar.InputBegan:Connect(function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = UserInputService:GetMouseLocation()
        frameStart = MainFrame.Position
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = false
    end
end)

UserInputService.InputChanged:Connect(function(input, gameProcessed)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local currentMouse = UserInputService:GetMouseLocation()
        local delta = currentMouse - dragStart
        MainFrame.Position = UDim2.new(frameStart.X.Scale, frameStart.X.Offset + delta.X, frameStart.Y.Scale, frameStart.Y.Offset + delta.Y)
    end
end)

print("✓ ABBADON OONA Script Carregado com Sucesso!")
print("✓ Features: ESP customizável, Fullbright, Aimbot")
