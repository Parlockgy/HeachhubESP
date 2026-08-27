-- =========================================================================
-- FUNHOUSE MULTI-FRAMEWORK (v8.5.0)
-- PROTEÇÃO ANTI-PERDA DE DADOS + RECUPERADOR DE SKINS + AUTO LOBBY
-- =========================================================================

local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()
local SaveManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/SaveManager.lua"))()
local InterfaceManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/InterfaceManager.lua"))()

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Lighting = game:GetService("Lighting")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- Gerenciador de ciclo de vida
local CurrentWindow = nil
local ActiveConnections = {}
local ActiveLoops = {}

local function cancelAllTasks()
    for _, conn in ipairs(ActiveConnections) do
        if conn and conn.Disconnect then
            pcall(function() conn:Disconnect() end)
        end
    end
    table.clear(ActiveConnections)
    
    for key, _ in pairs(ActiveLoops) do
        ActiveLoops[key] = false
    end
end

-- =========================================================
-- SISTEMA DE BOTÃO FLUTUANTE (ROUND, DRAGGABLE & BORDA PRETA)
-- =========================================================
local GuiParent = (gethui and gethui()) or (pcall(function() return game:GetService("CoreGui") end) and game:GetService("CoreGui")) or PlayerGui

local ToggleGui = Instance.new("ScreenGui")
ToggleGui.Name = "FunhouseFloatingWidget"
ToggleGui.ResetOnSpawn = false
ToggleGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ToggleGui.Parent = GuiParent

local ToggleButton = Instance.new("ImageButton")
ToggleButton.Name = "OpenButton"
ToggleButton.Size = UDim2.fromOffset(56, 56)
ToggleButton.Position = UDim2.new(0.04, 0, 0.45, 0)
ToggleButton.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
ToggleButton.BackgroundTransparency = 0.1
-- rbxthumb garante que a imagem carregue para qualquer player
ToggleButton.Image = "rbxthumb://type=Asset&id=100656561588778&w=420&h=420"
ToggleButton.Visible = false
ToggleButton.Active = true
ToggleButton.AutoButtonColor = false
ToggleButton.Parent = ToggleGui

local UICorner = Instance.new("UICorner")
UICorner.CornerRadius = UDim.new(1, 0)
UICorner.Parent = ToggleButton

local UIStroke = Instance.new("UIStroke")
UIStroke.Color = Color3.fromRGB(0, 0, 0) -- Borda Preta
UIStroke.Thickness = 2.5
UIStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
UIStroke.Parent = ToggleButton

local dragging, dragInput, dragStart, startPos

local function updateDrag(input)
    local delta = input.Position - dragStart
    local targetPos = UDim2.new(
        startPos.X.Scale,
        startPos.X.Offset + delta.X,
        startPos.Y.Scale,
        startPos.Y.Offset + delta.Y
    )
    TweenService:Create(ToggleButton, TweenInfo.new(0.08, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {Position = targetPos}):Play()
end

ToggleButton.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPos = ToggleButton.Position
        
        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

ToggleButton.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        updateDrag(input)
    end
end)

ToggleButton.MouseButton1Click:Connect(function()
    if CurrentWindow then
        ToggleButton.Visible = false
        if CurrentWindow.Minimize then
            CurrentWindow:Minimize()
        end
    end
end)

local function bindMinimizeListener(window)
    if not window then return end
    task.spawn(function()
        while CurrentWindow == window do
            if window.Minimized ~= nil then
                if window.Minimized and not ToggleButton.Visible then
                    ToggleButton.Visible = true
                elseif not window.Minimized and ToggleButton.Visible then
                    ToggleButton.Visible = false
                end
            end
            task.wait(0.2)
        end
    end)
end

-- =========================================================
-- FUNÇÕES DE UTILIDADE E ANTI-LAG
-- =========================================================
local function enableLagReducer()
    Lighting.GlobalShadows = false
    Lighting.FogEnd = 9e9
    Lighting.Brightness = 1
    settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
    
    for _, v in ipairs(Workspace:GetDescendants()) do
        if v:IsA("BasePart") and not v:IsA("MeshPart") then
            v.Material = Enum.Material.SmoothPlastic
            v.Reflectance = 0
        elseif v:IsA("Decal") or v:IsA("Texture") then
            v.Transparency = 0.5
        elseif v:IsA("ParticleEmitter") or v:IsA("Trail") then
            v.Lifetime = NumberRange.new(0)
            v.Enabled = false
        end
    end
end

