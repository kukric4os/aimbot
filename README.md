-- ABBADON OONA - ESP + AIMBOT (Enhanced UI with Sliders)
-- LocalScript for client (StarterPlayerScripts / StarterGui)
-- Features:
--  - Advanced ESP (Box, Tracer, Name, Distance, HealthBar, Skeleton)
--  - Color picker sliders for ESP Box, Skeleton, Name colors
--  - Fullbright option in MISC tab
--  - FOV slider 0-120 in MISC tab
--  - Zoom with bindable key in MISC tab
--  - Movable GUI (drag title bar), tabs (ESP / COLORS / AIMBOT / MISC)
--  - Animated open/close, toggle buttons

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
-- CONFIG: ESP (default colors)
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

    BoxColor = Color3.fromRGB(0, 255, 100),
    NameColor = Color3.fromRGB(255, 255, 255),
    SkeletonColor = Color3.fromRGB(0, 255, 255),

    HealthColorFull = Color3.fromRGB(0, 255, 0),
    HealthColorLow = Color3.fromRGB(255, 0, 0),

    RainbowSpeed = 2,
    Thickness = 2,
    Transparency = 0.7,
    TracerOrigin = "Bottom"
}

-- ====================
-- CONFIG: FULLBRIGHT
-- ====================
local FullbrightConfig = {
    Enabled = false,
    OriginalBrightness = Lighting.Brightness,
    OriginalAmbient = Lighting.Ambient,
    OriginalOutdoorAmbient = Lighting.OutdoorAmbient
}

-- ====================
-- CONFIG: ZOOM
-- ====================
local ZoomConfig = {
    Enabled = false,
    Bind = Enum.KeyCode.Z,
    BindName = "Z",
    ZoomLevel = 50
}

-- ====================
-- CONFIG: AIMBOT
-- ====================
local Aimbot = {
    Enabled = false,
    Bind = Enum.KeyCode.E,
    BindName = "E",
    Mode = "Hold",
    FOV = 0,
    ShowFOV = false,
    Smoothing = 0.12,
    Prioritize = "Distance",
    AimPart = "Head",
    VisibleCheck = true
}

-- ====================
-- State & helpers
-- ====================
local UseDrawing = pcall(function() return Drawing end)
local ESP_Containers = {}
local RainbowTime = 0

local listeningForBind = false
local listeningForZoomBind = false
local keyDown = false
local toggled = false

local fovCircle = nil
local fovGui = nil
local zoomActive = false

local function clamp(v, a, b) return math.clamp(v, a, b) end

local function safeSetVisible(obj, visible)
    if not obj then return end
    pcall(function()
        if obj.Visible ~= nil then obj.Visible = visible end
        if obj.Enabled ~= nil then obj.Enabled = visible end
    end)
end

local function safeRemove(obj)
    if not obj then return end
    pcall(function()
        local t = typeof(obj)
        if t == "userdata" or t == "Drawing" then
            if obj.Visible ~= nil then obj.Visible = false end
            if obj.Remove then pcall(obj.Remove, obj) end
            if obj.Destroy then pcall(obj.Destroy, obj) end
        else
            if obj.Destroy then obj:Destroy() end
        end
    end)
end

local function getRoot(character)
    if not character then return nil end
    return character:FindFirstChild("HumanoidRootPart") or character:FindFirstChild("UpperTorso") or character:FindFirstChild("Torso")
end

local function isValidTarget(plr)
    if not plr or plr == LocalPlayer then return false end
    local char = plr.Character
    if not char or not char.Parent then return false end
    local root = getRoot(char)
    if not root then return false end
    local hum = char:FindFirstChildWhichIsA("Humanoid")
    if not hum or hum.Health <= 0 then return false end
    if AdvancedESP.TeamCheck and (not AdvancedESP.ShowTeam) and plr.Team == LocalPlayer.Team then return false end
    local cam = Camera or Workspace.CurrentCamera
    if not cam then return false end
    local distance = (root.Position - cam.CFrame.Position).Magnitude
    if distance > AdvancedESP.MaxDistance then return false end
    return true, char, root, hum, distance
end

local function hsvRainbow(dt)
    RainbowTime = RainbowTime + dt * AdvancedESP.RainbowSpeed
    return Color3.fromHSV((RainbowTime % 1), 1, 1)
end

local function getColorForHealth(hum)
    if AdvancedESP.Rainbow then
        return Color3.fromHSV((tick() % 5) / 5, 1, 1)
    end
    if not hum then return AdvancedESP.BoxColor end
    local maxH = (hum.MaxHealth > 0) and hum.MaxHealth or 100
    local pct = clamp(hum.Health / maxH, 0, 1)
    return AdvancedESP.HealthColorFull:Lerp(AdvancedESP.HealthColorLow, 1 - pct)
end

-- Drawing constructors
local function newLine()
    if not UseDrawing then return nil end
    local ok, l = pcall(function() return Drawing.new("Line") end)
    if not ok or not l then return nil end
    pcall(function()
        if l.Thickness ~= nil then l.Thickness = AdvancedESP.Thickness end
        if l.Transparency ~= nil then l.Transparency = AdvancedESP.Transparency end
        if l.Color ~= nil then l.Color = AdvancedESP.BoxColor end
        l.Visible = false
    end)
    return l
end

local function newText()
    if not UseDrawing then return nil end
    local ok, t = pcall(function() return Drawing.new("Text") end)
    if not ok or not t then return nil end
    pcall(function()
        t.Size = 14
        t.Color = AdvancedESP.NameColor
        t.Center = true
        t.Outline = true
        t.Visible = false
    end)
    return t
end

local function createBillboardForCharacter(plr)
    local char = plr.Character
    if not char then return nil end
    local head = char:FindFirstChild("Head")
    if not head then return nil end
    local bill = Instance.new("BillboardGui")
    bill.Name = "ESP_Billboard"
    bill.Adornee = head
    bill.Size = UDim2.new(0, 200, 0, 64)
    bill.StudsOffset = Vector3.new(0, 2.6, 0)
    bill.AlwaysOnTop = true
    bill.Parent = head

    local nameLabel = Instance.new("TextLabel", bill)
    nameLabel.Size = UDim2.new(1,0,0,22)
    nameLabel.Position = UDim2.new(0,0,0,0)
    nameLabel.BackgroundTransparency = 1
    nameLabel.TextScaled = true
    nameLabel.Font = Enum.Font.SourceSansBold
    nameLabel.Text = plr.Name
    nameLabel.TextColor3 = AdvancedESP.NameColor

    local distLabel = Instance.new("TextLabel", bill)
    distLabel.Size = UDim2.new(1,0,0,18)
    distLabel.Position = UDim2.new(0,0,0,22)
    distLabel.BackgroundTransparency = 1
    distLabel.TextScaled = true
    distLabel.Font = Enum.Font.SourceSans
    distLabel.Text = ""
    distLabel.TextColor3 = Color3.fromRGB(200,200,200)

    local healthBarBG = Instance.new("Frame", bill)
    healthBarBG.Size = UDim2.new(0, 6, 0, 38)
    healthBarBG.Position = UDim2.new(0, -8, 0, 6)
    healthBarBG.BackgroundColor3 = Color3.fromRGB(50,50,50)
    healthBarBG.BorderSizePixel = 0

    local healthBar = Instance.new("Frame", healthBarBG)
    healthBar.Size = UDim2.new(1,0,0,0)
    healthBar.Position = UDim2.new(0,0,1,0)
    healthBar.AnchorPoint = Vector2.new(0,1)
    healthBar.BackgroundColor3 = AdvancedESP.HealthColorFull
    healthBar.BorderSizePixel = 0

    return {
        Billboard = bill,
        NameLabel = nameLabel,
        DistLabel = distLabel,
        HealthBarFrame = healthBar,
        HealthBarBG = healthBarBG
    }
