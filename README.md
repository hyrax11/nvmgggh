local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- ================== SERVIÇOS & VARIÁVEIS ==================
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local Mouse = LocalPlayer:GetMouse()

-- ================== CONFIGURAÇÕES GLOBAIS ==================
local Settings = {
    Aimlock = {
        Enabled = false,
        IsHeld = false, -- Controle interno de "segurando tecla"
        KeybindMode = "Hold", -- Hold (Segurar) ou Toggle (Alternar)
        Smoothing = 5,
        FOV = 100,
        TargetPart = "Head",
        ShowFOV = true
    },
    ESP = {
        Enabled = false,
        Chams = true,
        Names = true,
        FillColor = Color3.fromRGB(255, 0, 0),
        OutlineColor = Color3.fromRGB(255, 255, 255)
    }
}

-- Tabela de Limpeza
local Cleanup = { Connections = {}, ESPObjects = {} }

-- ================== SISTEMA DE AIMLOCK ==================
local DrawingLib = Drawing or require(script.Parent.Drawing)
local FOVCircle = DrawingLib.new("Circle")
_G.FOVCircle = FOVCircle
FOVCircle.Thickness = 2
FOVCircle.NumSides = 60
FOVCircle.Filled = false
FOVCircle.Transparency = 1

local function GetClosestPlayer()
    local target = nil
    local shortestDist = math.huge
    local mousePos = Vector2.new(Mouse.X, Mouse.Y)

    for _, v in pairs(Players:GetPlayers()) do
        if v ~= LocalPlayer and v.Character and v.Character:FindFirstChild("HumanoidRootPart") then
            -- Team Check
            if v.Team ~= LocalPlayer.Team or v.Team == nil then
                local part = v.Character:FindFirstChild(Settings.Aimlock.TargetPart)
                if part then
                    local screenPos, onScreen = Camera:WorldToViewportPoint(part.Position)
                    if onScreen then
                        local dist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                        if dist < Settings.Aimlock.FOV and dist < shortestDist then
                            shortestDist = dist
                            target = v
                        end
                    end
                end
            end
        end
    end
    return target
end

-- Loop Principal Aimlock
local AimLoop = RunService.RenderStepped:Connect(function()
    -- Atualiza Visual do FOV
    FOVCircle.Position = Vector2.new(Mouse.X, Mouse.Y + 36)
    FOVCircle.Radius = Settings.Aimlock.FOV
    FOVCircle.Visible = Settings.Aimlock.ShowFOV and Settings.Aimlock.Enabled
    FOVCircle.Color = Color3.fromRGB(255, 255, 255)

    -- Lógica de Mira Ativa
    -- Só mira se o sistema estiver habilitado E a tecla estiver sendo pressionada (IsHeld)
    if Settings.Aimlock.Enabled and Settings.Aimlock.IsHeld then
        local target = GetClosestPlayer()
        if target and target.Character then
            local targetPart = target.Character[Settings.Aimlock.TargetPart]
            local currentCF = Camera.CFrame
            local targetPos = targetPart.Position
            
            -- Cálculo de Rotação
            local lookAtCF = CFrame.new(currentCF.Position, targetPos)
            
            -- Suavização
            local alpha = 1 / math.max(Settings.Aimlock.Smoothing, 1)
            Camera.CFrame = currentCF:Lerp(lookAtCF, alpha)
        end
    end
end)
table.insert(Cleanup.Connections, AimLoop)

-- ================== SISTEMA DE ESP ==================
local function UpdateESP()
    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            if not Cleanup.ESPObjects[player] then Cleanup.ESPObjects[player] = {} end
            local visuals = Cleanup.ESPObjects[player]
            
            local isTeammate = (player.Team == LocalPlayer.Team and player.Team ~= nil)
            
            if not Settings.ESP.Enabled or isTeammate then
                if visuals.Highlight then visuals.Highlight:Destroy(); visuals.Highlight = nil end
                if visuals.Billboard then visuals.Billboard:Destroy(); visuals.Billboard = nil end
                continue
            end

            -- Chams
            if Settings.ESP.Chams then
                if not visuals.Highlight then
                    local hl = Instance.new("Highlight")
                    hl.FillTransparency = 0.5
                    hl.OutlineTransparency = 0
                    hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
                    hl.Parent = player.Character
                    visuals.Highlight = hl
                end
                visuals.Highlight.FillColor = Settings.ESP.FillColor
                visuals.Highlight.OutlineColor = Settings.ESP.OutlineColor
                visuals.Highlight.Parent = player.Character
            else
                if visuals.Highlight then visuals.Highlight:Destroy(); visuals.Highlight = nil end
            end

            -- Nomes
            if Settings.ESP.Names then
                if not visuals.Billboard then
                    local bg = Instance.new("BillboardGui")
                    bg.AlwaysOnTop = true
                    bg.Size = UDim2.new(0, 200, 0, 50)
                    bg.StudsOffset = Vector3.new(0, 3.5, 0)
                    bg.Parent = player.Character.Head
                    local lbl = Instance.new("TextLabel", bg)
                    lbl.BackgroundTransparency = 1
                    lbl.Size = UDim2.new(1,0,1,0)
                    lbl.TextColor3 = Color3.new(1,1,1)
                    lbl.TextStrokeTransparency = 0
                    lbl.Font = Enum.Font.GothamBold
                    visuals.Billboard = bg
                end
                local dist = math.floor((LocalPlayer.Character.HumanoidRootPart.Position - player.Character.HumanoidRootPart.Position).Magnitude)
                visuals.Billboard.Parent = player.Character.Head
                visuals.Billboard.Info.Text = string.format("%s\n[%dm]", player.Name, dist)
            else
                if visuals.Billboard then visuals.Billboard:Destroy(); visuals.Billboard = nil end
            end
        end
    end
