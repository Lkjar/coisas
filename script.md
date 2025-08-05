-- Script de GUI para Gamepasses do Roblox
-- Criado por: snow
-- Discord: slay_hear

local Players = game:GetService("Players")
local MarketplaceService = game:GetService("MarketplaceService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

-- Variáveis globais
local gamepasses = {}
local selectedGamepass = nil

-- Função para detectar gamepasses do jogo
local function detectGamepasses()
    local foundGamepasses = {}
    
    -- Método 1: Procurar por scripts no workspace que mencionem IDs de gamepass
    local function searchForGamepassIds()
        local gamepassIds = {}
        
        -- Procura em todos os scripts do jogo por possíveis IDs de gamepass
        local function searchInInstance(instance)
            for _, child in pairs(instance:GetChildren()) do
                if child:IsA("Script") or child:IsA("LocalScript") or child:IsA("ModuleScript") then
                    local success, source = pcall(function()
                        return child.Source
                    end)
                    
                    if success and source then
                        -- Procura por padrões de ID de gamepass no código
                        for id in string.gmatch(source, "GamePassService:PlayerHasPass%(.*,%s*(%d+)%)") do
                            table.insert(gamepassIds, tonumber(id))
                        end
                        for id in string.gmatch(source, "MarketplaceService:PlayerOwnsAsset%(.*,%s*(%d+)%)") do
                            table.insert(gamepassIds, tonumber(id))
                        end
                        for id in string.gmatch(source, "gamepass[%s_]*[iI][dD][%s_]*=[%s_]*(%d+)") do
                            table.insert(gamepassIds, tonumber(id))
                        end
                    end
                end
                
                if child:GetChildren() and #child:GetChildren() > 0 then
                    searchInInstance(child)
                end
            end
        end
        
        pcall(function()
            searchInInstance(game.Workspace)
            searchInInstance(game.ServerScriptService)
            searchInInstance(game.ReplicatedStorage)
        end)
        
        return gamepassIds
    end
    
    -- Método 2: Testar IDs sequenciais mais amplos
    local function testSequentialIds()
        local gamepassIds = {}
        
        -- Testa uma faixa maior de IDs
        for i = 1, 200 do
            local success, info = pcall(function()
                return MarketplaceService:GetProductInfo(i, Enum.InfoType.GamePass)
            end)
            
            if success and info and info.Name then
                table.insert(gamepassIds, i)
            end
            
            wait(0.05) -- Reduz o delay mas ainda evita rate limiting
        end
        
        return gamepassIds
    end
    
    -- Método 3: Procurar por gamepasses relacionadas ao jogo atual
    local function getGameRelatedPasses()
        local gamepassIds = {}
        local gameId = game.PlaceId
        
        -- Tenta encontrar gamepasses baseadas no ID do jogo
        local startId = math.floor(gameId / 1000) * 1000
        local endId = startId + 1000
        
        for i = startId, endId, 10 do -- Testa a cada 10 IDs para ser mais rápido
            local success, info = pcall(function()
                return MarketplaceService:GetProductInfo(i, Enum.InfoType.GamePass)
            end)
            
            if success and info and info.Name then
                table.insert(gamepassIds, i)
            end
            
            wait(0.03)
        end
        
        return gamepassIds
    end
    
    -- Combina todos os métodos
    local allIds = {}
    
    -- Executa os métodos de detecção
    spawn(function()
        local scriptIds = searchForGamepassIds()
        for _, id in pairs(scriptIds) do
            table.insert(allIds, id)
        end
    end)
    
    local sequentialIds = testSequentialIds()
    for _, id in pairs(sequentialIds) do
        table.insert(allIds, id)
    end
    
    local gameRelatedIds = getGameRelatedPasses()
    for _, id in pairs(gameRelatedIds) do
        table.insert(allIds, id)
    end
    
    -- Remove duplicatas
    local uniqueIds = {}
    local seen = {}
    for _, id in pairs(allIds) do
        if not seen[id] then
            seen[id] = true
            table.insert(uniqueIds, id)
        end
    end
    
    -- Obtém informações detalhadas de cada gamepass encontrada
    for _, id in pairs(uniqueIds) do
        local success, info = pcall(function()
            return MarketplaceService:GetProductInfo(id, Enum.InfoType.GamePass)
        end)
        
        if success and info and info.Name then
            table.insert(foundGamepasses, {
                id = id,
                name = info.Name,
                description = info.Description or "Sem descrição disponível",
                price = info.PriceInRobux or 0
            })
        end
        
        wait(0.1)
    end
    
    return foundGamepasses
end

-- Função para criar a GUI principal
local function createMainGUI()
    -- ScreenGui principal
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "GamepassGUI"
    screenGui.Parent = playerGui
    screenGui.ResetOnSpawn = false
    
    -- Frame principal
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 500, 0, 400)
    mainFrame.Position = UDim2.new(0.5, -250, 0.5, -200)
    mainFrame.BackgroundColor3 = Color3.fromRGB(35, 35, 35)
    mainFrame.BorderSizePixel = 0
    mainFrame.Parent = screenGui
    
    -- Cantos arredondados
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 10)
    corner.Parent = mainFrame
    
    -- Título
    local titleLabel = Instance.new("TextLabel")
    titleLabel.Name = "Title"
    titleLabel.Size = UDim2.new(1, 0, 0, 50)
    titleLabel.Position = UDim2.new(0, 0, 0, 0)
    titleLabel.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
    titleLabel.BorderSizePixel = 0
    titleLabel.Text = "🎮 Gamepass Manager"
    titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    titleLabel.TextSize = 18
    titleLabel.Font = Enum.Font.GothamBold
    titleLabel.Parent = mainFrame
    
    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 10)
    titleCorner.Parent = titleLabel
    
    -- Frame para abas
    local tabFrame = Instance.new("Frame")
    tabFrame.Name = "TabFrame"
    tabFrame.Size = UDim2.new(1, 0, 0, 40)
    tabFrame.Position = UDim2.new(0, 0, 0, 55)
    tabFrame.BackgroundTransparency = 1
    tabFrame.Parent = mainFrame
    
    -- Botão aba Gamepasses
    local gamepassTab = Instance.new("TextButton")
    gamepassTab.Name = "GamepassTab"
    gamepassTab.Size = UDim2.new(0, 120, 1, 0)
    gamepassTab.Position = UDim2.new(0, 10, 0, 0)
    gamepassTab.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    gamepassTab.BorderSizePixel = 0
    gamepassTab.Text = "Gamepasses"
    gamepassTab.TextColor3 = Color3.fromRGB(255, 255, 255)
    gamepassTab.TextSize = 14
    gamepassTab.Font = Enum.Font.Gotham
    gamepassTab.Parent = tabFrame
    
    local gamepassTabCorner = Instance.new("UICorner")
    gamepassTabCorner.CornerRadius = UDim.new(0, 5)
    gamepassTabCorner.Parent = gamepassTab
    
    -- Botão aba Créditos
    local creditsTab = Instance.new("TextButton")
    creditsTab.Name = "CreditsTab"
    creditsTab.Size = UDim2.new(0, 120, 1, 0)
    creditsTab.Position = UDim2.new(0, 140, 0, 0)
    creditsTab.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    creditsTab.BorderSizePixel = 0
    creditsTab.Text = "Créditos"
    creditsTab.TextColor3 = Color3.fromRGB(255, 255, 255)
    creditsTab.TextSize = 14
    creditsTab.Font = Enum.Font.Gotham
    creditsTab.Parent = tabFrame
    
    local creditsTabCorner = Instance.new("UICorner")
    creditsTabCorner.CornerRadius = UDim.new(0, 5)
    creditsTabCorner.Parent = creditsTab
    
    return screenGui, mainFrame, gamepassTab, creditsTab