end

-- Skeleton helper
local function getSkeletonPairs(char)
    if not char then return {} end
    local pairsOut = {}
    local function pos(name)
        local p = char:FindFirstChild(name)
        if p and p:IsA("BasePart") then return p.Position end
        return nil
    end

    local head = pos("Head")
    local upperTorso = pos("UpperTorso")
    local lowerTorso = pos("LowerTorso")
    local root = pos("HumanoidRootPart") or lowerTorso
    local lUpperArm = pos("LeftUpperArm")
    local lLowerArm = pos("LeftLowerArm")
    local lHand = pos("LeftHand")
    local rUpperArm = pos("RightUpperArm")
    local rLowerArm = pos("RightLowerArm")
    local rHand = pos("RightHand")
    local lUpperLeg = pos("LeftUpperLeg")
    local lLowerLeg = pos("LeftLowerLeg")
    local lFoot = pos("LeftFoot")
    local rUpperLeg = pos("RightUpperLeg")
    local rLowerLeg = pos("RightLowerLeg")
    local rFoot = pos("RightFoot")

    if head and upperTorso and lowerTorso then
        local function add(a, b) if a and b then table.insert(pairsOut, {a, b}) end end
        add(head, upperTorso)
        add(upperTorso, lowerTorso)
        add(lowerTorso, root)
        add(upperTorso, lUpperArm); add(lUpperArm, lLowerArm); add(lLowerArm, lHand)
        add(upperTorso, rUpperArm); add(rUpperArm, rLowerArm); add(rLowerArm, rHand)
        add(lowerTorso, lUpperLeg); add(lUpperLeg, lLowerLeg); add(lLowerLeg, lFoot)
        add(lowerTorso, rUpperLeg); add(rUpperLeg, rLowerLeg); add(rLowerLeg, rFoot)
        return pairsOut
    end

    local torso = char:FindFirstChild("Torso")
    local headR6 = char:FindFirstChild("Head")
    local leftArm = char:FindFirstChild("Left Arm") or char:FindFirstChild("LeftArm")
    local rightArm = char:FindFirstChild("Right Arm") or char:FindFirstChild("RightArm")
    local leftLeg = char:FindFirstChild("Left Leg") or char:FindFirstChild("LeftLeg")
    local rightLeg = char:FindFirstChild("Right Leg") or char:FindFirstChild("RightLeg")
    if torso and headR6 then
        if headR6 then table.insert(pairsOut, {headR6.Position, torso.Position}) end
        if leftArm and torso then table.insert(pairsOut, {leftArm.Position, torso.Position}) end
        if rightArm and torso then table.insert(pairsOut, {rightArm.Position, torso.Position}) end
        if leftLeg and torso then table.insert(pairsOut, {leftLeg.Position, torso.Position}) end
        if rightLeg and torso then table.insert(pairsOut, {rightLeg.Position, torso.Position}) end
        return pairsOut
    end

    return pairsOut
end

-- Create ESP container
local function createESP(plr)
    if not plr or plr == LocalPlayer then return end
    if ESP_Containers[plr] then return end

    local cont = {}
    ESP_Containers[plr] = cont

    cont.BoxLines = {}
    if AdvancedESP.Box then for i = 1, 8 do cont.BoxLines[i] = newLine() end end
    cont.Tracer = AdvancedESP.Tracer and newLine() or nil
    cont.HealthBar = AdvancedESP.HealthBar and newLine() or nil
    cont.NameText = AdvancedESP.Name and newText() or nil
    cont.DistText = AdvancedESP.Distance and newText() or nil

    cont.SkeletonLines = {}
    if AdvancedESP.Skeleton then for i = 1, 20 do cont.SkeletonLines[i] = newLine() end end

    plr.CharacterAdded:Connect(function(char)
        task.wait(0.25)
        if not UseDrawing then
            if cont.Billboard and cont.Billboard.Billboard and cont.Billboard.Billboard.Parent then
                pcall(function() cont.Billboard.Billboard:Destroy() end)
            end
            cont.Billboard = createBillboardForCharacter(plr)
        end
    end)

    plr.AncestryChanged:Connect(function(_, parent)
        if not parent then
            if cont.BoxLines then for _, v in ipairs(cont.BoxLines) do safeRemove(v) end end
            safeRemove(cont.Tracer); safeRemove(cont.HealthBar); safeRemove(cont.NameText); safeRemove(cont.DistText)
            if cont.Billboard and cont.Billboard.Billboard then pcall(function() cont.Billboard.Billboard:Destroy() end) end
            if cont.SkeletonLines then for _, l in ipairs(cont.SkeletonLines) do safeRemove(l) end end
            ESP_Containers[plr] = nil
        end
    end)
end

for _, plr in ipairs(Players:GetPlayers()) do if plr ~= LocalPlayer then createESP(plr) end end
Players.PlayerAdded:Connect(function(plr) if plr ~= LocalPlayer then createESP(plr) end end)

-- FOV visual
local function createFOVVisual()
    safeRemove(fovCircle)
    if fovGui and fovGui.Parent then fovGui:Destroy(); fovGui = nil end

    if Aimbot.FOV == 0 or not Aimbot.ShowFOV then return end

    if UseDrawing then
        local ok, c = pcall(function() return Drawing.new("Circle") end)
        if ok and c then
            c.Radius = Aimbot.FOV
            c.Thickness = 2
            c.Transparency = 0.8
            c.Color = Color3.fromRGB(255,255,255)
            c.Filled = false
            c.Visible = Aimbot.ShowFOV and Aimbot.FOV > 0
            fovCircle = c
            return
        end
    end
end

local function updateFOVVisual()
    if UseDrawing and fovCircle then
        local centerX = (Camera.ViewportSize.X/2)
        local centerY = (Camera.ViewportSize.Y/2)
        pcall(function()
            fovCircle.Position = Vector2.new(centerX, centerY)
            fovCircle.Radius = Aimbot.FOV
            fovCircle.Visible = Aimbot.ShowFOV and Aimbot.FOV > 0
        end)
    end
end

