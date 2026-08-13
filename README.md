-- 👤 Stalker + Seek Music
-- sem texto Activated/créditos | resto igual

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Debris = game:GetService("Debris")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local camera = Workspace.CurrentCamera
local humanoid = char:WaitForChild("Humanoid")

local stalkerImages = {
	"rbxassetid://71725176156204",
	"rbxassetid://122094429760163",
	"rbxassetid://100917760105588",
	"rbxassetid://127842346062233"
}

local currentStalker = nil
local activated = false
local achievementGiven = false
local touching = false
local lookTime = 0
local attacking = false

local spawnSound = 136833080474934
local randomScarySounds = {136350971091939, 113917217579668}
local JUMPSCARE_IMG = "rbxassetid://88898243238361"
local JUMPSCARE_SOUND = "rbxassetid://9125472062"

local function caption(text)
	local MainUI = player:WaitForChild("PlayerGui"):WaitForChild("MainUI")
	local func = require(MainUI.Initiator.Main_Game)
	func.caption(text, true)
end

local function playSound(id, volume)
	local sound = Instance.new("Sound", Workspace)
	sound.SoundId = "rbxassetid://" .. tostring(id):gsub("rbxassetid://", "")
	sound.Volume = volume or 3.5
	sound:Play()
	Debris:AddItem(sound, 6)
end

local function giveAchievement()
	if achievementGiven then return end
	achievementGiven = true

	local achievementGiver = loadstring(game:HttpGet(
		"https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
	))()
	achievementGiver({
		Title = "Find a Stalker",
		Desc = "Let him watch you.",
		Reason = "But don't get close...",
		Image = "rbxassetid://99486859440104"
	})
end

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness

local function startDarkFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(10, 10, 10)

	TweenService:Create(Lighting, TweenInfo.new(1.2, Enum.EasingStyle.Quad), {
		FogEnd = 18,
		FogStart = 0,
		Brightness = 0.1
	}):Play()
end

local function endDarkFog()
	TweenService:Create(Lighting, TweenInfo.new(1.8, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

local function makeSmoke(part)
	local emitter = Instance.new("ParticleEmitter")
	emitter.Texture = "rbxassetid://243660364"
	emitter.Color = ColorSequence.new(Color3.fromRGB(40, 40, 40), Color3.fromRGB(90, 90, 90))
	emitter.Size = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 2.5),
		NumberSequenceKeypoint.new(1, 6)
	})
	emitter.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0.25),
		NumberSequenceKeypoint.new(1, 1)
	})
	emitter.Lifetime = NumberRange.new(1.4, 2.2)
	emitter.Rate = 50
	emitter.Speed = NumberRange.new(1, 3)
	emitter.SpreadAngle = Vector2.new(50, 50)
	emitter.Parent = part
	return emitter
end

local function vanishWithSmoke(part)
	if not part or not part.Parent then return end

	local smoke = makeSmoke(part)

	local groundPos = Vector3.new(part.Position.X, part.Position.Y - 16, part.Position.Z)
	local downTween = TweenService:Create(part, TweenInfo.new(1.6, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		Position = groundPos
	})
	downTween:Play()
	downTween.Completed:Wait()

	smoke.Enabled = false
	task.wait(0.4)

	if part and part.Parent then
		part:Destroy()
	end
end

local function screenJumpscare()
	local gui = Instance.new("ScreenGui")
	gui.Name = "StalkerJS"
	gui.IgnoreGuiInset = true
	gui.DisplayOrder = 100
	gui.Parent = player:WaitForChild("PlayerGui")

	local img = Instance.new("ImageLabel")
	img.Size = UDim2.new(1.1, 0, 1.1, 0)
	img.Position = UDim2.new(-0.05, 0, -0.05, 0)
	img.BackgroundTransparency = 1
	img.Image = JUMPSCARE_IMG
	img.ScaleType = Enum.ScaleType.Crop
	img.Parent = gui

	local sound = Instance.new("Sound", Workspace)
	sound.SoundId = JUMPSCARE_SOUND
	sound.Volume = 4
	sound:Play()
	Debris:AddItem(sound, 8)

	local shaking = true
	task.spawn(function()
		while shaking and img and img.Parent do
			img.Position = UDim2.new(
				-0.05 + math.random(-8, 8) / 200,
				0,
				-0.05 + math.random(-8, 8) / 200,
				0
			)
			task.wait(0.03)
		end
	end)

	task.wait(1.2)
	shaking = false

	local fade = TweenService:Create(img, TweenInfo.new(1.4, Enum.EasingStyle.Quad), {
		ImageTransparency = 1
	})
	fade:Play()
	fade.Completed:Wait()

	gui:Destroy()
end

local function isLookingAt(targetPos)
	local camPos = camera.CFrame.Position
	local camLook = camera.CFrame.LookVector
	local dir = (targetPos - camPos)
	if dir.Magnitude > 60 then return false end
	dir = dir.Unit
	return camLook:Dot(dir) > 0.85
end

local function hitPlayer(stalker)
	if touching or not stalker or not stalker.Parent then return end
	touching = true
	attacking = false
	currentStalker = nil

	startDarkFog()
	caption("...")
	humanoid:TakeDamage(20)

	task.spawn(screenJumpscare)
	vanishWithSmoke(stalker)

	task.wait(0.5)
	endDarkFog()

	giveAchievement()
	touching = false
	lookTime = 0
end