local function safeClickGui(guiElement)
    if not guiElement then return end
    if firesignal then
        pcall(function() firesignal(guiElement.MouseButton1Click) end)
        pcall(function() firesignal(guiElement.MouseButton1Down) end)
        pcall(function() firesignal(guiElement.Activated) end)
    end
    local pos = guiElement.AbsolutePosition + (guiElement.AbsoluteSize / 2)
    VirtualInputManager:SendMouseButtonEvent(pos.X, pos.Y, 0, true, game, 0)
    task.wait(0.005)
    VirtualInputManager:SendMouseButtonEvent(pos.X, pos.Y, 0, false, game, 0)
end

local function fastScreenClick(x, y)
    local vp = Workspace.CurrentCamera.ViewportSize
    x = x or (vp.X / 2)
    y = y or (vp.Y / 2)
    VirtualInputManager:SendMouseButtonEvent(x, y, 0, true, game, 0)
    VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.Space, false, game)
    task.wait(0.005)
    VirtualInputManager:SendMouseButtonEvent(x, y, 0, false, game, 0)
    VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.Space, false, game)
end

local function getRootPart()
    local character = LocalPlayer.Character
    return character and character:FindFirstChild("HumanoidRootPart")
end

local function isLobby()
    return Workspace:FindFirstChild("Lobby") ~= nil 
        or Workspace:FindFirstChild("Grinwell") ~= nil 
        or PlayerGui:FindFirstChild("LobbyUI") ~= nil 
        or PlayerGui:FindFirstChild("LobbyButtons") ~= nil
end

-- =========================================================================
-- MÓDULO DE RECUPERAÇÃO E SINCRONIZAÇÃO DE SKINS
-- =========================================================================
local function forceRecoverSkins()
    Fluent:Notify({
        Title = "Recuperador de Skins",
        Content = "Iniciando verificação e re-sincronização do servidor...",
        Duration = 4
    })
    
    local shopRemotes = ReplicatedStorage:FindFirstChild("ShopRemotes")
    if not shopRemotes then
        Fluent:Notify({ Title = "Erro", Content = "ShopRemotes não encontrado!", Duration = 3 })
        return
    end

    local getShopList = shopRemotes:FindFirstChild("GetShopList")
    local getShopVariants = shopRemotes:FindFirstChild("GetShopVariants")
    local buyVariant = shopRemotes:FindFirstChild("BuyVariant")
    local getCoins = shopRemotes:FindFirstChild("GetCoins") or shopRemotes:FindFirstChild("GetRemnants")
    local isOwned = shopRemotes:FindFirstChild("IsOwned")

    -- 1. Força a leitura do servidor para atualizar o cache
    pcall(function()
        if getCoins then getCoins:InvokeServer() end
        if getShopList then getShopList:InvokeServer() end
    end)
    task.wait(0.5)

    -- 2. Tenta re-vincular e restaurar cada variante de cada personagem
    local allCharacters = {
        ["1"] = "Splits",
        ["2"] = "Manny",
        ["3"] = "Anton",
        ["4"] = "Freddie",
        ["5"] = "Mabel"
    }

    local knownVariants = {
        ["Splits"] = {"Default", "Mime", "Cinematic", "Duality", "Old"},
        ["Manny"] = {"Default", "JustAConcept", "Vincible", "Bee", "Purple", "Pizza"},
        ["Anton"] = {"Default", "OFF", "Fireant", "Clone", "Scout", "Slushie"},
        ["Freddie"] = {"Default", "Fazbutter", "Popcorn"},
        ["Mabel"] = {"Default", "Mabelicious"},
        ["Testy"] = {"Default", "Red"}
    }

    local restoredCount = 0

    for charId, charName in pairs(allCharacters) do
        -- Sincroniza variantes com o servidor
        pcall(function()
            if getShopVariants then
                getShopVariants:InvokeServer(charId)
                getShopVariants:InvokeServer(charName)
            end
        end)
        task.wait(0.1)

        -- Tenta ativar/reivindicar as skins padrão e de custo 0 caso tenham desvinculado
        if buyVariant then
            local list = knownVariants[charName] or {"Default"}
            for _, varName in ipairs(list) do
                pcall(function()
                    -- Tenta por ID e por Nome para cobrir ambos os tipos de remote do jogo
                    buyVariant:InvokeServer(charId, varName)
                    buyVariant:InvokeServer(charName, varName)
                    restoredCount = restoredCount + 1
                end)
                task.wait(0.08)
            end
        end
    end

    -- 3. Atualiza o ShopUI da PlayerGui se existir
    local shopGui = PlayerGui:FindFirstChild("Shop")
    if shopGui then
        local shopScript = shopGui:FindFirstChild("ShopUI")
        if shopScript and shopScript.Enabled then
            shopScript.Enabled = false
            task.wait(0.1)
            shopScript.Enabled = true
        end
    end

    Fluent:Notify({
        Title = "Sincronização Concluída",
        Content = "Tentativas de revalidação enviadas ao servidor!",
        Duration = 5
    })