-- Fullbright toggle
local function setFullbright(enabled)
    if enabled then
        FullbrightConfig.OriginalBrightness = Lighting.Brightness
        FullbrightConfig.OriginalAmbient = Lighting.Ambient
        FullbrightConfig.OriginalOutdoorAmbient = Lighting.OutdoorAmbient
        Lighting.Brightness = 2
        Lighting.Ambient = Color3.fromRGB(255, 255, 255)
        Lighting.OutdoorAmbient = Color3.fromRGB(255, 255, 255)
    else
        Lighting.Brightness = FullbrightConfig.OriginalBrightness
        Lighting.Ambient = FullbrightConfig.OriginalAmbient
        Lighting.OutdoorAmbient = FullbrightConfig.OriginalOutdoorAmbient
    end
end

-- Aimbot target selection & aim
local function getAimbotTarget()
    if Aimbot.FOV == 0 then return nil end
    local cam = Camera or Workspace.CurrentCamera
    if not cam then return nil end
    local center = Vector2.new(cam.ViewportSize.X/2, cam.ViewportSize.Y/2)
    local best, bestDist = nil, math.huge
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer then
            local valid, char, root, hum, dist = isValidTarget(plr)
            if valid and char then
                local part = char:FindFirstChild(Aimbot.AimPart) or getRoot(char)
                if part then
                    local screen3 = cam:WorldToViewportPoint(part.Position)
                    local sp = Vector2.new(screen3.X, screen3.Y)
                    local d = (sp - center).Magnitude
                    if d <= Aimbot.FOV then
                        if Aimbot.VisibleCheck and not screen3 then
                            -- skip
                        else
                            if d < bestDist then
                                bestDist = d
                                best = {player = plr, char = char, part = part, screenDist = d, worldPos = part.Position}
                            end
                        end
                    end
                end
            end
        end
    end
    return best
end

local function aimAtTarget(target, delta)
    if not target or not target.worldPos then return end
    local cam = Camera or Workspace.CurrentCamera
    if not cam then return end
    local camPos = cam.CFrame.Position
    local aimPos = target.worldPos
    local targetCFrame = CFrame.new(camPos, aimPos)
    local lerpFactor = clamp(1 - Aimbot.Smoothing, 0, 1) * clamp(delta * 60, 0, 1)
    pcall(function() cam.CFrame = cam.CFrame:Lerp(targetCFrame, lerpFactor) end)
end

-- Zoom feature
local function applyZoom(active)
    zoomActive = active
    if active then
        Camera.FieldOfView = 15
    else
        Camera.FieldOfView = 70
    end
end

