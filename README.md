-- Servicios
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService("CoreGui")

-- Local
local LocalPlayer = Players.LocalPlayer

-- ⚙️ CONFIGURACIÓN
local HITBOX_COLOR = Color3.new(1, 1, 1) -- Blanco
local HITBOX_TRANSPARENCY = 0.45 -- Transparencia: 0=opaco | 1=invisible
local MIN_SIZE = 2
local MAX_SIZE = 35
local DEFAULT_SIZE = 6

-- Estado
local HitboxEnabled = true
local HitboxSizeValue = DEFAULT_SIZE
local Hitboxes = {}
local Allies = {}

-- GUI
local ScreenGui = nil
local MainFrame = nil
local ToggleBtn = nil
local SizeLabel = nil
local SliderBg = nil
local SliderFill = nil
local SliderBtn = nil
local isDragging = false

-- ==============================================
-- FUNCIONES
-- ==============================================

local function ValidateTarget(player)
    if not player or player == LocalPlayer or not player.Character then return false end
    if Allies[player.UserId] then return false end
    local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
    return humanoid and humanoid.Health > 0
end

local function ClearHitboxes()
    for _, hb in pairs(Hitboxes) do pcall(function() hb:Destroy() end) end
    table.clear(Hitboxes)
end

local function CreateHitbox(player)
    if not ValidateTarget(player) then return end
    if Hitboxes[player] then return end

    local root = player.Character:FindFirstChild("HumanoidRootPart")
    if not root then return end

    local size = math.clamp(HitboxSizeValue, MIN_SIZE, MAX_SIZE)
    local hb = Instance.new("Part")
    hb.Name = "SAK_Hitbox"
    hb.Anchored = false
    hb.CanCollide = false
    hb.CanQuery = true
    hb.CanTouch = false
    hb.Color = HITBOX_COLOR
    hb.Size = Vector3.new(size, size, size)
    hb.Transparency = HITBOX_TRANSPARENCY
    hb.Material = Enum.Material.Neon
    hb.Massless = true
    hb.Parent = player.Character

    local weld = Instance.new("Weld")
    weld.Part0 = root
    weld.Part1 = hb
    weld.Parent = hb

    Hitboxes[player] = hb
end

local function UpdateHitboxes()
    if not HitboxEnabled then ClearHitboxes() return end
    local size = math.clamp(HitboxSizeValue, MIN_SIZE, MAX_SIZE)

    -- Limpiar jugadores que se fueron
    for pl, hb in pairs(Hitboxes) do
        if not pl.Parent or not pl.Character or not ValidateTarget(pl) then
            pcall(function() hb:Destroy() end)
            Hitboxes[pl] = nil
        elseif hb and hb.Parent then
            hb.Size = Vector3.new(size, size, size)
        end
    end

    -- Crear nuevas
    for _, pl in ipairs(Players:GetPlayers()) do
        if not Hitboxes[pl] and ValidateTarget(pl) then
            CreateHitbox(pl)
        end
    end
end

-- ✅ Función para actualizar el valor del slider
local function UpdateSliderFromX(x)
    if not SliderBg then return end
    local absPos = x - SliderBg.AbsolutePosition.X
    local ratio = math.clamp(absPos / SliderBg.AbsoluteSize.X, 0, 1)
    HitboxSizeValue = math.floor(MIN_SIZE + ratio * (MAX_SIZE - MIN_SIZE))
    SizeLabel.Text = "Tamaño: " .. HitboxSizeValue
    SliderFill.Size = UDim2.new(ratio, 0, 1, 0)
    SliderBtn.Position = UDim2.new(ratio, -8, 0.5, -8)
end

-- ==============================================
-- CREAR GUI
-- ==============================================