end

-- Função para criar a aba de gamepasses
local function createGamepassTab(parent)
    local gamepassFrame = Instance.new("Frame")
    gamepassFrame.Name = "GamepassFrame"
    gamepassFrame.Size = UDim2.new(1, -20, 1, -110)
    gamepassFrame.Position = UDim2.new(0, 10, 0, 100)
    gamepassFrame.BackgroundTransparency = 1
    gamepassFrame.Parent = parent
    
    -- ScrollingFrame para lista de gamepasses
    local scrollFrame = Instance.new("ScrollingFrame")
    scrollFrame.Name = "GamepassList"
    scrollFrame.Size = UDim2.new(1, 0, 1, -50)
    scrollFrame.Position = UDim2.new(0, 0, 0, 0)
    scrollFrame.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
    scrollFrame.BorderSizePixel = 0
    scrollFrame.ScrollBarThickness = 8
    scrollFrame.Parent = gamepassFrame
    
    local scrollCorner = Instance.new("UICorner")
    scrollCorner.CornerRadius = UDim.new(0, 8)
    scrollCorner.Parent = scrollFrame
    
    -- Layout para a lista
    local listLayout = Instance.new("UIListLayout")
    listLayout.SortOrder = Enum.SortOrder.LayoutOrder
    listLayout.Padding = UDim.new(0, 5)
    listLayout.Parent = scrollFrame
    
    local listPadding = Instance.new("UIPadding")
    listPadding.PaddingTop = UDim.new(0, 10)
    listPadding.PaddingBottom = UDim.new(0, 10)
    listPadding.PaddingLeft = UDim.new(0, 10)
    listPadding.PaddingRight = UDim.new(0, 10)
    listPadding.Parent = scrollFrame
    
    -- Botão "Pegar Gamepass"
    local getGamepassBtn = Instance.new("TextButton")
    getGamepassBtn.Name = "GetGamepassButton"
    getGamepassBtn.Size = UDim2.new(1, 0, 0, 40)
    getGamepassBtn.Position = UDim2.new(0, 0, 1, -40)
    getGamepassBtn.BackgroundColor3 = Color3.fromRGB(0, 200, 100)
    getGamepassBtn.BorderSizePixel = 0
    getGamepassBtn.Text = "🎁 Pegar Gamepass"
    getGamepassBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    getGamepassBtn.TextSize = 16
    getGamepassBtn.Font = Enum.Font.GothamBold
    getGamepassBtn.Parent = gamepassFrame
    
    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 8)
    btnCorner.Parent = getGamepassBtn
    
    return gamepassFrame, scrollFrame, getGamepassBtn
