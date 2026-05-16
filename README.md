-- Limpa a interface anterior para evitar conflitos de loops rodando juntos
local oldGui = game:GetService("CoreGui"):FindFirstChild("ClassicSoccerSmartFix")
if oldGui then oldGui:Destroy() end

-- 1. Interface Visual Cinza Nativa
local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "ClassicSoccerSmartFix"
ScreenGui.Parent = game:GetService("CoreGui")

local MainFrame = Instance.new("Frame")
MainFrame.Size = UDim2.new(0, 290, 0, 155)
MainFrame.Position = UDim2.new(0.5, -145, 0.4, -77)
MainFrame.BackgroundColor3 = Color3.fromRGB(40, 40, 40) -- Cinza Escuro
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true -- Permite arrastar a tela no celular/PC
MainFrame.Parent = ScreenGui

local Corner = Instance.new("UICorner")
Corner.CornerRadius = UDim.new(0, 10)
Corner.Parent = MainFrame

local Title = Instance.new("TextLabel")
Title.Size = UDim2.new(1, 0, 0, 35)
Title.Text = "Classic Soccer - Auto-Liberação"
Title.TextColor3 = Color3.fromRGB(255, 255, 255)
Title.BackgroundTransparency = 1
Title.TextSize = 13
Title.Font = Enum.Font.SourceSansBold
Title.Parent = MainFrame

-- Retângulo Cinza para Ajuste de Curva
local TextBox = Instance.new("TextBox")
TextBox.Size = UDim2.new(1, -20, 0, 35)
TextBox.Position = UDim2.new(0, 10, 0, 40)
TextBox.Text = "25" -- Valor padrão da curva lateral
TextBox.TextColor3 = Color3.fromRGB(255, 255, 255)
TextBox.BackgroundColor3 = Color3.fromRGB(60, 60, 60) -- Cinza Claro
TextBox.TextSize = 16
TextBox.Font = Enum.Font.SourceSansBold
TextBox.PlaceholderText = "Largura da Curva (Arco)"
TextBox.Parent = MainFrame

local BoxCorner = Instance.new("UICorner")
BoxCorner.CornerRadius = UDim.new(0, 6)
BoxCorner.Parent = TextBox

-- Botão Salvar Posição Alvo
local SaveTargetBtn = Instance.new("TextButton")
SaveTargetBtn.Size = UDim2.new(1, -20, 0, 35)
SaveTargetBtn.Position = UDim2.new(0, 10, 0, 85)
SaveTargetBtn.Text = "Salvar Destino no Meu Personagem"
SaveTargetBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
SaveTargetBtn.BackgroundColor3 = Color3.fromRGB(0, 120, 215) -- Azul padrão
SaveTargetBtn.TextSize = 14
SaveTargetBtn.Font = Enum.Font.SourceSansBold
SaveTargetBtn.Parent = MainFrame

local BtnCorner1 = Instance.new("UICorner")
BtnCorner1.CornerRadius = UDim.new(0, 6)
BtnCorner1.Parent = SaveTargetBtn

-- Texto de Status da Interface
local StatusLabel = Instance.new("TextLabel")
StatusLabel.Size = UDim2.new(1, -20, 0, 25)
StatusLabel.Position = UDim2.new(0, 10, 0, 125)
StatusLabel.Text = "Aguardando definição de destino..."
StatusLabel.TextColor3 = Color3.fromRGB(180, 180, 180)
StatusLabel.BackgroundTransparency = 1
StatusLabel.TextSize = 13
StatusLabel.Font = Enum.Font.SourceSansItalic
StatusLabel.Parent = MainFrame

-- 2. Lógica de Interceptação Inteligente
local savedTargetPosition = nil
local curveIntensity = 25
local ballSpeed = 3.2 -- Velocidade padrão aprovada por você
local isTransporting = false
local lastKnownBallPosition = Vector3.new(0,0,0)

local acaoDeChuteDetectada = false
local tempoUltimaAcao = 0

-- Atualiza a intensidade da curva quando você digita no retângulo cinza
TextBox.FocusLost:Connect(function()
	local num = tonumber(TextBox.Text)
	if num then curveIntensity = num else TextBox.Text = tostring(curveIntensity) end
end)

-- Salva o destino baseado onde seu personagem está pisando
SaveTargetBtn.MouseButton1Click:Connect(function()
	local player = game:GetService("Players").LocalPlayer
	if player and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
		savedTargetPosition = player.Character.HumanoidRootPart.Position
		StatusLabel.Text = "Destino Salvo com Sucesso!"
		StatusLabel.TextColor3 = Color3.fromRGB(0, 255, 128)
	end
end)

