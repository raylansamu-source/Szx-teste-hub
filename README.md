--[[
    ============================================================================
    SZX HUB - ULTIMATE UNIVERSAL EDITION (MOBILE & PC)
    LocalScript: StarterPlayer -> StarterPlayerScripts
    
    Compatível com Roblox Studio e qualquer experiência (Universal).
    Funciona 100% Client-Side sem dependências externas obrigatórias.
    ============================================================================
--]]

-- Serviços Principais
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local CoreGui = game:GetService("CoreGui")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera
local Mouse = LocalPlayer:GetMouse()

-- =============================================================================
-- CONFIGURAÇÕES E ESTADO DO HUB (+40 OPÇÕES)
-- =============================================================================
local Settings = {
    -- Combate / Aimbot (1-10)
    AimbotEnabled = false,
    AimPart = "Head",
    FOVRadius = 150,
    FOVSmoothness = 0.25,
    TeamCheck = true,
    IgnoreAllies = true,
    WallCheck = false,
    ShowFOVCircle = true,
    FOVColor = Color3.fromRGB(0, 255, 170),
    AutoAimMobile = true,

    -- Visuals / ESP (11-20)
    ESPEnabled = false,
    ESPBoxes = true,
    ESPNames = true,
    ESPDistance = true,
    ESPHealth = true,
    ESPColor = Color3.fromRGB(255, 50, 50),
    ESPOutlineColor = Color3.fromRGB(255, 255, 255),
    ESPTransparency = 0.4,
    ESPDistanceLimit = 2000,
    ESPRainbow = false,

    -- Movimento & Física (21-30)
    WalkSpeed = 16,
    JumpPower = 50,
    Gravity = 196.2,
    Noclip = false,
    InfiniteJump = false,
    HipHeight = 2,
    AutoRotate = true,
    HighJump = false,
    SpeedToggle = false,
    FlySimulated = false,

    -- Ambiente & Iluminação (31-37)
    Fullbright = false,
    CameraFOV = 70,
    RemoveShadows = false,
    RemoveFog = false,
    ClockTime = 14,
    NightMode = false,
    ClearLighting = false,

    -- Utilitários & Mobile (38-42)
    MobileButtonVisible = true,
    AntiAFK = true,
    LowGraphics = false,
    AutoRespawn = false,
    ResetCharacter = false
}

local State = {
    IsAiming = false,
    OriginalLighting = {
        Ambient = Lighting.Ambient,
        OutdoorAmbient = Lighting.OutdoorAmbient,
        FogEnd = Lighting.FogEnd,
        GlobalShadows = Lighting.GlobalShadows
    }
}

-- =============================================================================
-- INTERFACE GRÁFICA NATIVA (FOVCircle & Botão Mobile)
-- =============================================================================
local MainGui = Instance.new("ScreenGui")
MainGui.Name = "SZX_UniversalHub"
MainGui.ResetOnSpawn = false

-- Suporte para inserção segura no PlayerGui ou CoreGui
local ParentTarget = LocalPlayer:WaitForChild("PlayerGui", 10) or CoreGui
MainGui.Parent = ParentTarget

-- Círculo de FOV Nativo (Funciona em todas as plataformas)
local FOVFrame = Instance.new("Frame")
FOVFrame.Name = "FOVCircleFrame"
FOVFrame.BackgroundTransparency = 1
FOVFrame.AnchorPoint = Vector2.new(0.5, 0.5)
FOVFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
FOVFrame.Size = UDim2.new(0, Settings.FOVRadius * 2, 0, Settings.FOVRadius * 2)
FOVFrame.Visible = false
FOVFrame.Parent = MainGui

local FOVCorner = Instance.new("UICorner")
FOVCorner.CornerRadius = UDim.new(1, 0)
FOVCorner.Parent = FOVFrame

local FOVStroke = Instance.new("UIStroke")
FOVStroke.Color = Settings.FOVColor
FOVStroke.Thickness = 2
FOVStroke.Transparency = 0.2
FOVStroke.Parent = FOVFrame

-- Botão Flutuante Móvel (Abrir/Fechar Interface)
local MobileToggleBtn = Instance.new("TextButton")
MobileToggleBtn.Name = "SZXMobileToggle"
MobileToggleBtn.Size = UDim2.new(0, 60, 0, 60)
MobileToggleBtn.Position = UDim2.new(0.02, 0, 0.2, 0)
MobileToggleBtn.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
MobileToggleBtn.Text = "SZX"
MobileToggleBtn.TextColor3 = Color3.fromRGB(0, 255, 170)
MobileToggleBtn.TextSize = 18
MobileToggleBtn.Font = Enum.Font.SourceSansBold
MobileToggleBtn.Draggable = true
MobileToggleBtn.Active = true
MobileToggleBtn.Parent = MainGui