-- Main update loop
local function UpdateAll(delta)
    Camera = Workspace.CurrentCamera or Camera
    if not Camera then return end

    if AdvancedESP.Rainbow then hsvRainbow(delta) end
    updateFOVVisual()

    for plr, cont in pairs(ESP_Containers) do
        local valid, char, root, hum, distance = isValidTarget(plr)
        if not valid then
            if cont.BoxLines then for _, l in ipairs(cont.BoxLines) do safeSetVisible(l, false) end end
            safeSetVisible(cont.Tracer, false)
            safeSetVisible(cont.HealthBar, false)
            safeSetVisible(cont.NameText, false)
            safeSetVisible(cont.DistText, false)
            if cont.SkeletonLines then for _, l in ipairs(cont.SkeletonLines) do safeSetVisible(l, false) end end
            if cont.Billboard and cont.Billboard.Billboard then pcall(function() cont.Billboard.Billboard.Enabled = false end) end
        else
            if not (char and char.Parent) then
            else
                local head = char:FindFirstChild("Head")
                if not head then
                else
                    local head3 = head.Position + Vector3.new(0, 0.8, 0)
                    local leg3 = root.Position - Vector3.new(0, 3.5, 0)

                    local headScreen, headOnScreen = Camera:WorldToViewportPoint(head3)
                    local legScreen, legOnScreen = Camera:WorldToViewportPoint(leg3)

                    if not (headOnScreen or legOnScreen) then
                        if cont.BoxLines then for _, l in ipairs(cont.BoxLines) do safeSetVisible(l, false) end end
                        safeSetVisible(cont.Tracer, false)
                        safeSetVisible(cont.HealthBar, false)
                        safeSetVisible(cont.NameText, false)
                        safeSetVisible(cont.DistText, false)
                        if cont.SkeletonLines then for _, l in ipairs(cont.SkeletonLines) do safeSetVisible(l, false) end end
                        if cont.Billboard and cont.Billboard.Billboard then pcall(function() cont.Billboard.Billboard.Enabled = false end) end
                    else
                        local height = math.abs(headScreen.Y - legScreen.Y)
                        if height < 10 then height = 10 end
                        local width = math.clamp(height * 0.55, 8, 600)

                        local topLeft = Vector2.new(headScreen.X - width/2, headScreen.Y)
                        local topRight = Vector2.new(headScreen.X + width/2, headScreen.Y)
                        local botLeft = Vector2.new(legScreen.X - width/2, legScreen.Y)
                        local botRight = Vector2.new(legScreen.X + width/2, legScreen.Y)

                        local colorBox = AdvancedESP.Rainbow and Color3.fromHSV((tick()%1),1,1) or AdvancedESP.BoxColor
                        local colorName = AdvancedESP.NameColor
                        local colorSkel = AdvancedESP.SkeletonColor
                        if hum then colorBox = getColorForHealth(hum) end

                        -- Box (Complete)
                        if AdvancedESP.Box and cont.BoxLines then
                            local segs = {
                                {From = topLeft, To = Vector2.new(topRight.X, topLeft.Y)},
                                {From = topRight, To = Vector2.new(topRight.X, botRight.Y)},
                                {From = botRight, To = Vector2.new(botLeft.X, botRight.Y)},
                                {From = botLeft, To = Vector2.new(botLeft.X, topLeft.Y)},
                                -- Corners
                                {From = topLeft, To = Vector2.new(topLeft.X + width/6, topLeft.Y)},
                                {From = topLeft, To = Vector2.new(topLeft.X, topLeft.Y + height/6)},
                                {From = topRight, To = Vector2.new(topRight.X - width/6, topRight.Y)},
                                {From = topRight, To = Vector2.new(topRight.X, topRight.Y + height/6)}
                            }
                            for i = 1, 8 do
                                local line = cont.BoxLines[i]
                                if line then
                                    pcall(function()
                                        line.From = segs[i].From
                                        line.To = segs[i].To
                                        line.Color = colorBox
                                        line.Thickness = AdvancedESP.Thickness
                                        line.Transparency = AdvancedESP.Transparency
                                        line.Visible = AdvancedESP.Enabled
                                    end)
                                end
                            end
                        end

                        -- Tracer
                        if AdvancedESP.Tracer and cont.Tracer then
                            local fromPos
                            if AdvancedESP.TracerOrigin == "Bottom" then
                                fromPos = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y)
                            elseif AdvancedESP.TracerOrigin == "Center" then
                                local v = Camera.ViewportSize / 2
                                fromPos = Vector2.new(v.X, v.Y)
                            elseif AdvancedESP.TracerOrigin == "Mouse" then
                                fromPos = UserInputService:GetMouseLocation()
                            else
                                fromPos = Vector2.new(Camera.ViewportSize.X/2, Camera.ViewportSize.Y)
                            end
                            pcall(function()
                                cont.Tracer.From = fromPos
                                cont.Tracer.To = Vector2.new((botLeft.X + botRight.X)/2, botLeft.Y)
                                cont.Tracer.Color = AdvancedESP.Rainbow and hsvRainbow(delta) or AdvancedESP.BoxColor
                                cont.Tracer.Visible = AdvancedESP.Enabled
                            end)
                        end

                        -- HealthBar
                        if AdvancedESP.HealthBar and cont.HealthBar then
                            local maxH = (hum and hum.MaxHealth > 0) and hum.MaxHealth or 100
                            local pct = hum and math.clamp(hum.Health / maxH, 0, 1) or 0
                            local barH = height * pct
                            pcall(function()
                                cont.HealthBar.From = Vector2.new(topLeft.X - 8, topLeft.Y + height)
                                cont.HealthBar.To = Vector2.new(topLeft.X - 8, topLeft.Y + height - barH)
                                cont.HealthBar.Color = getColorForHealth(hum)
                                cont.HealthBar.Thickness = 4
                                cont.HealthBar.Visible = AdvancedESP.Enabled
                            end)
                        end

                        -- Texts
                        if UseDrawing then
                            if AdvancedESP.Name and cont.NameText then
                                pcall(function()
                                    cont.NameText.Text = plr.Name
                                    cont.NameText.Position = Vector2.new(headScreen.X, headScreen.Y - 18)
                                    cont.NameText.Color = colorName
                                    cont.NameText.Visible = AdvancedESP.Enabled
                                end)
                            end
                            if AdvancedESP.Distance and cont.DistText then
                                pcall(function()
                                    cont.DistText.Text = math.floor(distance) .. " studs"
                                    cont.DistText.Position = Vector2.new(headScreen.X, headScreen.Y - 34)
                                    cont.DistText.Color = Color3.fromRGB(200,200,200)
                                    cont.DistText.Visible = AdvancedESP.Enabled
                                end)
                            end
                        else
                            if cont.Billboard and cont.Billboard.Billboard then
                                pcall(function()
                                    cont.Billboard.Billboard.Enabled = AdvancedESP.Enabled
                                    if cont.Billboard.NameLabel then
                                        cont.Billboard.NameLabel.Text = plr.Name
                                        cont.Billboard.NameLabel.TextColor3 = colorName
                                    end
                                    if cont.Billboard.DistLabel then
                                        cont.Billboard.DistLabel.Text = math.floor(distance) .. " studs"
                                    end
                                    if cont.Billboard.HealthBarFrame and hum then
                                        local maxH = hum.MaxHealth > 0 and hum.MaxHealth or 100
                                        local pct = math.clamp(hum.Health / maxH, 0, 1)
                                        cont.Billboard.HealthBarFrame.Size = UDim2.new(1,0,pct * 1,0)
                                        cont.Billboard.HealthBarFrame.BackgroundColor3 = getColorForHealth(hum)
                                    end
                                end)
                            end
                        end

                        -- Skeleton
                        if AdvancedESP.Skeleton and cont.SkeletonLines then
                            local pairs3 = getSkeletonPairs(char)
                            for i = 1, #cont.SkeletonLines do
                                local line = cont.SkeletonLines[i]
                                local pair = pairs3[i]
                                if line and pair then
                                    local a3, b3 = pair[1], pair[2]
                                    local a2 = Camera:WorldToViewportPoint(a3)
                                    local b2 = Camera:WorldToViewportPoint(b3)
                                    local aOn = a2 and a2.Z > 0
                                    local bOn = b2 and b2.Z > 0
                                    pcall(function()
                                        line.From = Vector2.new(a2.X, a2.Y)
                                        line.To = Vector2.new(b2.X, b2.Y)
                                        line.Color = AdvancedESP.Rainbow and hsvRainbow(delta) or AdvancedESP.SkeletonColor
                                        line.Thickness = AdvancedESP.Thickness
                                        line.Transparency = AdvancedESP.Transparency
                                        line.Visible = AdvancedESP.Enabled and AdvancedESP.Skeleton and (aOn or bOn)
                                    end)
                                elseif line then
                                    safeSetVisible(line, false)
                                end
                            end
                        else
                            if cont.SkeletonLines then for _, l in ipairs(cont.SkeletonLines) do safeSetVisible(l, false) end end
                        end
                    end
                end
            end
        end
    end

    -- Aimbot logic
    if Aimbot.Enabled and Aimbot.FOV > 0 then
        local shouldAim = (Aimbot.Mode == "Hold" and keyDown) or (Aimbot.Mode == "Toggle" and toggled)
        if shouldAim then
            local target = getAimbotTarget()
            if target then aimAtTarget(target, delta) end
        end
    end
end

if RunService and RunService.RenderStepped then
    RunService.RenderStepped:Connect(UpdateAll)
elseif RunService and RunService.Heartbeat then
    RunService.Heartbeat:Connect(function(dt) UpdateAll(dt) end)
else
    warn("No RunService stepping available")
end

-- ====================
-- GUI (Compact with Sliders)
-- ====================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "AbbadonOonaGui"
screenGui.ResetOnSpawn = false
screenGui.Parent = PlayerGui

local targetSize = UDim2.new(0, 450, 0, 500)
local closedSize = UDim2.new(0, 0, 0, 0)

local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = closedSize
mainFrame.Position = UDim2.new(0, 20, 0, 80)
mainFrame.BackgroundColor3 = Color3.fromRGB(12,12,12)
mainFrame.BorderSizePixel = 1
mainFrame.BorderColor3 = Color3.fromRGB(50, 50, 50)
mainFrame.Active = true
mainFrame.ClipsDescendants = true
mainFrame.Parent = screenGui

Instance.new("UICorner", mainFrame).CornerRadius = UDim.new(0, 8)

-- TitleBar
local titleBar = Instance.new("Frame", mainFrame)
titleBar.Name = "TitleBar"
titleBar.Size = UDim2.new(1, 0, 0, 50)
titleBar.BackgroundTransparency = 1

local titleLabel = Instance.new("TextLabel", titleBar)
titleLabel.Name = "TitleLabel"
titleLabel.Size = UDim2.new(0.5, -10, 1, 0)
titleLabel.Position = UDim2.new(0, 10, 0, 0)
titleLabel.BackgroundTransparency = 1
titleLabel.Text = "ABBADON OONA"
titleLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
titleLabel.Font = Enum.Font.SourceSansBold
titleLabel.TextSize = 16
titleLabel.TextXAlignment = Enum.TextXAlignment.Left
titleLabel.TextStrokeTransparency = 0.3
titleLabel.TextStrokeColor3 = Color3.fromRGB(255, 255, 255)