-- Identifica cliques do mouse ou toques na tela (Celular)
game:GetService("UserInputService").InputBegan:Connect(function(input, processed)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		acaoDeChuteDetectada = true
		tempoUltimaAcao = tick()
	end
end)

-- Monitora teclas comuns de atalho (PC)
game:GetService("UserInputService").InputBegan:Connect(function(input, processed)
	if processed then return end
	local key = input.KeyCode
	if key == Enum.KeyCode.X or key == Enum.KeyCode.C or key == Enum.KeyCode.V or key == Enum.KeyCode.F then
		acaoDeChuteDetectada = true
		tempoUltimaAcao = tick()
	end
end)

-- Filtro inteligente: diferencia chute real de toques bobos de corrida
local function verificarContextoDeChute()
	local player = game:GetService("Players").LocalPlayer
	if not player or not player.Character then return false end
	
	if acaoDeChuteDetectada and (tick() - tempoUltimaAcao) < 0.4 then
		return true
	end
	
	for _, child in ipairs(player.Character:GetChildren()) do
		if child:IsA("Tool") then
			local name = child.Name:lower()
			if name:find("shoot") or name:find("pass") or name:find("long") or name:find("kick") or name:find("lob") then
				return true
			end
		end
	end
	
	return false
end

-- Escaneia e localiza a bola real no The Classic Soccer por formato esférico
local function encontrarBolaReal()
	for _, obj in ipairs(workspace:GetDescendants()) do
		if obj:IsA("BasePart") then
			if obj.Name == "Ball" or obj.Name == "TPS" or obj.Name == "FakeBall" or obj.Name == "Football" or obj.Parent.Name == "Balls" or obj.Parent.Name == "Football" then
				if obj.Size.X == obj.Size.Y and obj.Size.Y == obj.Size.Z and obj.Size.X > 1 and obj.Size.X < 6 then
					return obj
				end
			end
		end
	end
	return nil
end

-- Faz a bola viajar fazendo a curva em formato de lua e libera automaticamente no final
local function guiarBola(ball)
	if isTransporting or not savedTargetPosition then return end
	isTransporting = true
	
	task.spawn(function()
		local startPos = ball.Position
		local totalDistanceVector = (savedTargetPosition - startPos)
		local totalDistance = Vector3.new(totalDistanceVector.X, 0, totalDistanceVector.Z).Magnitude
		
		local progress = 0
		
		ball.Velocity = Vector3.new(0,0,0)
		ball.RotVelocity = Vector3.new(0,0,0)

		while progress < 1 and ball and ball.Parent do
			task.wait(0.01)
			progress = progress + (ballSpeed / totalDistance)
			if progress > 1 then progress = 1 end
			
			local currentLinearPos = startPos:Lerp(savedTargetPosition, progress)
			
			local dirToTarget = totalDistanceVector.Unit
			local sideVector = Vector3.new(-dirToTarget.Z, 0, dirToTarget.X)
			local arcOffset = math.sin(progress * math.pi) * curveIntensity
			
			-- Trajetória em formato de lua (parábola de altura)
			local heightOffset = math.sin(progress * math.pi) * 12 
			
			local finalPos = currentLinearPos + (sideVector * arcOffset) + Vector3.new(0, heightOffset, 0)
			ball.CFrame = CFrame.new(finalPos)
			
			if progress >= 1 then break end
		end
		
		-- AUTO-LIBERAÇÃO: Dá um pequeno empurrão natural para frente ao chegar e solta a bola
		if ball and ball.Parent then
			ball.Velocity = totalDistanceVector.Unit * 15 
			StatusLabel.Text = "Trajeto Concluído! Bola Liberada."
			StatusLabel.TextColor3 = Color3.fromRGB(0, 255, 255)
		end
		
		task.wait(0.2)
		isTransporting = false
		acaoDeChuteDetectada = false
	end)
end

-- 3. Monitoramento de frames com filtros anti-bug para o respawn do :pb
game:GetService("RunService").Heartbeat:Connect(function()
	if not savedTargetPosition or isTransporting then return end
	
	local ball = encontrarBolaReal()
	if ball then
		local currentPos = ball.Position
		local movementDelta = (currentPos - lastKnownBallPosition).Magnitude
		
		-- Filtro de segurança: se mover entre 0.4 e 10 studs significa que foi chutada. Se for mais, foi :pb.
		if movementDelta > 0.4 and movementDelta < 10 and lastKnownBallPosition ~= Vector3.new(0,0,0) then
			if verificarContextoDeChute() then
				guiarBola(ball)
			end
		end
		
		lastKnownBallPosition = currentPos
	else
		lastKnownBallPosition = Vector3.new(0,0,0)
	end
end)