local function CreateGUI()
    if ScreenGui and ScreenGui.Parent then return end
    local container = CoreGui
    pcall(function() if gethui then container = gethui() end end)

    ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "SAK_HitboxGUI"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    ScreenGui.Parent = container

    -- Marco principal
    MainFrame = Instance.new("Frame")
    MainFrame.Size = UDim2.new(0, 240, 0, 160)
    MainFrame.Position = UDim2.new(0.02, 0, 0.5, -80)
    MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 30)
    MainFrame.Active = true
    MainFrame.Draggable = true
    MainFrame.ClipsDescendants = false
    MainFrame.Parent = ScreenGui
    Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 12)

    local stroke = Instance.new("UIStroke")
    stroke.Thickness = 1.5
    stroke.Color = Color3.fromRGB(180, 180, 180)
    stroke.Parent = MainFrame

    -- Título
    local Title = Instance.new("TextLabel")
    Title.Size = UDim2.new(1, 0, 0, 35)
    Title.BackgroundTransparency = 1
    Title.Text = "🤍 HITBOX BLANCA"
    Title.Font = Enum.Font.GothamBold
    Title.TextSize = 15
    Title.TextColor3 = Color3.new(1, 1, 1)
    Title.Parent = MainFrame

    -- Botón Activar/Desactivar
    ToggleBtn = Instance.new("TextButton")
    ToggleBtn.Size = UDim2.new(1, -24, 0, 40)
    ToggleBtn.Position = UDim2.new(0, 12, 0, 40)
    ToggleBtn.BackgroundColor3 = Color3.fromRGB(50, 150, 70)
    ToggleBtn.Text = "✔ ACTIVADO"
    ToggleBtn.Font = Enum.Font.GothamBold
    ToggleBtn.TextSize = 14
    ToggleBtn.TextColor3 = Color3.new(1, 1, 1)
    ToggleBtn.AutoButtonColor = false
    ToggleBtn.Parent = MainFrame
    Instance.new("UICorner", ToggleBtn).CornerRadius = UDim.new(0, 8)

    -- Etiqueta de tamaño
    SizeLabel = Instance.new("TextLabel")
    SizeLabel.Size = UDim2.new(1, -24, 0, 22)
    SizeLabel.Position = UDim2.new(0, 12, 0, 92)
    SizeLabel.BackgroundTransparency = 1
    SizeLabel.Text = "Tamaño: " .. HitboxSizeValue
    SizeLabel.Font = Enum.Font.Gotham
    SizeLabel.TextSize = 12
    SizeLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    SizeLabel.TextXAlignment = Enum.TextXAlignment.Left
    SizeLabel.Parent = MainFrame

    -- Fondo del slider
    SliderBg = Instance.new("Frame")
    SliderBg.Size = UDim2.new(1, -24, 0, 10)
    SliderBg.Position = UDim2.new(0, 12, 0, 120)
    SliderBg.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
    SliderBg.Parent = MainFrame
    Instance.new("UICorner", SliderBg).CornerRadius = UDim.new(1, 0)

    -- Barra de progreso
    local initialRatio = (HitboxSizeValue - MIN_SIZE) / (MAX_SIZE - MIN_SIZE)
    SliderFill = Instance.new("Frame")
    SliderFill.Size = UDim2.new(initialRatio, 0, 1, 0)
    SliderFill.BackgroundColor3 = HITBOX_COLOR
    SliderFill.Parent = SliderBg
    Instance.new("UICorner", SliderFill).CornerRadius = UDim.new(1, 0)

    -- Botón del slider
    SliderBtn = Instance.new("TextButton")
    SliderBtn.Size = UDim2.new(0, 20, 0, 20)
    SliderBtn.Position = UDim2.new(initialRatio, -10, 0.5, -10)
    SliderBtn.BackgroundColor3 = Color3.new(1, 1, 1)
    SliderBtn.Text = ""
    SliderBtn.AutoButtonColor = false
    SliderBtn.ZIndex = 10
    SliderBtn.Parent = SliderBg
    Instance.new("UICorner", SliderBtn).CornerRadius = UDim.new(1, 0)

    -- ==============================================
    -- EVENTOS DEL SLIDER ✅ ARREGLADOS
    -- ==============================================

    SliderBtn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or 
           input.UserInputType == Enum.UserInputType.Touch then
            isDragging = true
        end
    end)

    SliderBg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or 
           input.UserInputType == Enum.UserInputType.Touch then
            isDragging = true
            UpdateSliderFromX(input.Position.X)
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or 
           input.UserInputType == Enum.UserInputType.Touch then
            isDragging = false
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if isDragging and (input.UserInputType == Enum.UserInputType.MouseMovement or 
                            input.UserInputType == Enum.UserInputType.Touch) then
            UpdateSliderFromX(input.Position.X)
        end
    end)

    -- Botón de activar/desactivar
    ToggleBtn.MouseButton1Click:Connect(function()
        HitboxEnabled = not HitboxEnabled
        if HitboxEnabled then
            ToggleBtn.Text = "✔ ACTIVADO"
            ToggleBtn.BackgroundColor3 = Color3.fromRGB(50, 150, 70)
            for _, p in ipairs(Players:GetPlayers()) do task.spawn(function() CreateHitbox(p) end) end
        else
            ToggleBtn.Text = "✖ DESACTIVADO"
            ToggleBtn.BackgroundColor3 = Color3.fromRGB(170, 45, 45)
            ClearHitboxes()
        end
    end)
end

-- ==============================================
-- INICIO
-- ==============================================

CreateGUI()

-- Bucle principal
RunService.RenderStepped:Connect(UpdateHitboxes)

-- Recargar al reaparecer
LocalPlayer.CharacterAdded:Connect(function()
    task.wait(0.6)
    ClearHitboxes()
    if HitboxEnabled then
        task.wait(0.3)
        for _, p in ipairs(Players:GetPlayers()) do task.spawn(function() CreateHitbox(p) end) end
    end
end)

-- Actualizar cuando se una un jugador nuevo
Players.PlayerAdded:Connect(function(player)
    player.CharacterAdded:Connect(function()
        task.wait(0.5)
        if HitboxEnabled then
            CreateHitbox(player)
        end
    end)
end)