local tabsHolder = Instance.new("Frame", titleBar)
tabsHolder.Name = "Tabs"
tabsHolder.Size = UDim2.new(0.5, -10, 1, 0)
tabsHolder.Position = UDim2.new(0.5, 0, 0, 0)
tabsHolder.BackgroundTransparency = 1

local function createTab(parent, text, pos, size)
    local btn = Instance.new("TextButton", parent)
    btn.Size = size
    btn.Position = pos
    btn.Text = text
    btn.Font = Enum.Font.SourceSans
    btn.TextSize = 10
    btn.BackgroundColor3 = Color3.fromRGB(28,28,28)
    btn.TextColor3 = Color3.new(1,1,1)
    btn.AutoButtonColor = false
    btn.BorderSizePixel = 0
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)
    return btn
end

local espTab = createTab(tabsHolder, "ESP", UDim2.new(0, 0, 0.2, 0), UDim2.new(0.24, -2, 0.6, 0))
local colorTab = createTab(tabsHolder, "Colors", UDim2.new(0.25, -1, 0.2, 0), UDim2.new(0.24, -2, 0.6, 0))
local aimTab = createTab(tabsHolder, "Aim", UDim2.new(0.5, -1, 0.2, 0), UDim2.new(0.24, -2, 0.6, 0))
local miscTab = createTab(tabsHolder, "Misc", UDim2.new(0.75, -1, 0.2, 0), UDim2.new(0.24, -2, 0.6, 0))

local contentHolder = Instance.new("Frame", mainFrame)
contentHolder.Name = "ContentHolder"
contentHolder.Size = UDim2.new(1, -16, 1, -66)
contentHolder.Position = UDim2.new(0, 8, 0, 58)
contentHolder.BackgroundTransparency = 1
contentHolder.ClipsDescendants = true

local espFrame = Instance.new("ScrollingFrame", contentHolder)
espFrame.Size = UDim2.new(1, 0, 1, 0)
espFrame.BackgroundTransparency = 1
espFrame.Visible = true
espFrame.ScrollBarThickness = 4
espFrame.CanvasSize = UDim2.new(0, 0, 0, 300)

local colorFrame = Instance.new("ScrollingFrame", contentHolder)
colorFrame.Size = UDim2.new(1, 0, 1, 0)
colorFrame.BackgroundTransparency = 1
colorFrame.Visible = false
colorFrame.ScrollBarThickness = 4
colorFrame.CanvasSize = UDim2.new(0, 0, 0, 400)

local aimFrame = Instance.new("ScrollingFrame", contentHolder)
aimFrame.Size = UDim2.new(1, 0, 1, 0)
aimFrame.BackgroundTransparency = 1
aimFrame.Visible = false
aimFrame.ScrollBarThickness = 4
aimFrame.CanvasSize = UDim2.new(0, 0, 0, 400)

local miscFrame = Instance.new("ScrollingFrame", contentHolder)
miscFrame.Size = UDim2.new(1, 0, 1, 0)
miscFrame.BackgroundTransparency = 1
miscFrame.Visible = false
miscFrame.ScrollBarThickness = 4
miscFrame.CanvasSize = UDim2.new(0, 0, 0, 350)

espTab.MouseButton1Click:Connect(function()
    espFrame.Visible = true
    colorFrame.Visible = false
    aimFrame.Visible = false
    miscFrame.Visible = false
    espTab.BackgroundColor3 = Color3.fromRGB(60,60,60)
    colorTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
    aimTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
    miscTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
end)
colorTab.MouseButton1Click:Connect(function()
    espFrame.Visible = false
    colorFrame.Visible = true
    aimFrame.Visible = false
    miscFrame.Visible = false
    espTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
    colorTab.BackgroundColor3 = Color3.fromRGB(60,60,60)
    aimTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
    miscTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
end)
aimTab.MouseButton1Click:Connect(function()
    espFrame.Visible = false
    colorFrame.Visible = false
    aimFrame.Visible = true
    miscFrame.Visible = false
    espTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
    colorTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
    aimTab.BackgroundColor3 = Color3.fromRGB(60,60,60)
    miscTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
end)
miscTab.MouseButton1Click:Connect(function()
    espFrame.Visible = false
    colorFrame.Visible = false
    aimFrame.Visible = false
    miscFrame.Visible = true
    espTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
    colorTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
    aimTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
    miscTab.BackgroundColor3 = Color3.fromRGB(60,60,60)
end)

-- ESP tab toggles
local function createToggleButtonSimple(parent, label, key, posY)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(1, -8, 0, 28)
    btn.Position = UDim2.new(0, 4, 0, posY)
    btn.BackgroundColor3 = Color3.fromRGB(28,28,28)
    btn.BorderSizePixel = 0
    btn.Text = label
    btn.Font = Enum.Font.SourceSans
    btn.TextColor3 = Color3.new(1,1,1)
    btn.TextSize = 12
    btn.AutoButtonColor = false
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)

    local indicator = Instance.new("Frame", btn)
    indicator.Size = UDim2.new(0, 36, 0, 20)
    indicator.Position = UDim2.new(1, -42, 0.5, -10)
    indicator.BackgroundColor3 = AdvancedESP[key] and Color3.fromRGB(0,170,70) or Color3.fromRGB(80,80,80)
    Instance.new("UICorner", indicator).CornerRadius = UDim.new(0.5,0)

    btn.MouseButton1Click:Connect(function()
        AdvancedESP[key] = not AdvancedESP[key]
        indicator.BackgroundColor3 = AdvancedESP[key] and Color3.fromRGB(0,170,70) or Color3.fromRGB(80,80,80)
    end)
    return btn
end

local espOptions = {
    {label="Master", key="Enabled", y=0},
    {label="Box", key="Box", y=32},
    {label="Tracer", key="Tracer", y=64},
    {label="Name", key="Name", y=96},
    {label="Distance", key="Distance", y=128},
    {label="Health Bar", key="HealthBar", y=160},
    {label="Skeleton", key="Skeleton", y=192},
    {label="Rainbow", key="Rainbow", y=224},
    {label="Team Check", key="TeamCheck", y=256}
}
for _, opt in ipairs(espOptions) do createToggleButtonSimple(espFrame, opt.label, opt.key, opt.y) end