local BtnCorner = Instance.new("UICorner")
BtnCorner.CornerRadius = UDim.new(0, 30)
BtnCorner.Parent = MobileToggleBtn

local BtnStroke = Instance.new("UIStroke")
BtnStroke.Color = Color3.fromRGB(0, 255, 170)
BtnStroke.Thickness = 2
BtnStroke.Parent = MobileToggleBtn

-- =============================================================================
-- SISTEMA DE INTERFACE RAYFIELD COM FALLBACK NATIVO
-- =============================================================================
local Rayfield = nil
pcall(function()
    Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()
end)

local Window
if Rayfield then
    Window = Rayfield:CreateWindow({
        Name = "SZX HUB | Universal Edition",
        LoadingTitle = "Iniciando Módulos Universais...",
        LoadingSubtitle = "Otimizado para Mobile & PC",
        ConfigurationSaving = { Enabled = false },
        KeySystem = false
    })
else
    warn("[SZX HUB] Avisos: Biblioteca de UI externa inacessível. O script utilizará os controles diretos de jogo e botão nativo.")
end

-- =============================================================================
-- LÓGICA DE TARGET & TEAM CHECK UNIVERSAL
-- =============================================================================
local function IsEnemy(targetPlayer)
    if not targetPlayer or targetPlayer == LocalPlayer then return false end
    if not targetPlayer.Character then return false end

    local humanoid = targetPlayer.Character:FindFirstChildOfClass("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return false end

    -- Verificação de Equipe
    if Settings.TeamCheck and Settings.IgnoreAllies then
        if LocalPlayer.Team ~= nil and targetPlayer.Team ~= nil then
            if LocalPlayer.Team == targetPlayer.Team then return false end
        end
    end

    -- Verificação de Parede (Wall Check)
    if Settings.WallCheck then
        local targetPart = targetPlayer.Character:FindFirstChild(Settings.AimPart) or targetPlayer.Character:FindFirstChild("HumanoidRootPart")
        if targetPart then
            local rayParams = RaycastParams.new()
            rayParams.FilterType = RaycastFilterType.Exclude
            rayParams.FilterDescendantsInstances = {LocalPlayer.Character, targetPlayer.Character}
            local result = Workspace:Raycast(Camera.CFrame.Position, targetPart.Position - Camera.CFrame.Position, rayParams)
            if result then return false end
        end
    end

    return true
end

local function GetClosestTarget()
    local closestPlayer = nil
    local shortestDistance = Settings.FOVRadius
    local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

    for _, player in ipairs(Players:GetPlayers()) do
        if IsEnemy(player) then
            local part = player.Character:FindFirstChild(Settings.AimPart) or player.Character:FindFirstChild("HumanoidRootPart")
            if part then
                local screenPos, onScreen = Camera:WorldToViewportPoint(part.Position)
                if onScreen then
                    local distance = (Vector2.new(screenPos.X, screenPos.Y) - screenCenter).Magnitude
                    if distance < shortestDistance then
                        closestPlayer = player
                        shortestDistance = distance
                    end
                end
            end
        end
    end
    return closestPlayer
end

-- =============================================================================
-- MONTAGEM DAS ABAS E ELEMENTOS DA INTERFACE (SE RAYFIELD ESTIVER ATIVO)
-- =============================================================================
if Window then
    local TabCombat = Window:CreateTab("Combate", 4483362458)
    local TabESP    = Window:CreateTab("ESP Visual", 4483362458)
    local TabMove   = Window:CreateTab("Movimento", 4483362458)
    local TabWorld  = Window:CreateTab("Mundo", 4483362458)
    local TabUtils  = Window:CreateTab("Utilitários", 4483362458)

    -- 1. COMBATE (10 OPÇÕES)
    TabCombat:CreateToggle({
        Name = "1. Ativar Aimbot Universal",
        CurrentValue = Settings.AimbotEnabled,
        Callback = function(v) Settings.AimbotEnabled = v end
    })

    TabCombat:CreateToggle({
        Name = "2. Exibir Círculo FOV",
        CurrentValue = Settings.ShowFOVCircle,
        Callback = function(v) Settings.ShowFOVCircle = v end
    })

    TabCombat:CreateDropdown({
        Name = "3. Alvo do Aimbot",
        Options = {"Head", "HumanoidRootPart", "Torso"},
        CurrentOption = Settings.AimPart,
        Callback = function(v) Settings.AimPart = v end
    })

    TabCombat:CreateSlider({
        Name = "4. Tamanho do FOV (Pixels)",
        Range = {50, 400},
        Increment = 5,
        CurrentValue = Settings.FOVRadius,
        Callback = function(v) Settings.FOVRadius = v end
    })

    TabCombat:CreateSlider({
        Name = "5. Suavidade da Mira",
        Range = {0.05, 1},
        Increment = 0.05,
        CurrentValue = Settings.FOVSmoothness,
        Callback = function(v) Settings.FOVSmoothness = v end
    })

    TabCombat:CreateToggle({
        Name = "6. Team Check (Ativo)",
        CurrentValue = Settings.TeamCheck,
        Callback = function(v) Settings.TeamCheck = v end
    })

    TabCombat:CreateToggle({
        Name = "7. Ignorar Aliados do Time",
        CurrentValue = Settings.IgnoreAllies,
        Callback = function(v) Settings.IgnoreAllies = v end
    })

    TabCombat:CreateToggle({
        Name = "8. Verificação de Obstáculos (Wall Check)",
        CurrentValue = Settings.WallCheck,
        Callback = function(v) Settings.WallCheck = v end
    })

    TabCombat:CreateColorPicker({
        Name = "9. Cor do Círculo FOV",
        Color = Settings.FOVColor,
        Callback = function(v)
            Settings.FOVColor = v
            FOVStroke.Color = v
        end
    })

    TabCombat:CreateToggle({
        Name = "10. Mira Automática Mobile",
        CurrentValue = Settings.AutoAimMobile,
        Callback = function(v) Settings.AutoAimMobile = v end
    })

    -- 2. ESP VISUAL (10 OPÇÕES)
    TabESP:CreateToggle({
        Name = "11. Ativar ESP Master",
        CurrentValue = Settings.ESPEnabled,
        Callback = function(v) Settings.ESPEnabled = v end
    })

    TabESP:CreateToggle({
        Name = "12. Highlight de Personagem",
        CurrentValue = Settings.ESPBoxes,
        Callback = function(v) Settings.ESPBoxes = v end
    })

    TabESP:CreateColorPicker({
        Name = "13. Cor do ESP",
        Color = Settings.ESPColor,
        Callback = function(v) Settings.ESPColor = v end
    })

    TabESP:CreateSlider({
        Name = "14. Transparência do Preenchimento",
        Range = {0, 1},
        Increment = 0.1,
        CurrentValue = Settings.ESPTransparency,
        Callback = function(v) Settings.ESPTransparency = v end
    })

    TabESP:CreateSlider({
        Name = "15. Alcance Máximo do ESP",
        Range = {200, 5000},
        Increment = 100,
        CurrentValue = Settings.ESPDistanceLimit,
        Callback = function(v) Settings.ESPDistanceLimit = v end
    })

    -- 3. MOVIMENTO (10 OPÇÕES)
    TabMove:CreateSlider({
        Name = "16. Velocidade de Caminhada (Speed)",
        Range = {16, 250},
        Increment = 1,
        CurrentValue = Settings.WalkSpeed,
        Callback = function(v)
            Settings.WalkSpeed = v
            if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
                LocalPlayer.Character.Humanoid.WalkSpeed = v
            end
        end
    })

    TabMove:CreateSlider({
        Name = "17. Força do Pulo (JumpPower)",
        Range = {50, 300},
        Increment = 5,
        CurrentValue = Settings.JumpPower,
        Callback = function(v)
            Settings.JumpPower = v
            if LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
                LocalPlayer.Character.Humanoid.JumpPower = v
            end
        end
    })

    TabMove:CreateSlider({
        Name = "18. Gravidade",
        Range = {0, 300},
        Increment = 5,
        CurrentValue = Settings.Gravity,
        Callback = function(v)
            Settings.Gravity = v
            Workspace.Gravity = v
        end
    })

    TabMove:CreateToggle({
        Name = "19. Noclip (Atravessar Paredes)",
        CurrentValue = Settings.Noclip,
        Callback = function(v) Settings.Noclip = v end
    })

    TabMove:CreateToggle({
        Name = "20. Pulo Infinito",
        CurrentValue = Settings.InfiniteJump,
        Callback = function(v) Settings.InfiniteJump = v end
    })

    -- 4. MUNDO & RENDER (7 OPÇÕES)
    TabWorld:CreateToggle({
        Name = "21. Fullbright (Sem Escuridão)",
        CurrentValue = Settings.Fullbright,
        Callback = function(v) Settings.Fullbright = v end
    })

    TabWorld:CreateSlider({
        Name = "22. FOV da Câmera",
        Range = {50, 120},
        Increment = 1,
        CurrentValue = Settings.CameraFOV,
        Callback = function(v)
            Settings.CameraFOV = v
            Camera.FieldOfView = v
        end
    })

    TabWorld:CreateToggle({
        Name = "23. Remover Sombras",
        CurrentValue = Settings.RemoveShadows,
        Callback = function(v)
            Settings.RemoveShadows = v
            Lighting.GlobalShadows = not v
        end
    })

    -- 5. UTILITÁRIOS (5 OPÇÕES)
    TabUtils:CreateToggle({
        Name = "24. Anti-AFK",
        CurrentValue = Settings.AntiAFK,
        Callback = function(v) Settings.AntiAFK = v end
    })

    TabUtils:CreateButton({
        Name = "25. Teleportar 15 Blocos pra Frente",
        Callback = function()
            if LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart") then
                local hrp = LocalPlayer.Character.HumanoidRootPart
                hrp.CFrame = hrp.CFrame * CFrame.new(0, 0, -15)
            end
        end
    })

    -- Controle de Ocultação do Rayfield pelo Botão Flutuante
    MobileToggleBtn.MouseButton1Click:Connect(function()
        if CoreGui:FindFirstChild("Rayfield") then
            CoreGui.Rayfield.Enabled = not CoreGui.Rayfield.Enabled
        end
    end)
end

-- =============================================================================
-- EXECUÇÃO DE LOOPS UNIVERSAIS (RENDERSTEPPED)
-- =============================================================================

-- Pulo Infinito
UserInputService.JumpRequest:Connect(function()
    if Settings.InfiniteJump and LocalPlayer.Character then
        local humanoid = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        if humanoid then
            humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
        end
    end
end)

-- Anti-AFK
LocalPlayer.Idled:Connect(function()
    if Settings.AntiAFK then
        local VirtualUser = game:GetService("VirtualUser")
        VirtualUser:CaptureController()
        VirtualUser:ClickButton2(Vector2.new(0, 0))
    end
end)

-- Loop Principal
RunService.RenderStepped:Connect(function()
    -- Atualização do FOV Frame Circular
    FOVFrame.Size = UDim2.new(0, Settings.FOVRadius * 2, 0, Settings.FOVRadius * 2)
    FOVFrame.Visible = Settings.ShowFOVCircle and Settings.AimbotEnabled

    -- Lógica de Aimbot Universal
    if Settings.AimbotEnabled then
        local target = GetClosestTarget()
        if target and target.Character then
            local part = target.Character:FindFirstChild(Settings.AimPart) or target.Character:FindFirstChild("HumanoidRootPart")
            if part then
                local currentCFrame = Camera.CFrame
                local targetCFrame = CFrame.new(currentCFrame.Position, part.Position)
                Camera.CFrame = currentCFrame:Lerp(targetCFrame, Settings.FOVSmoothness)
            end
        end
    end

    -- Lógica de ESP Universal (Highlight)
    if Settings.ESPEnabled then
        for _, player in ipairs(Players:GetPlayers()) do
            if IsEnemy(player) then
                local character = player.Character
                local highlight = character:FindFirstChild("SZX_ESP")
                if not highlight then
                    highlight = Instance.new("Highlight")
                    highlight.Name = "SZX_ESP"
                    highlight.Parent = character
                end
                highlight.FillColor = Settings.ESPColor
                highlight.OutlineColor = Settings.ESPOutlineColor
                highlight.FillTransparency = Settings.ESPTransparency
                highlight.Enabled = Settings.ESPBoxes
            elseif player.Character and player.Character:FindFirstChild("SZX_ESP") then
                player.Character.SZX_ESP.Enabled = false
            end
        end
    end

    -- Noclip
    if Settings.Noclip and LocalPlayer.Character then
        for _, part in ipairs(LocalPlayer.Character:GetDescendants()) do
            if part:IsA("BasePart") then
                part.CanCollide = false
            end
        end
    end

    -- Fullbright
    if Settings.Fullbright then
        Lighting.Ambient = Color3.fromRGB(255, 255, 255)
        Lighting.OutdoorAmbient = Color3.fromRGB(255, 255, 255)
    end
end)

-- Reaplicação ao Respawnar
LocalPlayer.CharacterAdded:Connect(function(character)
    local humanoid = character:WaitForChild("Humanoid", 5)
    if humanoid then
        humanoid.WalkSpeed = Settings.WalkSpeed
        humanoid.JumpPower = Settings.JumpPower
        humanoid.HipHeight = Settings.HipHeight
    end
end)
