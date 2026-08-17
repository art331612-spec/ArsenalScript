--[[
    ARSENAL PRO v.46.2 – FOV ПОЛНОСТЬЮ ОТКЛЮЧАЕТСЯ ПРИ СВЁРТКЕ
    - При сворачивании меню ползунок FOV становится неактивным
    - Все события ползунка проверяют menuVisible
    - Баг с касаниями на телефоне исправлен
--]]

local player = game:GetService("Players").LocalPlayer
local runService = game:GetService("RunService")
local userInputService = game:GetService("UserInputService")
local workspace = game:GetService("Workspace")
local camera = workspace.CurrentCamera
local tweenService = game:GetService("TweenService")

-- ==========================================
-- === ВСПОМОГАТЕЛЬНЫЕ ФУНКЦИИ (без изменений) ===
-- ==========================================
local function getEnemies()
    local enemies = {}
    local myTeam = player.Team
    for _, plr in pairs(game:GetService("Players"):GetPlayers()) do
        if plr ~= player and plr.Character and plr.Character:FindFirstChild("HumanoidRootPart") then
            if myTeam and plr.Team and plr.Team ~= myTeam then
                table.insert(enemies, plr)
            elseif not myTeam then
                table.insert(enemies, plr)
            end
        end
    end
    return enemies
end

local function isVisible(targetPart)
    local origin = camera.CFrame.Position
    local direction = (targetPart.Position - origin).unit
    local distance = (targetPart.Position - origin).magnitude
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Blacklist
    params.FilterDescendantsInstances = {player.Character, camera}
    local result = workspace:Raycast(origin, direction * distance, params)
    if result then
        local hit = result.Instance
        if hit and hit:IsDescendantOf(targetPart.Parent) then
            return true
        else
            return false
        end
    end
    return true
end

-- ==========================================
-- === РЕЖИМ "ТЕНЬ" (без изменений) ===
-- ==========================================
local shadowActive = false

local function teleportToSpawn()
    local char = player.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end
    local spawn = workspace:FindFirstChild("SpawnLocation")
    if spawn then
        root.CFrame = spawn.CFrame + Vector3.new(0, 3, 0)
    else
        root.CFrame = CFrame.new(0, 5, 0)
    end
end

local function updateShadow()
    if not shadowActive then return end
    local enemies = getEnemies()
    if #enemies == 0 then
        teleportToSpawn()
        return
    end

    local char = player.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    local closest = nil
    local closestDist = math.huge
    local origin = root.Position
    local maxDist = 2000

    for _, enemy in pairs(enemies) do
        local eRoot = enemy.Character and enemy.Character:FindFirstChild("HumanoidRootPart")
        if eRoot then
            local humanoid = enemy.Character:FindFirstChild("Humanoid")
            if humanoid and humanoid.Health > 0 then
                local dist = (eRoot.Position - origin).magnitude
                if dist < closestDist and dist < maxDist then
                    closestDist = dist
                    closest = enemy
                end
            end
        end
    end

    if closest and closest.Character then
        local targetRoot = closest.Character:FindFirstChild("HumanoidRootPart")
        if targetRoot then
            local targetPos = targetRoot.Position - targetRoot.CFrame.LookVector * 2.5 + Vector3.new(0, 1.5, 0)
            root.CFrame = CFrame.new(targetPos, targetRoot.Position)
        end
    else
        teleportToSpawn()
    end
end

-- ==========================================
-- === МОДЫ ===
-- ==========================================
local aimbotActive = false
local espActive = false
local espBoxes = {}
local speedModActive = false
local jumpModActive = false
local fovValue = 0
local fovLockActive = true

-- ВЫСОКИЙ ПРЫЖОК
local function applyJumpImpulse()
    if not jumpModActive then return end
    local char = player.Character
    if not char then return end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return end
    local bv = Instance.new("BodyVelocity")
    bv.Velocity = Vector3.new(0, 70, 0)
    bv.MaxForce = Vector3.new(0, 4000, 0)
    bv.Parent = root
    game:GetService("Debris"):AddItem(bv, 0.1)
end

local function setupJump(char)
    local humanoid = char:FindFirstChild("Humanoid")
    if humanoid then
        humanoid.Jumping:Connect(applyJumpImpulse)
    end
end