end

-- Função para criar a aba de créditos
local function createCreditsTab(parent)
    local creditsFrame = Instance.new("Frame")
    creditsFrame.Name = "CreditsFrame"
    creditsFrame.Size = UDim2.new(1, -20, 1, -110)
    creditsFrame.Position = UDim2.new(0, 10, 0, 100)
    creditsFrame.BackgroundColor3 = Color3.fromRGB(45, 45, 45)
    creditsFrame.BorderSizePixel = 0
    creditsFrame.Visible = false
    creditsFrame.Parent = parent
    
    local creditsCorner = Instance.new("UICorner")
    creditsCorner.CornerRadius = UDim.new(0, 8)
    creditsCorner.Parent = creditsFrame
    
    -- Título dos créditos
    local creditsTitle = Instance.new("TextLabel")
    creditsTitle.Size = UDim2.new(1, 0, 0, 50)
    creditsTitle.Position = UDim2.new(0, 0, 0, 20)
    creditsTitle.BackgroundTransparency = 1
    creditsTitle.Text = "✨ Créditos"
    creditsTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
    creditsTitle.TextSize = 24
    creditsTitle.Font = Enum.Font.GothamBold
    creditsTitle.TextXAlignment = Enum.TextXAlignment.Center
    creditsTitle.Parent = creditsFrame
    
    -- Informações do criador
    local creatorInfo = Instance.new("TextLabel")
    creatorInfo.Size = UDim2.new(1, -40, 0, 80)
    creatorInfo.Position = UDim2.new(0, 20, 0, 100)
    creatorInfo.BackgroundTransparency = 1
    creatorInfo.Text = "🔧 Criado por: snow\n\n💬 Discord: slay_hear"
    creatorInfo.TextColor3 = Color3.fromRGB(200, 200, 200)
    creatorInfo.TextSize = 18
    creatorInfo.Font = Enum.Font.Gotham
    creatorInfo.TextXAlignment = Enum.TextXAlignment.Center
    creatorInfo.TextYAlignment = Enum.TextYAlignment.Top
    creatorInfo.Parent = creditsFrame
    
    -- Linha decorativa
    local line = Instance.new("Frame")
    line.Size = UDim2.new(0.8, 0, 0, 2)
    line.Position = UDim2.new(0.1, 0, 0, 200)
    line.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    line.BorderSizePixel = 0
    line.Parent = creditsFrame
    
    -- Informações adicionais
    local additionalInfo = Instance.new("TextLabel")
    additionalInfo.Size = UDim2.new(1, -40, 0, 60)
    additionalInfo.Position = UDim2.new(0, 20, 0, 220)
    additionalInfo.BackgroundTransparency = 1
    additionalInfo.Text = "⚡ Script para simulação de Gamepasses\n🎮 Use com responsabilidade!"
    additionalInfo.TextColor3 = Color3.fromRGB(150, 150, 150)
    additionalInfo.TextSize = 14
    additionalInfo.Font = Enum.Font.Gotham
    additionalInfo.TextXAlignment = Enum.TextXAlignment.Center
    additionalInfo.TextYAlignment = Enum.TextYAlignment.Top
    additionalInfo.Parent = creditsFrame
    
    return creditsFrame
