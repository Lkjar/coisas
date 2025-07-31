local player = game.Players.LocalPlayer
local mouse = player:GetMouse()
local tool = script.Parent
local anims = {}
local stamina = 100
local debounce = false

-- Função para carregar as animações
local function loadAnims()
	local character = player.Character or player.CharacterAdded:Wait()
	local humanoid = character:WaitForChild("Humanoid")
	
	-- Usar os objetos Animation que já existem na ferramenta
	local leftAnimation = tool:WaitForChild("Left")
	local rightAnimation = tool:WaitForChild("Right")
	
	-- Carregar as animações no Humanoid
	anims.left = humanoid:LoadAnimation(leftAnimation)
	anims.right = humanoid:LoadAnimation(rightAnimation)
	
	-- Definir prioridade das animações (opcional)
	anims.left.Priority = Enum.AnimationPriority.Action
	anims.right.Priority = Enum.AnimationPriority.Action
end

-- Carregar as animações ao pegar a ferramenta
tool.Equipped:Connect(function()
	loadAnims()
end)

-- Função para realizar o soco
local function punch(direction)
	if debounce or stamina < 10 then return end
	debounce = true
	
	local anim = direction == "left" and anims.left or anims.right
	
	-- Verificar se a animação foi carregada corretamente
	if anim then
		anim:Play()
	else
		warn("Animação não foi carregada corretamente!")
	end
	
	-- Reduz a stamina ao socar
	stamina = stamina - 10
	print("Stamina restante: " .. stamina) -- Debug
	
	wait(1)  -- Delay entre socos
	debounce = false
end

-- Conectar o botão esquerdo do mouse para realizar o soco à esquerda
mouse.Button1Down:Connect(function()
	if not debounce then
		punch("left")  -- Realiza o soco à esquerda com o botão esquerdo do mouse
	end
end)

-- Conectar o botão direito do mouse para realizar o soco à direita
mouse.Button2Down:Connect(function()
	if not debounce then
		punch("right")  -- Realiza o soco à direita com o botão direito do mouse
	end
end)