-- Color Picker with Sliders
local function createColorSlider(parent, label, configKey, yPos)
    local container = Instance.new("Frame", parent)
    container.Size = UDim2.new(1, -8, 0, 90)
    container.Position = UDim2.new(0, 4, 0, yPos)
    container.BackgroundColor3 = Color3.fromRGB(18,18,18)
    container.BorderSizePixel = 0
    Instance.new("UICorner", container).CornerRadius = UDim.new(0, 4)

    local labelText = Instance.new("TextLabel", container)
    labelText.Size = UDim2.new(1, -8, 0, 18)
    labelText.Position = UDim2.new(0, 4, 0, 3)
    labelText.BackgroundTransparency = 1
    labelText.Text = label
    labelText.Font = Enum.Font.SourceSansBold
    labelText.TextSize = 11
    labelText.TextColor3 = Color3.new(1,1,1)
    labelText.TextXAlignment = Enum.TextXAlignment.Left

    local colorPreview = Instance.new("Frame", container)
    colorPreview.Size = UDim2.new(0, 40, 0, 40)
    colorPreview.Position = UDim2.new(0, 4, 0, 22)
    colorPreview.BackgroundColor3 = AdvancedESP[configKey]
    colorPreview.BorderSizePixel = 1
    colorPreview.BorderColor3 = Color3.fromRGB(100, 100, 100)
    Instance.new("UICorner", colorPreview).CornerRadius = UDim.new(0, 3)

    -- R Slider
    local rLabel = Instance.new("TextLabel", container)
    rLabel.Size = UDim2.new(0, 12, 0, 16)
    rLabel.Position = UDim2.new(0, 50, 0, 24)
    rLabel.BackgroundTransparency = 1
    rLabel.Text = "R"
    rLabel.Font = Enum.Font.SourceSans
    rLabel.TextSize = 10
    rLabel.TextColor3 = Color3.fromRGB(255, 100, 100)

    local rSlider = Instance.new("Frame", container)
    rSlider.Size = UDim2.new(0.5, -56, 0, 16)
    rSlider.Position = UDim2.new(0, 66, 0, 24)
    rSlider.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    rSlider.BorderSizePixel = 0
    Instance.new("UICorner", rSlider).CornerRadius = UDim.new(0, 2)

    local rBar = Instance.new("Frame", rSlider)
    rBar.Size = UDim2.new(0.5, 0, 1, 0)
    rBar.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
    rBar.BorderSizePixel = 0

    local rValue = Instance.new("TextLabel", container)
    rValue.Size = UDim2.new(0, 30, 0, 16)
    rValue.Position = UDim2.new(1, -34, 0, 24)
    rValue.BackgroundTransparency = 1
    rValue.Font = Enum.Font.SourceSans
    rValue.TextSize = 10
    rValue.TextColor3 = Color3.new(1,1,1)

    -- G Slider
    local gLabel = Instance.new("TextLabel", container)
    gLabel.Size = UDim2.new(0, 12, 0, 16)
    gLabel.Position = UDim2.new(0, 50, 0, 42)
    gLabel.BackgroundTransparency = 1
    gLabel.Text = "G"
    gLabel.Font = Enum.Font.SourceSans
    gLabel.TextSize = 10
    gLabel.TextColor3 = Color3.fromRGB(100, 255, 100)

    local gSlider = Instance.new("Frame", container)
    gSlider.Size = UDim2.new(0.5, -56, 0, 16)
    gSlider.Position = UDim2.new(0, 66, 0, 42)
    gSlider.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    gSlider.BorderSizePixel = 0
    Instance.new("UICorner", gSlider).CornerRadius = UDim.new(0, 2)

    local gBar = Instance.new("Frame", gSlider)
    gBar.Size = UDim2.new(0.5, 0, 1, 0)
    gBar.BackgroundColor3 = Color3.fromRGB(0, 255, 0)
    gBar.BorderSizePixel = 0

    local gValue = Instance.new("TextLabel", container)
    gValue.Size = UDim2.new(0, 30, 0, 16)
    gValue.Position = UDim2.new(1, -34, 0, 42)
    gValue.BackgroundTransparency = 1
    gValue.Font = Enum.Font.SourceSans
    gValue.TextSize = 10
    gValue.TextColor3 = Color3.new(1,1,1)

    -- B Slider
    local bLabel = Instance.new("TextLabel", container)
    bLabel.Size = UDim2.new(0, 12, 0, 16)
    bLabel.Position = UDim2.new(0, 50, 0, 60)
    bLabel.BackgroundTransparency = 1
    bLabel.Text = "B"
    bLabel.Font = Enum.Font.SourceSans
    bLabel.TextSize = 10
    bLabel.TextColor3 = Color3.fromRGB(100, 100, 255)

    local bSlider = Instance.new("Frame", container)
    bSlider.Size = UDim2.new(0.5, -56, 0, 16)
    bSlider.Position = UDim2.new(0, 66, 0, 60)
    bSlider.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    bSlider.BorderSizePixel = 0
    Instance.new("UICorner", bSlider).CornerRadius = UDim.new(0, 2)

    local bBar = Instance.new("Frame", bSlider)
    bBar.Size = UDim2.new(0.5, 0, 1, 0)
    bBar.BackgroundColor3 = Color3.fromRGB(0, 0, 255)
    bBar.BorderSizePixel = 0

    local bValue = Instance.new("TextLabel", container)
    bValue.Size = UDim2.new(0, 30, 0, 16)
    bValue.Position = UDim2.new(1, -34, 0, 60)
    bValue.BackgroundTransparency = 1
    bValue.Font = Enum.Font.SourceSans
    bValue.TextSize = 10
    bValue.TextColor3 = Color3.new(1,1,1)

    local function updateSliders()
        local r = math.floor(AdvancedESP[configKey].R * 255)
        local g = math.floor(AdvancedESP[configKey].G * 255)
        local b = math.floor(AdvancedESP[configKey].B * 255)
        rBar.Size = UDim2.new(r / 255, 0, 1, 0)
        gBar.Size = UDim2.new(g / 255, 0, 1, 0)
        bBar.Size = UDim2.new(b / 255, 0, 1, 0)
        rValue.Text = tostring(r)
        gValue.Text = tostring(g)
        bValue.Text = tostring(b)
        colorPreview.BackgroundColor3 = AdvancedESP[configKey]
    end

    local function sliderDrag(slider, bar, setValue)
        local dragging = false
        slider.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                dragging = true
            end
        end)
        UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                dragging = false
            end
        end)
        UserInputService.InputChanged:Connect(function(input)
            if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
                local mouse = UserInputService:GetMouseLocation()
                local sliderPos = slider.AbsolutePosition.X
                local sliderSize = slider.AbsoluteSize.X
                local relative = clamp(mouse.X - sliderPos, 0, sliderSize)
                local percent = relative / sliderSize
                setValue(math.floor(percent * 255))
                updateSliders()
            end
        end)
    end

    sliderDrag(rSlider, rBar, function(val)
        local r = val
        local g = math.floor(AdvancedESP[configKey].G * 255)
        local b = math.floor(AdvancedESP[configKey].B * 255)
        AdvancedESP[configKey] = Color3.fromRGB(r, g, b)
    end)

    sliderDrag(gSlider, gBar, function(val)
        local r = math.floor(AdvancedESP[configKey].R * 255)
        local g = val
        local b = math.floor(AdvancedESP[configKey].B * 255)
        AdvancedESP[configKey] = Color3.fromRGB(r, g, b)
    end)

    sliderDrag(bSlider, bBar, function(val)
        local r = math.floor(AdvancedESP[configKey].R * 255)
        local g = math.floor(AdvancedESP[configKey].G * 255)
        local b = val
        AdvancedESP[configKey] = Color3.fromRGB(r, g, b)
    end)

    updateSliders()