end

-- Função para criar item de gamepass na lista
local function createGamepassItem(gamepass, parent)
    local itemFrame = Instance.new("Frame")
    itemFrame.Name = "GamepassItem_" .. gamepass.id
    itemFrame.Size = UDim2.new(1, 0, 0, 60)
    itemFrame.BackgroundColor3 = Color3.fromRGB(55, 55, 55)
    itemFrame.BorderSizePixel = 0
    itemFrame.Parent = parent
    
    local itemCorner = Instance.new("UICorner")
    itemCorner.CornerRadius = UDim.new(0, 6)
    itemCorner.Parent = itemFrame
    
    -- Botão selecionável
    local selectBtn = Instance.new("TextButton")
    selectBtn.Size = UDim2.new(1, 0, 1, 0)
    selectBtn.BackgroundTransparency = 1
    selectBtn.Text = ""
    selectBtn.Parent = itemFrame
    
    -- Nome da gamepass
    local nameLabel = Instance.new("TextLabel")
    nameLabel.Size = UDim2.new(0.6, 0, 0.5, 0)
    nameLabel.Position = UDim2.new(0, 15, 0, 5)
    nameLabel.BackgroundTransparency = 1
    nameLabel.Text = gamepass.name
    nameLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    nameLabel.TextSize = 16
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextXAlignment = Enum.TextXAlignment.Left
    nameLabel.Parent = itemFrame
    
    -- Descrição
    local descLabel = Instance.new("TextLabel")
    descLabel.Size = UDim2.new(0.6, 0, 0.5, 0)
    descLabel.Position = UDim2.new(0, 15, 0.5, 0)
    descLabel.BackgroundTransparency = 1
    descLabel.Text = gamepass.description
    descLabel.TextColor3 = Color3.fromRGB(180, 180, 180)
    descLabel.TextSize = 12
    descLabel.Font = Enum.Font.Gotham
    descLabel.TextXAlignment = Enum.TextXAlignment.Left
    descLabel.Parent = itemFrame
    
    -- Preço
    local priceLabel = Instance.new("TextLabel")
    priceLabel.Size = UDim2.new(0.3, 0, 1, 0)
    priceLabel.Position = UDim2.new(0.7, 0, 0, 0)
    priceLabel.BackgroundTransparency = 1
    priceLabel.Text = "💎 " .. gamepass.price .. " R$"
    priceLabel.TextColor3 = Color3.fromRGB(0, 200, 100)
    priceLabel.TextSize = 14
    priceLabel.Font = Enum.Font.GothamBold
    priceLabel.TextXAlignment = Enum.TextXAlignment.Center
    priceLabel.Parent = itemFrame
    
    -- Indicador de seleção
    local selectedIndicator = Instance.new("Frame")
    selectedIndicator.Name = "SelectedIndicator"
    selectedIndicator.Size = UDim2.new(0, 4, 1, 0)
    selectedIndicator.Position = UDim2.new(0, 0, 0, 0)
    selectedIndicator.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
    selectedIndicator.BorderSizePixel = 0
    selectedIndicator.Visible = false
    selectedIndicator.Parent = itemFrame
    
    -- Função de seleção
    selectBtn.MouseButton1Click:Connect(function()
        -- Remove seleção anterior
        if selectedGamepass then
            local oldIndicator = parent:FindFirstChild("GamepassItem_" .. selectedGamepass.id):FindFirstChild("SelectedIndicator")
            if oldIndicator then
                oldIndicator.Visible = false
            end
        end
        
        -- Define nova seleção
        selectedGamepass = gamepass
        selectedIndicator.Visible = true
        
        print("Gamepass selecionada:", gamepass.name)
    end)
    
    return itemFrame
end

