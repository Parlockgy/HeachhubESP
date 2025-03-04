local Frame = Instance.new("Frame")
local UICorner = Instance.new("UICorner")
local TextButton = Instance.new("TextButton")
local UICorner_2 = Instance.new("UICorner")

Frame.Parent = game.StarterGui.ScreenGui
Frame.BackgroundColor3 = Color3.fromRGB(229, 36, 2)
Frame.BorderColor3 = Color3.fromRGB(0, 0, 0)
Frame.BorderSizePixel = 0
Frame.Position = UDim2.new(0.119402982, 0, 0.221544713, 0)
Frame.Size = UDim2.new(0, 242, 0, 59)

UICorner.Parent = Frame

TextButton.Parent = Frame
TextButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
TextButton.BorderColor3 = Color3.fromRGB(0, 0, 0)
TextButton.BorderSizePixel = 0
TextButton.Position = UDim2.new(0.050439883, 0, 0.154584721, 0)
TextButton.Size = UDim2.new(0, 217, 0, 39)
TextButton.Font = Enum.Font.Gotham
TextButton.Text = "ESP: Off"
TextButton.TextColor3 = Color3.fromRGB(255, 0, 0)
TextButton.TextScaled = true
TextButton.TextSize = 14.000
TextButton.TextTransparency = 0.170
TextButton.TextWrapped = true

UICorner_2.Parent = TextButton

local function AMYU_fake_script()
	local script = Instance.new('LocalScript', Frame)

	local UIS = game:GetService('UserInputService')
	local frame = script.Parent
	local dragToggle = nil
	local dragSpeed = 0.25
	local dragStart = nil
	local startPos = nil
	
	local function updateInput(input)
		local delta = input.Position - dragStart
		local position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X,
			startPos.Y.Scale, startPos.Y.Offset + delta.Y)
		game:GetService('TweenService'):Create(frame, TweenInfo.new(dragSpeed), {Position = position}):Play()
	end
	
	frame.InputBegan:Connect(function(input)
		if (input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch) then 
			dragToggle = true
			dragStart = input.Position
			startPos = frame.Position
			input.Changed:Connect(function()
				if input.UserInputState == Enum.UserInputState.End then
					dragToggle = false
				end
			end)
		end
	end)
	
	UIS.InputChanged:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
			if dragToggle then
				updateInput(input)
			end
		end
	end)
end
coroutine.wrap(AMYU_fake_script)()
local function QZXUQHD_fake_script()
	local script = Instance.new('Script', TextButton)

	local Players = game:GetService("Players")
	local button = script.Parent
	local textLabel = button:WaitForChild("TextLabel")
	local isESPActive = false
	local lines = {}
	local boxes = {}
	local billboardGUIs = {}
	
	local function createESP(player)
		local line = Instance.new("Part")
		line.Size = Vector3.new(0.1, 0.1, 1)
		line.Anchored = true
		line.CanCollide = false
		line.Color = Color3.fromRGB(0, 255, 0)
		line.Parent = workspace
	
		local box = Instance.new("Part")
		box.Size = player.Character:WaitForChild("HumanoidRootPart").Size
		box.Anchored = true
		box.CanCollide = false
		box.Color = Color3.fromRGB(0, 255, 0)
		box.Parent = workspace
	
		local billboardGUI = Instance.new("BillboardGui")
		billboardGUI.Adornee = player.Character:WaitForChild("HumanoidRootPart")
		billboardGUI.Size = UDim2.new(0, 100, 0, 50)
		billboardGUI.StudsOffset = Vector3.new(0, 3, 0)
	
		local textLabel = Instance.new("TextLabel")
		textLabel.Size = UDim2.new(1, 0, 1, 0)
		textLabel.Text = player.Name
		textLabel.BackgroundTransparency = 1
		textLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
		textLabel.TextSize = 14
		textLabel.Parent = billboardGUI
		billboardGUI.Parent = workspace
	
		table.insert(lines, line)
		table.insert(boxes, box)
		table.insert(billboardGUIs, billboardGUI)
	
		return line, box, billboardGUI
	end
	
	local function removeESP()
		for _, line in ipairs(lines) do
			line:Destroy()
		end
		for _, box in ipairs(boxes) do
			box:Destroy()
		end
		for _, billboardGUI in ipairs(billboardGUIs) do
			billboardGUI:Destroy()
		end
	
		lines = {}
		boxes = {}
		billboardGUIs = {}
	end
	
	local function updateESP()
		for _, player in ipairs(Players:GetPlayers()) do
			if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
				local line, box, billboardGUI = createESP(player)
	
				line.CFrame = CFrame.new(player.Character.HumanoidRootPart.Position, game.Players.LocalPlayer.Character.HumanoidRootPart.Position)
				box.CFrame = CFrame.new(player.Character.HumanoidRootPart.Position)
			end
		end
	end
	
	button.MouseButton1Click:Connect(function()
		if isESPActive then
			textLabel.Text = "ESP: Off"
			textLabel.TextColor3 = Color3.fromRGB(255, 0, 0)
			removeESP()
		else
			textLabel.Text = "ESP: On"
			textLabel.TextColor3 = Color3.fromRGB(0, 255, 0)
			updateESP()
		end
		isESPActive = not isESPActive
	end)
	
end
coroutine.wrap(QZXUQHD_fake_script)()