end

createColorSlider(colorFrame, "Box Color", "BoxColor", 0)
createColorSlider(colorFrame, "Skeleton Color", "SkeletonColor", 100)
createColorSlider(colorFrame, "Name Color", "NameColor", 200)

-- AIMBOT tab
local function createAimbotToggle(parent, label, getState, setState, yPos)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(1, -8, 0, 28)
    btn.Position = UDim2.new(0, 4, 0, yPos)
    btn.BackgroundColor3 = Color3.fromRGB(28,28,28)
    btn.BorderSizePixel = 0
    btn.Text = label
    btn.Font = Enum.Font.SourceSans
    btn.TextColor3 = Color3.new(1,1,1)
    btn.TextSize = 12
    btn.AutoButtonColor = false
    Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 4)

    local indicator = Instance.new("Frame", btn)
    indicator.Size = UDim2.new(0, 36, 0, 20)
    indicator.Position = UDim2.new(1, -42, 0.5, -10)
    indicator.BackgroundColor3 = getState() and Color3.fromRGB(0,170,70) or Color3.fromRGB(80,80,80)
    Instance.new("UICorner", indicator).CornerRadius = UDim.new(0.5,0)

    btn.MouseButton1Click:Connect(function()
        setState(not getState())
        indicator.BackgroundColor3 = getState() and Color3.fromRGB(0,170,70) or Color3.fromRGB(80,80,80)
    end)
    return btn
end

createAimbotToggle(aimFrame, "Aimbot Master", function() return Aimbot.Enabled end, function(v) Aimbot.Enabled = v end, 0)

local modeBtn = Instance.new("TextButton", aimFrame)
modeBtn.Size = UDim2.new(1, -8, 0, 28)
modeBtn.Position = UDim2.new(0, 4, 0, 32)
modeBtn.BackgroundColor3 = Color3.fromRGB(28,28,28)
modeBtn.BorderSizePixel = 0
modeBtn.Text = "Mode: " .. Aimbot.Mode
modeBtn.Font = Enum.Font.SourceSans
modeBtn.TextColor3 = Color3.new(1,1,1)
modeBtn.TextSize = 12
modeBtn.AutoButtonColor = false
Instance.new("UICorner", modeBtn).CornerRadius = UDim.new(0, 4)
modeBtn.MouseButton1Click:Connect(function()
    if Aimbot.Mode == "Hold" then Aimbot.Mode = "Toggle" else Aimbot.Mode = "Hold" end
    modeBtn.Text = "Mode: " .. Aimbot.Mode
end)

local bindBtn = Instance.new("TextButton", aimFrame)
bindBtn.Size = UDim2.new(1, -8, 0, 28)
bindBtn.Position = UDim2.new(0, 4, 0, 64)
bindBtn.BackgroundColor3 = Color3.fromRGB(28,28,28)
bindBtn.BorderSizePixel = 0
bindBtn.Text = "Bind: " .. Aimbot.BindName
bindBtn.Font = Enum.Font.SourceSans
bindBtn.TextColor3 = Color3.new(1,1,1)
bindBtn.TextSize = 12
bindBtn.AutoButtonColor = false
Instance.new("UICorner", bindBtn).CornerRadius = UDim.new(0, 4)
bindBtn.MouseButton1Click:Connect(function()
    listeningForBind = true
    bindBtn.Text = "Press any key..."
end)

createAimbotToggle(aimFrame, "Show FOV", function() return Aimbot.ShowFOV end, function(v) Aimbot.ShowFOV = v createFOVVisual() end, 96)

local aimPartBtn = Instance.new("TextButton", aimFrame)
aimPartBtn.Size = UDim2.new(1, -8, 0, 28)
aimPartBtn.Position = UDim2.new(0, 4, 0, 128)
aimPartBtn.BackgroundColor3 = Color3.fromRGB(28,28,28)
aimPartBtn.BorderSizePixel = 0
aimPartBtn.Text = "Aim Part: " .. Aimbot.AimPart
aimPartBtn.Font = Enum.Font.SourceSans
aimPartBtn.TextColor3 = Color3.new(1,1,1)
aimPartBtn.TextSize = 12
aimPartBtn.AutoButtonColor = false
Instance.new("UICorner", aimPartBtn).CornerRadius = UDim.new(0, 4)
aimPartBtn.MouseButton1Click:Connect(function()
    if Aimbot.AimPart == "Head" then Aimbot.AimPart = "HumanoidRootPart" else Aimbot.AimPart = "Head" end
    aimPartBtn.Text = "Aim Part: " .. Aimbot.AimPart
end)

-- Smoothing Slider
local smoothContainer = Instance.new("Frame", aimFrame)
smoothContainer.Size = UDim2.new(1, -8, 0, 40)
smoothContainer.Position = UDim2.new(0, 4, 0, 160)
smoothContainer.BackgroundColor3 = Color3.fromRGB(18,18,18)
smoothContainer.BorderSizePixel = 0
Instance.new("UICorner", smoothContainer).CornerRadius = UDim.new(0, 4)

local smoothLabel = Instance.new("TextLabel", smoothContainer)
smoothLabel.Size = UDim2.new(1, -8, 0, 16)
smoothLabel.Position = UDim2.new(0, 4, 0, 2)
smoothLabel.BackgroundTransparency = 1
smoothLabel.Text = "Smoothing: " .. string.format("%.2f", Aimbot.Smoothing)
smoothLabel.Font = Enum.Font.SourceSans
smoothLabel.TextSize = 10
smoothLabel.TextColor3 = Color3.new(1,1,1)

local smoothSlider = Instance.new("Frame", smoothContainer)
smoothSlider.Size = UDim2.new(1, -8, 0, 12)
smoothSlider.Position = UDim2.new(0, 4, 0, 20)
smoothSlider.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
smoothSlider.BorderSizePixel = 0
Instance.new("UICorner", smoothSlider).CornerRadius = UDim.new(0, 2)

local smoothBar = Instance.new("Frame", smoothSlider)
smoothBar.Size = UDim2.new(Aimbot.Smoothing / 0.95, 0, 1, 0)
smoothBar.BackgroundColor3 = Color3.fromRGB(100, 200, 255)
smoothBar.BorderSizePixel = 0

local smoothDragging = false
smoothSlider.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        smoothDragging = true
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        smoothDragging = false
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if smoothDragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local mouse = UserInputService:GetMouseLocation()
        local sliderPos = smoothSlider.AbsolutePosition.X
        local sliderSize = smoothSlider.AbsoluteSize.X
        local relative = clamp(mouse.X - sliderPos, 0, sliderSize)
        local percent = relative / sliderSize
        Aimbot.Smoothing = percent * 0.95
        smoothBar.Size = UDim2.new(percent, 0, 1, 0)
        smoothLabel.Text = "Smoothing: " .. string.format("%.2f", Aimbot.Smoothing)
    end
