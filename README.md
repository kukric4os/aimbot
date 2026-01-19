-- GLORIUS GANG HUB - ESP + AIMBOT (Color UI removed, restored simpler config)
-- LocalScript for client (StarterPlayerScripts / StarterGui)
-- This version removes the RGB color-editing UI and returns to the previous default color configuration.
-- Features preserved:
--  - Advanced ESP (Box, Tracer, Name, Distance, HealthBar, Skeleton)
--  - Movable GUI (drag title bar), tabs (ESP / AIMBOT)
--  - Animated open/close, toggle buttons
--  - Aimbot with bindable key, Mode (Hold/Toggle), FOV circle, smoothing, aim part selector
-- Notes:
--  - To change colors now, edit the AdvancedESP table at the top (BoxColor, NameColor, SkeletonColor)
--  - Drawing API is used when available, otherwise Billboard fallback is used

-- Services
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

-- Cached refs
local LocalPlayer = Players.LocalPlayer or Players.PlayerAdded:Wait()
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local Camera = Workspace.CurrentCamera

-- ====================
-- CONFIG: ESP (default colors only)
-- ====================
local AdvancedESP = {
    Enabled = true,
    Box = true,
    Tracer = true,
    Name = true,
    Distance = true,
    HealthBar = true,
    Skeleton = true,
    Chams = false, -- removed usage in UI, stays false by default
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
    Thickness = 1.5,
    Transparency = 0.9,
    TracerOrigin = "Bottom"
}