local function spawnStalker()
	if currentStalker then currentStalker:Destroy() end
	touching = false
	attacking = false
	lookTime = 0

	local stalkerPart = Instance.new("Part")
	stalkerPart.Name = "Stalker"
	stalkerPart.Size = Vector3.new(9, 12, 2.5)
	stalkerPart.Transparency = 1
	stalkerPart.Anchored = true
	stalkerPart.CanCollide = false
	stalkerPart.Parent = Workspace

	local randomImage = stalkerImages[math.random(1, #stalkerImages)]

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local decal = Instance.new("Decal")
		decal.Texture = randomImage
		decal.Face = face
		decal.Parent = stalkerPart
	end

	stalkerPart.CFrame = hrp.CFrame * CFrame.new(0, 3, 24)
	currentStalker = stalkerPart

	task.spawn(function()
		while stalkerPart.Parent do
			if not attacking then
				stalkerPart.CFrame = CFrame.new(stalkerPart.Position, camera.CFrame.Position)
			end
			task.wait(0.03)
		end
	end)

	playSound(spawnSound, 4)

	if math.random(1, 3) == 1 then
		playSound(randomScarySounds[math.random(1, #randomScarySounds)], 3.5)
	end

	task.delay(math.random(45, 90), function()
		if currentStalker == stalkerPart and stalkerPart.Parent and not attacking and not touching then
			currentStalker = nil
			vanishWithSmoke(stalkerPart)
			print("Stalker sumiu (tempo parado)")
		end
	end)
end

local function jumpscare()
	local js = Instance.new("Part")
	js.Name = "StalkerJumpscare"
	js.Size = Vector3.new(7, 10, 1.5)
	js.Transparency = 1
	js.Anchored = true
	js.CanCollide = false
	js.Parent = Workspace

	local img = "rbxassetid://73629850429939"

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local decal = Instance.new("Decal")
		decal.Texture = img
		decal.Face = face
		decal.Parent = js
	end

	js.CFrame = hrp.CFrame * CFrame.new(0, 3, -6)

	task.spawn(function()
		while js.Parent do
			js.CFrame = CFrame.new(js.Position, camera.CFrame.Position)
			task.wait(0.03)
		end
	end)

	task.wait(1.4)
	js:Destroy()
end

task.spawn(function()
	while true do
		if currentStalker and not touching then
			local stalker = currentStalker
			local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
			if root and stalker.Parent then
				local distance = (stalker.Position - root.Position).Magnitude

				if distance < 9 then
					hitPlayer(stalker)
				else
					if isLookingAt(stalker.Position) then
						lookTime = lookTime + 0.1

						if lookTime >= 5 and not attacking then
							attacking = true
							print("Olhou demais! Vindo rápido!")

							task.spawn(function()
								while stalker.Parent and attacking do
									local r = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
									if not r then break end

									local dir = (r.Position - stalker.Position)
									if dir.Magnitude < 9 then
										hitPlayer(stalker)
										break
									end

									if dir.Magnitude > 0.5 then
										dir = dir.Unit
										stalker.CFrame = CFrame.new(stalker.Position + dir * 22 * 0.05, r.Position)
									end
									task.wait(0.03)
								end
							end)
						end
					else
						lookTime = math.max(0, lookTime - 0.15)
					end
				end
			end
		else
			lookTime = 0
		end
		task.wait(0.1)
	end
end)

local NEW_SEEK_ID = "rbxassetid://125959136412325"
local changedSounds = {}

local function changeSeekMusic()
	for _, obj in pairs(Workspace:GetDescendants()) do
		if obj:IsA("Sound") and not changedSounds[obj] then
			local nameLower = obj.Name:lower()
			local parentName = obj.Parent and obj.Parent.Name:lower() or ""

			if nameLower:find("seek") or nameLower:find("chase") or parentName:find("seek") or parentName:find("chase") then
				changedSounds[obj] = true

				local wasPlaying = obj.IsPlaying
				local oldTime = obj.TimePosition

				obj.SoundId = NEW_SEEK_ID
				obj.Volume = 3.8
				obj.PlaybackSpeed = 1
				obj.Looped = true

				if wasPlaying then
					obj:Stop()
					task.wait(0.05)
					obj.TimePosition = oldTime
					obj:Play()
				end
			end
		end
	end
end

task.spawn(function()
	while true do
		changeSeekMusic()
		task.wait(2)
	end
end)

ReplicatedStorage.GameData.LatestRoom.Changed:Connect(function()
	task.wait(1)
	changeSeekMusic()
end)

task.wait(3)
changeSeekMusic()

-- ativa SEM texto
ReplicatedStorage.GameData.LatestRoom.Changed:Connect(function()
	if not activated then
		activated = true
	end
end)

task.spawn(function()
	while true do
		task.wait(math.random(35, 70))
		if activated and math.random(1, 4) == 1 then
			spawnStalker()
		end
	end
end)

task.spawn(function()
	while true do
		task.wait(math.random(40, 85))
		if activated and math.random(1, 5) == 1 then
			jumpscare()
		end
	end
end)

player.Chatted:Connect(function(msg)
	local m = msg:lower()
	if m == "/stalker" then
		spawnStalker()
	elseif m == "/stalker3" then
		jumpscare()
	end
end)

print("✅ Stalker | sem texto Activated/créditos | resto normal")
print("/stalker | /stalker3")

-- 👤 Window Entity (Rework)
-- sala com janela = mais chance | 2 versões (sobe / desce)
-- abrir cedo = chase + névoa estilo Stalker
-- /window

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Debris = game:GetService("Debris")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

local WINDOW_IMAGE_UP = "rbxassetid://136542774742776"
local WINDOW_IMAGE_DOWN = "rbxassetid://98800851128806"
local CHASE_IMAGE = "rbxassetid://107999364222287"
local LOOK_SOUND = "rbxassetid://78494358244371"
local CHASE_MUSIC = "rbxassetid://91203761863073"
local ACHIEVEMENT_IMAGE = "rbxassetid://133276883616111"

local currentWindowEntity = nil
local cooldownActive = false
local cooldownEnd = 0
local chaseEntity = nil
local chaseMusic = nil
local achievementGiven = false

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness

local function getHRP()
	local c = player.Character
	return c and c:FindFirstChild("HumanoidRootPart")
end

local function getHumanoid()
	local c = player.Character
	return c and c:FindFirstChildOfClass("Humanoid")
end

local function caption(text)
	pcall(function()
		local MainUI = player:WaitForChild("PlayerGui"):WaitForChild("MainUI")
		local func = require(MainUI.Initiator.Main_Game)
		func.caption(text, true)
	end)
end

local function giveAchievement()
	if achievementGiven then return end
	achievementGiven = true
	pcall(function()
		local achievementGiver = loadstring(game:HttpGet(
			"https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
		))()
		achievementGiver({
			Title = "Don't Go Outside",
			Desc = "You waited long enough.",
			Reason = "The window was watching...",
			Image = ACHIEVEMENT_IMAGE
		})
	end)
end

local function playSound(id, volume, speed)
	local s = Instance.new("Sound", Workspace)
	s.SoundId = id
	s.Volume = volume or 3
	s.PlaybackSpeed = speed or 1
	s:Play()
	Debris:AddItem(s, 10)
	return s
end

local function startDarkFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(10, 10, 10)

	TweenService:Create(Lighting, TweenInfo.new(1.2, Enum.EasingStyle.Quad), {
		FogEnd = 18,
		FogStart = 0,
		Brightness = 0.1
	}):Play()
end

local function endDarkFog()
	TweenService:Create(Lighting, TweenInfo.new(1.8, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

local function findWindowsNear()
	local hrp = getHRP()
	if not hrp then return {} end

	local rooms = Workspace:FindFirstChild("CurrentRooms")
	if not rooms then return {} end

	local candidates = {}
	for _, v in pairs(rooms:GetDescendants()) do
		local name = v.Name:lower()
		if v:IsA("BasePart") and (name:find("window") or name:find("glass") or name:find("pane")) then
			if (v.Position - hrp.Position).Magnitude < 90 then
				table.insert(candidates, v)
			end
		end
	end
	return candidates
end

local function hasWindowRoom()
	return #findWindowsNear() > 0
end

local function isLookingAt(targetPos)
	local camPos = camera.CFrame.Position
	local camLook = camera.CFrame.LookVector
	local dir = (targetPos - camPos)
	if dir.Magnitude > 45 then return false end
	dir = dir.Unit
	return camLook:Dot(dir) > 0.92
end

local function spawnChase()
	if chaseEntity then return end

	local hrp = getHRP()
	if not hrp then return end

	startDarkFog() -- névoa estilo Stalker

	chaseMusic = Instance.new("Sound", Workspace)
	chaseMusic.SoundId = CHASE_MUSIC
	chaseMusic.Volume = 4
	chaseMusic.Looped = true
	chaseMusic:Play()

	local part = Instance.new("Part")
	part.Name = "WindowChase"
	part.Size = Vector3.new(8, 11, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.Parent = Workspace

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = CHASE_IMAGE
		d.Face = face
		d.Parent = part
	end

	part.CFrame = hrp.CFrame * CFrame.new(0, 3, 16)
	chaseEntity = part

	print("Window Entity PERSEGUINDO + névoa!")

	task.spawn(function()
		while part.Parent do
			local root = getHRP()
			local hum = getHumanoid()
			if root then
				local dir = (root.Position - part.Position)
				if dir.Magnitude > 0.5 then
					dir = dir.Unit
					part.CFrame = CFrame.new(part.Position + dir * 10 * 0.05, root.Position)
				end
				if (part.Position - root.Position).Magnitude < 8 then
					if hum then hum.Health = 0 end
					part:Destroy()
					chaseEntity = nil
					if chaseMusic then
						chaseMusic:Stop()
						chaseMusic:Destroy()
						chaseMusic = nil
					end
					endDarkFog()
					break
				end
			end
			task.wait(0.03)
		end
	end)
end

local function startCountdown()
	cooldownActive = true
	cooldownEnd = tick() + 10

	caption("dont go outside")

	task.spawn(function()
		for i = 10, 0, -1 do
			if not cooldownActive then return end
			caption(tostring(i))
			task.wait(1)
		end

		if cooldownActive and tick() >= cooldownEnd then
			cooldownActive = false
			caption("you can go")
			giveAchievement()
			print("Esperou → conquista!")
		end
	end)
end

local function spawnOnWindow()
	if currentWindowEntity or chaseEntity or cooldownActive then return end

	local hrp = getHRP()
	if not hrp then return end

	local windows = findWindowsNear()
	local window = (#windows > 0) and windows[math.random(1, #windows)] or nil

	local spawnCF
	if window then
		spawnCF = window.CFrame * CFrame.new(0, 0, 1.2)
	else
		-- sem janela: ainda pode, mas mais atrás
		spawnCF = hrp.CFrame * CFrame.new(0, 3, -14)
	end

	-- 2 versões: sobe ou desce
	local goesUp = math.random(1, 2) == 1
	local faceImage = goesUp and WINDOW_IMAGE_UP or WINDOW_IMAGE_DOWN

	local part = Instance.new("Part")
	part.Name = "WindowEntity"
	part.Size = Vector3.new(6, 8, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = spawnCF
	part.Parent = Workspace

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = faceImage
		d.Face = face
		d.Parent = part
	end

	currentWindowEntity = part
	print("Window Entity | versão: " .. (goesUp and "SOBE" or "DESCE"))

	local looked = false
	task.spawn(function()
		while part.Parent and not looked do
			part.CFrame = CFrame.new(part.Position, camera.CFrame.Position)

			if isLookingAt(part.Position) then
				looked = true
				playSound(LOOK_SOUND, 3.5, 0.20)

				local offset = goesUp and Vector3.new(0, 12, 0) or Vector3.new(0, -14, 0)
				local endPos = part.Position + offset

				local tween = TweenService:Create(part, TweenInfo.new(1.2, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
					Position = endPos
				})
				tween:Play()
				tween.Completed:Wait()

				if part and part.Parent then part:Destroy() end
				currentWindowEntity = nil
				startCountdown()
				break
			end
			task.wait(0.05)
		end
	end)
end

-- abriu porta cedo demais → chase + névoa
ReplicatedStorage.GameData.LatestRoom.Changed:Connect(function()
	if cooldownActive and tick() < cooldownEnd then
		cooldownActive = false
		print("Abriu cedo demais!")
		task.wait(0.4)
		spawnChase()
	end
end)

-- Spawn: mais chance se tem janela perto
task.spawn(function()
	while true do
		ReplicatedStorage.GameData.LatestRoom.Changed:Wait()
		task.wait(math.random(4, 10))

		if currentWindowEntity or chaseEntity or cooldownActive then
			continue
		end

		local hasWin = hasWindowRoom()
		-- com janela: 1/4 | sem janela: 1/14 (bem menos)
		local chance = hasWin and 4 or 14
		if math.random(1, chance) == 1 then
			print("Window spawn | janela=" .. tostring(hasWin))
			spawnOnWindow()
		end
	end
end)

player.Chatted:Connect(function(msg)
	if msg:lower() == "/window" then
		spawnOnWindow()
	end
end)

print("✅ Window Rework!")
print("Janela = mais chance | 2 versões (sobe/desce) | chase com névoa")
print("/window")

-- 👤 Squater
-- conquista 94315435216396 | mais baixo | só portas ~10 e ~20 | raro
-- /squater

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

local IMAGES = {
	"rbxassetid://129136443605749",
	"rbxassetid://89338642934638",
	"rbxassetid://99583828045469",
	"rbxassetid://122219006388427",
	"rbxassetid://103511714895149"
}

local SOUND_FACE3 = "rbxassetid://1841178191"
local SOUND_FACE4 = "rbxassetid://140028279221307"
local SOUND_JUMPSCARE = "rbxassetid://139310882854462"

local current = nil
local active = false
local camConn = nil
local locked = false
local shaking = false
local savedWalk, savedJump = 16, 50
local achievementGiven = false

local function playSound(id, volume)
	local s = Instance.new("Sound", Workspace)
	s.SoundId = id
	s.Volume = volume or 4
	s:Play()
	Debris:AddItem(s, 12)
	return s
end

local function getHRP()
	local c = player.Character
	return c and c:FindFirstChild("HumanoidRootPart")
end

local function getHumanoid()
	local c = player.Character
	return c and c:FindFirstChild("Humanoid")
end

local function setTexture(part, texture)
	for _, d in pairs(part:GetChildren()) do
		if d:IsA("Decal") then
			d.Texture = texture
		end
	end
end

local function getRoom(roomValue)
	local rooms = Workspace:FindFirstChild("CurrentRooms")
	if not rooms then return nil end
	return rooms:FindFirstChild(tostring(roomValue)) or rooms:FindFirstChild(roomValue)
end

local function getRoomCenter(room)
	if not room then return nil end

	if room:FindFirstChild("RoomEntrance") and room:FindFirstChild("RoomExit") then
		local a = room.RoomEntrance.Position
		local b = room.RoomExit.Position
		return Vector3.new((a.X + b.X) / 2, (a.Y + b.Y) / 2 + 1.2, (a.Z + b.Z) / 2)
	end

	local parts = {}
	for _, v in pairs(room:GetDescendants()) do
		if v:IsA("BasePart") then
			local n = v.Name:lower()
			if not n:find("door") and not n:find("entrance") and not n:find("exit") then
				table.insert(parts, v)
			end
		end
	end

	if #parts == 0 then
		for _, v in pairs(room:GetDescendants()) do
			if v:IsA("BasePart") then table.insert(parts, v) end
		end
	end

	if #parts == 0 then return nil end

	local sum = Vector3.zero
	for _, p in pairs(parts) do
		sum = sum + p.Position
	end
	local c = sum / #parts
	return Vector3.new(c.X, c.Y + 1.2, c.Z)
end

-- só janelas de porta 10 e 20 (com margem)
local function isAllowedRoom(num)
	num = tonumber(num)
	if not num then return false end
	-- porta ~10 (9 a 12) ou ~20 (18 a 22)
	if num >= 9 and num <= 12 then return true end
	if num >= 18 and num <= 22 then return true end
	return false
end

local function giveAchievement()
	if achievementGiven then return end
	achievementGiven = true

	pcall(function()
		local achievementGiver = loadstring(game:HttpGet(
			"https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
		))()

		achievementGiver({
			Title = "Still Standing",
			Desc = "You survived Squater.",
			Reason = "Dodge the charge. Don't freeze forever.",
			Image = "rbxassetid://94315435216396"
		})
	end)
end

local function unlockAll()
	locked = false
	shaking = false

	if camConn then
		camConn:Disconnect()
		camConn = nil
	end

	camera.CameraType = Enum.CameraType.Custom

	local h = getHumanoid()
	if h then
		h.WalkSpeed = savedWalk > 0 and savedWalk or 16
		h.JumpPower = savedJump > 0 and savedJump or 50
	end
end

local function lockOn(part)
	if locked then return end
	locked = true
	shaking = false

	local hum = getHumanoid()
	if hum then
		savedWalk = hum.WalkSpeed
		savedJump = hum.JumpPower
		hum.WalkSpeed = 0
		hum.JumpPower = 0
	end

	camera.CameraType = Enum.CameraType.Scriptable

	if camConn then camConn:Disconnect() end
	camConn = RunService.RenderStepped:Connect(function()
		if not locked then return end
		if not part or not part.Parent then return end
		local root = getHRP()
		if not root then return end

		local base = part.Position + (root.Position - part.Position).Unit * 10 + Vector3.new(0, 2, 0)
		local cf = CFrame.new(base, part.Position)

		if shaking then
			cf = cf * CFrame.new(
				math.random(-10, 10) / 12,
				math.random(-10, 10) / 12,
				0
			)
		end

		camera.CFrame = cf
	end)
end

local function spawnInRoom(roomValue)
	if active then return end
	active = true

	task.wait(1.2)

	local root = getHRP()
	if not root then
		active = false
		return
	end

	local room = getRoom(roomValue)
	local center = getRoomCenter(room)
	local spawnCF

	if center then
		spawnCF = CFrame.new(center, root.Position)
		print("Squater no MEIO da sala: " .. tostring(roomValue))
	else
		spawnCF = root.CFrame * CFrame.new(0, 1.2, -18)
		print("Squater fallback")
	end

	local part = Instance.new("Part")
	part.Name = "Squater"
	part.Size = Vector3.new(7, 10, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = spawnCF
	part.Parent = Workspace
	current = part

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = IMAGES[1]
		d.Face = face
		d.Parent = part
	end

	local survived = true

	task.spawn(function()
		while part.Parent and active do
			if not locked then
				local r = getHRP()
				if r then
					part.CFrame = CFrame.new(part.Position, r.Position)
				end
			end
			task.wait(0.04)
		end
	end)

	task.wait(2.8)
	if not part.Parent then active = false unlockAll() return end
	setTexture(part, IMAGES[2])

	task.wait(2.8)
	if not part.Parent then active = false unlockAll() return end
	setTexture(part, IMAGES[3])
	playSound(SOUND_FACE3, 4)
	lockOn(part)

	task.wait(2.5)
	if not part.Parent then active = false unlockAll() return end
	setTexture(part, IMAGES[4])
	playSound(SOUND_FACE4, 4.5)
	shaking = true

	task.wait(3.2)
	shaking = false
	if not part.Parent then active = false unlockAll() return end
	setTexture(part, IMAGES[5])
	unlockAll()
	task.wait(0.15)

	local chargeDir = nil
	local r = getHRP()
	if r then
		chargeDir = (r.Position - part.Position)
		chargeDir = Vector3.new(chargeDir.X, 0, chargeDir.Z)
		if chargeDir.Magnitude > 0.1 then
			chargeDir = chargeDir.Unit
		else
			chargeDir = r.CFrame.LookVector
		end
	end

	local speed = 28
	local start = tick()

	while part.Parent and (tick() - start) < 4 do
		local rr = getHRP()
		local hum = getHumanoid()

		if hum and hum.WalkSpeed < 1 then
			hum.WalkSpeed = savedWalk > 0 and savedWalk or 16
			hum.JumpPower = savedJump > 0 and savedJump or 50
		end

		if chargeDir then
			part.CFrame = CFrame.new(part.Position + chargeDir * speed * 0.05, part.Position + chargeDir)
		end

		if rr and (part.Position - rr.Position).Magnitude < 8 then
			survived = false
			playSound(SOUND_JUMPSCARE, 5)
			if hum then hum.Health = 0 end
			break
		end
		task.wait(0.03)
	end

	unlockAll()
	if part and part.Parent then part:Destroy() end
	current = nil
	active = false

	if survived then
		local hum = getHumanoid()
		if hum and hum.Health > 0 then
			giveAchievement()
		end
	end

	print("Squater acabou!")
end

-- só portas ~10 e ~20 + chance rara
task.spawn(function()
	while true do
		local room = ReplicatedStorage.GameData.LatestRoom.Changed:Wait()
		local num = tonumber(room) or room

		if isAllowedRoom(num) then
			task.wait(math.random(2, 6))
			-- bem raro mesmo nas janelas permitidas
			if not active and math.random(1, 6) == 1 then
				print("Squater raro na porta " .. tostring(num))
				spawnInRoom(num)
			end
		end
	end
end)

player.Chatted:Connect(function(msg)
	if msg:lower() == "/squater" then
		if active then return end
		print("Aguardando próxima porta...")
		local room = ReplicatedStorage.GameData.LatestRoom.Changed:Wait()
		local num = tonumber(room) or room
		spawnInRoom(num)
	end
end)

print("✅ Squater | só portas ~10 e ~20 | raro | /squater")

-- 👤 Orimmy
-- "You stole my gem Now you will PAY" → chase
-- aleatório BEM raro | só 2 vezes | demora pra aparecer
-- /orimmy

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local RunService = game:GetService("RunService")
local Lighting = game:GetService("Lighting")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera
local playerGui = player:WaitForChild("PlayerGui")

local IMAGE1 = "rbxassetid://78643031092632"
local IMAGE2 = "rbxassetid://101206859267443"
local IMAGE3 = "rbxassetid://100811806838174"
local MUSIC_ID = "rbxassetid://89135045680633"
local SOUND_VANISH = "rbxassetid://134263165995764"
local ACHIEVEMENT_IMG = "rbxassetid://125938550140878"

local SPAWN_DIST = 28
local CHASE_SPEED = 10
local LOOK_DOT = 0.72
local CLOSE_DIST = 12
local KILL_DIST = 6
local LIVE_TIME = 15
local MAX_SPAWNS = 2

local active = false
local current = nil
local chasing = false
local locked = false
local fogOn = false
local killedPlayer = false
local camConn = nil
local music = nil
local blinkConn = nil
local savedWalk, savedJump = 16, 50
local oldFogEnd, oldFogStart, oldAmbient, oldBrightness, oldFogColor
local screenGui = nil
local screenVisible = false
local achievementGiven = false
local spawnCount = 0

local function caption(text)
	pcall(function()
		local MainUI = player:WaitForChild("PlayerGui"):WaitForChild("MainUI")
		local func = require(MainUI.Initiator.Main_Game)
		func.caption(text, true)
	end)
end

local function getHRP()
	local c = player.Character
	return c and c:FindFirstChild("HumanoidRootPart")
end

local function getHumanoid()
	local c = player.Character
	return c and c:FindFirstChildOfClass("Humanoid")
end

local function setTexture(part, id)
	for _, d in pairs(part:GetChildren()) do
		if d:IsA("Decal") then
			d.Texture = id
		end
	end
end

local function isLookingAt(part)
	if not part or not part.Parent then return false end
	local camPos = camera.CFrame.Position
	local look = camera.CFrame.LookVector
	local dir = (part.Position - camPos)
	if dir.Magnitude < 1 then return true end
	return look:Dot(dir.Unit) > LOOK_DOT
end

local function playOneShot(id, vol)
	local s = Instance.new("Sound", Workspace)
	s.SoundId = id
	s.Volume = vol or 4
	s:Play()
	Debris:AddItem(s, 10)
end

local function giveAchievement()
	if achievementGiven or killedPlayer then return end
	achievementGiven = true
	pcall(function()
		local achievementGiver = loadstring(game:HttpGet(
			"https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
		))()
		achievementGiver({
			Title = "Orimmy Overlooked",
			Desc = "You survived Orimmy.",
			Reason = "He looked back... and left.",
			Image = ACHIEVEMENT_IMG
		})
	end)
end

local function startMusic()
	if music then
		pcall(function() music:Stop() music:Destroy() end)
	end
	music = Instance.new("Sound", Workspace)
	music.Name = "OrimmyMusic"
	music.SoundId = MUSIC_ID
	music.Volume = 3.5
	music.Looped = true
	music.PlaybackSpeed = 0.32
	music:Play()
end

local function stopMusic()
	if not music then return end
	local m = music
	music = nil
	local tw = TweenService:Create(m, TweenInfo.new(0.5), {Volume = 0})
	tw:Play()
	tw.Completed:Wait()
	if m and m.Parent then
		m:Stop()
		m:Destroy()
	end
end

local function lockOn(part)
	if locked then return end
	locked = true
	local hum = getHumanoid()
	if hum then
		savedWalk = hum.WalkSpeed
		savedJump = hum.JumpPower
		hum.WalkSpeed = 0
		hum.JumpPower = 0
		pcall(function() hum.JumpHeight = 0 end)
	end
	camera.CameraType = Enum.CameraType.Scriptable
	if camConn then camConn:Disconnect() end
	camConn = RunService.RenderStepped:Connect(function()
		if not locked or not part or not part.Parent then return end
		local root = getHRP()
		if not root then return end
		local base = part.Position + (root.Position - part.Position).Unit * 12 + Vector3.new(0, 2, 0)
		camera.CFrame = CFrame.new(base, part.Position)
	end)
end

local function unlockAll()
	locked = false
	if camConn then
		camConn:Disconnect()
		camConn = nil
	end
	camera.CameraType = Enum.CameraType.Custom
	local hum = getHumanoid()
	if hum then
		hum.WalkSpeed = savedWalk > 0 and savedWalk or 16
		hum.JumpPower = savedJump > 0 and savedJump or 50
		pcall(function() hum.JumpHeight = 7.2 end)
	end
end

local function ensureScreen()
	if screenGui and screenGui.Parent then return end
	pcall(function() playerGui:FindFirstChild("OrimmyScreen"):Destroy() end)
	screenGui = Instance.new("ScreenGui")
	screenGui.Name = "OrimmyScreen"
	screenGui.IgnoreGuiInset = true
	screenGui.DisplayOrder = 180
	screenGui.ResetOnSpawn = false
	screenGui.Parent = playerGui
	local img = Instance.new("ImageLabel")
	img.Name = "BlinkImg"
	img.Size = UDim2.new(1, 0, 1, 0)
	img.BackgroundTransparency = 1
	img.Image = IMAGE3
	img.ScaleType = Enum.ScaleType.Stretch
	img.ImageTransparency = 1
	img.Visible = false
	img.Parent = screenGui
end

local function setScreenClose(on)
	ensureScreen()
	local img = screenGui and screenGui:FindFirstChild("BlinkImg")
	if not img then return end
	if on then
		if not screenVisible then
			screenVisible = true
			img.Visible = true
			if blinkConn then blinkConn:Disconnect() end
			blinkConn = RunService.Heartbeat:Connect(function()
				if not screenVisible or not img or not img.Parent then return end
				img.ImageTransparency = (tick() % 0.24 < 0.12) and 0.05 or 0.55
			end)
		end
	else
		if screenVisible then
			screenVisible = false
			if blinkConn then blinkConn:Disconnect() blinkConn = nil end
			img.ImageTransparency = 1
			img.Visible = false
		end
	end
end

local function destroyScreen()
	screenVisible = false
	if blinkConn then blinkConn:Disconnect() blinkConn = nil end
	if screenGui then screenGui:Destroy() screenGui = nil end
end

local function startFogPulse()
	if fogOn then return end
	fogOn = true
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness
	oldFogColor = Lighting.FogColor
	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(15, 15, 18)
	Lighting.Brightness = 0.35
	task.spawn(function()
		while fogOn and current and current.Parent do
			TweenService:Create(Lighting, TweenInfo.new(0.9, Enum.EasingStyle.Sine), {
				FogEnd = 18, FogStart = 0
			}):Play()
			task.wait(1)
			if not fogOn then break end
			TweenService:Create(Lighting, TweenInfo.new(0.9, Enum.EasingStyle.Sine), {
				FogEnd = 55, FogStart = 8
			}):Play()
			task.wait(1)
		end
	end)
end

local function whiteFogThenClear()
	fogOn = false
	Lighting.FogColor = Color3.fromRGB(255, 255, 255)
	Lighting.FogStart = 0
	Lighting.FogEnd = 22
	Lighting.Ambient = Color3.fromRGB(200, 200, 210)
	Lighting.Brightness = 0.6
	task.wait(3)
	TweenService:Create(Lighting, TweenInfo.new(1.2), {
		FogEnd = oldFogEnd or 100000,
		FogStart = oldFogStart or 0,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		Brightness = oldBrightness or 1
	}):Play()
	Lighting.FogColor = oldFogColor or Color3.fromRGB(192, 192, 192)
end

local function sinkAndDestroy(part)
	if not part or not part.Parent then return end
	local target = part.Position - Vector3.new(0, 18, 0)
	local tw = TweenService:Create(part, TweenInfo.new(1.4, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		CFrame = CFrame.new(target, target + part.CFrame.LookVector)
	})
	tw:Play()
	tw.Completed:Wait()
	if part and part.Parent then part:Destroy() end
end

local function cleanup()
	unlockAll()
	stopMusic()
	destroyScreen()
	playOneShot(SOUND_VANISH, 4.5)

	local part = current
	current = nil
	active = false
	chasing = false

	if part and part.Parent then
		task.spawn(function()
			sinkAndDestroy(part)
		end)
	end

	if not killedPlayer then
		giveAchievement()
	end

	task.spawn(whiteFogThenClear)
end

local function spawnEntity()
	if active then return end
	if spawnCount >= MAX_SPAWNS then
		print("Orimmy já apareceu 2 vezes — não spawna mais")
		return
	end

	active = true
	chasing = false
	killedPlayer = false
	spawnCount = spawnCount + 1

	local hrp = getHRP()
	if not hrp then
		active = false
		spawnCount = math.max(0, spawnCount - 1)
		return
	end

	local part = Instance.new("Part")
	part.Name = "Orimmy"
	part.Size = Vector3.new(7, 11, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = hrp.CFrame * CFrame.new(0, 1.5, -SPAWN_DIST)
	part.Parent = Workspace
	current = part

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = IMAGE1
		d.Face = face
		d.Parent = part
	end

	lockOn(part)

	-- fala ao aparecer
	caption("You stole my gem")
	task.wait(1.4)
	caption("Now you will PAY")
	task.wait(1.2)

	local lookConn
	lookConn = RunService.RenderStepped:Connect(function()
		if not part.Parent then
			if lookConn then lookConn:Disconnect() end
			return
		end
		local r = getHRP()
		if r then
			part.CFrame = CFrame.new(part.Position, r.Position + Vector3.new(0, 1, 0))
		end
	end)

	local seen = false
	local waitStart = tick()
	while part.Parent and tick() - waitStart < 20 do
		if isLookingAt(part) or locked then
			task.wait(0.8)
			seen = true
			break
		end
		task.wait(0.08)
	end

	if not part.Parent or not seen then
		if lookConn then lookConn:Disconnect() end
		cleanup()
		return
	end

	unlockAll()
	setTexture(part, IMAGE2)
	chasing = true
	startMusic()
	startFogPulse()

	local endTime = tick() + LIVE_TIME

	while part.Parent and tick() < endTime do
		local r = getHRP()
		local hum = getHumanoid()
		if not r or not hum or hum.Health <= 0 then
			if hum and hum.Health <= 0 then killedPlayer = true end
			break
		end

		local dist = (r.Position - part.Position).Magnitude
		local dir = (r.Position - part.Position)
		if dir.Magnitude > 0.5 then
			dir = dir.Unit
			local newPos = part.Position + dir * CHASE_SPEED * 0.05
			part.CFrame = CFrame.new(newPos, r.Position + Vector3.new(0, 1, 0))
		end

		if dist <= CLOSE_DIST then
			setScreenClose(true)
		else
			setScreenClose(false)
		end

		if dist <= KILL_DIST then
			killedPlayer = true
			hum.Health = 0
			break
		end

		task.wait(0.04)
	end

	if lookConn then lookConn:Disconnect() end
	cleanup()
end

-- SPAWN BEM RARO
-- demora pra começar (depois da porta ~25+)
-- chance baixa | máx 2 vezes
task.spawn(function()
	-- espera o jogo avançar um pouco
	for _ = 1, 12 do
		ReplicatedStorage.GameData.LatestRoom.Changed:Wait()
	end

	while spawnCount < MAX_SPAWNS do
		ReplicatedStorage.GameData.LatestRoom.Changed:Wait()
		task.wait(math.random(20, 45)) -- demora bem entre tentativas
		if not active and spawnCount < MAX_SPAWNS and math.random(1, 28) == 1 then
			print("Orimmy raro spawnou! (" .. (spawnCount + 1) .. "/2)")
			spawnEntity()
		end
	end
end)

player.Chatted:Connect(function(msg)
	local m = msg:lower():gsub("%s+", "")
	if m == "/orimmy" or m == "orimmy" then
		spawnEntity()
	end
end)

print("✅ Orimmy | fala gem → chase | bem raro | máx 2x | demora pra aparecer")
print("Comando: /orimmy")

-- 📜 Stalker Mod Intro (1ª porta)
-- All English | sound | 4s | right side

local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local Workspace = game:GetService("Workspace")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

local SOUND_ID = "rbxassetid://138466523093279"

local showed = false

local function showIntro()
	if showed then return end
	showed = true

	pcall(function()
		playerGui:FindFirstChild("StalkerModIntro"):Destroy()
	end)

	local sound = Instance.new("Sound", Workspace)
	sound.Name = "StalkerIntroSound"
	sound.SoundId = SOUND_ID
	sound.Volume = 3
	sound.Looped = false
	sound:Play()
	Debris:AddItem(sound, 20)

	local gui = Instance.new("ScreenGui")
	gui.Name = "StalkerModIntro"
	gui.IgnoreGuiInset = true
	gui.DisplayOrder = 150
	gui.ResetOnSpawn = false
	gui.Parent = playerGui

	local credits = Instance.new("TextLabel")
	credits.AnchorPoint = Vector2.new(1, 0)
	credits.Position = UDim2.new(0.98, 0, 0.22, 0)
	credits.Size = UDim2.new(0.42, 0, 0.6, 0)
	credits.BackgroundTransparency = 1
	credits.TextColor3 = Color3.fromRGB(255, 255, 255)
	credits.TextStrokeTransparency = 0.5
	credits.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
	credits.Font = Enum.Font.Gotham
	credits.TextSize = 15
	credits.TextXAlignment = Enum.TextXAlignment.Right
	credits.TextYAlignment = Enum.TextYAlignment.Top
	credits.TextWrapped = true
	credits.RichText = true
	credits.TextTransparency = 1
	credits.Text = [[<b>Stalker mod made by: LEO -LDX- (guide)</b>

Credits to:

Twixxel's stalkers (Inspiration for all this)
omayoba (new entities and much more — all by him)
pyxlfunkin (ideas for this mod and much more!)
lohan0389_97689 (entity ideas and new mechanics)

[Thanks for playing]
This mod]]
	credits.Parent = gui

	TweenService:Create(credits, TweenInfo.new(1.0, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		TextTransparency = 0
	}):Play()

	task.wait(4)

	TweenService:Create(credits, TweenInfo.new(1.0, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		TextTransparency = 1
	}):Play()
	task.wait(1.1)

	if sound and sound.Parent then
		local fade = TweenService:Create(sound, TweenInfo.new(0.5), {Volume = 0})
		fade:Play()
		fade.Completed:Wait()
		sound:Stop()
		sound:Destroy()
	end

	gui:Destroy()
end

task.spawn(function()
	ReplicatedStorage.GameData.LatestRoom.Changed:Wait()
	task.wait(0.6)
	showIntro()
end)

print("✅ Intro all English | 4s | sound!")

-- 🦠 Virus
-- conquista ao sumir | spawn ULTRA raro | /virus
-- sons: ataque 139793062812354 | sumir 139682612041479

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local RunService = game:GetService("RunService")
local StarterGui = game:GetService("StarterGui")
local TextChatService = game:GetService("TextChatService")
local Debris = game:GetService("Debris")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local camera = Workspace.CurrentCamera

local IMAGE1 = "rbxassetid://92570423194999"
local IMAGE2 = "rbxassetid://92570423194999"
local IMAGE3 = "rbxassetid://131526416302994"
local MUSIC_ID = "rbxassetid://140648151329276"
local SOUND_ATTACK = "rbxassetid://139793062812354"
local SOUND_VANISH = "rbxassetid://139682612041479"
local ACHIEVEMENT_IMG = "rbxassetid://86705624572979"

local active = false
local music = nil
local softGlitchConn = nil
local softCC = nil
local chatSpam = false
local achievementGiven = false

local function getHumanoid()
	local c = player.Character
	return c and c:FindFirstChildOfClass("Humanoid")
end

local function playOneShot(id, volume)
	local s = Instance.new("Sound", Workspace)
	s.SoundId = id
	s.Volume = volume or 4
	s:Play()
	Debris:AddItem(s, 12)
	return s
end

local function giveAchievement()
	if achievementGiven then return end
	achievementGiven = true

	pcall(function()
		local achievementGiver = loadstring(game:HttpGet(
			"https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
		))()

		achievementGiver({
			Title = "Infected Mind",
			Desc = "You survived the Virus.",
			Reason = "It saw you... and left.",
			Image = ACHIEVEMENT_IMG
		})
	end)
end

local function chatBlack(msg)
	pcall(function()
		local channel = TextChatService:FindFirstChild("TextChannels")
			and TextChatService.TextChannels:FindFirstChild("RBXSystem")
		if channel then
			channel:DisplaySystemMessage('<font color="#000000">' .. msg .. '</font>')
		end
	end)
	pcall(function()
		StarterGui:SetCore("ChatMakeSystemMessage", {
			Text = msg,
			Color = Color3.fromRGB(0, 0, 0),
			Font = Enum.Font.SourceSansBold,
			TextSize = 18
		})
	end)
end

local function startChatSpam()
	if chatSpam then return end
	chatSpam = true
	task.spawn(function()
		while chatSpam and active do
			chatBlack("i can see you")
			task.wait(0.35)
		end
	end)
end

local function stopChatSpam()
	chatSpam = false
end

local function playMusic()
	if music then
		pcall(function() music:Stop() music:Destroy() end)
	end
	music = Instance.new("Sound", Workspace)
	music.Name = "VirusMusic"
	music.SoundId = MUSIC_ID
	music.Volume = 3.5
	music.Looped = true
	music.PlaybackSpeed = 0.55
	music:Play()
end

local function stopMusic()
	if not music then return end
	local m = music
	music = nil
	local tw = TweenService:Create(m, TweenInfo.new(0.7), {Volume = 0})
	tw:Play()
	tw.Completed:Wait()
	if m and m.Parent then
		m:Stop()
		m:Destroy()
	end
end

local function startSoftGlitch()
	if softCC then pcall(function() softCC:Destroy() end) end
	softCC = Instance.new("ColorCorrectionEffect")
	softCC.Name = "VirusSoftGlitch"
	softCC.Parent = Lighting

	if softGlitchConn then softGlitchConn:Disconnect() end
	softGlitchConn = RunService.RenderStepped:Connect(function()
		if not active or not softCC then return end
		if math.random() < 0.3 then
			softCC.Brightness = (math.random() - 0.5) * 0.2
			softCC.Contrast = math.random() * 0.25
			softCC.Saturation = -0.1 + math.random() * 0.15
			softCC.TintColor = Color3.fromRGB(
				math.random(210, 255),
				math.random(230, 255),
				math.random(210, 255)
			)
			camera.CFrame = camera.CFrame * CFrame.new(
				math.random(-2, 2) / 45,
				math.random(-2, 2) / 45,
				0
			)
		else
			softCC.Brightness = 0
			softCC.Contrast = 0
			softCC.Saturation = 0
			softCC.TintColor = Color3.fromRGB(255, 255, 255)
		end
	end)
end

local function stopSoftGlitch()
	if softGlitchConn then
		softGlitchConn:Disconnect()
		softGlitchConn = nil
	end
	if softCC then
		softCC:Destroy()
		softCC = nil
	end
end

local function shakeImageAndText(img, text, duration)
	local t0 = tick()
	while tick() - t0 < duration and img and img.Parent do
		img.Position = UDim2.new(0, math.random(-14, 14), 0, math.random(-10, 10))
		img.ImageTransparency = math.random() < 0.15 and 0.25 or 0

		if text and text.Visible then
			text.Position = UDim2.new(0.62, math.random(-10, 10), 0.48, math.random(-8, 8))
			text.TextTransparency = math.random() < 0.2 and 0.4 or 0
		end

		if math.random() < 0.4 then
			camera.CFrame = camera.CFrame * CFrame.new(
				math.random(-4, 4) / 28,
				math.random(-4, 4) / 28,
				0
			)
		end
		task.wait(0.04)
	end

	if img and img.Parent then
		img.Position = UDim2.new(0, 0, 0, 0)
		img.ImageTransparency = 0
	end
	if text and text.Parent then
		text.Position = UDim2.new(0.62, 0, 0.48, 0)
		text.TextTransparency = 0
	end
end

local function bigGlitchEnd()
	local gui = Instance.new("ScreenGui")
	gui.Name = "VirusGlitchEnd"
	gui.IgnoreGuiInset = true
	gui.DisplayOrder = 250
	gui.ResetOnSpawn = false
	gui.Parent = playerGui

	local flash = Instance.new("Frame")
	flash.Size = UDim2.new(1, 0, 1, 0)
	flash.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
	flash.BackgroundTransparency = 0.3
	flash.BorderSizePixel = 0
	flash.Parent = gui

	local cc = Instance.new("ColorCorrectionEffect")
	cc.Parent = Lighting

	local start = tick()
	while tick() - start < 1.3 do
		flash.BackgroundTransparency = 0.1 + math.random() * 0.55
		flash.BackgroundColor3 = math.random(1, 2) == 1
			and Color3.fromRGB(0, 0, 0)
			or Color3.fromRGB(40, 255, 60)

		cc.Brightness = (math.random() - 0.5) * 1
		cc.Contrast = math.random() * 1.3
		cc.Saturation = -0.5 + math.random() * 0.4
		cc.TintColor = Color3.fromRGB(
			math.random(40, 255),
			math.random(180, 255),
			math.random(40, 160)
		)

		camera.CFrame = camera.CFrame * CFrame.new(
			math.random(-7, 7) / 16,
			math.random(-7, 7) / 16,
			0
		)
		task.wait(0.03)
	end

	cc:Destroy()
	gui:Destroy()
end

local function applySlow()
	local hum = getHumanoid()
	if not hum then return end
	local old = hum.WalkSpeed
	hum.WalkSpeed = 5
	task.delay(5, function()
		local h = getHumanoid()
		if h then
			h.WalkSpeed = math.max(old, 16)
		end
	end)
end

local function spawnVirus()
	if active then return end
	active = true

	pcall(function()
		playerGui:FindFirstChild("VirusScreen"):Destroy()
	end)

	playMusic()
	startSoftGlitch()

	local gui = Instance.new("ScreenGui")
	gui.Name = "VirusScreen"
	gui.IgnoreGuiInset = true
	gui.DisplayOrder = 200
	gui.ResetOnSpawn = false
	gui.Parent = playerGui

	local img = Instance.new("ImageLabel")
	img.Size = UDim2.new(1, 0, 1, 0)
	img.Position = UDim2.new(0, 0, 0, 0)
	img.BackgroundTransparency = 1
	img.Image = IMAGE1
	img.ScaleType = Enum.ScaleType.Stretch
	img.ImageTransparency = 1
	img.Parent = gui

	local text = Instance.new("TextLabel")
	text.AnchorPoint = Vector2.new(0.5, 0.5)
	text.Position = UDim2.new(0.62, 0, 0.48, 0)
	text.Size = UDim2.new(0.45, 0, 0.08, 0)
	text.BackgroundTransparency = 1
	text.Text = "i can see you"
	text.TextColor3 = Color3.fromRGB(0, 0, 0)
	text.TextStrokeColor3 = Color3.fromRGB(255, 255, 255)
	text.TextStrokeTransparency = 0.35
	text.Font = Enum.Font.Gotham
	text.TextScaled = true
	text.Visible = false
	text.TextTransparency = 1
	text.ZIndex = 10
	text.Parent = gui

	TweenService:Create(img, TweenInfo.new(0.3), {ImageTransparency = 0}):Play()

	task.wait(3)
	if not gui.Parent then
		stopSoftGlitch()
		stopMusic()
		active = false
		return
	end

	img.Image = IMAGE2
	task.wait(2)
	if not gui.Parent then
		stopSoftGlitch()
		stopMusic()
		active = false
		return
	end

	img.Image = IMAGE3

	text.Visible = true
	TweenService:Create(text, TweenInfo.new(1.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		TextTransparency = 0
	}):Play()

	task.wait(5)
	if not gui.Parent then
		stopSoftGlitch()
		stopMusic()
		active = false
		return
	end

	-- ATACAR
	playOneShot(SOUND_ATTACK, 4.5)
	startChatSpam()
	shakeImageAndText(img, text, 1.8)

	TweenService:Create(img, TweenInfo.new(0.1), {ImageTransparency = 1}):Play()
	TweenService:Create(text, TweenInfo.new(0.1), {TextTransparency = 1}):Play()
	task.wait(0.12)
	gui:Destroy()

	-- SUMIR
	playOneShot(SOUND_VANISH, 4.5)
	bigGlitchEnd()
	stopChatSpam()
	stopSoftGlitch()
	applySlow()
	stopMusic()

	-- conquista ao sumir (1x)
	giveAchievement()

	active = false
	print("Virus sumiu + conquista!")
end

-- SPAWN ULTRA RARO (só ao abrir porta)
-- chance ~1/45 + delay longo
task.spawn(function()
	while true do
		ReplicatedStorage.GameData.LatestRoom.Changed:Wait()
		task.wait(math.random(8, 18))
		if not active and math.random(1, 45) == 1 then
			print("Virus ULTRA raro spawnou!")
			spawnVirus()
		end
	end
end)

player.Chatted:Connect(function(msg)
	local m = msg:lower():gsub("%s+", "")
	if m == "/virus" or m == "virus" or m == "/vírus" or m == "vírus" then
		spawnVirus()
	end
end)

print("✅ Virus | conquista 86705624572979 | ULTRA raro | /virus")

-- 👤 Tomentor + Conquista porta 51

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

local IMAGE = "rbxassetid://94197542013273"
local ACHIEVEMENT_IMAGE = "rbxassetid://78883047354754"

local SOUND_AMBIENT = "rbxassetid://136938798370680"
local SOUND_SMOKE = "rbxassetid://136966836614941"
local SOUND_DAMAGE = "rbxassetid://139973874184557"
local SOUND_JUMPSCARE = "rbxassetid://139310882854462"

local current = nil
local spawned = false
local touchCount = 0
local touching = false
local attacking = false
local chasing = false
local lookTime = 0
local ignoreTime = 0
local smokeSpots = {}
local camConn = nil
local ambient = nil
local achievementGiven = false

local function caption(text)
	pcall(function()
		local MainUI = player:WaitForChild("PlayerGui"):WaitForChild("MainUI")
		local func = require(MainUI.Initiator.Main_Game)
		func.caption(text, true)
	end)
end

local function playSound(id, volume)
	local s = Instance.new("Sound", Workspace)
	s.SoundId = id
	s.Volume = volume or 3.5
	s:Play()
	Debris:AddItem(s, 10)
	return s
end

local function giveAchievement()
	if achievementGiven then return end
	achievementGiven = true

	pcall(function()
		local achievementGiver = loadstring(game:HttpGet(
			"https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
		))()
		achievementGiver({
			Title = "Left Behind",
			Desc = "You escaped the library's mark.",
			Reason = "Door 51 was open...",
			Image = ACHIEVEMENT_IMAGE
		})
	end)
end

local function startAmbient()
	if ambient then return end
	ambient = Instance.new("Sound", Workspace)
	ambient.Name = "TomentorAmbient"
	ambient.SoundId = SOUND_AMBIENT
	ambient.Volume = 2.8
	ambient.Looped = true
	ambient:Play()
end

local function stopAmbient()
	if ambient then
		ambient:Stop()
		ambient:Destroy()
		ambient = nil
	end
end

local function getHRP()
	local c = player.Character
	return c and c:FindFirstChild("HumanoidRootPart")
end

local function getHumanoid()
	local c = player.Character
	return c and c:FindFirstChild("Humanoid")
end

local function makeRedSmoke(parent, rate)
	for i = 1, 3 do
		local emitter = Instance.new("ParticleEmitter")
		emitter.Texture = "rbxassetid://243660364"
		emitter.Color = ColorSequence.new({
			ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 30, 30)),
			ColorSequenceKeypoint.new(0.4, Color3.fromRGB(160, 0, 0)),
			ColorSequenceKeypoint.new(1, Color3.fromRGB(40, 0, 0))
		})
		emitter.Size = NumberSequence.new({
			NumberSequenceKeypoint.new(0, 3),
			NumberSequenceKeypoint.new(0.5, 6),
			NumberSequenceKeypoint.new(1, 10)
		})
		emitter.Transparency = NumberSequence.new({
			NumberSequenceKeypoint.new(0, 0.1),
			NumberSequenceKeypoint.new(0.7, 0.4),
			NumberSequenceKeypoint.new(1, 1)
		})
		emitter.Lifetime = NumberRange.new(2, 4)
		emitter.Rate = rate or 35
		emitter.Speed = NumberRange.new(2, 6)
		emitter.SpreadAngle = Vector2.new(80, 80)
		emitter.LightEmission = 0.6
		emitter.LightInfluence = 0
		emitter.Acceleration = Vector3.new(0, 2, 0)
		emitter.Parent = parent
	end

	local light = Instance.new("PointLight")
	light.Color = Color3.fromRGB(255, 0, 0)
	light.Brightness = 2
	light.Range = 16
	light.Parent = parent
end

local function redPulse()
	local oldA = Lighting.Ambient
	local oldB = Lighting.Brightness
	Lighting.Ambient = Color3.fromRGB(110, 0, 0)
	Lighting.Brightness = 0.25
	task.delay(1, function()
		Lighting.Ambient = oldA
		Lighting.Brightness = oldB
	end)
end

local function nearPlayerPos(distance)
	local root = getHRP()
	if not root then return Vector3.new(0, 5, 0) end
	local angle = math.rad(math.random(0, 360))
	local dist = distance or math.random(10, 16)
	return root.Position + Vector3.new(math.cos(angle) * dist, 3, math.sin(angle) * dist)
end

local function isLookingAt(targetPos)
	camera = Workspace.CurrentCamera
	local camPos = camera.CFrame.Position
	local camLook = camera.CFrame.LookVector
	local dir = (targetPos - camPos)
	if dir.Magnitude > 55 then return false end
	return camLook:Dot(dir.Unit) > 0.85
end

local function moveDownThenUp(part, newPos)
	if not part or not part.Parent then return end

	local downPos = Vector3.new(part.Position.X, part.Position.Y - 14, part.Position.Z)
	local downTween = TweenService:Create(part, TweenInfo.new(0.9, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		Position = downPos
	})
	downTween:Play()
	downTween.Completed:Wait()

	if not part.Parent then return end
	part.Position = Vector3.new(newPos.X, newPos.Y - 12, newPos.Z)

	local upTween = TweenService:Create(part, TweenInfo.new(1, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
		Position = newPos
	})
	upTween:Play()
	upTween.Completed:Wait()
end

local function vanishDown(part)
	if not part or not part.Parent then return end
	local downPos = Vector3.new(part.Position.X, part.Position.Y - 20, part.Position.Z)
	local tween = TweenService:Create(part, TweenInfo.new(1.4, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		Position = downPos
	})
	tween:Play()
	tween.Completed:Wait()
	if part and part.Parent then part:Destroy() end
end

local function lockCameraOn(part)
	local oldType = camera.CameraType
	camera.CameraType = Enum.CameraType.Scriptable

	local hum = getHumanoid()
	local walk, jump = 16, 50
	if hum then
		walk = hum.WalkSpeed
		jump = hum.JumpPower
		hum.WalkSpeed = 0
		hum.JumpPower = 0
	end

	camConn = RunService.RenderStepped:Connect(function()
		if not part or not part.Parent then return end
		local root = getHRP()
		if not root then return end
		local pos = part.Position + (part.Position - root.Position).Unit * -10 + Vector3.new(0, 2, 0)
		camera.CFrame = CFrame.new(pos, part.Position)
	end)

	return function()
		if camConn then
			camConn:Disconnect()
			camConn = nil
		end
		camera.CameraType = oldType
		local h = getHumanoid()
		if h then
			h.WalkSpeed = walk
			h.JumpPower = jump
		end
	end
end

local function leaveSmokeAtPlayer()
	local root = getHRP()
	if not root then return end

	local p = Instance.new("Part")
	p.Name = "TomentorSmokeSpot"
	p.Size = Vector3.new(5, 5, 5)
	p.Transparency = 1
	p.Anchored = true
	p.CanCollide = false
	p.Position = root.Position + Vector3.new(0, 1, 0)
	p.Parent = Workspace

	makeRedSmoke(p, 45)
	table.insert(smokeSpots, p)

	task.spawn(function()
		local hitCD = false
		while p and p.Parent do
			local r = getHRP()
			local hum = getHumanoid()
			if r and hum and not hitCD then
				if (p.Position - r.Position).Magnitude < 8 then
					hitCD = true
					hum:TakeDamage(4)
					playSound(SOUND_SMOKE, 3.5)
					redPulse()
					caption("The smoke burns...")
					task.delay(1.2, function() hitCD = false end)
				end
			end
			task.wait(0.25)
		end
	end)
end

local function clearAllSmoke()
	for _, s in pairs(smokeSpots) do
		if s and s.Parent then s:Destroy() end
	end
	smokeSpots = {}
end

local function startFinalAttack(oldPart)
	if chasing then return end
	chasing = true
	attacking = false
	touching = true

	if oldPart and oldPart.Parent then
		oldPart:Destroy()
	end
	current = nil

	local root = getHRP()
	if not root then
		chasing = false
		touching = false
		return
	end

	caption("HE IS COMING")
	redPulse()

	local part = Instance.new("Part")
	part.Name = "TomentorFinal"
	part.Size = Vector3.new(22, 32, 0.4)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = root.CFrame * CFrame.new(0, 6, 22)
	part.Parent = Workspace
	current = part

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = IMAGE
		d.Face = face
		d.Parent = part
	end

	makeRedSmoke(part, 80)
	local unlock = lockCameraOn(part)

	task.spawn(function()
		while part and part.Parent do
			local r = getHRP()
			local hum = getHumanoid()
			if not r then
				task.wait(0.1)
				continue
			end

			local dir = (r.Position - part.Position)
			if dir.Magnitude < 10 then
				playSound(SOUND_JUMPSCARE, 5)
				if hum then hum.Health = 0 end
				caption("...")
				unlock()
				stopAmbient()
				part:Destroy()
				current = nil
				chasing = false
				touching = false
				break
			end

			if dir.Magnitude > 0.5 then
				dir = dir.Unit
				part.CFrame = CFrame.new(part.Position + dir * 16 * 0.05, r.Position)
			end
			task.wait(0.03)
		end
		unlock()
		chasing = false
		touching = false
	end)
end

local function forceAppearFront(part)
	if touching or chasing or not part or not part.Parent then return end
	local root = getHRP()
	if not root then return end

	-- Som de jumpscare ao ignorar
	playSound(SOUND_JUMPSCARE, 4.5)
	caption("You can't ignore it...")
	redPulse()

	local frontPos = (root.CFrame * CFrame.new(0, 3, -10)).Position
	moveDownThenUp(part, frontPos)
	ignoreTime = 0
end

local function onTouch(part)
	if touching or chasing or not part or not part.Parent then return end
	touching = true
	attacking = false
	ignoreTime = 0
	touchCount = touchCount + 1

	local hum = getHumanoid()
	if hum then hum:TakeDamage(25) end
	playSound(SOUND_DAMAGE, 4)
	redPulse()
	leaveSmokeAtPlayer()
	print("Toque #" .. touchCount)

	if touchCount >= 3 then
		caption("No more running...")
		task.wait(0.5)
		startFinalAttack(part)
		return
	end

	caption("It marked you... (" .. touchCount .. "/3)")

	local newPos = nearPlayerPos(math.random(10, 15))
	part.Size = Vector3.new(10 + touchCount, 13 + touchCount * 1.5, 0.25)
	makeRedSmoke(part, 40 + touchCount * 10)
	moveDownThenUp(part, newPos)

	task.wait(0.8)
	touching = false
	lookTime = 0
end

local function spawnTomentor()
	if current then
		current:Destroy()
		current = nil
	end
	touchCount = 0
	touching = false
	attacking = false
	chasing = false
	lookTime = 0
	ignoreTime = 0

	local root = getHRP()
	if not root then return end

	startAmbient()

	local part = Instance.new("Part")
	part.Name = "Tomentor"
	part.Size = Vector3.new(8, 11, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = root.CFrame * CFrame.new(0, 3, -12)
	part.Parent = Workspace
	current = part

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = IMAGE
		d.Face = face
		d.Parent = part
	end

	makeRedSmoke(part, 30)
	caption("Something is watching from door 50...")
	print("Tomentor apareceu!")

	task.spawn(function()
		while part.Parent do
			if chasing then
				task.wait(0.1)
				continue
			end

			local r = getHRP()
			if not r then
				task.wait(0.1)
				continue
			end

			if not attacking and not touching then
				part.CFrame = CFrame.new(part.Position, r.Position)
			end

			local dist = (part.Position - r.Position).Magnitude
			local looking = isLookingAt(part.Position)

			if not touching and dist < 9 then
				onTouch(part)
			elseif not touching and not attacking and looking then
				ignoreTime = 0
				lookTime = lookTime + 0.1
				if lookTime >= 5 then
					attacking = true
					task.spawn(function()
						while part.Parent and attacking and not chasing do
							local rr = getHRP()
							if not rr then break end
							local dir = (rr.Position - part.Position)
							if dir.Magnitude < 9 then
								onTouch(part)
								break
							end
							if dir.Magnitude > 0.5 then
								dir = dir.Unit
								part.CFrame = CFrame.new(part.Position + dir * 18 * 0.05, rr.Position)
							end
							task.wait(0.03)
						end
						attacking = false
					end)
				end
			else
				lookTime = math.max(0, lookTime - 0.12)
				if not touching and not attacking and not looking then
					ignoreTime = ignoreTime + 0.05
					if ignoreTime >= 12 then
						forceAppearFront(part)
					end
				end
			end

			task.wait(0.05)
		end
	end)
end

pcall(function()
	ReplicatedStorage.GameData.LatestRoom.Changed:Connect(function(room)
		local num = tonumber(room) or room

		if (num == 50 or tostring(room) == "50") and not spawned then
			spawned = true
			task.wait(1.5)
			caption("Door 50...")
			task.wait(1)
			spawnTomentor()
		end

		if (num == 51 or tostring(room) == "51") then
			if current then
				caption("It leaves...")
				local part = current
				current = nil
				chasing = false
				attacking = false
				touching = false
				if camConn then camConn:Disconnect() camConn = nil end
				clearAllSmoke()
				stopAmbient()
				vanishDown(part)
				print("Tomentor sumiu (porta 51)")
			end

			-- Conquista só ao passar da 51
			if spawned then
				giveAchievement()
			end
		end
	end)
end)

player.Chatted:Connect(function(msg)
	if msg:lower() == "/tomentor" then
		spawnTomentor()
	end
end)

print("✅ Tomentor final + conquista!")
print("Porta 50 | /tomentor")

-- 👤 Presentation Entity (névoa vermelha + dano 70)

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local humanoid = char:WaitForChild("Humanoid")
local camera = Workspace.CurrentCamera

local IMAGE_FIRST = "rbxassetid://116330652591021"
local IMAGE_ATTACK = "rbxassetid://76656666763120"
local IMAGE_JS = "rbxassetid://98193802969656"

local SOUND_TEXT = "rbxassetid://139545496211288"
local SOUND_ATTACK = "rbxassetid://124513140438986"
local SOUND_AMBIENT = "rbxassetid://139807624552008"

local active = false
local camConn = nil
local ambient = nil

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness

local function startRedFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.fromRGB(120, 0, 0)
	Lighting.Ambient = Color3.fromRGB(80, 0, 0)

	TweenService:Create(Lighting, TweenInfo.new(1, Enum.EasingStyle.Quad), {
		FogEnd = 35,
		FogStart = 0,
		Brightness = 0.25
	}):Play()
end

local function endRedFog()
	TweenService:Create(Lighting, TweenInfo.new(1.5, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

local function playSound(id, volume, looped)
	local s = Instance.new("Sound", Workspace)
	s.SoundId = id
	s.Volume = volume or 3.5
	s.Looped = looped or false
	s:Play()
	if not looped then
		Debris:AddItem(s, 12)
	end
	return s
end

local function caption(text)
	pcall(function()
		local MainUI = player:WaitForChild("PlayerGui"):WaitForChild("MainUI")
		local func = require(MainUI.Initiator.Main_Game)
		func.caption(text, true)
	end)
end

local function createPNG(size, texture, cframe)
	local part = Instance.new("Part")
	part.Name = "PresentationEntity"
	part.Size = size
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = cframe
	part.Parent = Workspace

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = texture
		d.Face = face
		d.Parent = part
	end
	return part
end

local function lockCameraOn(part)
	local oldType = camera.CameraType
	camera.CameraType = Enum.CameraType.Scriptable

	camConn = RunService.RenderStepped:Connect(function()
		if not part or not part.Parent then return end
		local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
		if not root then return end
		local pos = part.Position + (part.Position - root.Position).Unit * -7 + Vector3.new(0, 1, 0)
		camera.CFrame = CFrame.new(pos, part.Position)
	end)

	local walk = humanoid.WalkSpeed
	local jump = humanoid.JumpPower
	humanoid.WalkSpeed = 0
	humanoid.JumpPower = 0

	return function()
		if camConn then
			camConn:Disconnect()
			camConn = nil
		end
		camera.CameraType = oldType
		if humanoid then
			humanoid.WalkSpeed = walk
			humanoid.JumpPower = jump
		end
	end
end

local function screenJumpscare()
	playSound(SOUND_ATTACK, 4.5)

	-- Dano 70
	local hum = player.Character and player.Character:FindFirstChild("Humanoid")
	if hum then
		hum:TakeDamage(70)
	end

	local gui = Instance.new("ScreenGui")
	gui.Name = "PresentationJS"
	gui.IgnoreGuiInset = true
	gui.DisplayOrder = 200
	gui.Parent = player:WaitForChild("PlayerGui")

	local img = Instance.new("ImageLabel")
	img.Size = UDim2.new(1.15, 0, 1.15, 0)
	img.Position = UDim2.new(-0.075, 0, -0.075, 0)
	img.BackgroundTransparency = 1
	img.Image = IMAGE_JS
	img.ScaleType = Enum.ScaleType.Crop
	img.Parent = gui

	local shaking = true
	task.spawn(function()
		while shaking and img and img.Parent do
			img.Position = UDim2.new(
				-0.075 + math.random(-10, 10) / 200,
				0,
				-0.075 + math.random(-10, 10) / 200,
				0
			)
			task.wait(0.03)
		end
	end)

	task.wait(1.6)
	shaking = false

	local fade = TweenService:Create(img, TweenInfo.new(1.2, Enum.EasingStyle.Quad), {
		ImageTransparency = 1
	})
	fade:Play()
	fade.Completed:Wait()
	gui:Destroy()
end

local function spawn()
	if active then return end
	active = true

	char = player.Character or char
	hrp = char:FindFirstChild("HumanoidRootPart") or char:WaitForChild("HumanoidRootPart")
	humanoid = char:FindFirstChild("Humanoid") or char:WaitForChild("Humanoid")

	-- Névoa vermelha desde o início
	startRedFog()
	ambient = playSound(SOUND_AMBIENT, 2.5, true)

	-- ========== 1. PRIMEIRA APARIÇÃO ==========
	local part = createPNG(Vector3.new(7, 10, 0.2), IMAGE_FIRST, hrp.CFrame * CFrame.new(0, 3, -12))
	local unlock = lockCameraOn(part)

	playSound(SOUND_TEXT, 4)
	caption("But before you go,")
	task.wait(2.2)

	playSound(SOUND_TEXT, 4)
	caption("time for a presentation here!")
	task.wait(2.5)

	local away = hrp.Position + (part.Position - hrp.Position).Unit * 55 + Vector3.new(0, 2, 0)
	local runTween = TweenService:Create(part, TweenInfo.new(1.4, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		Position = away
	})
	runTween:Play()
	runTween.Completed:Wait()

	unlock()
	task.wait(0.5)

	if part and part.Parent then
		part:Destroy()
	end

	task.wait(0.6)

	-- ========== 2. ATAQUE: vem em você ==========
	hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart") or hrp
	humanoid = player.Character and player.Character:FindFirstChild("Humanoid") or humanoid

	local attack = createPNG(
		Vector3.new(7, 10, 0.2),
		IMAGE_ATTACK,
		hrp.CFrame * CFrame.new(0, 3, 14)
	)

	local unlock2 = lockCameraOn(attack)

	task.spawn(function()
		while attack and attack.Parent do
			local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
			if not root then break end

			local dir = (root.Position - attack.Position)
			if dir.Magnitude < 8 then break end

			if dir.Magnitude > 0.5 then
				dir = dir.Unit
				attack.CFrame = CFrame.new(attack.Position + dir * 18 * 0.05, root.Position)
			end
			task.wait(0.03)
		end
	end)

	local start = tick()
	while attack and attack.Parent and (tick() - start) < 3.5 do
		local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
		if root and (attack.Position - root.Position).Magnitude < 9 then
			break
		end
		task.wait(0.05)
	end

	unlock2()

	if attack and attack.Parent then
		attack:Destroy()
	end

	task.wait(0.15)

	-- ========== 3. JUMPSCARE + DANO 70 ==========
	screenJumpscare()

	-- Some a névoa vermelha
	endRedFog()

	if ambient then
		ambient:Stop()
		ambient:Destroy()
		ambient = nil
	end

	active = false
	print("Presentation acabou!")
end

player.Chatted:Connect(function(msg)
	if msg:lower() == "/presentation" then
		spawn()
	end
end)

print("✅ Presentation - névoa vermelha + dano 70!")
print("Use /presentation")

-- 👤 Stalker Window

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local camera = Workspace.CurrentCamera

local IMAGE = "rbxassetid://87802065154223"

local current = nil

local function findWindow()
	local rooms = Workspace:FindFirstChild("CurrentRooms")
	if not rooms then return nil end

	local candidates = {}
	for _, v in pairs(rooms:GetDescendants()) do
		local name = v.Name:lower()
		if v:IsA("BasePart") and (name:find("window") or name:find("glass") or name:find("pane")) then
			if (v.Position - hrp.Position).Magnitude < 80 then
				table.insert(candidates, v)
			end
		end
	end

	if #candidates > 0 then
		return candidates[math.random(1, #candidates)]
	end
	return nil
end

local function isLookingAt(targetPos)
	local camPos = camera.CFrame.Position
	local camLook = camera.CFrame.LookVector
	local dir = (targetPos - camPos)
	if dir.Magnitude > 50 then return false end
	dir = dir.Unit
	return camLook:Dot(dir) > 0.90
end

local function spawn()
	if current then return end

	local window = findWindow()
	local spawnCF

	if window then
		spawnCF = CFrame.new(window.Position + Vector3.new(0, 0, 1.2), hrp.Position)
		print("StalkerWindow na janela!")
	else
		spawnCF = hrp.CFrame * CFrame.new(0, 3, -14)
		print("StalkerWindow fallback")
	end

	local part = Instance.new("Part")
	part.Name = "StalkerWindow"
	part.Size = Vector3.new(6, 8, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = spawnCF
	part.Parent = Workspace
	current = part

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = IMAGE
		d.Face = face
		d.Parent = part
	end

	local looked = false
	task.spawn(function()
		while part.Parent and not looked do
			part.CFrame = CFrame.new(part.Position, camera.CFrame.Position)

			if isLookingAt(part.Position) then
				looked = true

				-- Vai pra BAIXO (não pra cima)
				local downPos = part.Position + Vector3.new(0, -14, 0)
				local tween = TweenService:Create(part, TweenInfo.new(1.2, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
					Position = downPos
				})
				tween:Play()
				tween.Completed:Wait()

				if part then part:Destroy() end
				current = nil
				print("StalkerWindow sumiu pra baixo!")
				break
			end
			task.wait(0.05)
		end
	end)
end

-- Spawn aleatório um pouco raro
task.spawn(function()
	while true do
		task.wait(math.random(70, 140))
		if math.random(1, 4) == 1 and not current then
			spawn()
		end
	end
end)

player.Chatted:Connect(function(msg)
	if msg:lower() == "/stalkerwindow" then
		spawn()
	end
end)

print("✅ Stalker Window carregado!")
print("Use /stalkerwindow")

-- 👤 Stalker Jumpscare - Porta + névoa ao encostar

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local camera = Workspace.CurrentCamera

local images = {
	"rbxassetid://111565947739320",
	"rbxassetid://73509060375391",
	"rbxassetid://135807608825423"
}

local imageIndex = 1
local current = nil
local touching = false

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness

local function startDarkFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(10, 10, 10)

	TweenService:Create(Lighting, TweenInfo.new(1.2, Enum.EasingStyle.Quad), {
		FogEnd = 18,
		FogStart = 0,
		Brightness = 0.1
	}):Play()
end

local function endDarkFog()
	TweenService:Create(Lighting, TweenInfo.new(1.8, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

local function findDoorFrontOrBack()
	local rooms = Workspace:FindFirstChild("CurrentRooms")
	if not rooms then return nil, nil end

	local frontDoors = {}
	local backDoors = {}
	local look = hrp.CFrame.LookVector

	for _, v in pairs(rooms:GetDescendants()) do
		if not v:IsA("BasePart") then continue end

		local name = v.Name:lower()
		local parentName = v.Parent and v.Parent.Name:lower() or ""
		local full = name .. " " .. parentName

		if full:find("wardrobe") or full:find("locker") or full:find("closet") or full:find("cabinet") then
			continue
		end

		local isDoor = name == "door" or name:find("door") or parentName:find("door")
		if not isDoor then continue end
		if v.Size.Magnitude < 4 then continue end

		local dist = (v.Position - hrp.Position).Magnitude
		if dist < 5 or dist > 55 then continue end

		local toDoor = (v.Position - hrp.Position).Unit
		local dot = look:Dot(toDoor)

		if dot > 0.1 then
			table.insert(frontDoors, {part = v, dist = dist})
		elseif dot < -0.1 then
			table.insert(backDoors, {part = v, dist = dist})
		else
			table.insert(frontDoors, {part = v, dist = dist})
		end
	end

	local function pickClosest(list)
		table.sort(list, function(a, b) return a.dist < b.dist end)
		return list[1] and list[1].part or nil
	end

	if math.random(1, 2) == 1 then
		local d = pickClosest(frontDoors)
		if d then return d, "frente" end
		d = pickClosest(backDoors)
		if d then return d, "trás" end
	else
		local d = pickClosest(backDoors)
		if d then return d, "trás" end
		d = pickClosest(frontDoors)
		if d then return d, "frente" end
	end

	return nil, nil
end

local function spawn()
	if current then current:Destroy() end
	touching = false

	local door, side = findDoorFrontOrBack()
	local spawnCF

	if door then
		-- Colado no meio da porta, um pouco pro lado
		local mid = door.Position + Vector3.new(0, 1.3, 0)
		local sideOff = door.CFrame.RightVector * 1.1
		-- um pouco pra fora da porta (em direção ao player)
		local towardPlayer = (hrp.Position - door.Position)
		if towardPlayer.Magnitude > 0.1 then
			towardPlayer = towardPlayer.Unit * 0.8
		else
			towardPlayer = Vector3.new(0, 0, 0)
		end
		spawnCF = CFrame.new(mid + sideOff + towardPlayer, hrp.Position)
		print("Na PORTA (" .. (side or "?") .. "): " .. door.Name)
	else
		if math.random(1, 2) == 1 then
			spawnCF = hrp.CFrame * CFrame.new(1.2, 3, -8)
			print("Fallback frente")
		else
			spawnCF = hrp.CFrame * CFrame.new(1.2, 3, 11)
			print("Fallback trás")
		end
	end

	local img = images[imageIndex]
	imageIndex = imageIndex + 1
	if imageIndex > #images then imageIndex = 1 end

	local part = Instance.new("Part")
	part.Name = "StalkerJumpscareDoor"
	part.Size = Vector3.new(6, 9, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = spawnCF
	part.Parent = Workspace
	current = part

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = img
		d.Face = face
		d.Parent = part
	end

	print("Imagem: " .. img)

	-- Sempre te olha (NÃO some ao olhar)
	task.spawn(function()
		while part.Parent do
			local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
			if root then
				part.CFrame = CFrame.new(part.Position, root.Position)

				-- Só some se ENCOSTAR
				if not touching and (part.Position - root.Position).Magnitude < 9 then
					touching = true
					current = nil

					startDarkFog()
					print("Encostou → névoa!")

					part:Destroy()

					task.wait(2.5)
					endDarkFog()
					touching = false
					break
				end
			end
			task.wait(0.04)
		end
	end)
end

player.Chatted:Connect(function(msg)
	if msg:lower() == "/stalkerjumspecare" then
		spawn()
	end
end)

print("✅ Stalker Jumpscare atualizado!")
print("Use /stalkerjumspecare")

-- 👤 Stalker 2 - Só uma vez

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local Debris = game:GetService("Debris")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local camera = Workspace.CurrentCamera
local humanoid = char:WaitForChild("Humanoid")

local CHASE_IMAGE = "rbxassetid://99869789201897"
local CHASE_MUSIC = "rbxassetid://129085629036594"
local ACHIEVEMENT_IMAGE = "rbxassetid://80917066864552"

local chaseEntity = nil
local chaseMusic = nil
local redGui = nil
local redLabel = nil
local runShaking = false
local cameraShake = false
local shakeConn = nil
local achievementGiven = false
local alreadyAppeared = false -- só 1 vez

local function giveAchievement()
	if achievementGiven then return end
	achievementGiven = true

	local achievementGiver = loadstring(game:HttpGet(
		"https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
	))()

	achievementGiver({
		Title = "Too Far Away",
		Desc = "You outran the distance.",
		Reason = "499 blocks was not enough...",
		Image = ACHIEVEMENT_IMAGE
	})
end

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness

local function startDarkFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(10, 10, 10)

	TweenService:Create(Lighting, TweenInfo.new(1.2, Enum.EasingStyle.Quad), {
		FogEnd = 18,
		FogStart = 0,
		Brightness = 0.1
	}):Play()
end

local function endDarkFog()
	TweenService:Create(Lighting, TweenInfo.new(1.8, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

local function startCameraShake()
	if cameraShake then return end
	cameraShake = true
	shakeConn = RunService.RenderStepped:Connect(function()
		if not cameraShake then return end
		camera.CFrame = camera.CFrame * CFrame.new(
			math.random(-6, 6) / 18,
			math.random(-6, 6) / 18,
			0
		)
	end)
end

local function stopCameraShake()
	cameraShake = false
	if shakeConn then
		shakeConn:Disconnect()
		shakeConn = nil
	end
end

local function clearRedText()
	runShaking = false
	if redGui then
		redGui:Destroy()
		redGui = nil
		redLabel = nil
	end
end

local function redText(text, isRun)
	if not redGui or not redGui.Parent then
		redGui = Instance.new("ScreenGui")
		redGui.Name = "Stalker2Text"
		redGui.Parent = player:WaitForChild("PlayerGui")

		redLabel = Instance.new("TextLabel")
		redLabel.Size = UDim2.new(0.7, 0, 0, 32)
		redLabel.Position = UDim2.new(0.15, 0, 0.74, 0)
		redLabel.BackgroundTransparency = 1
		redLabel.TextColor3 = Color3.fromRGB(255, 45, 45)
		redLabel.TextStrokeTransparency = 0.35
		redLabel.TextStrokeColor3 = Color3.new(0, 0, 0)
		redLabel.TextScaled = true
		redLabel.Font = Enum.Font.Gotham
		redLabel.Parent = redGui
	end

	redLabel.Text = text

	if isRun and not runShaking then
		runShaking = true
		task.spawn(function()
			while runShaking and redLabel and redLabel.Parent do
				redLabel.Position = UDim2.new(
					0.15 + math.random(-6, 6) / 300,
					0,
					0.74 + math.random(-6, 6) / 300,
					0
				)
				task.wait(0.04)
			end
			if redLabel and redLabel.Parent then
				redLabel.Position = UDim2.new(0.15, 0, 0.74, 0)
			end
		end)
	elseif not isRun then
		runShaking = false
	end
end

local function vanishDown(part)
	if not part or not part.Parent then return end

	local smoke = Instance.new("ParticleEmitter")
	smoke.Texture = "rbxassetid://243660364"
	smoke.Color = ColorSequence.new(Color3.fromRGB(40, 40, 40), Color3.fromRGB(90, 90, 90))
	smoke.Size = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 2.5),
		NumberSequenceKeypoint.new(1, 6)
	})
	smoke.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 0.25),
		NumberSequenceKeypoint.new(1, 1)
	})
	smoke.Lifetime = NumberRange.new(1.4, 2.2)
	smoke.Rate = 50
	smoke.Speed = NumberRange.new(1, 3)
	smoke.SpreadAngle = Vector2.new(50, 50)
	smoke.Parent = part

	local groundPos = Vector3.new(part.Position.X, part.Position.Y - 16, part.Position.Z)
	local downTween = TweenService:Create(part, TweenInfo.new(1.6, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		Position = groundPos
	})
	downTween:Play()
	downTween.Completed:Wait()

	smoke.Enabled = false
	task.wait(0.35)

	if part and part.Parent then
		part:Destroy()
	end
end

local function startChaseMode()
	if chaseEntity then return end
	if alreadyAppeared then
		print("Stalker2 já apareceu nessa partida.")
		return
	end
	alreadyAppeared = true

	redText("It is 499 blocks away from you", false)
	startDarkFog()
	startCameraShake()

	chaseMusic = Instance.new("Sound", Workspace)
	chaseMusic.SoundId = CHASE_MUSIC
	chaseMusic.Volume = 3.2
	chaseMusic.Looped = true
	chaseMusic:Play()

	local chaser = Instance.new("Part")
	chaser.Name = "Stalker2"
	chaser.Size = Vector3.new(8, 12, 0.2)
	chaser.Transparency = 1
	chaser.Anchored = true
	chaser.CanCollide = false
	chaser.Parent = Workspace

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = CHASE_IMAGE
		d.Face = face
		d.Parent = chaser
	end

	chaser.CFrame = hrp.CFrame * CFrame.new(0, 4, 50)
	chaseEntity = chaser

	local speed = 7.5
	local startTime = tick()
	local duration = 20

	task.spawn(function()
		while chaseEntity and chaseEntity.Parent do
			if (tick() - startTime) >= duration then
				local entity = chaseEntity
				chaseEntity = nil

				stopCameraShake()
				if chaseMusic then
					chaseMusic:Stop()
					chaseMusic:Destroy()
				end
				clearRedText()

				vanishDown(entity)
				endDarkFog()

				redText("You survived", false)
				giveAchievement()
				task.delay(3, clearRedText)
				break
			end

			local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
			local hum = player.Character and player.Character:FindFirstChild("Humanoid")
			if not root then
				task.wait(0.2)
				continue
			end

			local realDist = (chaser.Position - root.Position).Magnitude
			local distance = math.max(0, math.floor(realDist * 2.2))

			if realDist <= 20 then
				redText("RUN", true)
				speed = 11
			else
				redText("It is " .. distance .. " blocks away from you", false)
			end

			Lighting.Ambient = Color3.fromRGB(140, 0, 0)
			Lighting.Brightness = 0.35

			local dir = (root.Position - chaser.Position)
			if dir.Magnitude > 0.5 then
				dir = dir.Unit
				local shakeOffset = Vector3.new(
					math.random(-4, 4) / 12,
					math.random(-3, 3) / 12,
					0
				)
				chaser.CFrame = CFrame.new(chaser.Position + dir * speed * 0.1 + shakeOffset, root.Position)
			end

			if realDist < 9 then
				if hum then
					hum:TakeDamage(40)
				end

				local entity = chaseEntity
				chaseEntity = nil

				stopCameraShake()
				if chaseMusic then
					chaseMusic:Stop()
					chaseMusic:Destroy()
				end
				clearRedText()

				vanishDown(entity)
				endDarkFog()

				Lighting.Ambient = Color3.new(1, 1, 1)
				Lighting.Brightness = 1
				break
			end

			task.wait(0.1)
		end
	end)
end

-- Spawn aleatório (só se ainda não apareceu)
task.spawn(function()
	while not alreadyAppeared do
		task.wait(math.random(150, 280))
		if not alreadyAppeared and math.random(1, 8) == 1 then
			startChaseMode()
		end
	end
end)

player.Chatted:Connect(function(msg)
	if msg:lower() == "/stalker2" then
		startChaseMode()
	end
end)

print("✅ Stalker 2 - Só 1 vez por partida!")
print("Use /stalker2")

-- 👤 StalkerDoor Final + Spawn bem bem raro

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local camera = Workspace.CurrentCamera

local DOOR_IMAGE   = "rbxassetid://89678769967446"
local WINDOW_IMG1  = "rbxassetid://83554491093997"
local WINDOW_IMG2  = "rbxassetid://71225962988620"
local MAIN_SOUND   = "rbxassetid://125434017517616"
local ACHIEVEMENT_IMAGE = "rbxassetid://114867313134951"

local current = nil
local active = false
local zooming = false
local zoomConn = nil
local achievementGiven = false

local function giveAchievement()
	if achievementGiven then return end
	achievementGiven = true

	local achievementGiver = loadstring(game:HttpGet(
		"https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
	))()

	achievementGiver({
		Title = "Behind The Door",
		Desc = "You survived the door watcher.",
		Reason = "He was always there...",
		Image = ACHIEVEMENT_IMAGE
	})
end

local function findDoorOrSpot()
	local rooms = Workspace:FindFirstChild("CurrentRooms")
	if not rooms then return nil, "fallback" end

	local doors = {}
	local spots = {}

	for _, v in pairs(rooms:GetDescendants()) do
		if not v:IsA("BasePart") then continue end

		local name = v.Name:lower()
		local parentName = v.Parent and v.Parent.Name:lower() or ""
		local full = name .. " " .. parentName
		local dist = (v.Position - hrp.Position).Magnitude

		if full:find("wardrobe") or full:find("locker") or full:find("closet") or full:find("cabinet") then
			continue
		end

		if (name:find("door") or parentName:find("door")) and dist > 5 and dist < 45 and v.Size.Magnitude > 4 then
			table.insert(doors, v)
		end

		if dist > 8 and dist < 35 then
			if name:find("wall") or name:find("shelf") or name:find("table") or name:find("desk")
				or name:find("book") or name:find("paint") or name:find("frame")
				or parentName:find("furniture") or parentName:find("decor") then
				if v.Size.Magnitude > 3 then
					table.insert(spots, v)
				end
			end
		end
	end

	if #doors > 0 and (math.random(1, 10) <= 7 or #spots == 0) then
		return doors[math.random(1, #doors)], "door"
	elseif #spots > 0 then
		return spots[math.random(1, #spots)], "spot"
	elseif #doors > 0 then
		return doors[1], "door"
	end

	return nil, "fallback"
end

local function findWindow()
	local rooms = Workspace:FindFirstChild("CurrentRooms")
	if not rooms then return nil end

	local candidates = {}
	for _, v in pairs(rooms:GetDescendants()) do
		local name = v.Name:lower()
		if v:IsA("BasePart") and (name:find("window") or name:find("glass") or name:find("pane")) then
			if (v.Position - hrp.Position).Magnitude < 80 then
				table.insert(candidates, v)
			end
		end
	end
	if #candidates > 0 then
		return candidates[math.random(1, #candidates)]
	end
	return nil
end

local function setFace(part, texture)
	for _, d in pairs(part:GetChildren()) do
		if d:IsA("Decal") then
			d.Texture = texture
		end
	end
end

local function createPNG(size, texture, cframe)
	local part = Instance.new("Part")
	part.Size = size
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = cframe
	part.Parent = Workspace

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = texture
		d.Face = face
		d.Parent = part
	end
	return part
end

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness
local oldCamType

local function startDarkFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(10, 10, 10)

	TweenService:Create(Lighting, TweenInfo.new(1.2, Enum.EasingStyle.Quad), {
		FogEnd = 18,
		FogStart = 0,
		Brightness = 0.1
	}):Play()
end

local function endDarkFog()
	TweenService:Create(Lighting, TweenInfo.new(1.8, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

local function startZoom(entity)
	zooming = true
	oldCamType = camera.CameraType
	camera.CameraType = Enum.CameraType.Scriptable

	zoomConn = RunService.RenderStepped:Connect(function()
		if not zooming or not entity or not entity.Parent then return end
		local targetPos = entity.Position + (entity.Position - hrp.Position).Unit * -6 + Vector3.new(0, 1, 0)
		camera.CFrame = CFrame.new(targetPos, entity.Position)
	end)
end

local function stopZoom()
	zooming = false
	if zoomConn then
		zoomConn:Disconnect()
		zoomConn = nil
	end
	if oldCamType then
		camera.CameraType = oldCamType
	end
end

local function spawn()
	if active then return end
	active = true

	local target, kind = findDoorOrSpot()
	local spawnCF

	if target and kind == "door" then
		local mid = target.Position + Vector3.new(0, 1.2, 0)
		local side = target.CFrame.RightVector * 1.3
		spawnCF = CFrame.new(mid + side, hrp.Position)
		print("Spawn na PORTA: " .. target.Name)
	elseif target and kind == "spot" then
		local pos = target.Position + Vector3.new(0, 2, 0) + (target.Position - hrp.Position).Unit * -1.5
		spawnCF = CFrame.new(pos, hrp.Position)
		print("Spawn em lugar visível: " .. target.Name)
	else
		spawnCF = hrp.CFrame * CFrame.new(1.5, 3, -10)
		print("Fallback")
	end

	local doorEntity = createPNG(Vector3.new(6, 9, 0.2), DOOR_IMAGE, spawnCF)
	current = doorEntity

	startZoom(doorEntity)

	task.spawn(function()
		while doorEntity.Parent do
			doorEntity.CFrame = CFrame.new(doorEntity.Position, hrp.Position)
			task.wait(0.03)
		end
	end)

	task.wait(2.5)

	local sidePos = doorEntity.Position + doorEntity.CFrame.RightVector * 12
	local sideTween = TweenService:Create(doorEntity, TweenInfo.new(1.2, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
		Position = sidePos
	})
	sideTween:Play()
	sideTween.Completed:Wait()

	stopZoom()

	if doorEntity then doorEntity:Destroy() end
	current = nil
	task.wait(0.3)

	local window = findWindow()
	local windowCF
	if window then
		windowCF = CFrame.new(window.Position + Vector3.new(0, 0, 1.2), hrp.Position)
	else
		windowCF = hrp.CFrame * CFrame.new(0, 3, -14)
	end

	local windowEntity = createPNG(Vector3.new(6, 8, 0.2), WINDOW_IMG1, windowCF)
	current = windowEntity
	print("StalkerDoor na janela!")

	local music = Instance.new("Sound", Workspace)
	music.SoundId = MAIN_SOUND
	music.Volume = 4
	music.PlaybackSpeed = 0.20
	music.Looped = false
	music:Play()

	local usingFirst = true
	local startTime = tick()
	local duration = 6

	task.spawn(function()
		while windowEntity and windowEntity.Parent do
			local elapsed = tick() - startTime
			if elapsed >= duration then break end

			windowEntity.CFrame = CFrame.new(windowEntity.Position, camera.CFrame.Position)

			usingFirst = not usingFirst
			setFace(windowEntity, usingFirst and WINDOW_IMG1 or WINDOW_IMG2)

			local progress = elapsed / duration
			local speed = math.max(0.04, 0.22 - (progress * 0.18))
			task.wait(speed)
		end
	end)

	task.wait(duration)

	if windowEntity and windowEntity.Parent then
		windowEntity:Destroy()
	end
	current = nil

	startDarkFog()
	print("Sumiu da janela → névoa!")

	local finished = false
	music.Ended:Connect(function()
		finished = true
	end)

	task.spawn(function()
		while music and music.Parent and not finished do
			if not music.IsPlaying and music.TimePosition > 0.3 then
				finished = true
				break
			end
			task.wait(0.15)
		end
		finished = true
	end)

	repeat task.wait(0.1) until finished

	endDarkFog()

	if music then
		music:Stop()
		music:Destroy()
	end

	giveAchievement()
	print("Som acabou → névoa sumiu + conquista!")

	active = false
end

-- Spawn ALEATÓRIO BEM BEM BEM RARO
task.spawn(function()
	while true do
		task.wait(math.random(180, 360)) -- 3 a 6 minutos
		if math.random(1, 8) == 1 and not active then
			spawn()
		end
	end
end)

player.Chatted:Connect(function(msg)
	if msg:lower() == "/stalkerdoor" then
		spawn()
	end
end)

print("✅ StalkerDoor Final + bem bem raro!")
print("Use /stalkerdoor")

-- 👤 Chica Entity

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local camera = Workspace.CurrentCamera

local IMAGE = "rbxassetid://95421528567921"

local current = nil

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness

local function startDarkFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(10, 10, 10)

	TweenService:Create(Lighting, TweenInfo.new(1.2, Enum.EasingStyle.Quad), {
		FogEnd = 18,
		FogStart = 0,
		Brightness = 0.1
	}):Play()
end

local function endDarkFog()
	TweenService:Create(Lighting, TweenInfo.new(2, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

local function findSpot()
	local rooms = Workspace:FindFirstChild("CurrentRooms")
	if not rooms then return nil end

	local candidates = {}

	for _, v in pairs(rooms:GetDescendants()) do
		if not v:IsA("BasePart") then continue end

		local name = v.Name:lower()
		local parentName = v.Parent and v.Parent.Name:lower() or ""
		local dist = (v.Position - hrp.Position).Magnitude

		local isGood =
			name:find("door") or parentName:find("door") or
			name:find("wardrobe") or name:find("locker") or name:find("closet") or
			name:find("wall") or name:find("shelf") or name:find("table") or
			name:find("cabinet") or name:find("frame")

		if isGood and dist > 8 and dist < 40 and v.Size.Magnitude > 2 then
			table.insert(candidates, v)
		end
	end

	if #candidates > 0 then
		return candidates[math.random(1, #candidates)]
	end
	return nil
end

local function spawn()
	if current then return end

	local spot = findSpot()
	local spawnCF

	if spot then
		local offset = spot.CFrame.RightVector * (math.random(0, 1) == 0 and 2.2 or -2.2)
		local back = (spot.Position - hrp.Position).Unit * -0.8
		local pos = spot.Position + offset + back + Vector3.new(0, 2.2, 0)
		spawnCF = CFrame.new(pos, hrp.Position)
		print("Chica em: " .. spot.Name)
	else
		spawnCF = hrp.CFrame * CFrame.new(4, 2.5, -6)
		print("Chica fallback")
	end

	local part = Instance.new("Part")
	part.Name = "ChicaEntity"
	part.Size = Vector3.new(5, 7, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.CFrame = spawnCF
	part.Parent = Workspace
	current = part

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = IMAGE
		d.Face = face
		d.Parent = part
	end

	print("Chica apareceu!")

	task.spawn(function()
		while part.Parent do
			local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
			if root then
				part.CFrame = CFrame.new(part.Position, root.Position)

				-- Chegou perto → some + névoa
				if (part.Position - root.Position).Magnitude < 9 then
					part:Destroy()
					current = nil
					print("Chica sumiu → névoa!")

					startDarkFog()
					task.wait(2.5)
					endDarkFog()
					break
				end
			end
			task.wait(0.04)
		end
	end)
end

-- Spawn aleatório (várias vezes)
task.spawn(function()
	while true do
		task.wait(math.random(45, 100))
		if math.random(1, 3) == 1 and not current then
			spawn()
		end
	end
end)

player.Chatted:Connect(function(msg)
	if msg:lower() == "/chica" then
		spawn()
	end
end)

print("✅ Chica carregada!")
print("Use /chica")

-- 👤 Stalker 4 - Final

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Lighting = game:GetService("Lighting")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local humanoid = char:WaitForChild("Humanoid")
local camera = Workspace.CurrentCamera

local IMAGE1 = "rbxassetid://101139774680573"
local IMAGE2 = "rbxassetid://94733115025990"
local SPAWN_SOUND = "rbxassetid://73171709370707"

local current = nil
local chaseMusic = nil
local shaking = false
local achievementGiven = false
local shakeConnection = nil
local alreadyDamaged = false
local spawnCount = 0
local MAX_SPAWNS = 2

local function caption(text)
	local MainUI = player:WaitForChild("PlayerGui"):WaitForChild("MainUI")
	local func = require(MainUI.Initiator.Main_Game)
	func.caption(text, true)
end

local function giveAchievement()
	if achievementGiven then return end
	achievementGiven = true

	local achievementGiver = loadstring(game:HttpGet(
		"https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
	))()

	achievementGiver({
		Title = "Next time, he'll get you.",
		Desc = "You can't hide.",
		Reason = "survive your nightmare...",
		Image = "rbxassetid://128401329091126"
	})
end

local function startShake()
	if shaking then return end
	shaking = true

	shakeConnection = RunService.RenderStepped:Connect(function()
		if not shaking then return end
		camera.CFrame = camera.CFrame * CFrame.new(
			math.random(-10, 10) / 14,
			math.random(-10, 10) / 14,
			math.random(-4, 4) / 20
		)
	end)
end

local function stopShake()
	shaking = false
	if shakeConnection then
		shakeConnection:Disconnect()
		shakeConnection = nil
	end
end

local oldFogEnd, oldFogStart, oldFogColor, oldAmbient, oldBrightness

local function startDarkFog()
	oldFogEnd = Lighting.FogEnd
	oldFogStart = Lighting.FogStart
	oldFogColor = Lighting.FogColor
	oldAmbient = Lighting.Ambient
	oldBrightness = Lighting.Brightness

	Lighting.FogColor = Color3.new(0, 0, 0)
	Lighting.Ambient = Color3.fromRGB(15, 15, 15)

	TweenService:Create(Lighting, TweenInfo.new(1.2, Enum.EasingStyle.Quad), {
		FogEnd = 35,
		FogStart = 0,
		Brightness = 0.15
	}):Play()
end

local function endDarkFog()
	TweenService:Create(Lighting, TweenInfo.new(1.8, Enum.EasingStyle.Quad), {
		FogEnd = oldFogEnd or 1000,
		FogStart = oldFogStart or 0,
		Brightness = oldBrightness or 1,
		Ambient = oldAmbient or Color3.new(1, 1, 1),
		FogColor = oldFogColor or Color3.new(0.75, 0.75, 0.75)
	}):Play()
end

local function cleanup()
	stopShake()
	endDarkFog()

	if current then
		current:Destroy()
		current = nil
	end
	if chaseMusic then
		chaseMusic:Stop()
		chaseMusic:Destroy()
		chaseMusic = nil
	end
end

local function spawn()
	if current then return end
	if spawnCount >= MAX_SPAWNS then
		print("Stalker4 já apareceu 2 vezes nessa partida.")
		return
	end

	spawnCount = spawnCount + 1
	alreadyDamaged = false

	local part = Instance.new("Part")
	part.Name = "Stalker4"
	part.Size = Vector3.new(9, 12, 0.2)
	part.Transparency = 1
	part.Anchored = true
	part.CanCollide = false
	part.Parent = Workspace

	for _, face in pairs(Enum.NormalId:GetEnumItems()) do
		local d = Instance.new("Decal")
		d.Texture = IMAGE1
		d.Face = face
		d.Parent = part
	end

	part.CFrame = hrp.CFrame * CFrame.new(0, 3, 20)
	current = part

	caption("do not hide just run")

	chaseMusic = Instance.new("Sound", Workspace)
	chaseMusic.SoundId = SPAWN_SOUND
	chaseMusic.Volume = 3.5
	chaseMusic.PlaybackSpeed = 0.20
	chaseMusic.Looped = false
	chaseMusic:Play()

	task.spawn(function()
		while chaseMusic and chaseMusic.Parent do
			if not chaseMusic.IsPlaying then
				chaseMusic.TimePosition = 1.1
				chaseMusic:Play()
			end
			task.wait(0.08)
		end
	end)

	print("Stalker4 apareceu! (" .. spawnCount .. "/2)")

	-- Sempre te olhando
	task.spawn(function()
		while part.Parent do
			local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
			if root then
				part.CFrame = CFrame.new(part.Position, root.Position)
			end
			task.wait(0.03)
		end
	end)

	-- 4s → troca imagem + névoa + treme + persegue
	task.delay(4, function()
		if not part.Parent then return end

		for _, d in pairs(part:GetChildren()) do
			if d:IsA("Decal") then
				d.Texture = IMAGE2
			end
		end

		startDarkFog()
		startShake()

		local startTime = tick()

		while part.Parent and (tick() - startTime) < 20 do
			local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
			local hum = player.Character and player.Character:FindFirstChild("Humanoid")

			if root then
				local dir = (root.Position - part.Position)
				if dir.Magnitude > 0.5 then
					dir = dir.Unit
					local shakeOffset = Vector3.new(
						math.random(-3, 3) / 10,
						math.random(-3, 3) / 10,
						0
					)
					part.CFrame = CFrame.new(part.Position + dir * 10 * 0.05 + shakeOffset, root.Position)
				end

				-- Dano 60 → some na hora
				if not alreadyDamaged and (part.Position - root.Position).Magnitude < 8 then
					alreadyDamaged = true
					if hum then
						hum:TakeDamage(60)
					end
					print("Tomou 60 de dano → sumiu!")
					cleanup()
					giveAchievement()
					return
				end
			end
			task.wait(0.03)
		end

		-- Sobreviveu os 20s
		cleanup()
		giveAchievement()
		print("Stalker4 sumiu + conquista!")
	end)
end

-- Spawn aleatório BEM BEM RARO (mas aparece na partida)
task.spawn(function()
	task.wait(math.random(40, 90)) -- primeira chance depois de 40~90s
	while spawnCount < MAX_SPAWNS do
		if math.random(1, 3) == 1 then -- chance razoável de aparecer
			spawn()
		end
		task.wait(math.random(100, 180)) -- bem espaçado
	end
end)

player.Chatted:Connect(function(msg)
	if msg:lower() == "/stalker4" then
		spawn()
	end
end)

print("✅ Stalker4 Final!")
print("Máximo 2 vezes por partida | Use /stalker4")

-- 👤 Stalker5 - Bem Raro + Resto igual

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local humanoid = char:WaitForChild("Humanoid")
local camera = Workspace.CurrentCamera

local images = {
    "rbxassetid://117555949177010",
    "rbxassetid://117693738162384",
    "rbxassetid://127803001168102",
    "rbxassetid://72001283472257",
    "rbxassetid://106205359037024"
}

local SPAWN_SOUND = "rbxassetid://85765588267110"

local current = nil
local music = nil
local achievementGiven = false

local function giveAchievement()
    if achievementGiven then return end
    achievementGiven = true

    local achievementGiver = loadstring(game:HttpGet(
        "https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
    ))()

    achievementGiver({
        Title = "YELLING'S WONT STOP...",
        Desc = "You survived yelling creature",
        Reason = "I h&t= y&ou",
        Image = "rbxassetid://99854480974574"
    })
end

local function spawn()
    if current then current:Destroy() end
    if music then music:Stop() end

    local part = Instance.new("Part")
    part.Name = "NovaEntidade"
    part.Size = Vector3.new(9, 12, 0.2)
    part.Transparency = 1
    part.Anchored = true
    part.CanCollide = false
    part.Parent = Workspace

    for _, face in pairs(Enum.NormalId:GetEnumItems()) do
        local d = Instance.new("Decal")
        d.Texture = images[1]
        d.Face = face
        d.Parent = part
    end

    part.CFrame = hrp.CFrame * CFrame.new(0, 3, 20)
    current = part

    music = Instance.new("Sound", Workspace)
    music.SoundId = SPAWN_SOUND
    music.Volume = 3.5
    music.PlaybackSpeed = 0.30
    music.Looped = false
    music:Play()

    print("Nova entidade apareceu!")

    music.Ended:Connect(function()
        if current then
            current:Destroy()
            current = nil
        end
        if music then music:Stop() end
        giveAchievement()
        print("Som acabou → sumiu + conquista!")
    end)

    -- Sempre te olhando
    task.spawn(function()
        while part.Parent do
            part.CFrame = CFrame.new(part.Position, camera.CFrame.Position)
            task.wait(0.03)
        end
    end)

    -- Troca de imagem
    task.spawn(function()
        for i = 2, #images do
            task.wait(0.6)
            if not part.Parent then return end

            for _, d in pairs(part:GetChildren()) do
                if d:IsA("Decal") then
                    d.Texture = images[i]
                end
            end
        end

        task.wait(2)
        if not part.Parent then return end

        print("Indo atrás!")

        while part.Parent do
            local root = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
            if root then
                local dir = (root.Position - part.Position)
                if dir.Magnitude > 0.5 then
                    dir = dir.Unit
                    part.CFrame = CFrame.new(part.Position + dir * 10 * 0.05, root.Position)
                end

                if (part.Position - root.Position).Magnitude < 8 then
                    humanoid:TakeDamage(30)
                    part:Destroy()
                    current = nil
                    if music then music:Stop() end
                    giveAchievement()
                    break
                end
            end
            task.wait(0.03)
        end
    end)
end

-- Spawn aleatório BEM RARO
task.spawn(function()
    while true do
        task.wait(math.random(120, 240)) -- bem raro (2 a 4 minutos)
        if math.random(1, 6) == 1 then
            spawn()
        end
    end
end)

player.Chatted:Connect(function(msg)
    if msg:lower() == "/stalker5" then
        spawn()
    end
end)

print("✅ Stalker5 (bem raro)!")
print("Use /stalker5")

-- 👤 /bon - Final (Aleatório Bem Raro)

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local Debris = game:GetService("Debris")

local player = Players.LocalPlayer
local char = player.Character or player.CharacterAdded:Wait()
local hrp = char:WaitForChild("HumanoidRootPart")
local camera = Workspace.CurrentCamera

local IMAGE = "rbxassetid://83332839682670"
local SPAWN_SOUND = "rbxassetid://108665567548115"
local DESPAWN_SOUND = "rbxassetid://7428927488"

local current = nil
local achievementGiven = false

local function playSound(id)
    local s = Instance.new("Sound", Workspace)
    s.SoundId = id
    s.Volume = 3.5
    s:Play()
    Debris:AddItem(s, 5)
end

local function giveAchievement()
    if achievementGiven then return end
    achievementGiven = true

    local achievementGiver = loadstring(game:HttpGet(
        "https://raw.githubusercontent.com/RegularVynixu/Utilities/main/Doors/Custom%20Achievements/Source.lua"
    ))()

    achievementGiver({
        Title = "He loves watching you...",
        Desc = "He's coming back...",
        Reason = "Bon is looking at you.",
        Image = "rbxassetid://76440712166258"
    })
end

local function spawn()
    if current then current:Destroy() end

    local part = Instance.new("Part")
    part.Name = "Stalker"
    part.Size = Vector3.new(9, 12, 2.5)
    part.Transparency = 1
    part.Anchored = true
    part.CanCollide = false
    part.Parent = Workspace

    for _, face in pairs(Enum.NormalId:GetEnumItems()) do
        local decal = Instance.new("Decal")
        decal.Texture = IMAGE
        decal.Face = face
        decal.Parent = part
    end

    part.CFrame = hrp.CFrame * CFrame.new(0, 3, 20)
    current = part

    playSound(SPAWN_SOUND)
    print("Spawnou!")

    -- Sempre te olhando
    task.spawn(function()
        while part.Parent do
            part.CFrame = CFrame.new(part.Position, camera.CFrame.Position)
            task.wait(0.03)
        end
    end)

    -- Some quando olhar + som + conquista
    task.spawn(function()
        while part.Parent do
            local dir = (part.Position - camera.CFrame.Position).Unit
            if camera.CFrame.LookVector:Dot(dir) > 0.75 then
                playSound(DESPAWN_SOUND)
                giveAchievement()
                part:Destroy()
                current = nil
                print("Sumiu + conquista!")
                break
            end
            task.wait(0.1)
        end
    end)
end

-- Spawn aleatório BEM RARO
task.spawn(function()
    while true do
        task.wait(math.random(90, 180)) -- bem raro (1min30s a 3min)
        if math.random(1, 5) == 1 then
            spawn()
        end
    end
end)

player.Chatted:Connect(function(msg)
    if msg:lower() == "/bon" then
        spawn()
    end
end)

print("Pronto. Digite /bon")

local Lighting = game:GetService("Lighting")

local cc = Lighting:FindFirstChild("DarkMode")
if not cc then
	cc = Instance.new("ColorCorrectionEffect")
	cc.Name = "DarkMode"
	cc.Parent = Lighting
end

while task.wait(0.2) do
	Lighting.ClockTime = 0
	Lighting.Brightness = 1.5
	Lighting.ExposureCompensation = -0.35
	Lighting.Ambient = Color3.fromRGB(55, 55, 55)
	Lighting.OutdoorAmbient = Color3.fromRGB(45, 45, 45)
	Lighting.EnvironmentDiffuseScale = 0.45
	Lighting.EnvironmentSpecularScale = 0.2
	Lighting.ShadowSoftness = 0.7

	cc.Brightness = -0.03
	cc.Contrast = 0.08
	cc.Saturation = -0.02
end