end)

-- MISC tab
local fullbrightBtn = createAimbotToggle(miscFrame, "Fullbright", function() return FullbrightConfig.Enabled end, function(v) FullbrightConfig.Enabled = v setFullbright(v) end, 0)

-- FOV Slider (0-120)
local fovContainer = Instance.new("Frame", miscFrame)
fovContainer.Size = UDim2.new(1, -8, 0, 40)
fovContainer.Position = UDim2.new(0, 4, 0, 32)
fovContainer.BackgroundColor3 = Color3.fromRGB(18,18,18)
fovContainer.BorderSizePixel = 0
Instance.new("UICorner", fovContainer).CornerRadius = UDim.new(0, 4)

local fovLabel = Instance.new("TextLabel", fovContainer)
fovLabel.Size = UDim2.new(1, -8, 0, 16)
fovLabel.Position = UDim2.new(0, 4, 0, 2)
fovLabel.BackgroundTransparency = 1
fovLabel.Text = "FOV: " .. tostring(Aimbot.FOV)
fovLabel.Font = Enum.Font.SourceSans
fovLabel.TextSize = 10
fovLabel.TextColor3 = Color3.new(1,1,1)

local fovSlider = Instance.new("Frame", fovContainer)
fovSlider.Size = UDim2.new(1, -8, 0, 12)
fovSlider.Position = UDim2.new(0, 4, 0, 20)
fovSlider.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
fovSlider.BorderSizePixel = 0
Instance.new("UICorner", fovSlider).CornerRadius = UDim.new(0, 2)

local fovBar = Instance.new("Frame", fovSlider)
fovBar.Size = UDim2.new(0, 0, 1, 0)
fovBar.BackgroundColor3 = Color3.fromRGB(255, 150, 100)
fovBar.BorderSizePixel = 0

local fovDragging = false
fovSlider.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        fovDragging = true
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        fovDragging = false
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if fovDragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local mouse = UserInputService:GetMouseLocation()
        local sliderPos = fovSlider.AbsolutePosition.X
        local sliderSize = fovSlider.AbsoluteSize.X
        local relative = clamp(mouse.X - sliderPos, 0, sliderSize)
        local percent = relative / sliderSize
        Aimbot.FOV = math.floor(percent * 120)
        fovBar.Size = UDim2.new(percent, 0, 1, 0)
        fovLabel.Text = "FOV: " .. tostring(Aimbot.FOV)
        createFOVVisual()
    end
end)

-- Zoom bind button
local zoomBindBtn = Instance.new("TextButton", miscFrame)
zoomBindBtn.Size = UDim2.new(1, -8, 0, 28)
zoomBindBtn.Position = UDim2.new(0, 4, 0, 80)
zoomBindBtn.BackgroundColor3 = Color3.fromRGB(28,28,28)
zoomBindBtn.BorderSizePixel = 0
zoomBindBtn.Text = "Zoom Bind: " .. ZoomConfig.BindName
zoomBindBtn.Font = Enum.Font.SourceSans
zoomBindBtn.TextColor3 = Color3.new(1,1,1)
zoomBindBtn.TextSize = 12
zoomBindBtn.AutoButtonColor = false
Instance.new("UICorner", zoomBindBtn).CornerRadius = UDim.new(0, 4)
zoomBindBtn.MouseButton1Click:Connect(function()
    listeningForZoomBind = true
    zoomBindBtn.Text = "Press any key..."
end)

-- Bind handling
local function setBind(keyCode)
    if not keyCode then return end
    Aimbot.Bind = keyCode
    Aimbot.BindName = tostring(keyCode):gsub("Enum.KeyCode.", "") or tostring(keyCode)
    bindBtn.Text = "Bind: " .. Aimbot.BindName
end

local function setZoomBind(keyCode)
    if not keyCode then return end
    ZoomConfig.Bind = keyCode
    ZoomConfig.BindName = tostring(keyCode):gsub("Enum.KeyCode.", "") or tostring(keyCode)
    zoomBindBtn.Text = "Zoom Bind: " .. ZoomConfig.BindName
end

UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if listeningForBind then
        if input.UserInputType == Enum.UserInputType.Keyboard then
            setBind(input.KeyCode)
            listeningForBind = false
        end
        return
    end
    if listeningForZoomBind then
        if input.UserInputType == Enum.UserInputType.Keyboard then
            setZoomBind(input.KeyCode)
            listeningForZoomBind = false
        end
        return
    end
    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == Aimbot.Bind then
        keyDown = true
        if Aimbot.Mode == "Toggle" then toggled = not toggled end
    end
    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == ZoomConfig.Bind then
        applyZoom(true)
    end
end)
UserInputService.InputEnded:Connect(function(input, processed)
    if processed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == Aimbot.Bind then
        keyDown = false
    end
    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == ZoomConfig.Bind then
        applyZoom(false)
    end
end)

-- Menu open/close and dragging
local function tweenObject(obj, props, info)
    info = info or TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
    local ok, t = pcall(function() return TweenService:Create(obj, info, props) end)
    if ok and t then t:Play() end
end

local menuOpen = false
local function setMenuOpen(open)
    if open == menuOpen then return end
    menuOpen = open
    if open then
        mainFrame.Visible = true
        tweenObject(mainFrame, {Size = targetSize}, TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out))
    else
        local t = TweenService:Create(mainFrame, TweenInfo.new(0.15, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Size = closedSize})
        t:Play()
        t.Completed:Connect(function() mainFrame.Visible = false end)
    end
end

UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == Enum.KeyCode.P then
        setMenuOpen(not menuOpen)
    end
    if input.KeyCode == Enum.KeyCode.Escape then setMenuOpen(false) end
end)

-- Dragging
do
    local dragging = false
    local dragStart, startPos, dragInput
    local function update(input)
        local delta = input.Position - dragStart
        mainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
    titleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = mainFrame.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)
    titleBar.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            dragInput = input
        end
    end)
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input == dragInput then update(input) end
    end)
end

-- Sync UI periodically
spawn(function()
    while true do
        fovLabel.Text = "FOV: " .. tostring(Aimbot.FOV)
        modeBtn.Text = "Mode: " .. Aimbot.Mode
        bindBtn.Text = listeningForBind and "Press a key..." or ("Bind: " .. (Aimbot.BindName or tostring(Aimbot.Bind)))
        zoomBindBtn.Text = listeningForZoomBind and "Press a key..." or ("Zoom Bind: " .. ZoomConfig.BindName)
        aimPartBtn.Text = "Aim Part: " .. Aimbot.AimPart
        task.wait(0.15)
    end
end)

print("✨ ABBADON OONA loaded! Press 'P' to open the menu.")
