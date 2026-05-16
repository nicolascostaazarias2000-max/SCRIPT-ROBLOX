-- Carrega a interface Rayfield direto da fonte oficial
local Rayfield = loadstring(game:HttpGet('https://githubusercontent.com'))()

-- Configurações de controle da bola (Versão aprovada por você)
local savedTargetPosition = nil
local curveIntensity = 25
local ballSpeed = 3.2 
local isTransporting = false
local lastKnownBallPosition = Vector3.new(0,0,0)

local acaoDeChuteDetectada = false
local tempoUltimaAcao = 0

-- 1. Inicialização da Janela Rayfield
local Window = Rayfield:CreateWindow({
   Name = "Classic Soccer - Auto-Liberação",
   LoadingTitle = "Iniciando Interface...",
   LoadingSubtitle = "Modo Inteligente",
   Theme = "Default",
   DisableRayfieldPrompts = false,
   DisableBuildWarnings = false,
   ConfigurationSaving = {
      Enabled = false
   }
})

-- Cria a Aba Principal no Menu
local MainTab = Window:CreateTab("⚽ Controle", 4483362458)

-- Texto de Status na Interface
local StatusParagraph = MainTab:CreateParagraph({
    Title = "Status do Script", 
    Content = "Aguardando definição de destino..."
})

-- Botão para Salvar a Posição Alvo
MainTab:CreateButton({
   Name = "Salvar Destino no Meu Personagem",
   Callback = function()
        local player = game:GetService("Players").LocalPlayer
        if player and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            savedTargetPosition = player.Character.HumanoidRootPart.Position
            StatusParagraph:Set({
                Title = "Destino Registrado!", 
                Content = "A bola irá para cá no próximo chute."
            })
        end
   end,
})

-- Slider para controlar a intensidade da curva (Arco de Lua)
MainTab:CreateSlider({
   Name = "Largura da Curva (Arco)",
   Min = 5,
   Max = 100,
   CurrentValue = 25,
   Flag = "CurveSlider",
   Callback = function(Value)
        curveIntensity = Value
   end,
})

-- 2. Sistema de Captura de Inputs (Shoot, Pass, Long)
game:GetService("UserInputService").InputBegan:Connect(function(input, processed)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		acaoDeChuteDetectada = true
		tempoUltimaAcao = tick()
	end
end)

game:GetService("UserInputService").InputBegan:Connect(function(input, processed)
	if processed then return end
	local key = input.KeyCode
	if key == Enum.KeyCode.X or key == Enum.KeyCode.C or key == Enum.KeyCode.V or key == Enum.KeyCode.F then
		acaoDeChuteDetectada = true
		tempoUltimaAcao = tick()
	end
end)

-- Validador de Contexto (Diferencia Chute Real de simples esbarrões)
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

-- Identifica a bola real no The Classic Soccer por geometria
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

-- 3. Função de Movimentação em Formato de Lua (CFrame Interpolado)
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
			
			-- Efeito da curva de lua e da parábola de altura combinados
			local arcOffset = math.sin(progress * math.pi) * curveIntensity
			local heightOffset = math.sin(progress * math.pi) * 12 
			
			local finalPos = currentLinearPos + (sideVector * arcOffset) + Vector3.new(0, heightOffset, 0)
			ball.CFrame = CFrame.new(finalPos)
			
			if progress >= 1 then break end
		end
		
		-- AUTO-LIBERAÇÃO NATURAL: Deixa a bola livre ao atingir o alvo
		if ball and ball.Parent then
			ball.Velocity = totalDistanceVector.Unit * 15 
            StatusParagraph:Set({
                Title = "Trajeto Concluído", 
                Content = "A bola completou a curva e foi liberada automaticamente."
            })
		end
		
		task.wait(0.2)
		isTransporting = false
		acaoDeChuteDetectada = false
	end)
end

-- 4. Monitoramento Contínuo com proteção anti-bug para o comando :pb
game:GetService("RunService").Heartbeat:Connect(function()
	if not savedTargetPosition or isTransporting then return end
	
	local ball = encontrarBolaReal()
	if ball then
		local currentPos = ball.Position
		local movementDelta = (currentPos - lastKnownBallPosition).Magnitude
		
		-- Verifica se o movimento condiz com um chute válido (ignora o spawn do :pb)
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