player.CharacterAdded:Connect(function(char)
    task.wait(0.5)
    setupJump(char)
    if speedModActive then
        local h = char:FindFirstChild("Humanoid")
        if h then h.WalkSpeed = 70 end
    end
end)
if player.Character then setupJump(player.Character) end

-- АИМБОТ
runService.RenderStepped:Connect(function()
    if not aimbotActive then return end
    local enemies = getEnemies()
    if #enemies == 0 then return end
    local closestEnemy = nil
    local closestDist = math.huge
    for _, enemy in pairs(enemies) do
        local head = enemy.Character:FindFirstChild("Head")
        if head and isVisible(head) then
            local dist = (head.Position - camera.CFrame.Position).magnitude
            if dist < closestDist then
                closestDist = dist
                closestEnemy = enemy
            end
        end
    end
    if closestEnemy and closestEnemy.Character then
        local head = closestEnemy.Character.Head
        if head then
            camera.CFrame = CFrame.new(camera.CFrame.Position, head.Position)
        end
    end
end)

-- ESP
runService.Heartbeat:Connect(function()
    if espActive then
        local enemies = getEnemies()
        for plr, box in pairs(espBoxes) do
            if not table.find(enemies, plr) or not plr.Character or not plr.Character:FindFirstChild("HumanoidRootPart") then
                box:Destroy()
                espBoxes[plr] = nil
            end
        end
        for _, enemy in pairs(enemies) do
            if enemy.Character and enemy.Character:FindFirstChild("HumanoidRootPart") then
                local root = enemy.Character.HumanoidRootPart
                if not espBoxes[enemy] then
                    local box = Instance.new("BoxHandleAdornment")
                    box.Adornee = root
                    box.Size = Vector3.new(4, 6, 2.5)
                    box.Color3 = Color3.fromRGB(255, 0, 0)
                    box.Transparency = 0.2
                    box.AlwaysOnTop = true
                    box.ZIndex = 10
                    box.Parent = root
                    espBoxes[enemy] = box
                end
            end
        end
    else
        for _, box in pairs(espBoxes) do
            box:Destroy()
        end
        espBoxes = {}
    end
end)

-- СКОРОСТЬ
runService.Heartbeat:Connect(function()
    local humanoid = player.Character and player.Character:FindFirstChild("Humanoid")
    if humanoid and speedModActive then
        humanoid.WalkSpeed = 70
    end
end)

-- ПОЛЁТ (F)
local flying = false
local flyVelocity = Instance.new("BodyVelocity")
flyVelocity.MaxForce = Vector3.new(4000, 4000, 4000)

userInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.F then
        flying = not flying
        if flying then
            flyVelocity.Parent = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
            flyVelocity.Velocity = Vector3.new(0, 10, 0)
        else
            flyVelocity.Parent = nil
        end
    end
end)

runService.Heartbeat:Connect(function()
    if flying and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
        local dir = Vector3.new()
        if userInputService:IsKeyDown(Enum.KeyCode.W) then dir = dir + camera.CFrame.LookVector end
        if userInputService:IsKeyDown(Enum.KeyCode.S) then dir = dir - camera.CFrame.LookVector end
        if userInputService:IsKeyDown(Enum.KeyCode.A) then dir = dir - camera.CFrame.RightVector end
        if userInputService:IsKeyDown(Enum.KeyCode.D) then dir = dir + camera.CFrame.RightVector end
        if dir.Magnitude > 0 then
            flyVelocity.Velocity = dir.unit * 50
        end
    end
end)

-- ==========================================
-- === FOV (УСИЛЕННАЯ ФИКСАЦИЯ + ОТКЛЮЧЕНИЕ ПРИ СВЁРТКЕ) ===
-- ==========================================
-- Принудительная установка в RenderStepped
runService.RenderStepped:Connect(function()
    if fovLockActive then
        camera.FieldOfView = 70 + fovValue
    end
end)

-- Heartbeat для надёжности
runService.Heartbeat:Connect(function()
    if fovLockActive then
        camera.FieldOfView = 70 + fovValue
    end
end)

-- Перехват изменений
camera:GetPropertyChangedSignal("FieldOfView"):Connect(function()
    if fovLockActive and camera.FieldOfView ~= 70 + fovValue then
        camera.FieldOfView = 70 + fovValue
    end
end)