end
local ESPLoop = RunService.RenderStepped:Connect(UpdateESP)
table.insert(Cleanup.Connections, ESPLoop)

-- ================== INTERFACE (UI) ==================
local Window = Rayfield:CreateWindow({
    Name = "🔥 Perplexity | V4 Ultimate",
    LoadingTitle = "Injetando...",
    ConfigurationSaving = { Enabled = false },
    KeySystem = false,
})

-- [[ ABA COMBATE ]] --
local CombatTab = Window:CreateTab("🔫 Combate", 4483362458)
local AimMain = CombatTab:CreateSection("Aimlock Principal")

CombatTab:CreateToggle({
    Name = "Habilitar Aimlock (Master Switch)",
    CurrentValue = false,
    Callback = function(v) Settings.Aimlock.Enabled = v end,
})

-- KEYBIND (TECLA DE ATALHO) ADICIONADA AQUI
CombatTab:CreateKeybind({
    Name = "Tecla de Mira (Segurar)",
    CurrentKeybind = "RightClick", -- Padrão: Botão Direito
    HoldToInteract = true, -- true = Segurar para ativar | false = Apertar para ligar/desligar
    Flag = "AimKeybind", 
    Callback = function(Keybind)
        -- Essa função é chamada automaticamente quando segura/solta a tecla
        Settings.Aimlock.IsHeld = Keybind 
    end,
})

CombatTab:CreateSlider({
    Name = "Aim Smoothing (Suavização)",
    Range = {1, 20},
    Increment = 1,
    Suffix = "Smooth",
    CurrentValue = 5,
    Callback = function(v) Settings.Aimlock.Smoothing = v end,
})

CombatTab:CreateSlider({
    Name = "Tamanho do FOV",
    Range = {50, 600},
    Increment = 10,
    CurrentValue = 100,
    Callback = function(v) Settings.Aimlock.FOV = v end,
})

CombatTab:CreateToggle({
    Name = "Mostrar Círculo FOV",
    CurrentValue = true,
    Callback = function(v) Settings.Aimlock.ShowFOV = v end,
})

CombatTab:CreateDropdown({
    Name = "Focar Parte do Corpo",
    Options = {"Head", "HumanoidRootPart", "UpperTorso"},
    CurrentOption = {"Head"},
    Callback = function(Option)
        Settings.Aimlock.TargetPart = Option[1]
    end,
})

-- [[ ABA VISUALS ]] --
local VisualTab = Window:CreateTab("👁 Visuals", 4483362458)
local ESPSection = VisualTab:CreateSection("ESP & Chams")

VisualTab:CreateToggle({
    Name = "Ativar ESP Mestre",
    CurrentValue = false,
    Callback = function(v) Settings.ESP.Enabled = v end,
})

VisualTab:CreateToggle({
    Name = "Chams (Através da Parede)",
    CurrentValue = true,
    Callback = function(v) Settings.ESP.Chams = v end,
})

VisualTab:CreateColorPicker({
    Name = "Cor do Chams",
    Color = Color3.fromRGB(255, 0, 0),
    Callback = function(v) Settings.ESP.FillColor = v end,
})

VisualTab:CreateColorPicker({
    Name = "Cor da Borda",
    Color = Color3.fromRGB(255, 255, 255),
    Callback = function(v) Settings.ESP.OutlineColor = v end,
})

VisualTab:CreateToggle({
    Name = "Mostrar Nomes",
    CurrentValue = true,
    Callback = function(v) Settings.ESP.Names = v end,
})

-- [[ ABA CONFIG ]] --
local ConfigTab = Window:CreateTab("⚙ Config", 4483362458)

ConfigTab:CreateButton({
    Name = "❌ UNLOAD (Desinjetar)",
    Callback = function()
        for _, conn in pairs(Cleanup.Connections) do conn:Disconnect() end
        for _, visual in pairs(Cleanup.ESPObjects) do
            if visual.Highlight then visual.Highlight:Destroy() end
            if visual.Billboard then visual.Billboard:Destroy() end
        end
        if _G.FOVCircle then _G.FOVCircle:Remove() end
        Rayfield:Destroy()
    end,
})

Rayfield:Notify({
    Title = "Script Atualizado!",
    Content = "Configure sua tecla de mira na aba Combate.",
    Duration = 5,
})