-- ====================
-- CONFIG: AIMBOT
-- ====================
local Aimbot = {
    Enabled = false,
    Bind = Enum.KeyCode.E,
    BindName = "E",
    Mode = "Hold",     -- "Hold" or "Toggle"
    FOV = 140,
    ShowFOV = true,
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
local keyDown = false
local toggled = false

local fovCircle = nil
local fovGui = nil

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

-- FOV visual (drawing or gui fallback)
local function createFOVVisual()
    safeRemove(fovCircle)
    if fovGui and fovGui.Parent then fovGui:Destroy(); fovGui = nil end

    if UseDrawing then
        local ok, c = pcall(function() return Drawing.new("Circle") end)
        if ok and c then
            c.Radius = Aimbot.FOV
            c.Thickness = 2
            c.Transparency = 0.8
            c.Color = Color3.fromRGB(255,255,255)
            c.Filled = false
            c.Visible = Aimbot.ShowFOV
            fovCircle = c
            return
        end
    end

    local gui = Instance.new("ScreenGui")
    gui.Name = "AimbotFOVGui"
    gui.ResetOnSpawn = false
    gui.Parent = PlayerGui

    local holder = Instance.new("Frame", gui)
    holder.Name = "FOVHolder"
    holder.Size = UDim2.new(0, Aimbot.FOV * 2, 0, Aimbot.FOV * 2)
    holder.Position = UDim2.new(0.5, -Aimbot.FOV, 0.5, -Aimbot.FOV)
    holder.BackgroundTransparency = 1
    holder.BorderSizePixel = 0

    local circle = Instance.new("ImageLabel", holder)
    circle.Size = UDim2.new(1,0,1,0)
    circle.Position = UDim2.new(0,0,0,0)
    circle.BackgroundTransparency = 1
    circle.Image = "rbxassetid://3570695787"
    circle.ImageColor3 = Color3.fromRGB(255,255,255)
    circle.ImageTransparency = Aimbot.ShowFOV and 0.5 or 1
    circle.ScaleType = Enum.ScaleType.Slice
    circle.SliceCenter = Rect.new(128,128,128,128)

    fovGui = gui
end

local function updateFOVVisual()
    if UseDrawing and fovCircle then
        local centerX = (Camera.ViewportSize.X/2)
        local centerY = (Camera.ViewportSize.Y/2)
        pcall(function()
            fovCircle.Position = Vector2.new(centerX, centerY)
            fovCircle.Radius = Aimbot.FOV
            fovCircle.Visible = Aimbot.ShowFOV
        end)
    elseif fovGui and fovGui.Parent then
        local holder = fovGui:FindFirstChild("FOVHolder")
        if holder then
            holder.Size = UDim2.new(0, Aimbot.FOV * 2, 0, Aimbot.FOV * 2)
            holder.Position = UDim2.new(0.5, -Aimbot.FOV, 0.5, -Aimbot.FOV)
            local circle = holder:FindFirstChildOfClass("ImageLabel")
            if circle then circle.ImageTransparency = Aimbot.ShowFOV and 0.5 or 1 end
        end
    end
end

-- Aimbot target selection & aim
local function getAimbotTarget()
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

-- Main update loop (ESP + Aim)
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

                        -- Box
                        if AdvancedESP.Box and cont.BoxLines then
                            local segs = {
                                {From = topLeft, To = Vector2.new(topLeft.X + width/4, topLeft.Y)},
                                {From = topLeft, To = Vector2.new(topLeft.X, topLeft.Y + height/4)},
                                {From = topRight, To = Vector2.new(topRight.X - width/4, topRight.Y)},
                                {From = topRight, To = Vector2.new(topRight.X, topRight.Y + height/4)},
                                {From = botLeft, To = Vector2.new(botLeft.X + width/4, botLeft.Y)},
                                {From = botLeft, To = Vector2.new(botLeft.X, botLeft.Y - height/4)},
                                {From = botRight, To = Vector2.new(botRight.X - width/4, botRight.Y)},
                                {From = botRight, To = Vector2.new(botRight.X, botRight.Y - height/4)}
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
    if Aimbot.Enabled then
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
-- GUI (organized, color UI removed)
-- ====================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "GloriusGangHubGui"
screenGui.ResetOnSpawn = false
screenGui.Parent = PlayerGui

local targetSize = UDim2.new(0, 620, 0, 720)
local closedSize = UDim2.new(0, 0, 0, 0)

local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = closedSize
mainFrame.Position = UDim2.new(0, 20, 0, 80)
mainFrame.BackgroundColor3 = Color3.fromRGB(12,12,12)
mainFrame.BorderSizePixel = 0
mainFrame.Active = true
mainFrame.ClipsDescendants = true
mainFrame.Parent = screenGui

-- TitleBar and tabs
local titleBar = Instance.new("Frame", mainFrame)
titleBar.Name = "TitleBar"
titleBar.Size = UDim2.new(1, 0, 0, 64)
titleBar.BackgroundTransparency = 1

local titleLabel = Instance.new("TextLabel", titleBar)
titleLabel.Name = "TitleLabel"
titleLabel.Size = UDim2.new(0.6, -20, 1, 0)
titleLabel.Position = UDim2.new(0, 14, 0, 0)
titleLabel.BackgroundTransparency = 1
titleLabel.Text = "GLORIUS GANG HUB"
titleLabel.TextColor3 = Color3.new(1,1,1)
titleLabel.Font = Enum.Font.SourceSansBold
titleLabel.TextSize = 20
titleLabel.TextXAlignment = Enum.TextXAlignment.Left

local tabsHolder = Instance.new("Frame", titleBar)
tabsHolder.Name = "Tabs"
tabsHolder.Size = UDim2.new(0.4, -14, 1, 0)
tabsHolder.Position = UDim2.new(0.6, 0, 0, 0)
tabsHolder.BackgroundTransparency = 1

local espTab = Instance.new("TextButton", tabsHolder)
espTab.Size = UDim2.new(0.5, -6, 0.6, 0)
espTab.Position = UDim2.new(0, 6, 0.2, 0)
espTab.Text = "ESP"
espTab.Font = Enum.Font.SourceSans
espTab.TextSize = 14
espTab.BackgroundColor3 = Color3.fromRGB(60,60,60)
espTab.TextColor3 = Color3.new(1,1,1)
espTab.AutoButtonColor = false
Instance.new("UICorner", espTab).CornerRadius = UDim.new(0.1, 0)

local aimTab = Instance.new("TextButton", tabsHolder)
aimTab.Size = UDim2.new(0.5, -6, 0.6, 0)
aimTab.Position = UDim2.new(0.5, 0, 0.2, 0)
aimTab.Text = "Aimbot"
aimTab.Font = Enum.Font.SourceSans
aimTab.TextSize = 14
aimTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
aimTab.TextColor3 = Color3.new(1,1,1)
aimTab.AutoButtonColor = false
Instance.new("UICorner", aimTab).CornerRadius = UDim.new(0.1, 0)

local contentHolder = Instance.new("Frame", mainFrame)
contentHolder.Name = "ContentHolder"
contentHolder.Size = UDim2.new(1, -28, 1, -100)
contentHolder.Position = UDim2.new(0, 14, 0, 76)
contentHolder.BackgroundTransparency = 1

local espFrame = Instance.new("Frame", contentHolder)
espFrame.Size = UDim2.new(1, 0, 1, 0)
espFrame.BackgroundTransparency = 1
espFrame.Visible = true

local aimbotFrame = Instance.new("Frame", contentHolder)
aimbotFrame.Size = UDim2.new(1, 0, 1, 0)
aimbotFrame.BackgroundTransparency = 1
aimbotFrame.Visible = false

espTab.MouseButton1Click:Connect(function()
    espFrame.Visible = true
    aimbotFrame.Visible = false
    espTab.BackgroundColor3 = Color3.fromRGB(60,60,60)
    aimTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
end)
aimTab.MouseButton1Click:Connect(function()
    espFrame.Visible = false
    aimbotFrame.Visible = true
    espTab.BackgroundColor3 = Color3.fromRGB(28,28,28)
    aimTab.BackgroundColor3 = Color3.fromRGB(60,60,60)
end)

-- ESP tab: toggles grid
local espGrid = Instance.new("UIGridLayout", espFrame)
espGrid.CellPadding = UDim2.new(0, 10, 0, 10)
espGrid.CellSize = UDim2.new(0.48, 0, 0, 52)
espGrid.FillDirection = Enum.FillDirection.Horizontal
espGrid.SortOrder = Enum.SortOrder.LayoutOrder

local function createToggleButtonSimple(parent, label, key)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(1, 0, 0, 52)
    btn.BackgroundColor3 = Color3.fromRGB(28,28,28)
    btn.BorderSizePixel = 0
    btn.Text = label
    btn.Font = Enum.Font.SourceSans
    btn.TextColor3 = Color3.new(1,1,1)
    btn.TextSize = 16
    btn.AutoButtonColor = false

    local indicator = Instance.new("Frame", btn)
    indicator.Size = UDim2.new(0, 46, 0, 28)
    indicator.Position = UDim2.new(1, -56, 0.5, -14)
    indicator.BackgroundColor3 = AdvancedESP[key] and Color3.fromRGB(0,170,70) or Color3.fromRGB(80,80,80)
    Instance.new("UICorner", indicator).CornerRadius = UDim.new(0.5,0)

    btn.MouseButton1Click:Connect(function()
        AdvancedESP[key] = not AdvancedESP[key]
        indicator.BackgroundColor3 = AdvancedESP[key] and Color3.fromRGB(0,170,70) or Color3.fromRGB(80,80,80)
    end)
    return btn
end

local espOptions = {
    {label="Master", key="Enabled"},
    {label="Box", key="Box"},
    {label="Tracer", key="Tracer"},
    {label="Name", key="Name"},
    {label="Distance", key="Distance"},
    {label="Health Bar", key="HealthBar"},
    {label="Skeleton", key="Skeleton"},
    {label="Rainbow", key="Rainbow"},
    {label="Team Check", key="TeamCheck"}
}
for _, opt in ipairs(espOptions) do createToggleButtonSimple(espFrame, opt.label, opt.key) end

-- MaxDistance in ESP tab
local espDistFrame = Instance.new("Frame", espFrame)
espDistFrame.Size = UDim2.new(1, 0, 0, 64)
espDistFrame.BackgroundTransparency = 1
espDistFrame.LayoutOrder = 999

local espDistLabel = Instance.new("TextLabel", espDistFrame)
espDistLabel.Size = UDim2.new(1, -140, 0, 28)
espDistLabel.Position = UDim2.new(0, 10, 0, 8)
espDistLabel.BackgroundTransparency = 1
espDistLabel.Text = "MaxDistance: " .. tostring(AdvancedESP.MaxDistance)
espDistLabel.Font = Enum.Font.SourceSans
espDistLabel.TextSize = 16
espDistLabel.TextColor3 = Color3.new(1,1,1)
espDistLabel.TextXAlignment = Enum.TextXAlignment.Left

local espMinus = Instance.new("TextButton", espDistFrame)
espMinus.Size = UDim2.new(0, 64, 0, 28)
espMinus.Position = UDim2.new(1, -120, 0, 8)
espMinus.Text = "-"
espMinus.Font = Enum.Font.SourceSansBold
espMinus.TextSize = 20
espMinus.BackgroundColor3 = Color3.fromRGB(28,28,28)
espMinus.TextColor3 = Color3.new(1,1,1)
espMinus.BorderSizePixel = 0

local espPlus = Instance.new("TextButton", espDistFrame)
espPlus.Size = UDim2.new(0, 64, 0, 28)
espPlus.Position = UDim2.new(1, -48, 0, 8)
espPlus.Text = "+"
espPlus.Font = Enum.Font.SourceSansBold
espPlus.TextSize = 20
espPlus.BackgroundColor3 = Color3.fromRGB(28,28,28)
espPlus.TextColor3 = Color3.new(1,1,1)
espPlus.BorderSizePixel = 0

espMinus.MouseButton1Click:Connect(function()
    AdvancedESP.MaxDistance = math.max(100, AdvancedESP.MaxDistance - 100)
    espDistLabel.Text = "MaxDistance: " .. tostring(AdvancedESP.MaxDistance)
end)
espPlus.MouseButton1Click:Connect(function()
    AdvancedESP.MaxDistance = math.min(10000, AdvancedESP.MaxDistance + 100)
    espDistLabel.Text = "MaxDistance: " .. tostring(AdvancedESP.MaxDistance)
end)

-- AIMBOT tab: controls (Bind, Mode, FOV +/- , ShowFOV toggle, Smoothing +/- , AimPart)
local aimGrid = Instance.new("UIGridLayout", aimbotFrame)
aimGrid.CellPadding = UDim2.new(0, 10, 0, 10)
aimGrid.CellSize = UDim2.new(0.48, 0, 0, 52)
aimGrid.FillDirection = Enum.FillDirection.Horizontal
aimGrid.SortOrder = Enum.SortOrder.LayoutOrder

local function createAimbotToggle(parent, label, getState, setState)
    local btn = Instance.new("TextButton", parent)
    btn.Size = UDim2.new(1,0,0,52); btn.BackgroundColor3 = Color3.fromRGB(28,28,28); btn.BorderSizePixel = 0
    btn.Text = label; btn.Font = Enum.Font.SourceSans; btn.TextColor3 = Color3.new(1,1,1); btn.TextSize = 16; btn.AutoButtonColor = false
    local indicator = Instance.new("Frame", btn); indicator.Size = UDim2.new(0,46,0,28); indicator.Position = UDim2.new(1,-56,0.5,-14)
    indicator.BackgroundColor3 = getState() and Color3.fromRGB(0,170,70) or Color3.fromRGB(80,80,80); Instance.new("UICorner", indicator).CornerRadius = UDim.new(0.5,0)
    btn.MouseButton1Click:Connect(function()
        setState(not getState())
        indicator.BackgroundColor3 = getState() and Color3.fromRGB(0,170,70) or Color3.fromRGB(80,80,80)
    end)
    return btn
end

local aimbotMasterBtn = createAimbotToggle(aimbotFrame, "Aimbot Master", function() return Aimbot.Enabled end, function(v) Aimbot.Enabled = v end)
local modeBtn = Instance.new("TextButton", aimbotFrame)
modeBtn.Size = UDim2.new(1,0,0,52); modeBtn.BackgroundColor3 = Color3.fromRGB(28,28,28); modeBtn.BorderSizePixel = 0
modeBtn.Text = "Mode: " .. Aimbot.Mode; modeBtn.Font = Enum.Font.SourceSans; modeBtn.TextColor3 = Color3.new(1,1,1); modeBtn.TextSize = 16
modeBtn.MouseButton1Click:Connect(function()
    if Aimbot.Mode == "Hold" then Aimbot.Mode = "Toggle" else Aimbot.Mode = "Hold" end
    modeBtn.Text = "Mode: " .. Aimbot.Mode
end)

local bindBtn = Instance.new("TextButton", aimbotFrame)
bindBtn.Size = UDim2.new(1,0,0,52); bindBtn.BackgroundColor3 = Color3.fromRGB(28,28,28); bindBtn.BorderSizePixel = 0
bindBtn.Text = "Bind: " .. Aimbot.BindName; bindBtn.Font = Enum.Font.SourceSans; bindBtn.TextColor3 = Color3.new(1,1,1); bindBtn.TextSize = 16
bindBtn.MouseButton1Click:Connect(function()
    listeningForBind = true
    bindBtn.Text = "Press any key..."
end)

-- FOV Row
local fovRow = Instance.new("Frame", aimbotFrame)
fovRow.Size = UDim2.new(1,0,0,52); fovRow.BackgroundTransparency = 1
local fovLabel = Instance.new("TextLabel", fovRow); fovLabel.Size = UDim2.new(0.6,0,1,0); fovLabel.Position = UDim2.new(0,8,0,0)
fovLabel.BackgroundTransparency = 1; fovLabel.Text = "FOV: " .. tostring(Aimbot.FOV); fovLabel.Font = Enum.Font.SourceSans; fovLabel.TextSize = 16; fovLabel.TextColor3 = Color3.new(1,1,1); fovLabel.TextXAlignment = Enum.TextXAlignment.Left
local fovMinus = Instance.new("TextButton", fovRow); fovMinus.Size = UDim2.new(0,84,0,36); fovMinus.Position = UDim2.new(1,-184,0,8); fovMinus.Text = "-" ; fovMinus.Font = Enum.Font.SourceSansBold; fovMinus.TextSize = 22; fovMinus.BackgroundColor3 = Color3.fromRGB(28,28,28)
local fovPlus = Instance.new("TextButton", fovRow); fovPlus.Size = UDim2.new(0,84,0,36); fovPlus.Position = UDim2.new(1,-92,0,8); fovPlus.Text = "+"; fovPlus.Font = Enum.Font.SourceSansBold; fovPlus.TextSize = 22; fovPlus.BackgroundColor3 = Color3.fromRGB(28,28,28)
fovMinus.MouseButton1Click:Connect(function() Aimbot.FOV = clamp(Aimbot.FOV - 10, 20, 2000); fovLabel.Text = "FOV: " .. tostring(Aimbot.FOV); createFOVVisual() end)
fovPlus.MouseButton1Click:Connect(function() Aimbot.FOV = clamp(Aimbot.FOV + 10, 20, 2000); fovLabel.Text = "FOV: " .. tostring(Aimbot.FOV); createFOVVisual() end)
local showFOVBtn = createAimbotToggle(aimbotFrame, "Show FOV", function() return Aimbot.ShowFOV end, function(v) Aimbot.ShowFOV = v if UseDrawing and fovCircle then fovCircle.Visible = v end if fovGui and fovGui.Parent then local holder = fovGui:FindFirstChild("FOVHolder"); if holder then local circle = holder:FindFirstChildOfClass("ImageLabel"); if circle then circle.ImageTransparency = v and 0.5 or 1 end end end end)

-- Smoothing row
local smoothRow = Instance.new("Frame", aimbotFrame)
smoothRow.Size = UDim2.new(1,0,0,52); smoothRow.BackgroundTransparency = 1
local smoothLabel = Instance.new("TextLabel", smoothRow); smoothLabel.Size = UDim2.new(0.6,0,1,0); smoothLabel.Position = UDim2.new(0,8,0,0)
smoothLabel.BackgroundTransparency = 1; smoothLabel.Text = "Smoothing: " .. string.format("%.2f", Aimbot.Smoothing); smoothLabel.Font = Enum.Font.SourceSans; smoothLabel.TextSize = 16; smoothLabel.TextColor3 = Color3.new(1,1,1); smoothLabel.TextXAlignment = Enum.TextXAlignment.Left
local smoothMinus = Instance.new("TextButton", smoothRow); smoothMinus.Size = UDim2.new(0,84,0,36); smoothMinus.Position = UDim2.new(1,-184,0,8); smoothMinus.Text = "-" ; smoothMinus.Font = Enum.Font.SourceSansBold; smoothMinus.TextSize = 22; smoothMinus.BackgroundColor3 = Color3.fromRGB(28,28,28)
local smoothPlus = Instance.new("TextButton", smoothRow); smoothPlus.Size = UDim2.new(0,84,0,36); smoothPlus.Position = UDim2.new(1,-92,0,8); smoothPlus.Text = "+" ; smoothPlus.Font = Enum.Font.SourceSansBold; smoothPlus.TextSize = 22; smoothPlus.BackgroundColor3 = Color3.fromRGB(28,28,28)
smoothMinus.MouseButton1Click:Connect(function() Aimbot.Smoothing = clamp(Aimbot.Smoothing - 0.02, 0, 0.95); smoothLabel.Text = "Smoothing: " .. string.format("%.2f", Aimbot.Smoothing) end)
smoothPlus.MouseButton1Click:Connect(function() Aimbot.Smoothing = clamp(Aimbot.Smoothing + 0.02, 0, 0.95); smoothLabel.Text = "Smoothing: " .. string.format("%.2f", Aimbot.Smoothing) end)

-- Aim part selection
local aimPartBtn = Instance.new("TextButton", aimbotFrame)
aimPartBtn.Size = UDim2.new(1,0,0,52); aimPartBtn.BackgroundColor3 = Color3.fromRGB(28,28,28); aimPartBtn.BorderSizePixel = 0
aimPartBtn.Text = "Aim Part: " .. Aimbot.AimPart; aimPartBtn.Font = Enum.Font.SourceSans; aimPartBtn.TextColor3 = Color3.new(1,1,1); aimPartBtn.TextSize = 16
aimPartBtn.MouseButton1Click:Connect(function()
    if Aimbot.AimPart == "Head" then Aimbot.AimPart = "HumanoidRootPart" else Aimbot.AimPart = "Head" end
    aimPartBtn.Text = "Aim Part: " .. Aimbot.AimPart
end)

-- Bind handling
local function setBind(keyCode)
    if not keyCode then return end
    Aimbot.Bind = keyCode
    Aimbot.BindName = tostring(keyCode):gsub("Enum.KeyCode.", "") or tostring(keyCode)
    bindBtn.Text = "Bind: " .. Aimbot.BindName
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
    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == Aimbot.Bind then
        keyDown = true
        if Aimbot.Mode == "Toggle" then toggled = not toggled end
    end
end)
UserInputService.InputEnded:Connect(function(input, processed)
    if processed then return end
    if input.UserInputType == Enum.UserInputType.Keyboard and input.KeyCode == Aimbot.Bind then
        keyDown = false
    end
end)

-- Create initial FOV visual
createFOVVisual()

-- Menu open/close and dragging
local function tweenObject(obj, props, info) info = info or TweenInfo.new(0.18, Enum.EasingStyle.Quad, Enum.EasingDirection.Out); local ok, t = pcall(function() return TweenService:Create(obj, info, props) end); if ok and t then pcall(function() t:Play() end) end end
local menuOpen = false
local function setMenuOpen(open)
    if open == menuOpen then return end
    menuOpen = open
    if open then
        mainFrame.Visible = true
        tweenObject(mainFrame, {Size = targetSize}, TweenInfo.new(0.28, Enum.EasingStyle.Quad, Enum.EasingDirection.Out))
    else
        local t = TweenService:Create(mainFrame, TweenInfo.new(0.22, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {Size = closedSize})
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
        smoothLabel.Text = "Smoothing: " .. string.format("%.2f", Aimbot.Smoothing)
        bindBtn.Text = listeningForBind and "Press a key..." or ("Bind: " .. (Aimbot.BindName or tostring(Aimbot.Bind)))
        modeBtn.Text = "Mode: " .. Aimbot.Mode
        aimPartBtn.Text = "Aim Part: " .. Aimbot.AimPart
        espDistLabel.Text = "MaxDistance: " .. tostring(AdvancedESP.MaxDistance)
        -- ensure fov visuals updated
        if UseDrawing and fovCircle then fovCircle.Radius = Aimbot.FOV; fovCircle.Visible = Aimbot.ShowFOV end
        task.wait(0.15)
    end
end)

print("GLORIUS GANG HUB (default color UI restored) loaded. Press 'P' to open the menu. Drawing available:", UseDrawing)