-- Восстановление при респавне
player.CharacterAdded:Connect(function()
    task.wait(0.2)
    if fovLockActive then
        camera.FieldOfView = 70 + fovValue
    end
end)

-- ==========================================
-- === МЕНЮ (без изменений, но с флагом menuVisible) ===
-- ==========================================
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "ArsenalPro"
screenGui.ResetOnSpawn = false
screenGui.Parent = gethui() or game:GetService("CoreGui")

local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 220, 0, 320)
mainFrame.Position = UDim2.new(0.5, -110, 0.5, -160)
mainFrame.BackgroundColor3 = Color3.fromRGB(4, 4, 12)
mainFrame.BackgroundTransparency = 0.1
mainFrame.BorderSizePixel = 0
mainFrame.ClipsDescendants = true
mainFrame.Active = true
mainFrame.Draggable = true
mainFrame.Parent = screenGui

local corner = Instance.new("UICorner")
corner.CornerRadius = UDim.new(0, 10)
corner.Parent = mainFrame

local blur = Instance.new("BlurEffect")
blur.Size = 4
blur.Parent = mainFrame

-- Неоновая рамка
local border = Instance.new("Frame")
border.Size = UDim2.new(1, 4, 1, 4)
border.Position = UDim2.new(0, -2, 0, -2)
border.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
border.BackgroundTransparency = 0.7
border.BorderSizePixel = 0
border.ZIndex = -1
border.Parent = mainFrame
local borderCorner = Instance.new("UICorner")
borderCorner.CornerRadius = UDim.new(0, 12)
borderCorner.Parent = border

local borderGrad = Instance.new("UIGradient")
borderGrad.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(0, 200, 255)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255, 0, 128)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 200, 255))
})
borderGrad.Rotation = 0
borderGrad.Parent = border
local gradTween = tweenService:Create(borderGrad, TweenInfo.new(2.5, Enum.EasingStyle.Linear, Enum.EasingDirection.InOut, -1, true), {Rotation = 360})
gradTween:Play()

local glow = Instance.new("Frame")
glow.Size = UDim2.new(1, 10, 1, 10)
glow.Position = UDim2.new(0, -5, 0, -5)
glow.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
glow.BackgroundTransparency = 0.95
glow.BorderSizePixel = 0
glow.ZIndex = -2
glow.Parent = mainFrame
local glowCorner = Instance.new("UICorner")
glowCorner.CornerRadius = UDim.new(0, 14)
glowCorner.Parent = glow
local glowTween = tweenService:Create(glow, TweenInfo.new(1.8, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {BackgroundTransparency = 0.85})
glowTween:Play()

-- Уголки
local function createCorner(x, y, rot)
    local c = Instance.new("Frame")
    c.Size = UDim2.new(0, 8, 0, 8)
    c.Position = UDim2.new(x, 0, y, 0)
    c.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
    c.BackgroundTransparency = 0.5
    c.BorderSizePixel = 0
    c.Parent = mainFrame
    local cCorner = Instance.new("UICorner")
    cCorner.CornerRadius = UDim.new(0, 2)
    cCorner.Parent = c
    c.Rotation = rot
end
createCorner(0, 0, 0)
createCorner(1, 0, 90)
createCorner(0, 1, 270)
createCorner(1, 1, 180)

-- Заголовок
local titleBg = Instance.new("TextLabel")
titleBg.Size = UDim2.new(1, 0, 0, 28)
titleBg.Position = UDim2.new(0, 0, 0, 0)
titleBg.BackgroundTransparency = 1
titleBg.Text = "ARSENAL"
titleBg.TextColor3 = Color3.fromRGB(255, 0, 128)
titleBg.TextScaled = true
titleBg.Font = Enum.Font.GothamBold
titleBg.TextStrokeColor3 = Color3.fromRGB(0, 200, 255)
titleBg.TextStrokeTransparency = 0.7
titleBg.Parent = mainFrame

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, 0, 0, 28)
title.Position = UDim2.new(0.01, 0, 0.01, 0)
title.BackgroundTransparency = 1
title.Text = "ARSENAL"
title.TextColor3 = Color3.fromRGB(255, 255, 255)
title.TextScaled = true
title.Font = Enum.Font.GothamBold
title.TextStrokeColor3 = Color3.fromRGB(0, 200, 255)
title.TextStrokeTransparency = 0.3
title.Parent = mainFrame

