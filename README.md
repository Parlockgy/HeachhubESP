local player = game.Players.LocalPlayer
local runService = game:GetService("RunService")
local espFolder = Instance.new("Folder", workspace)
espFolder.Name = "ActiveESP"

-- Cria a interface de botões
local screenGui = Instance.new("ScreenGui", player:WaitForChild("PlayerGui"))
screenGui.Name = "ESP_GUI"

local function createButton(name, posX)
	local button = Instance.new("TextButton")
	button.Size = UDim2.new(0, 80, 0, 30)
	button.Position = UDim2.new(0, posX, 1, -40)
	button.Text = name
	button.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
	button.TextColor3 = Color3.new(1, 1, 1)
	button.BorderSizePixel = 0
	button.Name = name .. "_Button"
	button.Parent = screenGui
	return button
end

local spawnBtn = createButton("Spawn", 10)
local deleteBtn = createButton("Delete", 100)

-- Função para criar ESP em um jogador
function createESP(target)
	if target == player then return end
	if not target.Character or not target.Character:FindFirstChild("HumanoidRootPart") then return end

	local char = target.Character
	local hrp = char:FindFirstChild("HumanoidRootPart")

	local function applyESP()
		if not char or not hrp then return end
		if workspace.ActiveESP:FindFirstChild(target.Name) then
			workspace.ActiveESP:FindFirstChild(target.Name):Destroy()
		end

		local isBear = hrp:FindFirstChildOfClass("BillboardGui") ~= nil
		local color = isBear and Color3.fromRGB(255, 0, 0) or Color3.fromRGB(0, 255, 0)
		local role = isBear and "Bear" or "Survivor"

		local container = Instance.new("Folder", workspace.ActiveESP)
		container.Name = target.Name

		-- Highlight
		local highlight = Instance.new("Highlight")
		highlight.FillTransparency = 1
		highlight.OutlineColor = color
		highlight.OutlineTransparency = 0
		highlight.Adornee = isBear and hrp or char
		highlight.Parent = container

		-- BillboardGui com texto
		local billboard = Instance.new("BillboardGui")
		billboard.Size = UDim2.new(0, 200, 0, 50)
		billboard.StudsOffset = Vector3.new(0, 3, 0)
		billboard.AlwaysOnTop = true
		billboard.Name = "ESP_Name"

		local label = Instance.new("TextLabel")
		label.Size = UDim2.new(1, 0, 1, 0)
		label.BackgroundTransparency = 1
		label.TextColor3 = color
		label.TextStrokeTransparency = 0
		label.TextStrokeColor3 = Color3.new(0, 0, 0)
		label.Text = target.Name .. " (" .. role .. ")"
		label.Font = Enum.Font.SourceSansBold
		label.TextScaled = true
		label.Parent = billboard

		billboard.Parent = hrp
		billboard.Parent = container

		-- Beam
		local a0 = Instance.new("Attachment", player.Character:WaitForChild("HumanoidRootPart"))
		local a1 = Instance.new("Attachment", hrp)

		local beam = Instance.new("Beam")
		beam.Attachment0 = a0
		beam.Attachment1 = a1
		beam.Width0 = 0.1
		beam.Width1 = 0.1
		beam.FaceCamera = true
		beam.Color = ColorSequence.new(color)
		beam.Transparency = NumberSequence.new(0)
		beam.Parent = container
	end

	-- Aplica o ESP e escuta mudanças no HRP
	applyESP()

	local hrpConn
	hrpConn = hrp.ChildAdded:Connect(function(child)
		if child:IsA("BillboardGui") then
			wait()
			applyESP()
		end
	end)

	hrp.ChildRemoved:Connect(function(child)
		if child:IsA("BillboardGui") then
			wait()
			applyESP()
		end
	end)

	-- Remove ESP quando o jogador morre ou some
	char.AncestryChanged:Connect(function()
		local esp = workspace.ActiveESP:FindFirstChild(target.Name)
		if esp then esp:Destroy() end
		if hrpConn then hrpConn:Disconnect() end
	end)
end

-- Botão Spawn
spawnBtn.MouseButton1Click:Connect(function()
	for _, plr in pairs(game.Players:GetPlayers()) do
		if plr ~= player then
			createESP(plr)
		end
	end
end)

-- Botão Delete
deleteBtn.MouseButton1Click:Connect(function()
	if workspace:FindFirstChild("ActiveESP") then
		workspace.ActiveESP:ClearAllChildren()
	end
end)

-- Atualiza ESP para novos jogadores
game.Players.PlayerAdded:Connect(function(plr)
	plr.CharacterAdded:Connect(function()
		wait(1)
		if workspace:FindFirstChild("ActiveESP") and #workspace.ActiveESP:GetChildren() > 0 then
			createESP(plr)
		end
	end)
end)