end

-- =========================================================================
-- MÓDULO 1: LOBBY & SHOP HUB
-- =========================================================================
local function loadLobbyScript()
    cancelAllTasks()
    if CurrentWindow then
        pcall(function() CurrentWindow:Destroy() end)
    end

    CurrentWindow = Fluent:CreateWindow({
        Title = "Funhouse [LOBBY HUB]",
        SubTitle = "v8.5.0 (Auto Shop & Safe Data)",
        TabWidth = 160,
        Size = UDim2.fromOffset(580, 460),
        Acrylic = true,
        Theme = "Dark",
        MinimizeKey = Enum.KeyCode.LeftControl
    })

    bindMinimizeListener(CurrentWindow)

    local Tabs = {
        LobbyAuto = CurrentWindow:AddTab({ Title = "Party & Matchmaking", Icon = "play" }),
        ShopChars = CurrentWindow:AddTab({ Title = "Characters & Skins", Icon = "shirt" }),
        ShopPerks = CurrentWindow:AddTab({ Title = "Perks & Upgrades", Icon = "zap" }),
        ShopStickers = CurrentWindow:AddTab({ Title = "Stickers Shop", Icon = "smile" }),
        Recovery = CurrentWindow:AddTab({ Title = "Anti-Bug & Restore", Icon = "shield-check" }),
        Performance = CurrentWindow:AddTab({ Title = "FPS & Anti-Lag", Icon = "cpu" }),
        Settings = CurrentWindow:AddTab({ Title = "Configs", Icon = "settings" })
    }

    -- 1. Auto Create Lobby & Matchmaking
    Tabs.LobbyAuto:AddSection("Auto Create Lobby Configuration")

    local lobbyMaxPlayers = 1
    local lobbyFriendsOnly = false

    Tabs.LobbyAuto:AddSlider("MaxPlayersSlider", {
        Title = "Max Players",
        Description = "1 = Solo / Instantâneo | 2 até 6 = Com amigos ou outros",
        Default = 1,
        Min = 1,
        Max = 6,
        Rounding = 0,
        Callback = function(Value)
            lobbyMaxPlayers = Value
        end
    })

    Tabs.LobbyAuto:AddToggle("FriendsOnlyToggle", {
        Title = "Friends Only (Apenas Amizades)",
        Description = "Ative para bloquear a entrada de pessoas aleatórias",
        Default = false,
        Callback = function(Value)
            lobbyFriendsOnly = Value
        end
    })

    local function runAutoCreateLobby()
        local lobbyUI = PlayerGui:FindFirstChild("LobbyUI")
        if not lobbyUI then return end
        
        local mainFrame = lobbyUI:FindFirstChild("MainFrame")
        
        if not (mainFrame and mainFrame.Visible) then
            local enterPart = Workspace:FindFirstChild("Enter", true)
            local root = getRootPart()
            if enterPart and root then
                local clicker = enterPart:FindFirstChildWhichIsA("ClickDetector", true)
                if clicker and fireclickdetector then
                    pcall(function() fireclickdetector(clicker) end)
                end
                pcall(function()
                    firetouchinterest(root, enterPart, 0)
                    firetouchinterest(root, enterPart, 1)
                end)
                task.wait(0.2)
            end
        end

        mainFrame = lobbyUI:FindFirstChild("MainFrame")
        if mainFrame and mainFrame.Visible then
            -- 1. Friends Only
            local friendsContainer = mainFrame:FindFirstChild("FriendsOnlyContainer")
            local buttonIndicator = friendsContainer and friendsContainer:FindFirstChild("ButtonIndicator")
            if buttonIndicator then
                local isOn = buttonIndicator:GetAttribute("IsOn")
                if isOn == nil then
                    local idOn = buttonIndicator:FindFirstChild("IdON")
                    isOn = idOn and idOn.Visible
                end
                if isOn ~= lobbyFriendsOnly then
                    safeClickGui(buttonIndicator)
                    task.wait(0.1)
                end
            end

            -- 2. Quantidade de jogadores
            local countContainer = mainFrame:FindFirstChild("PlayerCountContainer")
            if countContainer then
                local countDisplay = countContainer:FindFirstChild("CountDisplay")
                local minusBtn = countContainer:FindFirstChild("PlayerMinus")
                local plusBtn = countContainer:FindFirstChild("PlayerPlus")
                
                if countDisplay and minusBtn and plusBtn then
                    local currentVal = tonumber(countDisplay.Text) or 1
                    local attempts = 0
                    while currentVal ~= lobbyMaxPlayers and attempts < 10 do
                        attempts = attempts + 1
                        if currentVal < lobbyMaxPlayers then
                            safeClickGui(plusBtn)
                        elseif currentVal > lobbyMaxPlayers then
                            safeClickGui(minusBtn)
                        end
                        task.wait(0.06)
                        currentVal = tonumber(countDisplay.Text) or lobbyMaxPlayers
                    end
                end
            end

            -- 3. Clica em Create Lobby
            local createBtn = mainFrame:FindFirstChild("CreateButton")
            if createBtn then
                safeClickGui(createBtn)
                task.wait(0.6)
            end
        end
    end

    Tabs.LobbyAuto:AddToggle("AutoCreateLobbyToggle", {
        Title = "Auto Create Lobby",
        Description = "Cria e inicia o lobby automaticamente",
        Default = false,
        Callback = function(Value)
            ActiveLoops["AutoCreateLobby"] = Value
            if Value then
                task.spawn(function()
                    while ActiveLoops["AutoCreateLobby"] do
                        runAutoCreateLobby()
                        task.wait(1.5)
                    end
                end)
            end
        end
    })

    -- 2. Loja de Personagens e Skins com Proteção Anti-Bug
    Tabs.ShopChars:AddSection("Pals & Skins (Com Proteção de Saldo e Posse)")

    local charMap = {
        ["Splits"] = "1",
        ["Manny"] = "2",
        ["Anton"] = "3",
        ["Freddie"] = "4",
        ["Mabel"] = "5"
    }

    local selectedCharToBuy = "All"

    Tabs.ShopChars:AddDropdown("CharSelectDropdown", {
        Title = "Select Character To Buy",
        Values = {"All", "Splits", "Manny", "Anton", "Freddie", "Mabel"},
        Multi = false,
        Default = "All",
        Callback = function(v) selectedCharToBuy = v end
    })

    Tabs.ShopChars:AddToggle("SafeAutoBuyCharToggle", {
        Title = "Safe Auto Buy Character(s)",
        Description = "Checa se você já possui antes de enviar a compra para não bugar o inventário",
        Default = false,
        Callback = function(v)
            ActiveLoops["AutoBuyChar"] = v
            if v then
                task.spawn(function()
                    local buyRemote = ReplicatedStorage:FindFirstChild("ShopRemotes") and ReplicatedStorage.ShopRemotes:FindFirstChild("BuyCharacter")
                    local getShopList = ReplicatedStorage:FindFirstChild("ShopRemotes") and ReplicatedStorage.ShopRemotes:FindFirstChild("GetShopList")
                    
                    while ActiveLoops["AutoBuyChar"] do
                        if buyRemote and getShopList then
                            local shopData = nil
                            pcall(function() shopData = getShopList:InvokeServer() end)
                            
                            if type(shopData) == "table" then
                                for idOrName, data in pairs(shopData) do
                                    local cName = type(data) == "table" and data.name or idOrName
                                    local isOwned = type(data) == "table" and data.owned == true
                                    
                                    if (selectedCharToBuy == "All" or selectedCharToBuy == cName) and not isOwned then
                                        pcall(function() buyRemote:InvokeServer(idOrName) end)
                                        pcall(function() buyRemote:InvokeServer(cName) end)
                                        task.wait(2.5) -- Delay de segurança anti-corrupção
                                    end
                                end
                            end
                        end
                        task.wait(3.5)
                    end
                end)
            end
        end
    })

    Tabs.ShopChars:AddToggle("SafeAutoBuyVariantsToggle", {
        Title = "Safe Auto Buy Skins/Variantes",
        Description = "Compre apenas variantes que você ainda não tem",
        Default = false,
        Callback = function(v)
            ActiveLoops["AutoBuyVar"] = v
            if v then
                task.spawn(function()
                    local buyVarRemote = ReplicatedStorage:FindFirstChild("ShopRemotes") and ReplicatedStorage.ShopRemotes:FindFirstChild("BuyVariant")
                    local getVars = ReplicatedStorage:FindFirstChild("ShopRemotes") and ReplicatedStorage.ShopRemotes:FindFirstChild("GetShopVariants")
                    
                    while ActiveLoops["AutoBuyVar"] do
                        if buyVarRemote and getVars then
                            for cName, cId in pairs(charMap) do
                                local variantList = nil
                                pcall(function() variantList = getVars:InvokeServer(cId) or getVars:InvokeServer(cName) end)
                                
    