local function createDot(x, y)
    local dot = Instance.new("Frame")
    dot.Size = UDim2.new(0, 3, 0, 3)
    dot.Position = UDim2.new(x, 0, y, 0)
    dot.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
    dot.BackgroundTransparency = 0.5
    dot.BorderSizePixel = 0
    dot.Parent = mainFrame
    local dotCorner = Instance.new("UICorner")
    dotCorner.CornerRadius = UDim.new(1, 0)
    dotCorner.Parent = dot
    tweenService:Create(dot, TweenInfo.new(1.2, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {BackgroundTransparency = 0.2}):Play()
end
createDot(0.05, 0.02)
createDot(0.95, 0.02)

local subtitle = Instance.new("TextLabel")
subtitle.Size = UDim2.new(1, 0, 0, 14)
subtitle.Position = UDim2.new(0, 0, 0, 28)
subtitle.BackgroundTransparency = 1
subtitle.Text = "• N E O N •"
subtitle.TextColor3 = Color3.fromRGB(0, 200, 255)
subtitle.TextScaled = true
subtitle.Font = Enum.Font.GothamMedium
subtitle.TextXAlignment = Enum.TextXAlignment.Center
subtitle.Parent = mainFrame

local line = Instance.new("Frame")
line.Size = UDim2.new(0.8, 0, 0, 1)
line.Position = UDim2.new(0.1, 0, 0, 46)
line.BackgroundColor3 = Color3.fromRGB(255, 0, 128)
line.BackgroundTransparency = 0.4
line.BorderSizePixel = 0
line.Parent = mainFrame
tweenService:Create(line, TweenInfo.new(1.5, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {BackgroundTransparency = 0.15}):Play()

-- Контейнер для кнопок
local btnContainer = Instance.new("Frame")
btnContainer.Size = UDim2.new(1, -16, 1, -60)
btnContainer.Position = UDim2.new(0, 8, 0, 54)
btnContainer.BackgroundTransparency = 1
btnContainer.Parent = mainFrame

-- Функция создания кнопки
local function createNeonButton(text, callback, yPos)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, 0, 0, 24)
    btn.Position = UDim2.new(0, 0, 0, yPos)
    btn.BackgroundColor3 = Color3.fromRGB(8, 8, 20)
    btn.BackgroundTransparency = 0.2
    btn.Text = text .. ": Выкл"
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.TextScaled = true
    btn.Font = Enum.Font.GothamMedium
    btn.BorderSizePixel = 1
    btn.BorderColor3 = Color3.fromRGB(0, 200, 255)
    btn.Parent = btnContainer
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 4)
    btnCorner.Parent = btn

    local btnGlow = Instance.new("Frame")
    btnGlow.Size = UDim2.new(1, 4, 1, 4)
    btnGlow.Position = UDim2.new(0, -2, 0, -2)
    btnGlow.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
    btnGlow.BackgroundTransparency = 0.9
    btnGlow.BorderSizePixel = 0
    btnGlow.ZIndex = -1
    btnGlow.Parent = btn
    local btnGlowCorner = Instance.new("UICorner")
    btnGlowCorner.CornerRadius = UDim.new(0, 6)
    btnGlowCorner.Parent = btnGlow

    local dot = Instance.new("Frame")
    dot.Size = UDim2.new(0, 4, 0, 4)
    dot.Position = UDim2.new(0, 6, 0.5, -2)
    dot.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
    dot.BackgroundTransparency = 0.2
    dot.BorderSizePixel = 0
    dot.Parent = btn
    local dotCorner = Instance.new("UICorner")
    dotCorner.CornerRadius = UDim.new(1, 0)
    dotCorner.Parent = dot

    btn.MouseEnter:Connect(function()
        tweenService:Create(btn, TweenInfo.new(0.1), {Size = UDim2.new(1.04, 0, 0, 26)}):Play()
        tweenService:Create(btnGlow, TweenInfo.new(0.1), {BackgroundTransparency = 0.7}):Play()
    end)
    btn.MouseLeave:Connect(function()
        tweenService:Create(btn, TweenInfo.new(0.1), {Size = UDim2.new(1, 0, 0, 24)}):Play()
        tweenService:Create(btnGlow, TweenInfo.new(0.1), {BackgroundTransparency = 0.9}):Play()
    end)

    local state = false
    btn.MouseButton1Click:Connect(function()
        state = not state
        callback(state)
        if state then
            btn.Text = text .. ": Вкл"
            btn.BorderColor3 = Color3.fromRGB(40, 200, 40)
            btnGlow.BackgroundColor3 = Color3.fromRGB(40, 200, 40)
            dot.BackgroundColor3 = Color3.fromRGB(40, 200, 40)
        else
            btn.Text = text .. ": Выкл"
            btn.BorderColor3 = Color3.fromRGB(0, 200, 255)
            btnGlow.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
            dot.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
        end
    end)
    return btn