-- Função para simular a obtenção da gamepass
local function simulateGamepassPurchase(gamepass)
    if not gamepass then
        return false
    end
    
    -- Simula o evento de compra da gamepass
    print("🎉 Gamepass obtida:", gamepass.name)
    
    -- Aqui você pode adicionar lógica específica para cada gamepass
    -- Por exemplo, ativar benefícios específicos
    
    -- Notificação visual
    local notification = Instance.new("ScreenGui")
    notification.Name = "GamepassNotification"
    notification.Parent = playerGui
    
    local notifFrame = Instance.new("Frame")
    notifFrame.Size = UDim2.new(0, 300, 0, 80)
    notifFrame.Position = UDim2.new(1, 0, 0, 50)
    notifFrame.BackgroundColor3 = Color3.fromRGB(0, 200, 100)
    notifFrame.BorderSizePixel = 0
    notifFrame.Parent = notification
    
    local notifCorner = Instance.new("UICorner")
    notifCorner.CornerRadius = UDim.new(0, 10)
    notifCorner.Parent = notifFrame
    
    local notifText = Instance.new("TextLabel")
    notifText.Size = UDim2.new(1, -20, 1, 0)
    notifText.Position = UDim2.new(0, 10, 0, 0)
    notifText.BackgroundTransparency = 1
    notifText.Text = "🎉 Gamepass Obtida!\n" .. gamepass.name
    notifText.TextColor3 = Color3.fromRGB(255, 255, 255)
    notifText.TextSize = 14
    notifText.Font = Enum.Font.GothamBold
    notifText.TextXAlignment = Enum.TextXAlignment.Center
    notifText.Parent = notifFrame
    
    -- Animação da notificação
    local slideIn = TweenService:Create(notifFrame, TweenInfo.new(0.5, Enum.EasingStyle.Back), {Position = UDim2.new(1, -320, 0, 50)})
    local slideOut = TweenService:Create(notifFrame, TweenInfo.new(0.5, Enum.EasingStyle.Back), {Position = UDim2.new(1, 0, 0, 50)})
    
    slideIn:Play()
    wait(3)
    slideOut:Play()
    slideOut.Completed:Connect(function()
        notification:Destroy()
    end)
    
    return true
end

-- Função principal
local function initializeGUI()
    print("🚀 Inicializando Gamepass Manager...")
    
    -- Detecta gamepasses
    gamepasses = detectGamepasses()
    print("📦 Gamepasses detectadas:", #gamepasses)
    
    -- Cria a GUI
    local screenGui, mainFrame, gamepassTab, creditsTab = createMainGUI()
    local gamepassFrame, scrollFrame, getGamepassBtn = createGamepassTab(mainFrame)
    local creditsFrame = createCreditsTab(mainFrame)
    
    -- Popula a lista de gamepasses
    if #gamepasses > 0 then
        for _, gamepass in ipairs(gamepasses) do
            createGamepassItem(gamepass, scrollFrame)
        end
        
        -- Atualiza o tamanho do scroll
        scrollFrame.CanvasSize = UDim2.new(0, 0, 0, (#gamepasses * 65) + 20)
    else
        -- Mostra mensagem se não encontrar gamepasses
        local noGamepassLabel = Instance.new("TextLabel")
        noGamepassLabel.Size = UDim2.new(1, 0, 1, 0)
        noGamepassLabel.BackgroundTransparency = 1
        noGamepassLabel.Text = "🔍 Nenhuma gamepass detectada neste jogo\n\nTente em outro jogo ou verifique se\no jogo possui gamepasses ativas."
        noGamepassLabel.TextColor3 = Color3.fromRGB(180, 180, 180)
        noGamepassLabel.TextSize = 16
        noGamepassLabel.Font = Enum.Font.Gotham
        noGamepassLabel.TextXAlignment = Enum.TextXAlignment.Center
        noGamepassLabel.Parent = scrollFrame
    end
    
    -- Eventos das abas
    gamepassTab.MouseButton1Click:Connect(function()
        gamepassFrame.Visible = true
        creditsFrame.Visible = false
        gamepassTab.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
        creditsTab.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    end)
    
    creditsTab.MouseButton1Click:Connect(function()
        gamepassFrame.Visible = false
        creditsFrame.Visible = true
        creditsTab.BackgroundColor3 = Color3.fromRGB(0, 162, 255)
        gamepassTab.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    end)
    
    -- Evento do botão "Pegar Gamepass"
    getGamepassBtn.MouseButton1Click:Connect(function()
        if selectedGamepass then
            simulateGamepassPurchase(selectedGamepass)
        else
            print("⚠️ Nenhuma gamepass selecionada!")
        end
    end)
    
    -- Botão para fechar/abrir GUI (tecla G)
    UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        
        if input.KeyCode == Enum.KeyCode.G then
            screenGui.Enabled = not screenGui.Enabled
        end
    end)
    
    print("✅ Gamepass Manager inicializado! Pressione 'G' para abrir/fechar.")
end

-- Inicia o script
wait(2) -- Aguarda o jogo carregar
initializeGUI()