end

-- Создаём кнопки
local yPos = 0
createNeonButton("Аимбот", function(state) aimbotActive = state end, yPos)
yPos = yPos + 30
createNeonButton("ESP", function(state) espActive = state end, yPos)
yPos = yPos + 30
createNeonButton("Скорость", function(state)
    speedModActive = state
    if state and player.Character then
        local h = player.Character:FindFirstChild("Humanoid")
        if h then h.WalkSpeed = 70 end
    end
end, yPos)
yPos = yPos + 30
createNeonButton("Прыжок", function(state) jumpModActive = state end, yPos)
yPos = yPos + 30
createNeonButton("Тень", function(state)
    shadowActive = state
    if state then updateShadow() end
end, yPos)
yPos = yPos + 30

-- ==========================================
-- === СЛАЙДЕР FOV С ОТКЛЮЧЕНИЕМ ПРИ СВЁРТКЕ ===
-- ==========================================
local sliderFrame = Instance.new("Frame")
sliderFrame.Size = UDim2.new(1, 0, 0, 24)
sliderFrame.Position = UDim2.new(0, 0, 0, yPos + 4)
sliderFrame.BackgroundColor3 = Color3.fromRGB(8, 8, 20)
sliderFrame.BackgroundTransparency = 0.2
sliderFrame.BorderSizePixel = 1
sliderFrame.BorderColor3 = Color3.fromRGB(0, 200, 255)
sliderFrame.Parent = btnContainer
local sliderCorner = Instance.new("UICorner")
sliderCorner.CornerRadius = UDim.new(0, 4)
sliderCorner.Parent = sliderFrame

local sliderGlow = Instance.new("Frame")
sliderGlow.Size = UDim2.new(1, 4, 1, 4)
sliderGlow.Position = UDim2.new(0, -2, 0, -2)
sliderGlow.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
sliderGlow.BackgroundTransparency = 0.9
sliderGlow.BorderSizePixel = 0
sliderGlow.ZIndex = -1
sliderGlow.Parent = sliderFrame
local sliderGlowCorner = Instance.new("UICorner")
sliderGlowCorner.CornerRadius = UDim.new(0, 6)
sliderGlowCorner.Parent = sliderGlow

local sliderLabel = Instance.new("TextLabel")
sliderLabel.Size = UDim2.new(0.5, 0, 1, 0)
sliderLabel.Position = UDim2.new(0, 6, 0, 0)
sliderLabel.BackgroundTransparency = 1
sliderLabel.Text = "FOV: 0"
sliderLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
sliderLabel.TextScaled = true
sliderLabel.Font = Enum.Font.GothamMedium
sliderLabel.TextXAlignment = Enum.TextXAlignment.Left
sliderLabel.Parent = sliderFrame

local sliderBtn = Instance.new("TextButton")
sliderBtn.Size = UDim2.new(0.18, 0, 0.5, 0)
sliderBtn.Position = UDim2.new(0.08, 0, 0.25, 0)
sliderBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
sliderBtn.BackgroundTransparency = 0.2
sliderBtn.Text = ""
sliderBtn.BorderSizePixel = 1
sliderBtn.BorderColor3 = Color3.fromRGB(255, 255, 255)
sliderBtn.Parent = sliderFrame
local sliderBtnCorner = Instance.new("UICorner")
sliderBtnCorner.CornerRadius = UDim.new(0, 4)
sliderBtnCorner.Parent = sliderBtn

local dragging = false
local menuVisible = true  -- флаг видимости меню

local function updateFOV(value)
    fovValue = math.clamp(value, 0, 150)
    sliderLabel.Text = "FOV: " .. math.floor(fovValue)
    local percent = fovValue / 150
    local minPos = 0.08
    local maxPos = 0.78
    local newPos = minPos + (maxPos - minPos) * percent
    sliderBtn.Position = UDim2.new(newPos, 0, 0.25, 0)
end

local function onMouseMove()
    -- Если меню свёрнуто, не обрабатываем движение
    if not dragging or not menuVisible then return end
    local mouse = player:GetMouse()
    local sliderX = mouse.X - sliderFrame.AbsolutePosition.X
    local width = sliderFrame.AbsoluteSize.X
    if width <= 0 then return end
    local percent = math.clamp(sliderX / width, 0, 1)
    local newValue = math.floor(percent * 150)
    updateFOV(newValue)
end

-- Обработчики событий с проверкой menuVisible
sliderBtn.MouseButton1Down:Connect(function()
    if menuVisible then
        dragging = true
    end
end)

sliderBtn.MouseButton1Up:Connect(function()
    dragging = false
end)

-- Отключаем реакцию на движение, если меню свёрнуто
runService.RenderStepped:Connect(function()
    if dragging and menuVisible then
        onMouseMove()
    end
end)

sliderBtn.MouseEnter:Connect(function()
    if menuVisible then
        tweenService:Create(sliderBtn, TweenInfo.new(0.1), {Size = UDim2.new(0.22, 0, 0.55, 0)}):Play()
    end
end)
sliderBtn.MouseLeave:Connect(function()
    if menuVisible then
        tweenService:Create(sliderBtn, TweenInfo.new(0.1), {Size = UDim2.new(0.18, 0, 0.5, 0)}):Play()
    end
end)

-- Кнопка сворачивания
local toggleBtn = Instance.new("TextButton")
toggleBtn.Size = UDim2.new(0, 22, 0, 22)
toggleBtn.Position = UDim2.new(1, -28, 0, 3)
toggleBtn.BackgroundColor3 = Color3.fromRGB(255, 0, 128)
toggleBtn.BackgroundTransparency = 0.3
toggleBtn.Text = "−"
toggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
toggleBtn.TextScaled = true
toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.BorderSizePixel = 0
toggleBtn.Parent = mainFrame
local toggleCorner = Instance.new("UICorner")
toggleCorner.CornerRadius = UDim.new(0, 6)
toggleCorner.Parent = toggleBtn

local toggleGlow = Instance.new("Frame")
toggleGlow.Size = UDim2.new(1, 6, 1, 6)
toggleGlow.Position = UDim2.new(0, -3, 0, -3)
toggleGlow.BackgroundColor3 = Color3.fromRGB(255, 0, 128)
toggleGlow.BackgroundTransparency = 0.85
toggleGlow.BorderSizePixel = 0
toggleGlow.ZIndex = -1
toggleGlow.Parent = toggleBtn
local toggleGlowCorner = Instance.new("UICorner")
toggleGlowCorner.CornerRadius = UDim.new(0, 8)
toggleGlowCorner.Parent = toggleGlow

local allControls = {mainFrame:GetChildren()}
toggleBtn.MouseButton1Click:Connect(function()
    menuVisible = not menuVisible
    if menuVisible then
        toggleBtn.Text = "−"
        toggleBtn.BackgroundColor3 = Color3.fromRGB(255, 0, 128)
        toggleGlow.BackgroundColor3 = Color3.fromRGB(255, 0, 128)
        for _, child in pairs(allControls) do
            child.Visible = true
        end
        mainFrame.Size = UDim2.new(0, 220, 0, 320)
        -- Включаем активность ползунка
        sliderBtn.Active = true
    else
        toggleBtn.Text = "+"
        toggleBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
        toggleGlow.BackgroundColor3 = Color3.fromRGB(0, 200, 255)
        for _, child in pairs(allControls) do
            if child ~= toggleBtn and child ~= toggleGlow and child ~= mainFrame then
                child.Visible = false
            end
        end
        mainFrame.Size = UDim2.new(0, 220, 0, 28)
        -- ОТКЛЮЧАЕМ ПОЛЗУНОК
        sliderBtn.Active = false
        dragging = false
    end
end)

-- Запуск теневого цикла
runService.RenderStepped:Connect(function()
    if shadowActive then
        updateShadow()
    end
end)

print("Arsenal PRO v.46.2 – FOV полностью отключается при свёртке меню.")
