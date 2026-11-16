-- mm2 updated script --

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local CoreGui = game:GetService('CoreGui')
local StarterGui = game:GetService('StarterGui')
local GuiService = game:GetService('GuiService')
local VIM = game:GetService('VirtualInputManager')
local RunService = game:GetService('RunService')
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local HttpService = game:GetService("HttpService")

local LastTradePartner = nil

-- ===== СПИСОК ИМЁН =====
local FakeNames = {
	"xX_ShadowSlayer_Xx", "BloxyKing2008", "NoobMaster69", "PixelKnightz",
	"CrimsonReaperX", "MidnightFury77", "ZeroHavoc", "EpicGamer_LOL",
	"SilentStorm_YT", "FrostWolfie", "DragonHunter999", "SkyBreaker42",
	"VortexHaze", "PhantomRiderX", "NebulaCraze", "ToxicBubbles",
	"MysticBoba", "RobloxTrader01", "GamerGirl_Lyra", "SapphireWisp",
	"NinjaCookie123", "FluffyPandaUwU", "GoldenAegis", "VenomViperZ",
	"AstralFoxy", "MoonlightRose", "ChaosKnightX", "SilverScale99",
	"OmegaPredator", "EclipsedSoul", "EmeraldEcho", "CipherStorm",
	"PhoenixWraith", "ZephyrBlade", "InkyOctopus", "QuantumLynx",
	"DizzyDoodle", "NeonMango", "PiratePudding", "WaffleOverlord",
}
local SpawnedFakePlayers = {}
local FakePlayerIds = {}

-- ===== ДОБАВЛЕНИЕ В LEADERBOARD MM2 (ПЕРЕХВАТ) =====
local FakePlayersInLeaderboard = {}

local function AddFakePlayerToLeaderboard(username)
	if not username or username == "" then
		warn("[AddFakePlayer] No username provided")
		return false
	end

	if FakePlayersInLeaderboard[username] then
		return false
	end

	if Players:FindFirstChild(username) then
		warn("[AddFakePlayer] Player " .. username .. " already exists")
		return false
	end

	local success, fakePlayer = pcall(function()
		local newPlayer = Instance.new("Player")
		newPlayer.Name = username
		newPlayer.Parent = Players
		
		local fakeId = math.random(1000000, 9999999)
		newPlayer.UserId = fakeId
		
		FakePlayersInLeaderboard[username] = {
			player = newPlayer,
			userId = fakeId,
			created = os.time()
		}
		
		return newPlayer
	end)

	if success and fakePlayer then
		print("[AddFakePlayer] Successfully added " .. username .. " to MM2 leaderboard")
		
		pcall(function()
			for _, conn in pairs(getconnections(Players.PlayerAdded)) do
				pcall(function() conn.Function(fakePlayer) end)
			end
		end)
		
		return true
	else
		warn("[AddFakePlayer] Failed to add " .. username)
		return false
	end
end

local function RemoveFakePlayersFromLeaderboard()
	for username, data in pairs(FakePlayersInLeaderboard) do
		pcall(function()
			if data.player and data.player.Parent then
				data.player:Destroy()
			end
		end)
	end
	FakePlayersInLeaderboard = {}
	print("[AddFakePlayer] All fake players removed from MM2 leaderboard")
end

local function AddMultipleFakePlayers(count)
	count = count or 5
	local added = 0
	local usedNames = {}
	
	for i = 1, count do
		local name = FakeNames[math.random(1, #FakeNames)]
		local attempts = 0
		while (FakePlayersInLeaderboard[name] or usedNames[name]) and attempts < 10 do
			name = FakeNames[math.random(1, #FakeNames)]
			attempts = attempts + 1
		end
		if not FakePlayersInLeaderboard[name] and not usedNames[name] then
			if AddFakePlayerToLeaderboard(name) then
				added = added + 1
				usedNames[name] = true
			end
		end
	end
	print("[AddFakePlayer] Added " .. added .. " fake players to MM2 leaderboard")
	return added
end

local function GetFakePlayersList()
	local list = {}
	for name, data in pairs(FakePlayersInLeaderboard) do
		table.insert(list, {
			name = name,
			userId = data.userId,
			player = data.player,
			created = data.created
		})
	end
	return list
end

-- ===== ФУНКЦИЯ СЛЕДОВАНИЯ ДЛЯ БОТОВ =====
local function StartFakePlayerFollow(fakeData)
	if not fakeData or not fakeData.character then return end
	local character = fakeData.character
	local humanoid = character:FindFirstChild("Humanoid")
	if not humanoid then return end

	humanoid.WalkSpeed = 16
	local leashRadius = 20
	local wanderRadius = 10

	local animator = humanoid:FindFirstChild("Animator")
	if not animator then
		animator = Instance.new("Animator")
		animator.Parent = humanoid
	end

	local walkTrack, idleTrack
	local function loadAnimations()
		if walkTrack and idleTrack then return true end
		local success = pcall(function()
			if not walkTrack then
				local walkAnim = Instance.new("Animation")
				walkAnim.AnimationId = "rbxassetid://507777826"
				walkTrack = animator:LoadAnimation(walkAnim)
				walkTrack.Looped = true
				walkTrack.Priority = Enum.AnimationPriority.Movement
			end
			if not idleTrack then
				local idleAnim = Instance.new("Animation")
				idleAnim.AnimationId = "rbxassetid://507766666"
				idleTrack = animator:LoadAnimation(idleAnim)
				idleTrack.Looped = true
				idleTrack.Priority = Enum.AnimationPriority.Idle
			end
		end)
		return success and walkTrack and idleTrack
	end

	local maxAttempts = 10
	local attempt = 0
	while attempt < maxAttempts do
		if loadAnimations() then
			if idleTrack and not idleTrack.IsPlaying then
				idleTrack:Play(0.2)
			end
			break
		end
		attempt = attempt + 1
		task.wait(0.1)
	end

	local isWalking = false
	local state = 'idle'
	local wanderTarget = nil
	local wanderTimer = 0
	local idleTimer = 0
	local dt = 0.05
	local lastTargetPos = nil
	local lastMoveTime = 0
	local moveInterval = 0.15
	local firstIdle = true

	local function setWalking(walking)
		if walking == isWalking then return end
		isWalking = walking
		if walkTrack and idleTrack then
			if walking then
				if idleTrack.IsPlaying then idleTrack:Stop(0.2) end
				if not walkTrack.IsPlaying then walkTrack:Play(0.2) end
			else
				if walkTrack.IsPlaying then walkTrack:Stop(0.2) end
				if not idleTrack.IsPlaying then idleTrack:Play(0.2) end
			end
		end
	end

	local function pickNewWanderTarget(centerPos)
		local angle = math.random() * math.pi * 2
		local dist = math.random(4, wanderRadius)
		return centerPos + Vector3.new(math.cos(angle) * dist, 0, math.sin(angle) * dist)
	end

	local followConnection
	followConnection = RunService.Heartbeat:Connect(function()
		if not fakeData.active then
			followConnection:Disconnect()
			return
		end
		local localChar = Players.LocalPlayer.Character
		if not localChar then return end
		local localRoot = localChar:FindFirstChild("HumanoidRootPart")
		if not localRoot then return end

		local myRoot = character:FindFirstChild("HumanoidRootPart")
		if not myRoot then return end

		local myPos = myRoot.Position
		local playerPos = localRoot.Position
		local distToPlayer = (playerPos - myPos).Magnitude

		if distToPlayer > leashRadius then
			state = 'returning'
			wanderTarget = playerPos + Vector3.new(math.random(-8, 8), 0, math.random(-8, 8))
			if distToPlayer > leashRadius * 2.5 then
				local snapPos = playerPos + Vector3.new(math.random(-6, 6), 0, math.random(-6, 6))
				myRoot.CFrame = CFrame.new(snapPos)
				state = 'idle'
				idleTimer = 0
				firstIdle = true
				setWalking(false)
			end
		end

		if state == 'idle' then
			setWalking(false)
			idleTimer = idleTimer + dt
			if firstIdle then
				if idleTimer >= 10 then
					firstIdle = false
					idleTimer = 0
					state = 'wandering'
					wanderTarget = pickNewWanderTarget(myPos)
					wanderTimer = 0
					if (wanderTarget - playerPos).Magnitude > leashRadius then
						local dirToPlayer = (playerPos - myPos).Unit
						wanderTarget = playerPos - dirToPlayer * math.random(5, 12)
					end
				end
			else
				if idleTimer >= 10 then
					idleTimer = 0
					state = 'wandering'
					wanderTarget = pickNewWanderTarget(myPos)
					wanderTimer = 0
					if (wanderTarget - playerPos).Magnitude > leashRadius then
						local dirToPlayer = (playerPos - myPos).Unit
						wanderTarget = playerPos - dirToPlayer * math.random(5, 12)
					end
				end
			end
		elseif state == 'wandering' or state == 'returning' then
			if wanderTarget then
				local distToTarget = (wanderTarget - myPos).Magnitude
				if distToTarget > 2 then
					setWalking(true)
					local now = tick()
					if not lastTargetPos or (wanderTarget - lastTargetPos).Magnitude > 1 or (now - lastMoveTime) > moveInterval then
						humanoid:MoveTo(wanderTarget)
						lastTargetPos = wanderTarget
						lastMoveTime = now
					end
				else
					state = 'idle'
					idleTimer = 0
					firstIdle = true
					setWalking(false)
				end
			end
			wanderTimer = wanderTimer + dt
			if wanderTimer >= 10 and state == 'wandering' then
				state = 'idle'
				idleTimer = 0
				firstIdle = true
				setWalking(false)
			end
		end

		if not isWalking and state == 'idle' then
			setWalking(false)
		end
	end)

	fakeData.followConnection = followConnection
	fakeData.walkTrack = walkTrack
	fakeData.idleTrack = idleTrack
end

-- ===== СПАВН БОТА В МИРЕ =====
function SpawnFakePlayer(username)
	if not username or username == "" then
		warn("[SpawnFakePlayer] No username provided")
		return
	end

	local success, userId = pcall(function()
		return Players:GetUserIdFromNameAsync(username)
	end)

	if not success or not userId then
		warn("[SpawnFakePlayer] Could not resolve userId for " .. username)
		userId = math.random(1000000, 9999999)
	end

	local desc
	success, desc = pcall(function()
		return Players:GetHumanoidDescriptionFromUserId(userId)
	end)
	
	if not success or not desc then
		warn("[SpawnFakePlayer] Failed to get description for " .. username)
		desc = Instance.new("HumanoidDescription")
	end

	local folder = Instance.new("Folder")
	folder.Name = "FakePlayer_" .. username
	folder.Parent = workspace

	local character = Players:CreateHumanoidModelFromDescription(desc, Enum.HumanoidRigType.R15)
	if not character then
		warn("[SpawnFakePlayer] Failed to create character for " .. username)
		folder:Destroy()
		return
	end

	local localChar = Players.LocalPlayer.Character
	local rootPart = localChar and localChar:FindFirstChild("HumanoidRootPart")
	if rootPart then
		local offset = Vector3.new(math.random(-8, 8), 0, math.random(-8, 8))
		character:SetPrimaryPartCFrame(rootPart.CFrame * CFrame.new(offset))
	else
		character:SetPrimaryPartCFrame(CFrame.new(0, 5, 0))
	end

	character.Name = username
	character.Parent = folder

	local humanoid = character:FindFirstChild("Humanoid")
	if humanoid then
		humanoid.DisplayDistanceType = Enum.HumanoidDisplayDistanceType.Viewer
		humanoid.NameDisplayDistance = 100
		humanoid.HealthDisplayType = Enum.HumanoidHealthDisplayType.AlwaysOff
		humanoid.HealthDisplayDistance = 0
		humanoid.WalkSpeed = 16
		
		task.spawn(function()
			local animator = humanoid:FindFirstChild("Animator")
			if not animator then
				animator = Instance.new("Animator")
				animator.Parent = humanoid
			end
			local anim = Instance.new("Animation")
			anim.AnimationId = "rbxassetid://507766388"
			local track = animator:LoadAnimation(anim)
			if track then track:Play() end
		end)
	end

	for _, part in ipairs(character:GetDescendants()) do
		if part:IsA("BasePart") then
			part.CanCollide = false
			part.Massless = true
		end
	end

	local rootPart2 = character:FindFirstChild("HumanoidRootPart")
	if rootPart2 then
		local detector = Instance.new("ClickDetector")
		detector.Parent = rootPart2
		detector.MaxActivationDistance = 30
		detector.MouseClick:Connect(function(player)
			if player == Players.LocalPlayer then
				TradeTable.Player2.Player = username
				PartnerUserBox.Text = username
				pcall(function() TheirOffer.Username.Text = "(" .. username .. ")" end)
				print("[SpawnFakePlayer] " .. username .. " set as partner via click")
			end
		end)
	end

	local fakeData = {
		folder = folder,
		character = character,
		username = username,
		userId = userId,
		active = true,
		followConnection = nil,
		walkTrack = nil,
		idleTrack = nil,
	}
	table.insert(SpawnedFakePlayers, fakeData)

	StartFakePlayerFollow(fakeData)

	FakePlayerIds[userId] = true
	_G.fakePlayerIds = FakePlayerIds

	print("[SpawnFakePlayer] Spawned " .. username .. " (ID: " .. userId .. ")")
end

function DeleteAllFakePlayers()
	for _, fakeData in ipairs(SpawnedFakePlayers) do
		fakeData.active = false
		if fakeData.followConnection then
			fakeData.followConnection:Disconnect()
		end
		if fakeData.walkTrack then
			fakeData.walkTrack:Stop()
		end
		if fakeData.idleTrack then
			fakeData.idleTrack:Stop()
		end
		if fakeData.folder and fakeData.folder.Parent then
			fakeData.folder:Destroy()
		end
	end
	SpawnedFakePlayers = {}
	if not FakePlayerIds then FakePlayerIds = {} end
	print("[SpawnFakePlayer] All fake players deleted")
end

local function FormatValue(v)
	if v == nil then return "?" end
	if type(v) == "number" then
		local s = tostring(math.floor(v))
		local k
		repeat s, k = string.gsub(s, "^(-?%d+)(%d%d%d)", "%1,%2") until k == 0
		return s
	end
	return tostring(v)
end

setthreadidentity(2)
local ProfileData = require(game.ReplicatedStorage.Modules.ProfileData)
local InventoryModule = require(game.ReplicatedStorage.Modules.InventoryModule)
local ItemModule = require(game.ReplicatedStorage.Modules.ItemModule)
local Sync = require(game.ReplicatedStorage.Database.Sync)
local ItemPopupService = require(game.ReplicatedStorage.ClientServices.ItemPopupService)
setthreadidentity(8)

local TradeRemotes = game.ReplicatedStorage.Trade
local TradeGUI = game.Players.LocalPlayer.PlayerGui.TradeGUI
local TheirOffer = TradeGUI.Container.Trade.TheirOffer
local YourOffer = TradeGUI.Container.Trade.YourOffer

local SearchTextSignal
local TradeInventory
local functions = {}
local Config = { item = "", in_trade = false, player2 = nil }

-- ===== UNBOX ПОЛЛЫ =====
local godlyPool = {
    "TravelerGun","TreeGun2023","Constellation","TreeKnife2023","Turkey2023",
    "UFOKnife","VampireGun","Darkshot","Darksword","Raygun","SunsetGun",
    "Snowcannon","Bauble","SunsetKnife","HeartWand","WraithGun","WraithKnife",
    "Flora","Bloom","Rainbow_G","Rainbow_K","SnowDagger","FlowerwoodGun",
    "FlowerwoodKnife","XenoKnife","XenoGun","Watergun","Ocean_G","Waves_K",
    "Treat","Sweet","Blizzard",
}
local ancientPool = {
    "Gingerscope","TravelerAxe","VampireAxe","Celestial","Harvester","Icepiercer",
}
local chromaPool = {
    "TravelerGunChroma","TreeGun2023Chroma","TreeKnife2023Chroma",
    "BaubleChroma","ConstellationChroma","VampireGunChroma","UFOKnifeChroma",
    "RaygunChroma","SunsetGunChroma","SnowcannonChroma","BlizzardChroma",
    "SunsetKnifeChroma","SnowDaggerChroma","TreatChroma","HeartWandChroma",
    "SnowstormChroma","WatergunChroma","SweetChroma","BaubleKnifeChroma",
}

local WeaponCatalog = {}
local WeaponByKey = {}
local WeaponByName = {}
local RareWeaponKeys = {}
local RareRarities = { Godly = true, Ancient = true, Unique = true, Chroma = true, Legendary = true, Classic = true }
do
	local source = Sync.Weapons or Sync.Item
	for key, data in pairs(source) do
		if type(data) == "table" and (data.ItemType == "Knife" or data.ItemType == "Gun") then
			local rarity = data.Rarity or "Common"
			local isChroma = data.Chroma == true
			local effectiveRarity = isChroma and "Chroma" or rarity
			local entry = { key = key, name = data.ItemName or key, rarity = effectiveRarity, type = data.ItemType, chroma = isChroma }
			table.insert(WeaponCatalog, entry)
			WeaponByKey[key] = entry
			WeaponByName[string.lower(entry.name)] = entry
			if RareRarities[effectiveRarity] then table.insert(RareWeaponKeys, key) end
		end
	end
	local rarityOrder = { Chroma = 1, Godly = 2, Ancient = 3, Unique = 4, Legendary = 5, Classic = 6, Vintage = 7, Rare = 8, Uncommon = 9, Common = 10 }
	table.sort(WeaponCatalog, function(a, b)
		local ra = rarityOrder[a.rarity] or 99
		local rb = rarityOrder[b.rarity] or 99
		if ra ~= rb then return ra < rb end
		if a.type ~= b.type then return a.type < b.type end
		return a.name < b.name
	end)
end

local function resolveDBItem(id)
    if not Sync or not Sync.Weapons then return id, nil end
    local function isValid(data)
        if type(data) ~= "table" then return false end
        local r = data.Rarity or ""
        if r == "Unique" or r == "Common" or r == "Uncommon" or r == "Rare" then return false end
        return true
    end
    if Sync.Weapons[id] and isValid(Sync.Weapons[id]) then return id, Sync.Weapons[id] end
    local noSpace = id:gsub("%s+", "")
    if Sync.Weapons[noSpace] and isValid(Sync.Weapons[noSpace]) then return noSpace, Sync.Weapons[noSpace] end
    local stripped = id:gsub("[^%w]", "")
    if Sync.Weapons[stripped] and isValid(Sync.Weapons[stripped]) then return stripped, Sync.Weapons[stripped] end
    local strippedLower = stripped:lower()
    local bestID, bestData, bestScore = nil, nil, 0
    local rarityScore = {Ancient=5, Chroma=4, Godly=3, Legendary=2}
    for dbID, dbData in pairs(Sync.Weapons) do
        if isValid(dbData) then
            local idMatch = dbID:gsub("[^%w]",""):lower() == strippedLower
            local nameMatch = (dbData.ItemName or dbData.Name or ""):gsub("[^%w]",""):lower() == strippedLower
            if idMatch or nameMatch then
                local score = rarityScore[dbData.Rarity or ""] or 1
                if score > bestScore then
                    bestScore = score; bestID = dbID; bestData = dbData
                end
            end
        end
    end
    if bestID then return bestID, bestData end
    return id, nil
end

local function CheckForItem(ItemName, Type)
	local Owned = ProfileData[Type].Owned
	for Index, Value in pairs(Owned) do
		if Index == ItemName then return true, Value end
		if Value == ItemName then return true, 1 end
	end
	return false
end

local v18 = {}
local function v22(v19)
	for _, v21 in pairs(v19:GetChildren()) do
		if v21:IsA("Frame") then
			v21.Visible = false
			if v18[v21] then v18[v21]:Disconnect() v18[v21] = nil end
		end
	end
end

local TradeTable = {
	LastOffer = os.time(), Locked = false,
	Player1 = { Player = game.Players.LocalPlayer, Accepted = false, Offer = {} },
	Player2 = { Player = "m1cek3", Accepted = false, Offer = {} },
}

local FakeRequestActive = false

local function ShowFakeTradeRequest(senderName)
    local TradeModule = require(game.ReplicatedStorage.Modules.TradeModule)
    local requestFrame = TradeModule.GUI.RequestFrame
    if not requestFrame then return end

    FakeRequestActive = true

    local receivingFrame = requestFrame:FindFirstChild("ReceivingRequest")
    if not receivingFrame then
        FakeRequestActive = false
        return
    end

    receivingFrame.Username.Text = tostring(senderName)
    receivingFrame.Visible = true
    requestFrame.Visible = true

    local acceptBtn = receivingFrame:FindFirstChild("Accept")
    local declineBtn = receivingFrame:FindFirstChild("Decline")

    if acceptBtn then
        acceptBtn.MouseButton1Click:Connect(function()
            if not FakeRequestActive then return end
            FakeRequestActive = false
            requestFrame.Visible = false
            local partnerName = tostring(senderName)
            TradeTable.Player2.Player = partnerName
            if PartnerUserBox then PartnerUserBox.Text = partnerName end
            pcall(function() TheirOffer.Username.Text = "(" .. partnerName .. ")" end)
            Config.in_trade = false
            local ok, err = pcall(function() StartTrade() end)
            if not ok then warn("[FakeRequest] StartTrade error:", err) end
        end)
    end

    if declineBtn then
        declineBtn.MouseButton1Click:Connect(function()
            if not FakeRequestActive then return end
            FakeRequestActive = false
            requestFrame.Visible = false
        end)
    end
end

local function SpawnItem(ItemName, Amount, ItemType)
	Amount = Amount or 1
	ItemType = ItemType or "Weapons"
	pcall(function()
		if ProfileData[ItemType].Owned[ItemName] == nil then
			ProfileData[ItemType].Owned[ItemName] = Amount
		else
			ProfileData[ItemType].Owned[ItemName] = ProfileData[ItemType].Owned[ItemName] + Amount
		end
		game.ReplicatedStorage.Remotes.Inventory.InventoryDataChanged:Fire()
	end)
end

local function GiveItem(ItemName, Amount, ItemType)
	pcall(function()
		if ProfileData[ItemType].Owned[ItemName] == nil then
			ProfileData[ItemType].Owned[ItemName] = Amount
		else
			ProfileData[ItemType].Owned[ItemName] = ProfileData[ItemType].Owned[ItemName] + Amount
		end
		game.ReplicatedStorage.Remotes.Inventory.InventoryDataChanged:Fire()
	end)
end

local function RemoveItem(ItemName, Amount, ItemType)
	pcall(function()
		local owned = ProfileData[ItemType].Owned[ItemName]
		if not owned then return end
		if owned - Amount > 0 then
			ProfileData[ItemType].Owned[ItemName] = owned - Amount
		else
			ProfileData[ItemType].Owned[ItemName] = nil
		end
		game.ReplicatedStorage.Remotes.Inventory.InventoryDataChanged:Fire()
	end)
end

-- ============================================================
--  UNBOX SPIN WHEEL
-- ============================================================

openSpinWheel = function()
    local BoxModuleScript = ReplicatedStorage:FindFirstChild("Modules") and ReplicatedStorage.Modules:FindFirstChild("BoxModule")
    if not BoxModuleScript then
        warn("[Spinner] BoxModule not found")
        return
    end
    local Unboxing2Source = BoxModuleScript:FindFirstChild("Unboxing2")
    local NewItemSource = BoxModuleScript:FindFirstChild("NewItem")
    local GridSource = BoxModuleScript:FindFirstChild("UIGridLayout")
    if not Unboxing2Source or not NewItemSource then
        warn("[Spinner] Unboxing2 or NewItem not found")
        return
    end
    local rng = Random.new()
    local weaponPool = {}
    if Sync and Sync.Weapons then
        for id, data in pairs(Sync.Weapons) do
            if type(data) == "table" then
                weaponPool[id] = data
                local nameLower = (data.ItemName or data.Name or ""):lower():gsub("[^a-z0-9]","")
                weaponPool[nameLower] = data
            end
        end
    end
    local function pickRandomWeaponID()
        local roll = rng:NextInteger(1, 100)
        local pool
        if roll <= 10 and #chromaPool > 0 then
            pool = chromaPool
        elseif roll <= 35 and #ancientPool > 0 then
            pool = ancientPool
        else
            pool = #godlyPool > 0 and godlyPool or ancientPool
        end
        return pool[rng:NextInteger(1, #pool)]
    end
    local winID = pickRandomWeaponID()
    local resolvedWinID, winData = resolveDBItem(winID)
    local unboxGUI = Unboxing2Source:Clone()
    local guiContainer = unboxGUI:FindFirstChild("Container")
    local guiMain = guiContainer and guiContainer:FindFirstChild("Main")
    local innerContainer = guiMain and guiMain:FindFirstChild("Container")
    local background = innerContainer and innerContainer:FindFirstChild("Background")
    local itemContainer = background and background:FindFirstChild("ItemContainer")
    local offsetContainer = itemContainer and itemContainer:FindFirstChild("OffsetContainer")
    local mainContainer = offsetContainer and offsetContainer:FindFirstChild("MainContainer")
    if not mainContainer or not offsetContainer then
        warn("[Spinner] Could not navigate Unboxing2 structure")
        unboxGUI:Destroy()
        return
    end
    local stripCount = rng:NextInteger(20, 25)
    local winIndex = stripCount
    mainContainer:ClearAllChildren()
    local gridLayout
    if GridSource then
        gridLayout = GridSource:Clone()
    else
        gridLayout = Instance.new("UIGridLayout")
        gridLayout.CellSize = UDim2.new(0, 100, 1, 0)
        gridLayout.CellPadding = UDim2.new(0, 0, 0, 0)
        gridLayout.SortOrder = Enum.SortOrder.LayoutOrder
        gridLayout.FillDirection = Enum.FillDirection.Horizontal
    end
    gridLayout.Parent = mainContainer
    local cellWidth = gridLayout.CellSize.X.Offset
    offsetContainer.Size = UDim2.new(0, cellWidth * (stripCount + 5), 1, 0)
    offsetContainer.Position = UDim2.new(0.5, cellWidth / 2, 0, 0)
    for i = 1, stripCount + 5 do
        local itemData
        if i == winIndex then
            itemData = winData
        else
            local randID = pickRandomWeaponID()
            local _, randData = resolveDBItem(randID)
            itemData = randData
        end
        local cell = NewItemSource:Clone()
        cell.LayoutOrder = i
        cell.Visible = true
        cell.Parent = mainContainer
        if itemData and ItemModule and ItemModule.DisplayItem then
            pcall(function() ItemModule.DisplayItem(cell, itemData) end)
        end
    end
    pcall(function()
        local preItem1 = offsetContainer:FindFirstChild("PreItem1")
        if preItem1 then
            local randID = pickRandomWeaponID()
            local data = weaponPool[randID] or (Sync and Sync.Weapons and Sync.Weapons[randID])
            if data then ItemModule.DisplayItem(preItem1, data) end
        end
    end)
    pcall(function()
        local preItem2 = offsetContainer:FindFirstChild("PreItem2")
        if preItem2 then
            local randID = pickRandomWeaponID()
            local data = weaponPool[randID] or (Sync and Sync.Weapons and Sync.Weapons[randID])
            if data then ItemModule.DisplayItem(preItem2, data) end
        end
    end)
    unboxGUI.DisplayOrder = 1100
    unboxGUI.Parent = game.Players.LocalPlayer.PlayerGui
    local spinDuration = 3 + rng:NextNumber()
    local targetX = -(cellWidth * winIndex) + rng:NextInteger(-(cellWidth / 2 - 5), cellWidth / 2 - 5) + cellWidth - cellWidth / 2
    TweenService:Create(offsetContainer, TweenInfo.new(spinDuration, Enum.EasingStyle.Cubic, Enum.EasingDirection.Out), { Position = UDim2.new(0.5, targetX, 0, 0) }):Play()
    task.wait(spinDuration + 1)
    local itemName = winData and (winData.ItemName or winData.Name) or resolvedWinID
    pcall(function() unboxGUI:Destroy() end)
    
    -- ОРИГИНАЛЬНЫЙ CLAIM ЧЕРЕЗ ITEMPOPUPSERVICE.
    -- TradeGUI НЕ меняем и НЕ опускаем вниз.
    -- Оригинальный Claim просто появляется поверх трейда.
    local claimDone = false

    pcall(function()
        local claimConn
        claimConn = ItemPopupService.ItemClaimsComplete.Event:Connect(function()
            claimDone = true

            -- Только после нажатия ОРИГИНАЛЬНОЙ кнопки Claim
            -- добавляем выигранный предмет в текущий трейд.
            if Config.in_trade and TradeTable and not TradeTable.Locked and TradeTable.Player1 and TradeTable.Player1.Offer then
                local AlreadyOffered = 0
                for _, Item in pairs(TradeTable.Player1.Offer) do
                    if Item[1] == resolvedWinID and Item[3] == "Weapons" then
                        AlreadyOffered = Item[2] or 0
                        break
                    end
                end

                if AlreadyOffered == 0 then
                    if #TradeTable.Player1.Offer < 4 then
                        table.insert(TradeTable.Player1.Offer, { resolvedWinID, 1, "Weapons" })
                    end
                else
                    for Index, Item in pairs(TradeTable.Player1.Offer) do
                        if Item[1] == resolvedWinID and Item[3] == "Weapons" then
                            TradeTable.Player1.Offer[Index][2] = AlreadyOffered + 1
                            break
                        end
                    end
                end

                TradeTable.LastOffer = os.time()
                TradeTable.Player1.Accepted = false
                TradeTable.Player2.Accepted = false
                pcall(function() functions.UpdateTrade() end)
            end

            pcall(function() claimConn:Disconnect() end)
        end)

        -- Поднимаем ТОЛЬКО оригинальный слой Claim выше трейда.
        -- Сам TradeGUI не трогаем и его DisplayOrder не меняем.
        local crossPlatform = game.Players.LocalPlayer.PlayerGui:FindFirstChild("CrossPlatform")
        local oldCrossPlatformDisplayOrder = nil
        if crossPlatform and crossPlatform:IsA("ScreenGui") then
            oldCrossPlatformDisplayOrder = crossPlatform.DisplayOrder
            crossPlatform.DisplayOrder = 1200
        end

        -- Запускаем именно оригинальное окно Claim из ItemPopupService.
        ItemPopupService.ItemReceived:Fire(resolvedWinID, "Weapons", 1)

        local elapsed = 0
        while not claimDone and elapsed < 30 do
            task.wait(0.1)
            elapsed = elapsed + 0.1
        end

        pcall(function() claimConn:Disconnect() end)
        pcall(function()
            if crossPlatform and oldCrossPlatformDisplayOrder ~= nil then
                crossPlatform.DisplayOrder = oldCrossPlatformDisplayOrder
            end
        end)
    end)

    -- Если Claim не был нажат, ничего в трейд не добавляем.
end

-- ============================================================
--  ACCEPT TRADE
-- ============================================================

local function AcceptTrade()
	if not TradeTable then return end
	if TradeTable.Player1.Accepted and TradeTable.Player2.Accepted then
		TradeTable.Locked = true
		task.wait(0.2)
		
		if TradeTable.Player2.Offer then
			for _, item in pairs(TradeTable.Player2.Offer) do
				local itemID = item[1] or item.ItemID
				local amount = item[2] or item.Amount or 1
				local itemType = item[3] or item.ItemType or "Weapons"
				
				pcall(function() GiveItem(itemID, amount, itemType) end)
				
				pcall(function()
					if ItemPopupService then
						if ItemPopupService.AddNewItem then
							ItemPopupService:AddNewItem(itemID, itemType, amount)
						elseif ItemPopupService.ItemReceived then
							ItemPopupService.ItemReceived:Fire(itemID, itemType)
						end
					end
				end)
				
				pcall(function() _G.NewItem(itemID, "You Got...", nil, itemType, amount) end)
			end
		end
		
		if TradeTable.Player1.Offer then
			for _, item in pairs(TradeTable.Player1.Offer) do
					pcall(function() RemoveItem(item[1], item[2], item[3]) end)
			end
		end
		
		pcall(function() TradeGUI.Enabled = false end)
		local partner = TradeTable.Player2.Player
		if partner and partner ~= "" and partner ~= "m1cek3" then
			LastTradePartner = partner
			print("[mm2run] LastTradePartner saved: " .. partner)
		end
		TradeTable = {
			LastOffer = os.time(), Locked = false,
			Player1 = { Player = game.Players.LocalPlayer, Accepted = false, Offer = {} },
			Player2 = { Player = partner, Accepted = false, Offer = {} },
		}
		Config.in_trade = false
	end
end

local v84 = false

local function OfferItemLocalPlayer(ItemName, ItemType)
	if not TradeTable or TradeTable.Locked then return end
	local AlreadyOffered = 0
	for _, Item in pairs(TradeTable.Player1.Offer) do
		if Item[1] == ItemName and Item[3] == ItemType then AlreadyOffered = Item[2] end
	end
	local HasItem, Amount = CheckForItem(ItemName, ItemType)
	if HasItem and Amount - AlreadyOffered > 0 then
		if AlreadyOffered == 0 then
			if #TradeTable.Player1.Offer < 4 then
				table.insert(TradeTable.Player1.Offer, { ItemName, 1, ItemType })
			end
		else
			for Index, Item in pairs(TradeTable.Player1.Offer) do
				if Item[1] == ItemName then
					TradeTable.Player1.Offer[Index][2] = TradeTable.Player1.Offer[Index][2] + 1
					break
				end
			end
		end
	end
	TradeTable.LastOffer = os.time()
	TradeTable.Player1.Accepted = false
	TradeTable.Player2.Accepted = false
	pcall(function() functions.UpdateTrade() end)
end

local function RemoveItemLocalPlayer(ItemName, ItemType)
	if not TradeTable or TradeTable.Locked or TradeTable.Player1.Accepted then return end
	TradeTable.LastOffer = os.time()
	TradeTable.Player1.Accepted = false
	TradeTable.Player2.Accepted = false
	for Index, Item in pairs(TradeTable.Player1.Offer) do
		if Item[1] == ItemName and Item[3] == ItemType then
			TradeTable.Player1.Offer[Index][2] = TradeTable.Player1.Offer[Index][2] - 1
			if TradeTable.Player1.Offer[Index][2] <= 0 then
				table.remove(TradeTable.Player1.Offer, Index)
			end
			break
		end
	end
	pcall(function() functions.UpdateTrade() end)
end

local function OfferItemAnotherPlayer(ItemName, ItemType)
	if not ItemName or ItemName == "" or not TradeTable or TradeTable.Locked then return false end
	if #TradeTable.Player2.Offer >= 4 then
		local foundExisting = false
		for _, Item in pairs(TradeTable.Player2.Offer) do
			if Item[1] == ItemName and Item[3] == ItemType then foundExisting = true break end
		end
		if not foundExisting then return false end
	end
	local AlreadyOffered = 0
	for _, Item in pairs(TradeTable.Player2.Offer) do
		if Item[1] == ItemName and Item[3] == ItemType then AlreadyOffered = Item[2] end
	end
	if AlreadyOffered == 0 then
		table.insert(TradeTable.Player2.Offer, { ItemName, 1, ItemType })
	else
		for Index, Item in pairs(TradeTable.Player2.Offer) do
			if Item[1] == ItemName and Item[3] == ItemType then
				TradeTable.Player2.Offer[Index][2] = TradeTable.Player2.Offer[Index][2] + 1
				break
			end
		end
	end
	TradeTable.LastOffer = os.time()
	TradeTable.Player1.Accepted = false
	TradeTable.Player2.Accepted = false
	pcall(function() functions.UpdateTrade() end)
	return true
end

local function RemoveItemAnotherPlayer()
	if not TradeTable or not TradeTable.Player2 or not TradeTable.Player2.Offer then return end
	if #TradeTable.Player2.Offer > 0 and not TradeTable.Player2.Accepted then
		local LastIndex = #TradeTable.Player2.Offer
		TradeTable.Player2.Offer[LastIndex][2] = TradeTable.Player2.Offer[LastIndex][2] - 1
		if TradeTable.Player2.Offer[LastIndex][2] <= 0 then
			table.remove(TradeTable.Player2.Offer, LastIndex)
		end
		TradeTable.LastOffer = os.time()
		TradeTable.Player1.Accepted = false
		TradeTable.Player2.Accepted = false
		pcall(function() functions.UpdateTrade() end)
	end
end

local function v34(v23, v24)
	for v25, v26 in v24 do
		local ItemID = v26[1] or v26.ItemID
		local Amount = v26[2] or v26.Amount
		local ItemType = v26[3] or v26.ItemType

		local v33 = v23.Container["NewItem" .. v25]
		if not v33 then continue end

		local success = pcall(function()
			if Sync[ItemType] and Sync[ItemType][ItemID] then
				local v30 = {}
				for v31, v32 in pairs(Sync[ItemType][ItemID]) do
					v30[v31] = v32
				end
				v30.DataType = ItemType
				v30.Amount = Amount
				ItemModule.DisplayItem(v33, v30)
			end
		end)

		pcall(function()
			if v18[v33] then
				v18[v33]:Disconnect()
			end
			if v33.Container and v33.Container:FindFirstChild("ActionButton") then
				v18[v33] = v33.Container.ActionButton.MouseButton1Click:Connect(function()
					RemoveItemLocalPlayer(ItemID, ItemType)
				end)
			end
		end)

		v33.Visible = true
	end
end

local v85 = 6
local function ResetCooldown(arg1)
	if arg1 then
		TradeGUI.Container.Trade.Actions.Accept.Cooldown.Visible = false
		v85 = 0
		v84 = false
	else
		TradeGUI.Container.Trade.Actions.Accept.Cooldown.Visible = true
		v85 = 6
		TradeGUI.Container.Trade.Actions.Accept.Cooldown.Title.Text = " Please wait (" .. v85 .. ") before accepting."
		if not v84 then
			v84 = true
			repeat
				task.wait(1)
				v85 = v85 - 1
				TradeGUI.Container.Trade.Actions.Accept.Cooldown.Title.Text = " Please wait (" .. v85 .. ") before accepting."
			until v85 <= 0
			v84 = false
			TradeGUI.Container.Trade.Actions.Accept.Cooldown.Visible = false
		else
			v85 = 6
		end
	end
end

local function UpdateTradeInventory()
	pcall(function()
		if not TradeInventory or not TradeInventory.Data then return end
		local l_Offer_2 = TradeTable.Player1.Offer
		for v63, v64 in pairs(TradeInventory.Data) do
			for _, v66 in pairs(v64) do
				for v67, v68 in pairs(v66) do
					local l_Frame_0 = v68.Frame
					local l_Amount_0 = v68.Amount
					for _, v72 in pairs(l_Offer_2) do
						local v73 = v72[1] or v72.ItemID
						local v74 = v72[2] or v72.Amount
						local v75 = v72[3] or v72.ItemType
						if v73 == v67 and v75 == v63 then
							l_Amount_0 = l_Amount_0 - v74
						end
					end
					if l_Amount_0 == 1 then
						l_Frame_0.Container.Amount.Text = ""
						l_Frame_0.Visible = true
					elseif l_Amount_0 > 1 then
						l_Frame_0.Container.Amount.Text = "x" .. l_Amount_0
						l_Frame_0.Visible = true
					elseif l_Amount_0 < 1 then
						l_Frame_0.Visible = false
					end
				end
			end
		end
	end)
end

local v35 = "Accept"
functions.UpdateTrade = function()
	pcall(function()
		local Offer1 = TradeTable.Player1.Offer
		local Offer2 = TradeTable.Player2.Offer
		v22(YourOffer.Container)
		v22(TheirOffer.Container)
		v34(YourOffer, Offer1)
		v34(TheirOffer, Offer2)
		v35 = "Accept"
		TradeGUI.Container.Trade.Actions.Accept.Confirm.Visible = false
		TradeGUI.Container.Trade.Actions.Accept.Cancel.Visible = false
		YourOffer.Accepted.Visible = false
		TheirOffer.Accepted.Visible = false
		local l_AddItem_0 = TradeGUI.Container.Trade.Actions.Accept.AddItem
		local v44 = (#Offer1 < 1 and #Offer2 < 1)
		l_AddItem_0.Visible = v44
		UpdateTradeInventory()
		ResetCooldown(v44)
	end)
end

function DeclineTrade()
	pcall(function() TradeGUI.Enabled = false end)
	local partner = TradeTable.Player2.Player
	TradeTable = {
		LastOffer = os.time(), Locked = false,
		Player1 = { Player = game.Players.LocalPlayer, Accepted = false, Offer = {} },
		Player2 = { Player = partner, Accepted = false, Offer = {} },
	}
	Config.in_trade = false
	pcall(function() UnConnections() end)
end

local v87 = time()
local Connections = {}

function SetupConnections(v76)
	pcall(function()
		if v76 and v76.Data then
			for v77, v78 in pairs(v76.Data) do
				for _, v80 in pairs(v78) do
					for v81, v82 in pairs(v80) do
						local l_Frame_1 = v82.Frame
						if l_Frame_1 then
							Connections.Connection0 = l_Frame_1.Container.ActionButton.MouseButton1Click:Connect(function()
								OfferItemLocalPlayer(v81, v77)
							end)
						end
					end
				end
			end
		end
	end)
	pcall(function()
		Connections.Connection1 = TradeGUI.Container.Trade.Actions.Accept.ActionButton.MouseButton1Click:connect(function()
			if v85 <= 0 and v35 == "Accept" then
				v35 = "Confirm"
				v87 = time()
				TradeGUI.Container.Trade.Actions.Accept.Confirm.Visible = true
			end
		end)
	end)
	pcall(function()
		Connections.Connection2 = TradeGUI.Container.Trade.Actions.Accept.Confirm.ActionButton.MouseButton1Click:connect(function()
			if v85 <= 0 and time() - v87 >= 0.4 and v35 == "Confirm" then
				v35 = "Waiting"
				YourOffer.Accepted.Visible = true
				TradeGUI.Container.Trade.Actions.Accept.Cancel.Visible = true
				TradeTable.Player1.Accepted = true
				AcceptTrade()
			end
		end)
	end)
	pcall(function()
		Connections.Connection3 = TradeGUI.Container.Trade.Actions.Accept.Cancel.ActionButton.MouseButton1Click:connect(function()
			TradeTable.LastOffer = os.time()
			TradeTable.Player1.Accepted = false
			TradeTable.Player2.Accepted = false
			pcall(function() functions.UpdateTrade() end)
		end)
	end)
	pcall(function()
		Connections.Connection4 = TradeGUI.Container.Trade.Actions.Decline.ActionButton.MouseButton1Click:connect(function()
			DeclineTrade()
		end)
	end)
end

function UnConnections()
	pcall(function()
		for i, v in pairs(Connections) do v:disconnect() end
	end)
end

function StartTrade()
	if Config.in_trade then return end
	Config.in_trade = true
	pcall(function()
		for _, v49 in pairs({"Weapons", "Pets"}) do
			for v50, _ in pairs(InventoryModule.CreateBlankTradeInventoryTable()[v49]) do
				TradeGUI.Container.Items.Main:FindFirstChild(v49).Items.Container:FindFirstChild(v50).Container:ClearAllChildren()
			end
		end
	end)
	pcall(function() TradeInventory = InventoryModule.GenerateInventory(TradeGUI.Container.Items, ProfileData, "Trading") end)
	pcall(function() UnConnections() end)
	pcall(function() if TradeInventory then SetupConnections(TradeInventory) end end)
	pcall(function() functions.UpdateTrade(TradeTable) end)
	pcall(function() TheirOffer.Username.Text = "(" .. tostring(TradeTable.Player2.Player) .. ")" end)
	TradeGUI.Enabled = true
	pcall(function()
		if SearchTextSignal then SearchTextSignal:disconnect() end
		local SearchText = TradeGUI.Container.Items.Tabs.Search.Container.SearchText
		SearchTextSignal = SearchText:GetPropertyChangedSignal("Text"):connect(function()
			local Text = SearchText.Text
			Text = string.gsub(Text, "S", "")
			for _, v55 in pairs(TradeInventory.Data) do
				for _, v57 in pairs(v55.Current) do
					v57.Frame.Visible = string.find(string.lower(v57.Name), string.lower(Text))
					if v57.Frame.Parent.Parent:IsA("ScrollingFrame") then
						v57.Frame.Parent.Parent.CanvasPosition = Vector2.new(0, 0)
					else
						v57.Frame.Parent.Parent.Parent.CanvasPosition = Vector2.new(0, 0)
					end
				end
			end
		end)
	end)
end

local function partnerNameFromArgs(...)
	for _, a in ipairs({ ... }) do
		if typeof(a) == "Instance" and a:IsA("Player") then
			return a.Name
		end
		if type(a) == "number" then
			local p = game.Players:GetPlayerByUserId(a)
			if p then return p.Name end
		end
		if type(a) == "string" and a ~= "" and a ~= game.Players.LocalPlayer.Name then
			return a
		end
	end
end

TradeRemotes.StartTrade.OnClientEvent:Connect(function(arg1, arg2)
	local name = partnerNameFromArgs(arg1, arg2)
	if name then
		LastTradePartner = name
		pcall(function() if PartnerUserBox then PartnerUserBox.Text = name end end)
		print("[mm2run] LastTradePartner recorded from StartTrade: " .. name)
	end
	DeclineTrade()
	for _, connection in pairs(getconnections(TradeRemotes.StartTrade)) do
		if connection.Function then connection.Function(arg1, arg2) end
	end
end)

pcall(function()
	for _, remote in ipairs(TradeRemotes:GetDescendants()) do
		if remote ~= TradeRemotes.StartTrade and remote:IsA("RemoteEvent") then
			remote.OnClientEvent:Connect(function(...)
				local name = partnerNameFromArgs(...)
				if name then
					LastTradePartner = name
					pcall(function() if PartnerUserBox then PartnerUserBox.Text = name end end)
					print("[mm2run] LastTradePartner updated from " .. remote.Name .. ": " .. name)
				end
			end)
		end
	end
end)

-- ===== GUI СОЗДАНИЕ =====
do
local controlGui = Instance.new("ScreenGui")
controlGui.ResetOnSpawn = false
controlGui.DisplayOrder = 999999999
controlGui.Enabled = true
controlGui.Parent = game:GetService("CoreGui")

local mainFrame = Instance.new("Frame")
mainFrame.Size = UDim2.new(0, 230, 0, 480)
mainFrame.Position = UDim2.new(0, 10, 0.5, -240)
mainFrame.BackgroundColor3 = Color3.fromRGB(25, 20, 30)
mainFrame.BorderSizePixel = 0
mainFrame.ZIndex = 1
mainFrame.ClipsDescendants = true
mainFrame.Parent = controlGui

local mainCorner = Instance.new("UICorner")
mainCorner.CornerRadius = UDim.new(0, 8)
mainCorner.Parent = mainFrame

local mainStroke = Instance.new("UIStroke")
mainStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
mainStroke.Color = Color3.fromRGB(255, 100, 180)
mainStroke.Thickness = 2.5
mainStroke.Parent = mainFrame

local hue = 0
task.spawn(function()
	while mainStroke and mainStroke.Parent do
		local pink = Color3.fromRGB(255, 100 + math.sin(hue) * 50, 180 + math.sin(hue + 1) * 40)
		mainStroke.Color = pink
		hue = hue + 0.03
		task.wait(0.05)
	end
end)

local titleLabel = Instance.new("TextLabel")
titleLabel.Size = UDim2.new(1, 0, 0, 30)
titleLabel.Position = UDim2.new(0, 0, 0, 4)
titleLabel.BackgroundTransparency = 1
titleLabel.Text = "m1cek3 on discord"
titleLabel.Font = Enum.Font.FredokaOne
titleLabel.TextSize = 16
titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
titleLabel.TextXAlignment = Enum.TextXAlignment.Center
titleLabel.Parent = mainFrame

local titleStroke = Instance.new("UIStroke")
titleStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Contextual
titleStroke.Color = Color3.new(0, 0, 0)
titleStroke.Thickness = 1.5
titleStroke.Parent = titleLabel

local GUI_Locked = false

local lockButton = Instance.new("TextButton")
lockButton.Size = UDim2.new(0, 26, 0, 30)
lockButton.Position = UDim2.new(1, -8, 0, 4)
lockButton.AnchorPoint = Vector2.new(1, 0)
lockButton.BackgroundTransparency = 1
lockButton.Text = "🔓"
lockButton.Font = Enum.Font.SourceSansBold
lockButton.TextSize = 18
lockButton.TextColor3 = Color3.fromRGB(255, 255, 255)
lockButton.ZIndex = 20
lockButton.Parent = mainFrame

lockButton.MouseButton1Click:Connect(function()
	GUI_Locked = not GUI_Locked
	lockButton.Text = GUI_Locked and "🔒" or "🔓"
end)

local Drag = { mode = nil, startInput = nil, startPos = nil }

titleLabel.Active = true
titleLabel.InputBegan:Connect(function(input)
	if GUI_Locked then return end
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		Drag.mode = "move"
		Drag.startInput = input.Position
		Drag.startPos = mainFrame.Position
	end
end)

UserInputService.InputChanged:Connect(function(input)
	if not Drag.mode then return end
	if input.UserInputType ~= Enum.UserInputType.MouseMovement and input.UserInputType ~= Enum.UserInputType.Touch then return end
	local delta = input.Position - Drag.startInput
	if Drag.mode == "move" then
		mainFrame.Position = UDim2.new(Drag.startPos.X.Scale, Drag.startPos.X.Offset + delta.X, Drag.startPos.Y.Scale, Drag.startPos.Y.Offset + delta.Y)
	end
end)

UserInputService.InputEnded:Connect(function(input)
	if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
		Drag.mode = nil
	end
end)

local tabContainer = Instance.new("Frame")
tabContainer.Size = UDim2.new(0.95, 0, 0, 28)
tabContainer.Position = UDim2.new(0.025, 0, 0, 38)
tabContainer.BackgroundTransparency = 1
tabContainer.Parent = mainFrame

local tabs = {"Control", "Players", "Items", "Spawner", "Sets"}
local currentTab = "Control"
local tabFrames = {}
local tabButtons = {}
local activeTabPulseTween = nil

function setActiveTab(tabName)
	if currentTab == tabName then return end
	if activeTabPulseTween then activeTabPulseTween:Cancel() end
	currentTab = tabName
	for name, data in pairs(tabButtons) do
		local isActive = name == tabName
		TweenService:Create(data.button, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {
			BackgroundColor3 = isActive and Color3.fromRGB(60, 40, 60) or Color3.fromRGB(40, 30, 45)
		}):Play()
		local targetColor = isActive and Color3.fromRGB(255, 100, 200) or Color3.fromRGB(150, 80, 120)
		local targetThickness = isActive and 1.5 or 1.0
		TweenService:Create(data.stroke, TweenInfo.new(0.25, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), {
			Color = targetColor, Thickness = targetThickness
		}):Play()
		if isActive then
			local pulseInfo = TweenInfo.new(1.5, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)
			activeTabPulseTween = TweenService:Create(data.stroke, pulseInfo, {
				Color = targetColor:Lerp(Color3.fromRGB(255, 200, 255), 0.25), Thickness = 2.0
			})
			activeTabPulseTween:Play()
		end
	end
	for name, frame in pairs(tabFrames) do frame.Visible = name == tabName end
end

for i, tabName in ipairs(tabs) do
	local tabButton = Instance.new("TextButton")
	tabButton.Size = UDim2.new(1/#tabs - 0.015, 0, 1, 0)
	tabButton.Position = UDim2.new((i - 1) * (1/#tabs), (i == 1) and 0 or 0, 0, 0)
	tabButton.BackgroundColor3 = i == 1 and Color3.fromRGB(60, 40, 60) or Color3.fromRGB(40, 30, 45)
	tabButton.BackgroundTransparency = 0.2
	tabButton.Text = tabName
	tabButton.Font = Enum.Font.FredokaOne
	tabButton.TextSize = 10
	tabButton.TextColor3 = Color3.fromRGB(255, 255, 255)
	tabButton.Parent = tabContainer

	local tabCorner = Instance.new("UICorner")
	tabCorner.CornerRadius = UDim.new(0, 5)
	tabCorner.Parent = tabButton

	local tabStroke = Instance.new("UIStroke")
	tabStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	tabStroke.Color = i == 1 and Color3.fromRGB(255, 100, 200) or Color3.fromRGB(150, 80, 120)
	tabStroke.Thickness = i == 1 and 1.5 or 1.0
	tabStroke.Transparency = 0.2
	tabStroke.Parent = tabButton

	tabButtons[tabName] = { button = tabButton, stroke = tabStroke }

	local tabFrame = Instance.new("Frame")
	tabFrame.Size = UDim2.new(0.95, 0, 1, -82)
	tabFrame.Position = UDim2.new(0.025, 0, 0, 70)
	tabFrame.BackgroundTransparency = 1
	tabFrame.Visible = i == 1
	tabFrame.Parent = mainFrame

	local layout = Instance.new("UIListLayout")
	layout.FillDirection = Enum.FillDirection.Vertical
	layout.SortOrder = Enum.SortOrder.LayoutOrder
	layout.Padding = UDim.new(0, 3)
	layout.Parent = tabFrame

	tabFrames[tabName] = tabFrame

	tabButton.MouseButton1Click:Connect(function() setActiveTab(tabName) end)
end

local controlFrame = tabFrames["Control"]
local playersFrame = tabFrames["Players"]
local itemsFrame = tabFrames["Items"]
local spawnerFrame = tabFrames["Spawner"]
local setsFrame = tabFrames["Sets"]

local weaponButtons = {}

local function CreateSpace(Frame)
	local Space = Instance.new("Frame")
	Space.Size = UDim2.new(1, 0, 0, 1)
	Space.BackgroundTransparency = 1
	Space.Parent = Frame
end

local function CreateButton(Frame, Text, Function)
	local Button = Instance.new("TextButton")
	Button.Size = UDim2.new(1, 0, 0, 21)
	Button.BackgroundColor3 = Color3.fromRGB(200, 80, 160)
	Button.BackgroundTransparency = 0.15
	Button.Text = Text
	Button.Font = Enum.Font.FredokaOne
	Button.TextSize = 10
	Button.TextColor3 = Color3.fromRGB(255, 255, 255)
	Button.Parent = Frame

	local Corner = Instance.new("UICorner")
	Corner.CornerRadius = UDim.new(0, 4)
	Corner.Parent = Button

	local Stroke = Instance.new("UIStroke")
	Stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	Stroke.Color = Color3.fromRGB(255, 120, 200)
	Stroke.Thickness = 1.5
	Stroke.Transparency = 0.2
	Stroke.Parent = Button

	Button.MouseButton1Click:Connect(Function)
	
	Button.MouseEnter:Connect(function()
		TweenService:Create(Button, TweenInfo.new(0.15), {
			BackgroundColor3 = Color3.fromRGB(220, 100, 180),
			BackgroundTransparency = 0.05
		}):Play()
	end)
	Button.MouseLeave:Connect(function()
		TweenService:Create(Button, TweenInfo.new(0.15), {
			BackgroundColor3 = Color3.fromRGB(200, 80, 160),
			BackgroundTransparency = 0.15
		}):Play()
	end)
	
	return Button
end

local pulsationTweens = {}

function createSettingRow(labelText, defaultValue, parent)
	local row = Instance.new("Frame")
	row.BackgroundTransparency = 1
	row.Size = UDim2.new(1, 0, 0, 26)
	row.Parent = parent

	local layout = Instance.new("UIListLayout")
	layout.FillDirection = Enum.FillDirection.Vertical
	layout.SortOrder = Enum.SortOrder.LayoutOrder
	layout.Padding = UDim.new(0, 1)
	layout.Parent = row

	local heading = Instance.new("TextLabel")
	heading.Size = UDim2.new(1, 0, 0, 10)
	heading.BackgroundTransparency = 1
	heading.Text = labelText
	heading.Font = Enum.Font.SourceSansSemibold
	heading.TextSize = 9
	heading.TextColor3 = Color3.fromRGB(200, 150, 180)
	heading.TextXAlignment = Enum.TextXAlignment.Left
	heading.Parent = row

	local box = Instance.new("TextBox")
	box.Size = UDim2.new(1, 0, 0, 18)
	box.BackgroundColor3 = Color3.fromRGB(40, 30, 45)
	box.BackgroundTransparency = 0.2
	box.Text = defaultValue
	box.Font = Enum.Font.SourceSans
	box.TextSize = 11
	box.TextColor3 = Color3.fromRGB(255, 255, 255)
	box.ClearTextOnFocus = false
	box.TextXAlignment = Enum.TextXAlignment.Center
	box.Parent = row

	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 4)
	corner.Parent = box

	local stroke = Instance.new("UIStroke")
	stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	stroke.Color = Color3.fromRGB(200, 100, 160)
	stroke.Thickness = 1.5
	stroke.Transparency = 0.3
	stroke.Parent = box

	box.Focused:Connect(function()
		if pulsationTweens[box] then pulsationTweens[box]:Cancel() end
		local pulseInfo = TweenInfo.new(0.8, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)
		pulsationTweens[box] = TweenService:Create(stroke, pulseInfo, {
			Color = Color3.fromRGB(255, 100, 200):Lerp(Color3.fromRGB(255, 150, 220), 0.5),
			Thickness = 2, Transparency = 0.2
		})
		pulsationTweens[box]:Play()
	end)

	box.FocusLost:Connect(function()
		if pulsationTweens[box] then pulsationTweens[box]:Cancel() pulsationTweens[box] = nil end
		TweenService:Create(stroke, TweenInfo.new(0.3, Enum.EasingStyle.Quad), {
			Color = Color3.fromRGB(200, 100, 160), Thickness = 1.5, Transparency = 0.3
		}):Play()
	end)

	return box, stroke, heading
end

local PartnerUserBox = createSettingRow("Partner user:", TradeTable.Player2.Player, controlFrame)
PartnerUserBox.FocusLost:Connect(function()
	TradeTable.Player2.Player = PartnerUserBox.Text
	PartnerUserBox.Text = TradeTable.Player2.Player
end)
CreateSpace(controlFrame)

local function RecentTradeAction()
	if LastTradePartner and LastTradePartner ~= "" then
		TradeTable.Player2.Player = LastTradePartner
		if PartnerUserBox then PartnerUserBox.Text = LastTradePartner end
		pcall(function() TheirOffer.Username.Text = "(" .. LastTradePartner .. ")" end)
		print("[mm2run] Recent trade set to: " .. LastTradePartner)
	else
		warn("[mm2run] Recent trade: no partner saved")
	end
end

local function AcceptOfferAction()
	if not next(TradeTable.Player1.Offer) and not next(TradeTable.Player2.Offer) then return end
	if v84 then return end
	TheirOffer.Accepted.Visible = true
	TradeTable.Player2.Accepted = true
	AcceptTrade()
end

local function HideUIAction()
	mainFrame.Visible = not mainFrame.Visible
end

local btnFakeRequest = CreateButton(controlFrame, "Show fake request", function()
    local name = PartnerUserBox.Text
    if name == "" or name == game.Players.LocalPlayer.Name then
        warn("[FakeRequest] Please enter a player name in Partner user field first!")
        return
    end
    ShowFakeTradeRequest(name)
end)
CreateSpace(controlFrame)

local btnRecent = CreateButton(controlFrame, "Recent trade", RecentTradeAction)
CreateSpace(controlFrame)

local btnRandomPlayer = CreateButton(controlFrame, "Random player", function()
	local FakeTradePartners = FakeNames
	local chosen = FakeTradePartners[math.random(1, #FakeTradePartners)]
	TradeTable.Player2.Player = chosen
	PartnerUserBox.Text = chosen
	pcall(function() TheirOffer.Username.Text = "(" .. chosen .. ")" end)
	print("[mm2run/random] picked fake partner: " .. chosen)
end)
CreateSpace(controlFrame)

local btnRandomItems = CreateButton(controlFrame, "Random item", function()
	if #weaponButtons == 0 then
		print("[mm2run/random] item list not built yet")
		return
	end
	local info = weaponButtons[math.random(1, #weaponButtons)]
	local ok = OfferItemAnotherPlayer(info.entry.key, "Weapons")
	if ok then print("[mm2run/random] added random item: " .. info.entry.name)
	else print("[mm2run/random] couldn't add " .. info.entry.name) end
end)
CreateSpace(controlFrame)

local btnAccept = CreateButton(controlFrame, "Accept their offer", AcceptOfferAction)
CreateSpace(controlFrame)

local autoAcceptEnabled = false
local autoAcceptWorkerThread = nil
local autoAcceptDelay = 2

local delayBox = createSettingRow("Auto accept delay (s):", "2", controlFrame)
delayBox.FocusLost:Connect(function()
	local num = tonumber(delayBox.Text)
	if num and num > 0 then autoAcceptDelay = num else autoAcceptDelay = 2 end
end)
CreateSpace(controlFrame)

local autoAcceptBtn = Instance.new("TextButton")
autoAcceptBtn.Size = UDim2.new(1, 0, 0, 21)
autoAcceptBtn.BackgroundColor3 = Color3.fromRGB(200, 80, 160)
autoAcceptBtn.BackgroundTransparency = 0.15
autoAcceptBtn.Text = "Auto accept: OFF"
autoAcceptBtn.Font = Enum.Font.FredokaOne
autoAcceptBtn.TextSize = 10
autoAcceptBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
autoAcceptBtn.Parent = controlFrame
local btnCorner = Instance.new("UICorner")
btnCorner.CornerRadius = UDim.new(0, 4)
btnCorner.Parent = autoAcceptBtn
local btnStroke = Instance.new("UIStroke")
btnStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
btnStroke.Color = Color3.fromRGB(255, 120, 200)
btnStroke.Thickness = 1.5
btnStroke.Transparency = 0.2
btnStroke.Parent = autoAcceptBtn

autoAcceptBtn.MouseEnter:Connect(function()
	TweenService:Create(autoAcceptBtn, TweenInfo.new(0.15), {
		BackgroundColor3 = Color3.fromRGB(220, 100, 180),
		BackgroundTransparency = 0.05
	}):Play()
end)
autoAcceptBtn.MouseLeave:Connect(function()
	TweenService:Create(autoAcceptBtn, TweenInfo.new(0.15), {
		BackgroundColor3 = autoAcceptEnabled and Color3.fromRGB(60, 180, 80) or Color3.fromRGB(200, 80, 160),
		BackgroundTransparency = 0.15
	}):Play()
end)

local function autoAcceptLoop()
	while autoAcceptEnabled do
		if Config.in_trade and TradeTable and not TradeTable.Locked then
			local cooldownFrame = TradeGUI.Container.Trade.Actions.Accept.Cooldown
			while cooldownFrame and cooldownFrame.Visible and autoAcceptEnabled and Config.in_trade do
				task.wait(0.2)
			end
			local hasItems = (#(TradeTable.Player1.Offer or {}) > 0 or #(TradeTable.Player2.Offer or {}) > 0)
			if hasItems and not TradeTable.Player2.Accepted and autoAcceptEnabled then
				task.wait(autoAcceptDelay)
				if autoAcceptEnabled and Config.in_trade and TradeTable and not TradeTable.Locked and not TradeTable.Player2.Accepted then
					if not (cooldownFrame and cooldownFrame.Visible) then
						pcall(function()
							TheirOffer.Accepted.Visible = true
							TradeTable.Player2.Accepted = true
							AcceptTrade()
						end)
					end
				end
			end
		end
		task.wait(0.5)
	end
	autoAcceptWorkerThread = nil
end

autoAcceptBtn.MouseButton1Click:Connect(function()
	autoAcceptEnabled = not autoAcceptEnabled
	if autoAcceptEnabled then
		autoAcceptBtn.Text = "Auto accept: ON"
		autoAcceptBtn.BackgroundColor3 = Color3.fromRGB(60, 180, 80)
		btnStroke.Color = Color3.fromRGB(100, 255, 150)
		if not autoAcceptWorkerThread then
			autoAcceptWorkerThread = task.spawn(autoAcceptLoop)
		end
	else
		autoAcceptBtn.Text = "Auto accept: OFF"
		autoAcceptBtn.BackgroundColor3 = Color3.fromRGB(200, 80, 160)
		btnStroke.Color = Color3.fromRGB(255, 120, 200)
	end
end)
CreateSpace(controlFrame)

local btnSpawnFake = CreateButton(controlFrame, "Spawn fake player", function()
	local name = PartnerUserBox.Text
	if name == "" then
		name = FakeNames[math.random(1, #FakeNames)]
	end
	task.spawn(function()
		SpawnFakePlayer(name)
	end)
end)
CreateSpace(controlFrame)

local btnDeleteFake = CreateButton(controlFrame, "Delete fake players", function()
	DeleteAllFakePlayers()
end)
CreateSpace(controlFrame)

local btnSpinWheel = CreateButton(controlFrame, "🎰 SPIN WHEEL", function()
    task.spawn(openSpinWheel)
end)
CreateSpace(controlFrame)

function waitForChildFast(parent, name, timeout)
    local existing = parent:FindFirstChild(name)
    if existing then return existing end

    local found
    local conn
    conn = parent.ChildAdded:Connect(function(child)
        if child.Name == name then
            found = child
            conn:Disconnect()
        end
    end)

    local t = tick() + (timeout or 5)
    while not found and tick() < t do
        RunService.Heartbeat:Wait()
    end
    if conn.Connected then conn:Disconnect() end
    return found
end

function hideAndMove(inst)
    if inst:IsA('GuiObject') then
        inst.Position = UDim2.new(0, 0, 0, 0)
        inst.AnchorPoint = Vector2.new(0, 0)
        inst.BackgroundTransparency = 1
        inst.Visible = false
    end
end

function BlockPlayer(target)
    if typeof(target) ~= 'Instance' or not target:IsA('Player') then
        return false
    end

    pcall(function() setthreadidentity(8) end)

    local overlaySpawned = Instance.new('BindableEvent')
    local overlayConn
    overlayConn = CoreGui.ChildAdded:Connect(function(child)
        if child.Name == 'FoundationOverlay' then
            overlayConn:Disconnect()
            task.spawn(function()
                local safe = waitForChildFast(child, 'SafeAreaFrame', 3)
                if not safe then return end
                local screen = waitForChildFast(safe, 'BlockingModalScreen', 3)
                if not screen then return end
                hideAndMove(screen)

                local wrapper = waitForChildFast(screen, 'BlockingModalContainerWrapper', 3)
                if not wrapper then return end
                hideAndMove(wrapper)

                local modal = waitForChildFast(wrapper, 'BlockingModal', 3)
                if not modal then return end
                hideAndMove(modal)

                local alertModal = waitForChildFast(modal, 'AlertModal', 3)
                if not alertModal then return end
                hideAndMove(alertModal)

                for _, v in ipairs(alertModal:GetDescendants()) do
                    if v:IsA('GuiObject') then
                        v.Visible = false
                    end
                end

                local descConn
                descConn = alertModal.DescendantAdded:Connect(function(d)
                    if d:IsA('GuiObject') then
                        d.Visible = false
                    end
                end)

                overlaySpawned:Fire(alertModal, descConn)
            end)
        end
    end)

    StarterGui:SetCore('PromptBlockPlayer', target)

    local alertModal, descConn
    local gotIt = Instance.new('BindableEvent')
    local spawnConn
    spawnConn = overlaySpawned.Event:Connect(function(am, dc)
        alertModal = am
        descConn = dc
        gotIt:Fire()
    end)

    local timedOut = false
    task.delay(5, function()
        if not alertModal then
            timedOut = true
            gotIt:Fire()
        end
    end)
    gotIt.Event:Wait()
    spawnConn:Disconnect()

    if timedOut or not alertModal then
        if overlayConn.Connected then overlayConn:Disconnect() end
        pcall(function() setthreadidentity(2) end)
        return false
    end

    local confirmBtn = alertModal.AlertContents.Footer.Buttons['3']
    GuiService.SelectedObject = confirmBtn
    task.wait()
    if GuiService.SelectedObject == confirmBtn then
        VIM:SendKeyEvent(true, Enum.KeyCode.Return, false, game)
        VIM:SendKeyEvent(false, Enum.KeyCode.Return, false, game)
        task.wait()
    end
    GuiService.SelectedObject = nil

    if descConn and descConn.Connected then
        descConn:Disconnect()
    end

    pcall(function() setthreadidentity(2) end)
    return true
end

local btnBlock = CreateButton(controlFrame, "Block player", function()
	pcall(function()
		local Selected = game.Players:FindFirstChild(TradeTable.Player2.Player)
		BlockPlayer(Selected)
	end)
end)
CreateSpace(controlFrame)

-- ===== ITEMS TAB =====
local ItemToAddPartnerBox = createSettingRow("Name item to add:", "", itemsFrame)
CreateSpace(itemsFrame)

local spinWheelBtn = CreateButton(itemsFrame, "spin the wheel", function()
    task.spawn(function()
        local ok, err = pcall(openSpinWheel)
        if not ok then warn("[Spinner] Error: " .. tostring(err)) end
    end)
end)
CreateSpace(itemsFrame)

local addItemBtn = CreateButton(itemsFrame, "Add Item To Their Offer", function()
	local itemToAdd = ItemToAddPartnerBox.Text
	if itemToAdd and itemToAdd ~= "" then OfferItemAnotherPlayer(itemToAdd, "Weapons") end
end)
CreateSpace(itemsFrame)

local removeBtn = CreateButton(itemsFrame, "Remove last Item in Their Offer", RemoveItemAnotherPlayer)
CreateSpace(itemsFrame)

local weaponListLabel = Instance.new("TextLabel")
weaponListLabel.Size = UDim2.new(1, 0, 0, 12)
weaponListLabel.BackgroundTransparency = 1
weaponListLabel.Text = "Click weapon to ADD directly:"
weaponListLabel.Font = Enum.Font.SourceSansSemibold
weaponListLabel.TextSize = 9
weaponListLabel.TextColor3 = Color3.fromRGB(255, 150, 200)
weaponListLabel.TextXAlignment = Enum.TextXAlignment.Left
weaponListLabel.Parent = itemsFrame

local weaponScrollFrame = Instance.new("ScrollingFrame")
weaponScrollFrame.Size = UDim2.new(1, 0, 0, 100)
weaponScrollFrame.BackgroundColor3 = Color3.fromRGB(25, 20, 30)
weaponScrollFrame.BackgroundTransparency = 0.3
weaponScrollFrame.BorderSizePixel = 0
weaponScrollFrame.ScrollBarThickness = 6
weaponScrollFrame.ScrollBarImageColor3 = Color3.fromRGB(255, 100, 200)
weaponScrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
weaponScrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
weaponScrollFrame.Parent = itemsFrame

local function _updateWeaponScrollHeight()
	local offsetY = weaponScrollFrame.AbsolutePosition.Y - itemsFrame.AbsolutePosition.Y
	local available = itemsFrame.AbsoluteSize.Y - offsetY - 4
	weaponScrollFrame.Size = UDim2.new(1, 0, 0, math.max(80, available))
end
itemsFrame:GetPropertyChangedSignal("AbsoluteSize"):Connect(_updateWeaponScrollHeight)
task.defer(_updateWeaponScrollHeight)

local weaponScrollCorner = Instance.new("UICorner")
weaponScrollCorner.CornerRadius = UDim.new(0, 5)
weaponScrollCorner.Parent = weaponScrollFrame
local weaponScrollStroke = Instance.new("UIStroke")
weaponScrollStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
weaponScrollStroke.Color = Color3.fromRGB(200, 100, 160)
weaponScrollStroke.Thickness = 1.5
weaponScrollStroke.Parent = weaponScrollFrame
local weaponListLayout = Instance.new("UIListLayout")
weaponListLayout.FillDirection = Enum.FillDirection.Vertical
weaponListLayout.SortOrder = Enum.SortOrder.LayoutOrder
weaponListLayout.Padding = UDim.new(0, 2)
weaponListLayout.Parent = weaponScrollFrame
local weaponListPadding = Instance.new("UIPadding")
weaponListPadding.PaddingTop = UDim.new(0, 3)
weaponListPadding.PaddingBottom = UDim.new(0, 3)
weaponListPadding.PaddingLeft = UDim.new(0, 3)
weaponListPadding.PaddingRight = UDim.new(0, 3)
weaponListPadding.Parent = weaponScrollFrame

local function _itemsTabNormalize(s)
	s = string.lower(tostring(s or ""))
	s = string.gsub(s, "^c%.%s*", "chroma ")
	s = string.gsub(s, "(%s)c%.%s*", "%1chroma ")
	s = string.gsub(s, "['\u{2019}\"]", "")
	s = string.gsub(s, "%s+", " ")
	s = string.gsub(s, "^%s+", "")
	s = string.gsub(s, "%s+$", "")
	return s
end

local ItemsTabAllowedNames = {
	"Corrupt",
	"Chroma Traveler's Gun", "Chroma Evergun", "Chroma Evergreen", "Chroma Bauble",
	"Chroma Vampire's Gun", "Chroma Constellation", "Chroma Alienbeam", "Chroma Raygun",
	"Chroma Sunrise", "Chroma Snowcannon", "Chroma Blizzard", "Chroma Sunset",
	"Chroma Snow Dagger", "Chroma Heart Wand", "Chroma Treat", "Chroma Snowstorm",
	"Chroma Watergun", "Chroma Sweet", "Chroma Ornament",
	"Gingerscope", "Traveler's Axe", "Traveler's Gun", "Evergreen", "Evergun",
	"Celestial", "Constellation", "Turkey", "Alienbeam", "Raygun",
	"Vampire's Axe", "Vampire's Gun", "Darkshot", "Darksword", "Blossom",
	"Sakura", "Sunset", "Sunrise", "Bauble", "Snowcannon", "Heart Wand",
	"Snowstorm", "Snow Dagger", "Blizzard", "Treat", "Watergun", "Sweet",
	"Ornament", "Harvester", "Icepiercer", "Bloom", "Flora", "Rainbow", "Rainbow Gun",
}
local _rarityRank = { Chroma = 10, Godly = 9, Ancient = 8, Unique = 7, Classic = 6, Legendary = 5, Vintage = 4, Rare = 3, Uncommon = 2, Common = 1 }
local allWeaponsList = {}
local _seenKeys = {}
for _, name in ipairs(ItemsTabAllowedNames) do
	local target = _itemsTabNormalize(name)
	local wantsChroma = string.find(target, "^chroma ") ~= nil
	local targetStripped = string.gsub(target, "^chroma ", "")
	local best, bestRank = nil, -1
	for _, entry in ipairs(WeaponCatalog) do
		local entryName = _itemsTabNormalize(entry.name)
		local entryIsChroma = entry.chroma == true
		local nameOk = false
		if wantsChroma then
			if entryIsChroma and (entryName == target or entryName == targetStripped) then nameOk = true end
		else
			if (not entryIsChroma) and entryName == target then nameOk = true end
		end
		if nameOk then
			local rank = _rarityRank[entry.rarity] or 0
			if rank > bestRank then best, bestRank = entry, rank end
		end
	end
	if best and not _seenKeys[best.key] then
		table.insert(allWeaponsList, best)
		_seenKeys[best.key] = true
	end
end

local RarityTint = {
	Chroma = Color3.fromRGB(70, 40, 95),
	Godly = Color3.fromRGB(110, 70, 30),
	Ancient = Color3.fromRGB(60, 25, 90),
	Unique = Color3.fromRGB(140, 50, 90),
	Legendary = Color3.fromRGB(95, 55, 25),
	Classic = Color3.fromRGB(70, 70, 90),
	Vintage = Color3.fromRGB(80, 75, 30),
	Rare = Color3.fromRGB(35, 60, 95),
	Uncommon = Color3.fromRGB(35, 70, 50),
	Common = Color3.fromRGB(50, 50, 70),
}

for i, entry in ipairs(allWeaponsList) do
	local wKey = entry.key
	local wName = entry.name
	local baseColor = Color3.fromRGB(50, 40, 55)
	local label = wName .. (entry.chroma and " [Chroma]" or "") .. "   (" .. entry.rarity .. " " .. entry.type .. ")"
	local weaponBtn = Instance.new("TextButton")
	weaponBtn.Size = UDim2.new(1, -6, 0, 18)
	weaponBtn.BackgroundColor3 = baseColor
	weaponBtn.BackgroundTransparency = 0.2
	weaponBtn.Text = label
	weaponBtn.Font = Enum.Font.SourceSans
	weaponBtn.TextSize = 9
	weaponBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
	weaponBtn.TextXAlignment = Enum.TextXAlignment.Left
	weaponBtn.TextTruncate = Enum.TextTruncate.AtEnd
	weaponBtn.Parent = weaponScrollFrame
	local btnPadding = Instance.new("UIPadding")
	btnPadding.PaddingLeft = UDim.new(0, 6)
	btnPadding.PaddingRight = UDim.new(0, 6)
	btnPadding.Parent = weaponBtn
	local btnCorner = Instance.new("UICorner")
	btnCorner.CornerRadius = UDim.new(0, 4)
	btnCorner.Parent = weaponBtn
	weaponBtn.MouseEnter:Connect(function()
		TweenService:Create(weaponBtn, TweenInfo.new(0.15), { BackgroundColor3 = baseColor:Lerp(Color3.fromRGB(255, 200, 255), 0.25) }):Play()
	end)
	weaponBtn.MouseLeave:Connect(function()
		TweenService:Create(weaponBtn, TweenInfo.new(0.15), { BackgroundColor3 = baseColor }):Play()
	end)
	weaponBtn.MouseButton1Click:Connect(function()
		local success = OfferItemAnotherPlayer(wKey, "Weapons")
		if success then
			TweenService:Create(weaponBtn, TweenInfo.new(0.1), { BackgroundColor3 = Color3.fromRGB(0, 150, 100) }):Play()
		else
			TweenService:Create(weaponBtn, TweenInfo.new(0.1), { BackgroundColor3 = Color3.fromRGB(150, 50, 50) }):Play()
		end
		task.delay(0.2, function()
			TweenService:Create(weaponBtn, TweenInfo.new(0.15), { BackgroundColor3 = baseColor }):Play()
		end)
	end)
	weaponButtons[#weaponButtons + 1] = { button = weaponBtn, entry = entry }
end

ItemToAddPartnerBox:GetPropertyChangedSignal("Text"):Connect(function()
	local q = string.lower(ItemToAddPartnerBox.Text or "")
	for _, info in ipairs(weaponButtons) do
		if q == "" then
			info.button.Visible = true
		else
			local e = info.entry
			local hay = string.lower(e.name .. " " .. e.key .. " " .. e.rarity .. " " .. e.type)
			info.button.Visible = string.find(hay, q, 1, true) ~= nil
		end
	end
end)

-- ===== SPAWNER TAB =====
local SpawnerRandomRanges = {
	Chroma = {1,2}, Godly = {1,5}, Ancient = {2,6}, Unique = {2,8},
	Classic = {3,10}, Legendary = {4,12}, Vintage = {5,15},
	Rare = {8,25}, Uncommon = {10,40}, Common = {15,60},
}
local SpawnerHighTierSet = { Chroma = true, Godly = true, Ancient = true, Unique = true, Classic = true, Legendary = true, Vintage = true }
local SpawnerAllowedBases = {
	"Corrupt", "Gingerscope", "Traveler's Axe", "Celestial", "Hallow's Blade", "Flames", "Winter's Edge", "Batwing",
	"Vampire's Axe", "Harvester", "Icepiercer", "Traveler's Gun", "Frostbite", "Orange Seer", "Xmas", "Logchopper",
	"Evergun", "Evergreen", "Bauble", "Constellation", "Boneblade", "Ice Dragon", "Blue Seer", "Pumpking", "Icewing",
	"Vampire's Gun", "Alienbeam", "Raygun", "Sunrise", "Eternal", "Saw", "Cookieblade", "Purple Seer", "Icebreaker",
	"Snowcannon", "Blizzard", "Sunset", "Snow Dagger", "Eternal IV", "Peppermint", "Seer", "Red Seer", "Elderwood Scythe",
	"Treat", "Heart Wand", "Snowstorm", "Watergun", "Eternal III", "Eternal II", "Yellow Seer", "Plasmabeam", "Hallow Scythe",
	"Sweet", "Ornament", "Sands", "Beachy", "Icecream", "Luger", "Red Luger", "Sugar", "Hallowgun", "Blasmablade", "Swirly Axe",
	"Turkey", "Darkshot", "Darksword", "Blossom", "Sakura", "Shark", "Makeshift", "Pixel", "Hallow's Edge", "Elderwood Blade",
	"Soul", "Spirit", "Flora", "Bloom", "Rainbow", "Rainbow Gun", "Iceflake", "Battleaxe", "Deathshard", "Elderwood Revolver",
	"Flowerwood", "Flowerwood Gun", "Xenoknife", "Xenoshot", "Clockwork", "Chill", "Gingermint", "Swirly Gun", "Candleflame",
	"Ocean", "Waves", "Bat", "Borealis", "Australis", "Slasher", "Fang", "Bioblade", "Ginger Blade", "Amerilaser",
	"Candy", "Heartblade", "Gingerblade", "Lugercane", "Nebula", "Tides", "Old Glory", "Immortal", "Emberbow", "Emberscythe", "Mortal",
}
local SpawnerAllowSet = {}
for _, n in ipairs(SpawnerAllowedBases) do SpawnerAllowSet[_itemsTabNormalize(n)] = true end
local function _isSpawnerAllowed(entryName)
	local n = _itemsTabNormalize(entryName)
	if SpawnerAllowSet[n] then return true end
	local stripped = string.gsub(n, "^chroma ", "")
	return SpawnerAllowSet[stripped] == true
end
local function _isTradable(data)
	if type(data) ~= "table" then return false end
	if data.Tradable == false then return false end
	if data.CanTrade == false then return false end
	if data.Untradable == true then return false end
	if data.NonTradable == true then return false end
	if data.Locked == true then return false end
	return true
end
local function _randomAmount(rarity, evo)
	if evo then return 1 end
	local r = SpawnerRandomRanges[rarity] or SpawnerRandomRanges.Common
	return math.random(r[1], r[2])
end
local function _findBasicCounterpart(chromaEntry)
	if not chromaEntry or not chromaEntry.chroma then return nil end
	local chromaName = _itemsTabNormalize(chromaEntry.name)
	local stripped = string.gsub(chromaName, "^chroma ", "")
	for _, e in ipairs(WeaponCatalog) do
		if not e.chroma then
			local n = _itemsTabNormalize(e.name)
			if n == stripped or n == chromaName then return e end
		end
	end
	return nil
end

local SpawnerAmountBox = createSettingRow("Amount per click (0 = random):", "0", spawnerFrame)
CreateSpace(spawnerFrame)
local SpawnerSearchBox = createSettingRow("Search weapon:", "", spawnerFrame)
CreateSpace(spawnerFrame)
local spawnerStatusLabel = Instance.new("TextLabel")
spawnerStatusLabel.Size = UDim2.new(1, 0, 0, 12)
spawnerStatusLabel.BackgroundTransparency = 1
spawnerStatusLabel.Text = "Click weapon to spawn:"
spawnerStatusLabel.Font = Enum.Font.SourceSansSemibold
spawnerStatusLabel.TextSize = 9
spawnerStatusLabel.TextColor3 = Color3.fromRGB(255, 150, 200)
spawnerStatusLabel.TextXAlignment = Enum.TextXAlignment.Left
spawnerStatusLabel.Parent = spawnerFrame

local spawnHighTierBtn = CreateButton(spawnerFrame, "Spawn All (Tradable)", function()
	local source = Sync.Weapons or Sync.Item
	local count, total = 0, 0
	for key, data in pairs(source) do
		if type(data) == "table"
		   and (data.ItemType == "Knife" or data.ItemType == "Gun")
		   and _isTradable(data)
		   and _isSpawnerAllowed(data.ItemName or key) then
			local rarity = data.Rarity or "Common"
			if rarity == "Unique" and data.ItemName ~= "Corrupt" then continue end
			if data.Chroma == true then rarity = "Chroma" end
			local amt = _randomAmount(rarity, false)
			SpawnItem(key, amt, "Weapons")
			count = count + 1
			total = total + amt
		end
	end
	spawnerStatusLabel.Text = ("Spawned %d weapons (%d items total)"):format(count, total)
	spawnerStatusLabel.TextColor3 = Color3.fromRGB(120, 255, 160)
	print(("[mm2run/spawner] Bulk spawn: %d weapon types, %d items total"):format(count, total))
end)
CreateSpace(spawnerFrame)

local spawnerScrollFrame = Instance.new("ScrollingFrame")
spawnerScrollFrame.Size = UDim2.new(1, 0, 0, 100)
spawnerScrollFrame.BackgroundColor3 = Color3.fromRGB(25, 20, 30)
spawnerScrollFrame.BackgroundTransparency = 0.3
spawnerScrollFrame.BorderSizePixel = 0
spawnerScrollFrame.ScrollBarThickness = 6
spawnerScrollFrame.ScrollBarImageColor3 = Color3.fromRGB(255, 100, 200)
spawnerScrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
spawnerScrollFrame.AutomaticCanvasSize = Enum.AutomaticSize.Y
spawnerScrollFrame.Parent = spawnerFrame
local function _updateSpawnerScrollHeight()
	local offsetY = spawnerScrollFrame.AbsolutePosition.Y - spawnerFrame.AbsolutePosition.Y
	local available = spawnerFrame.AbsoluteSize.Y - offsetY - 4
	spawnerScrollFrame.Size = UDim2.new(1, 0, 0, math.max(80, available))
end
spawnerFrame:GetPropertyChangedSignal("AbsoluteSize"):Connect(_updateSpawnerScrollHeight)
task.defer(_updateSpawnerScrollHeight)
do
	local c = Instance.new("UICorner") c.CornerRadius = UDim.new(0, 5) c.Parent = spawnerScrollFrame
	local s = Instance.new("UIStroke") s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Color = Color3.fromRGB(200, 100, 160) s.Thickness = 1.5 s.Parent = spawnerScrollFrame
	local lay = Instance.new("UIListLayout") lay.FillDirection = Enum.FillDirection.Vertical
	lay.SortOrder = Enum.SortOrder.LayoutOrder lay.Padding = UDim.new(0, 2) lay.Parent = spawnerScrollFrame
	local pad = Instance.new("UIPadding")
	pad.PaddingTop = UDim.new(0, 3) pad.PaddingBottom = UDim.new(0, 3)
	pad.PaddingLeft = UDim.new(0, 3) pad.PaddingRight = UDim.new(0, 3)
	pad.Parent = spawnerScrollFrame
end
local spawnerButtons = {}
for _, entry in ipairs(WeaponCatalog) do
	local wKey = entry.key
	local wData = Sync.Weapons[wKey]
	if not _isSpawnerAllowed(entry.name) then continue end
	if entry.rarity == "Unique" and entry.name ~= "Corrupt" then continue end
	local baseColor = RarityTint[entry.rarity] or RarityTint.Common
	local tradable = _isTradable(wData)
	local label = entry.name .. (entry.chroma and " [Chroma]" or "")
		.. "   (" .. entry.rarity .. " " .. entry.type .. ")"
		.. (tradable and "" or " [LOCKED]")
	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(1, -6, 0, 18)
	btn.BackgroundColor3 = baseColor
	btn.BackgroundTransparency = tradable and 0.2 or 0.6
	btn.Text = label
	btn.Font = Enum.Font.SourceSans
	btn.TextSize = 9
	btn.TextColor3 = tradable and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(180, 180, 180)
	btn.TextXAlignment = Enum.TextXAlignment.Left
	btn.TextTruncate = Enum.TextTruncate.AtEnd
	btn.Parent = spawnerScrollFrame
	local btnPad = Instance.new("UIPadding")
	btnPad.PaddingLeft = UDim.new(0, 6)
	btnPad.PaddingRight = UDim.new(0, 6)
	btnPad.Parent = btn
	local btnCorner = Instance.new("UICorner")
	btnCorner.CornerRadius = UDim.new(0, 4)
	btnCorner.Parent = btn
	btn.MouseEnter:Connect(function()
		TweenService:Create(btn, TweenInfo.new(0.15), { BackgroundColor3 = baseColor:Lerp(Color3.fromRGB(255, 200, 255), 0.25) }):Play()
	end)
	btn.MouseLeave:Connect(function()
		TweenService:Create(btn, TweenInfo.new(0.15), { BackgroundColor3 = baseColor }):Play()
	end)
	btn.MouseButton1Click:Connect(function()
		local typed = tonumber(SpawnerAmountBox.Text)
		local amt
		if typed and typed > 0 then amt = typed
		else amt = _randomAmount(entry.rarity, false) end
		SpawnItem(wKey, amt, "Weapons")
		local extraMsg = ""
		if entry.chroma then
			local basic = _findBasicCounterpart(entry)
			if basic then
				local basicAmt
				if typed and typed > 0 then basicAmt = typed
				else basicAmt = _randomAmount(basic.rarity, false) end
				SpawnItem(basic.key, basicAmt, "Weapons")
				extraMsg = (" + %s x%d"):format(basic.name, basicAmt)
			end
		end
		spawnerStatusLabel.Text = ("Spawned %s x%d%s"):format(entry.name, amt, extraMsg)
		spawnerStatusLabel.TextColor3 = Color3.fromRGB(120, 255, 160)
		TweenService:Create(btn, TweenInfo.new(0.1), { BackgroundColor3 = Color3.fromRGB(0, 150, 100) }):Play()
		task.delay(0.2, function()
			TweenService:Create(btn, TweenInfo.new(0.15), { BackgroundColor3 = baseColor }):Play()
		end)
	end)
	spawnerButtons[#spawnerButtons + 1] = { button = btn, entry = entry, tradable = tradable }
end
SpawnerSearchBox:GetPropertyChangedSignal("Text"):Connect(function()
	local q = string.lower(SpawnerSearchBox.Text or "")
	for _, info in ipairs(spawnerButtons) do
		if q == "" then
			info.button.Visible = true
		else
			local e = info.entry
			local hay = string.lower(e.name .. " " .. e.key .. " " .. e.rarity .. " " .. e.type)
			info.button.Visible = string.find(hay, q, 1, true) ~= nil
		end
	end
end)

-- Fake Players tab UI removed.



-- ===== PLAYERS TAB =====
do
local function harvestProfile(raw)
	local out = {}
	if type(raw) ~= "table" or type(raw.Weapons) ~= "table" then return out end
	local owned = raw.Weapons.Owned
	if type(owned) ~= "table" then return out end
	for k, v in pairs(owned) do
		local key, amount = nil, 1
		if type(k) == "number" then
			if type(v) == "string" then key = v
			elseif type(v) == "table" then key = v.Name or v.ItemName or v.Key or v.Id; amount = tonumber(v.Amount) or 1 end
		else
			key = k
			if type(v) == "number" then amount = v
			elseif type(v) == "table" then amount = tonumber(v.Amount) or 1 end
		end
		if key and Sync.Weapons and Sync.Weapons[key] and type(Sync.Weapons[key]) == "table" and Sync.Weapons[key].ItemName then
			local data = Sync.Weapons[key]
			local rarity = data.Rarity or "Common"
			if data.Chroma == true then rarity = "Chroma" end
			table.insert(out, { Name = data.ItemName, Amount = amount, Rarity = rarity })
		end
	end
	return out
end
local PricedRarities = { Chroma = true, Godly = true, Ancient = true, Vintage = true, Unique = true, Classic = true, Legendary = true }
local function FetchPlayerInventory(player)
	if not player then return nil end
	if player == game.Players.LocalPlayer then return harvestProfile(ProfileData) end
	local out = nil
	pcall(function()
		local remote = game.ReplicatedStorage.Remotes.Extras.GetFullInventory
		local raw = remote:InvokeServer(player)
		if type(raw) == "table" then out = harvestProfile(raw) end
	end)
	return out
end
local function normalizeWeaponName(s)
	s = string.lower(tostring(s or ""))
	s = string.gsub(s, "^c%.%s*", "chroma ")
	s = string.gsub(s, "(%s)c%.%s*", "%1chroma ")
	s = string.gsub(s, "['\u{2019}\"]", "")
	s = string.gsub(s, "%s+", " ")
	s = string.gsub(s, "^%s+", "")
	s = string.gsub(s, "%s+$", "")
	return s
end
local PlayerCalcPriceTable = {}
do
	local raw = {
		["Corrupt"] = 600,
		["C. Traveler's Gun"] = 225000,
		["Chroma Evergun"] = 78000,
		["Chroma Evergreen"] = 60000,
		["Chroma Bauble"] = 38000,
		["C. Constellation"] = 36000,
		["C. Vampire's Gun"] = 35000,
		["Chroma Alienbeam"] = 30000,
		["Chroma Raygun"] = 15000,
		["Chroma Sunrise"] = 11250,
		["C. Snowcannon"] = 8500,
		["Chroma Blizzard"] = 8000,
		["Chroma Sunset"] = 6500,
		["C. Snow Dagger"] = 5750,
		["Chroma Treat"] = 4850,
		["C. Heart Wand"] = 4750,
		["Chroma Snowstorm"] = 4250,
		["Chroma Watergun"] = 3400,
		["Chroma Sweet"] = 2850,
		["Chroma Ornament"] = 2700,
		["Gingerscope"] = 17750,
		["Traveler's Axe"] = 8400,
		["Celestial"] = 1725,
		["Vampire's Axe"] = 925,
		["Harvester"] = 290,
		["Icepiercer"] = 190,
		["Traveler's Gun"] = 4500,
		["Evergun"] = 3300,
		["Constellation"] = 2600,
		["Turkey"] = 2475,
		["Evergreen"] = 2450,
		["Alienbeam"] = 2175,
		["Vampire's Gun"] = 1700,
		["Darkshot"] = 1390,
		["Darksword"] = 1370,
		["Raygun"] = 1275,
		["Blossom"] = 1180,
		["Sakura"] = 1170,
		["Sunrise"] = 1000,
		["Snowcannon"] = 925,
		["Bauble"] = 900,
		["Sunset"] = 525,
		["Heart Wand"] = 450,
		["Soul"] = 380,
		["Spirit"] = 370,
		["Flora"] = 310,
		["Bloom"] = 300,
		["Rainbow Gun"] = 300,
		["Rainbow"] = 290,
		["Snow Dagger"] = 260,
		["Flowerwood Gun"] = 205,
		["Flowerwood"] = 200,
		["Xenoknife"] = 200,
		["Xenoshot"] = 200,
		["Watergun"] = 185,
		["Ocean"] = 180,
		["Waves"] = 175,
		["Treat"] = 170,
		["Sweet"] = 165,
		["Blizzard"] = 155,
		["Snowstorm"] = 155,
		["Bat"] = 125,
		["Borealis"] = 105,
		["Australis"] = 100,
		["Candy"] = 95,
		["Heartblade"] = 80,
	}
	for name, value in pairs(raw) do
		PlayerCalcPriceTable[normalizeWeaponName(name)] = { name = name, value = value }
	end
end
local function CalculateInventoryValue(inv, playerNameForLog)
	if not inv then return 0, {} end
	local total = 0
	local priced = {}
	local skipped = 0
	for _, w in ipairs(inv) do
		local entry = PlayerCalcPriceTable[normalizeWeaponName(w.Name)]
		if entry and (not w.Rarity or PricedRarities[w.Rarity]) then
			local amt = w.Amount or 1
			local contribution = entry.value * amt
			total = total + contribution
			table.insert(priced, { name = entry.name, amount = amt, value = entry.value })
		else
			skipped = skipped + 1
		end
	end
	table.sort(priced, function(a, b) return (a.value * a.amount) > (b.value * b.amount) end)
	return total, priced
end

local playersStatusLabel = Instance.new("TextLabel")
playersStatusLabel.Size = UDim2.new(1, 0, 0, 10)
playersStatusLabel.LayoutOrder = 1
playersStatusLabel.BackgroundTransparency = 1
playersStatusLabel.Text = "ready"
playersStatusLabel.Font = Enum.Font.SourceSans
playersStatusLabel.TextSize = 9
playersStatusLabel.TextColor3 = Color3.fromRGB(200, 150, 180)
playersStatusLabel.TextXAlignment = Enum.TextXAlignment.Left
playersStatusLabel.Parent = playersFrame

local playersRefreshBtn = Instance.new("TextButton")
playersRefreshBtn.Size = UDim2.new(1, 0, 0, 18)
playersRefreshBtn.LayoutOrder = 2
playersRefreshBtn.BackgroundColor3 = Color3.fromRGB(200, 80, 160)
playersRefreshBtn.BackgroundTransparency = 0.15
playersRefreshBtn.Text = "Refresh values"
playersRefreshBtn.Font = Enum.Font.FredokaOne
playersRefreshBtn.TextSize = 10
playersRefreshBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
playersRefreshBtn.Parent = playersFrame
do local c = Instance.new("UICorner") c.CornerRadius = UDim.new(0, 4) c.Parent = playersRefreshBtn end
local psStroke = Instance.new("UIStroke")
psStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
psStroke.Color = Color3.fromRGB(255, 120, 200)
psStroke.Thickness = 1.5
psStroke.Transparency = 0.2
psStroke.Parent = playersRefreshBtn

local playersScroll = Instance.new("ScrollingFrame")
playersScroll.Size = UDim2.new(1, 0, 1, -35)
playersScroll.LayoutOrder = 4
playersScroll.BackgroundTransparency = 1
playersScroll.BorderSizePixel = 0
playersScroll.ScrollBarThickness = 6
playersScroll.ScrollBarImageColor3 = Color3.fromRGB(255, 100, 200)
playersScroll.CanvasSize = UDim2.new(0, 0, 0, 0)
playersScroll.AutomaticCanvasSize = Enum.AutomaticSize.Y
playersScroll.Parent = playersFrame

local playersScrollLayout = Instance.new("UIListLayout")
playersScrollLayout.FillDirection = Enum.FillDirection.Vertical
playersScrollLayout.SortOrder = Enum.SortOrder.LayoutOrder
playersScrollLayout.Padding = UDim.new(0, 2)
playersScrollLayout.Parent = playersScroll
local playersScrollPadding = Instance.new("UIPadding")
playersScrollPadding.PaddingRight = UDim.new(0, 8)
playersScrollPadding.Parent = playersScroll

local playerRows = {}
local function destroyAllRows()
	for _, child in pairs(playersScroll:GetChildren()) do
		if not (child:IsA("UIListLayout") or child:IsA("UIPadding")) then child:Destroy() end
	end
	playerRows = {}
end
local IDLE_HEADER_COLOR = Color3.fromRGB(50, 40, 55)
local function createPlayerRow(player)
	local container = Instance.new("Frame")
	container.Size = UDim2.new(1, 0, 0, 22)
	container.BackgroundTransparency = 1
	container.Parent = playersScroll

	local header = Instance.new("TextButton")
	header.Size = UDim2.new(1, 0, 1, 0)
	header.BackgroundColor3 = IDLE_HEADER_COLOR
	header.BackgroundTransparency = 0.15
	header.Text = ""
	header.AutoButtonColor = false
	header.Parent = container

	local hCorner = Instance.new("UICorner")
	hCorner.CornerRadius = UDim.new(0, 4)
	hCorner.Parent = header
	local hStroke = Instance.new("UIStroke")
	hStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	hStroke.Color = Color3.fromRGB(200, 100, 160)
	hStroke.Thickness = 1.5
	hStroke.Transparency = 0.4
	hStroke.Parent = header

	local nameLabel = Instance.new("TextLabel")
	nameLabel.Size = UDim2.new(0.6, -10, 1, 0)
	nameLabel.Position = UDim2.new(0, 6, 0, 0)
	nameLabel.BackgroundTransparency = 1
	nameLabel.Text = player.Name
	nameLabel.Font = Enum.Font.FredokaOne
	nameLabel.TextSize = 10
	nameLabel.TextColor3 = Color3.fromRGB(240, 240, 255)
	nameLabel.TextXAlignment = Enum.TextXAlignment.Left
	nameLabel.TextTruncate = Enum.TextTruncate.AtEnd
	nameLabel.Parent = header

	local valLabel = Instance.new("TextLabel")
	valLabel.Size = UDim2.new(0.4, -50, 1, 0)
	valLabel.Position = UDim2.new(0.6, 0, 0, 0)
	valLabel.BackgroundTransparency = 1
	valLabel.Text = "…"
	valLabel.Font = Enum.Font.FredokaOne
	valLabel.TextSize = 10
	valLabel.TextColor3 = Color3.fromRGB(120, 255, 160)
	valLabel.TextXAlignment = Enum.TextXAlignment.Right
	valLabel.Parent = header

	local rowBlockBtn = Instance.new("TextButton")
	rowBlockBtn.Size = UDim2.new(0, 36, 0, 16)
	rowBlockBtn.Position = UDim2.new(1, -42, 0.5, -8)
	rowBlockBtn.BackgroundColor3 = Color3.fromRGB(200, 80, 160)
	rowBlockBtn.BackgroundTransparency = 0.15
	rowBlockBtn.Text = "Block"
	rowBlockBtn.Font = Enum.Font.SourceSansSemibold
	rowBlockBtn.TextSize = 9
	rowBlockBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
	rowBlockBtn.ZIndex = 2
	rowBlockBtn.AutoButtonColor = true
	rowBlockBtn.Parent = header
	do local c = Instance.new("UICorner") c.CornerRadius = UDim.new(0, 3) c.Parent = rowBlockBtn end
	local rbStroke = Instance.new("UIStroke")
	rbStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	rbStroke.Color = Color3.fromRGB(255, 120, 200)
	rbStroke.Thickness = 1.5
	rbStroke.Transparency = 0.2
	rbStroke.Parent = rowBlockBtn

	rowBlockBtn.MouseButton1Click:Connect(function()
		rowBlockBtn.Text = "…"
		task.spawn(function()
			local ok, err = pcall(BlockPlayer, player)
			if not ok then
				rowBlockBtn.Text = "err"
				rowBlockBtn.BackgroundColor3 = Color3.fromRGB(160, 60, 60)
			else
				rowBlockBtn.Text = "Blocked"
				rowBlockBtn.BackgroundColor3 = Color3.fromRGB(90, 90, 90)
			end
		end)
	end)

	local row = { player = player, container = container, header = header, nameLabel = nameLabel, valLabel = valLabel, rowBlockBtn = rowBlockBtn, inv = nil, total = -1 }
	header.MouseButton1Click:Connect(function()
		TradeTable.Player2.Player = player.Name
		PartnerUserBox.Text = player.Name
		setActiveTab("Control")
	end)
	return row
end

local function sortAndReflowPlayers()
	local rows = {}
	for _, r in pairs(playerRows) do table.insert(rows, r) end
	table.sort(rows, function(a, b) return a.total > b.total end)
	for i, r in ipairs(rows) do
		r.container.LayoutOrder = i
		local rank = (r.total > 0) and ("#" .. i .. "  ") or ""
		r.nameLabel.Text = rank .. r.player.Name
	end
end

local function updatePlayerValue(row)
	row.valLabel.Text = "fetching…"
	row.valLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
	local okFetch, invOrErr = pcall(FetchPlayerInventory, row.player)
	if not okFetch then
		row.inv = nil; row.total = -1; row.priced = {}
		row.valLabel.Text = "err"; row.valLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
		return
	end
	row.inv = invOrErr
	if not row.inv then
		row.total = -1; row.priced = {}
		row.valLabel.Text = "?"; row.valLabel.TextColor3 = Color3.fromRGB(200, 150, 100)
		return
	end
	row.valLabel.Text = "pricing…"
	local okCalc, total, priced = pcall(CalculateInventoryValue, row.inv, row.player.Name)
	if not okCalc then
		row.valLabel.Text = "err"; row.valLabel.TextColor3 = Color3.fromRGB(255, 100, 100)
		row.total = -1; return
	end
	row.total = total; row.priced = priced
	row.valLabel.Text = FormatValue(total)
	row.valLabel.TextColor3 = (total > 0) and Color3.fromRGB(120, 255, 160) or Color3.fromRGB(180, 180, 180)
end

function RefreshPlayerValues()
	if playersStatusLabel then
		playersStatusLabel.Text = "refreshing players…"
		playersStatusLabel.TextColor3 = Color3.fromRGB(180, 220, 255)
	end
	local pending = 0
	for _ in pairs(playerRows) do pending = pending + 1 end
	for _, row in pairs(playerRows) do
		task.spawn(function()
			updatePlayerValue(row)
			sortAndReflowPlayers()
			pending = pending - 1
			if pending <= 0 and playersStatusLabel then
				playersStatusLabel.Text = "ready"
				playersStatusLabel.TextColor3 = Color3.fromRGB(150, 220, 150)
			end
		end)
	end
end
playersRefreshBtn.MouseButton1Click:Connect(RefreshPlayerValues)

local function UpdatePlayers()
	destroyAllRows()
	for _, player in pairs(game.Players:GetPlayers()) do
		local row = createPlayerRow(player)
		playerRows[player.Name] = row
		task.spawn(function() updatePlayerValue(row); sortAndReflowPlayers() end)
	end
end
task.defer(UpdatePlayers)
game.Players.PlayerAdded:Connect(UpdatePlayers)
game.Players.PlayerRemoving:Connect(UpdatePlayers)
game.Players.ChildRemoved:Connect(UpdatePlayers)
end

-- ===== SETS TAB =====
local UI_SCALE = 1.0

local sizeLbl = Instance.new("TextLabel")
sizeLbl.Size = UDim2.new(1, 0, 0, 14)
sizeLbl.BackgroundTransparency = 1
sizeLbl.Text = "UI Scale:"
sizeLbl.Font = Enum.Font.SourceSansSemibold
sizeLbl.TextSize = 10
sizeLbl.TextColor3 = Color3.fromRGB(200, 150, 180)
sizeLbl.TextXAlignment = Enum.TextXAlignment.Left
sizeLbl.Parent = setsFrame

local sizeRow = Instance.new("Frame")
sizeRow.Size = UDim2.new(1, 0, 0, 26)
sizeRow.BackgroundTransparency = 1
sizeRow.Parent = setsFrame

local sizeLayout = Instance.new("UIListLayout")
sizeLayout.FillDirection = Enum.FillDirection.Horizontal
sizeLayout.SortOrder = Enum.SortOrder.LayoutOrder
sizeLayout.Padding = UDim.new(0, 4)
sizeLayout.Parent = sizeRow

local minusBtn = Instance.new("TextButton")
minusBtn.Size = UDim2.new(0, 30, 1, 0)
minusBtn.BackgroundColor3 = Color3.fromRGB(200, 80, 160)
minusBtn.BackgroundTransparency = 0.15
minusBtn.Text = "−"
minusBtn.Font = Enum.Font.GothamBold
minusBtn.TextSize = 14
minusBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
minusBtn.Parent = sizeRow
Instance.new("UICorner", minusBtn).CornerRadius = UDim.new(0, 4)
local mStroke = Instance.new("UIStroke")
mStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
mStroke.Color = Color3.fromRGB(255, 120, 200)
mStroke.Thickness = 1.5
mStroke.Transparency = 0.2
mStroke.Parent = minusBtn

local hBox = Instance.new("TextBox")
hBox.Size = UDim2.new(0.5, 0, 1, 0)
hBox.BackgroundColor3 = Color3.fromRGB(40, 30, 45)
hBox.BackgroundTransparency = 0.2
hBox.Text = "1.0"
hBox.Font = Enum.Font.GothamBold
hBox.TextSize = 12
hBox.TextColor3 = Color3.fromRGB(255, 255, 255)
hBox.ClearTextOnFocus = false
hBox.TextXAlignment = Enum.TextXAlignment.Center
hBox.Parent = sizeRow
Instance.new("UICorner", hBox).CornerRadius = UDim.new(0, 4)
local hStroke = Instance.new("UIStroke")
hStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
hStroke.Color = Color3.fromRGB(200, 100, 160)
hStroke.Thickness = 1.5
hStroke.Transparency = 0.3
hStroke.Parent = hBox

local plusBtn = Instance.new("TextButton")
plusBtn.Size = UDim2.new(0, 30, 1, 0)
plusBtn.BackgroundColor3 = Color3.fromRGB(200, 80, 160)
plusBtn.BackgroundTransparency = 0.15
plusBtn.Text = "+"
plusBtn.Font = Enum.Font.GothamBold
plusBtn.TextSize = 14
plusBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
plusBtn.Parent = sizeRow
Instance.new("UICorner", plusBtn).CornerRadius = UDim.new(0, 4)
local pStroke = Instance.new("UIStroke")
pStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
pStroke.Color = Color3.fromRGB(255, 120, 200)
pStroke.Thickness = 1.5
pStroke.Transparency = 0.2
pStroke.Parent = plusBtn

local function applyScale(s)
    s = math.clamp(math.floor(s * 10) / 10, 0.5, 2.0)
    hBox.Text = string.format("%.1f", s)
    UI_SCALE = s
    mainFrame.Size = UDim2.new(0, 230 * s, 0, 480 * s)
    mainFrame.Position = UDim2.new(0, 10, 0.5, -240 * s)
    titleLabel.TextSize = 16 * s
    for _, child in ipairs(mainFrame:GetDescendants()) do
        if child:IsA("TextButton") or child:IsA("TextLabel") or child:IsA("TextBox") then
            if child ~= titleLabel and child ~= lockButton then
                local baseSize = 10
                if child:IsA("TextLabel") then
                    baseSize = 9
                elseif child:IsA("TextBox") then
                    baseSize = 11
                end
                child.TextSize = baseSize * s
            end
        end
    end
    titleLabel.TextSize = 16 * s
    lockButton.TextSize = 18 * s
end

minusBtn.MouseButton1Click:Connect(function()
    local cur = tonumber(hBox.Text) or 1.0
    applyScale(cur - 0.1)
end)

plusBtn.MouseButton1Click:Connect(function()
    local cur = tonumber(hBox.Text) or 1.0
    applyScale(cur + 0.1)
end)

hBox.FocusLost:Connect(function()
    local cur = tonumber(hBox.Text) or 1.0
    applyScale(cur)
end)

CreateSpace(setsFrame)

local Keybinds = {
	ShowFakeRequest = Enum.KeyCode.E,
	BlockPlayer = Enum.KeyCode.B,
	AddRandomItem = Enum.KeyCode.R,
	RemoveItem = Enum.KeyCode.Y,
	RecentTrade = Enum.KeyCode.Q,
	AcceptOffer = Enum.KeyCode.T,
	HideUI = Enum.KeyCode.RightShift,
	SpinWheel = Enum.KeyCode.U,
}
local keybindButtons = {}
local keybindCapturing = false
local capturingKey = nil

local keybindLabel = Instance.new("TextLabel")
keybindLabel.Size = UDim2.new(1, 0, 0, 14)
keybindLabel.BackgroundTransparency = 1
keybindLabel.Text = "Keybinds (click to change):"
keybindLabel.Font = Enum.Font.SourceSansSemibold
keybindLabel.TextSize = 10
keybindLabel.TextColor3 = Color3.fromRGB(200, 150, 180)
keybindLabel.TextXAlignment = Enum.TextXAlignment.Left
keybindLabel.Parent = setsFrame

local actionsList = {"ShowFakeRequest", "BlockPlayer", "AddRandomItem", "RemoveItem", "RecentTrade", "AcceptOffer", "HideUI", "SpinWheel"}
local actionLabels = {
	ShowFakeRequest = "Show Fake Request",
	BlockPlayer = "Block Player",
	AddRandomItem = "Add Random Item",
	RemoveItem = "Remove Item",
	RecentTrade = "Recent Trade",
	AcceptOffer = "Accept Offer",
	HideUI = "Hide UI",
	SpinWheel = "Spin Unbox Wheel",
}

for _, action in ipairs(actionsList) do
	local row = Instance.new("Frame")
	row.Size = UDim2.new(1, 0, 0, 20)
	row.BackgroundTransparency = 1
	row.Parent = setsFrame

	local label = Instance.new("TextLabel")
	label.Size = UDim2.new(0.6, 0, 1, 0)
	label.BackgroundTransparency = 1
	label.Text = actionLabels[action]
	label.Font = Enum.Font.SourceSans
	label.TextSize = 9
	label.TextColor3 = Color3.fromRGB(240, 240, 240)
	label.TextXAlignment = Enum.TextXAlignment.Left
	label.Parent = row

	local btn = Instance.new("TextButton")
	btn.Size = UDim2.new(0.3, 0, 1, 0)
	btn.Position = UDim2.new(0.7, 0, 0, 0)
	btn.BackgroundColor3 = Color3.fromRGB(200, 80, 160)
	btn.BackgroundTransparency = 0.15
	btn.Text = tostring(Keybinds[action]):gsub("Enum.KeyCode.", "")
	btn.Font = Enum.Font.FredokaOne
	btn.TextSize = 10
	btn.TextColor3 = Color3.fromRGB(255, 255, 255)
	btn.Parent = row
	local btnCorner2 = Instance.new("UICorner")
	btnCorner2.CornerRadius = UDim.new(0, 4)
	btnCorner2.Parent = btn
	local btnStroke2 = Instance.new("UIStroke")
	btnStroke2.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	btnStroke2.Color = Color3.fromRGB(255, 120, 200)
	btnStroke2.Thickness = 1.5
	btnStroke2.Transparency = 0.2
	btnStroke2.Parent = btn

	keybindButtons[action] = btn

	btn.MouseButton1Click:Connect(function()
		if keybindCapturing then return end
		keybindCapturing = true
		capturingKey = action
		btn.Text = "..."
		btn.BackgroundColor3 = Color3.fromRGB(220, 100, 180)
	end)
end

local function keyCaptureHandler(input, gameProcessed)
	if gameProcessed then return end
	if input.UserInputType == Enum.UserInputType.Keyboard then
		local key = input.KeyCode
		if keybindCapturing and capturingKey then
			Keybinds[capturingKey] = key
			keybindCapturing = false
			local btn = keybindButtons[capturingKey]
			if btn then
				btn.Text = tostring(key):gsub("Enum.KeyCode.", "")
				btn.BackgroundColor3 = Color3.fromRGB(200, 80, 160)
			end
			capturingKey = nil
		end
	end
end
UserInputService.InputBegan:Connect(keyCaptureHandler)

local function hotkeyHandler(input, gameProcessed)
	if gameProcessed then return end
	if input.UserInputType == Enum.UserInputType.Keyboard then
		local key = input.KeyCode
		if keybindCapturing then return end
		if key == Keybinds.ShowFakeRequest then
			local name = PartnerUserBox.Text
			if name == "" or name == game.Players.LocalPlayer.Name then
				warn("[FakeRequest] Please enter a player name in Partner user field first!")
				return
			end
			ShowFakeTradeRequest(name)
		elseif key == Keybinds.BlockPlayer then
			local player = game.Players:FindFirstChild(TradeTable.Player2.Player)
			if player then BlockPlayer(player) end
		elseif key == Keybinds.AddRandomItem then
			if #weaponButtons > 0 then
				local info = weaponButtons[math.random(1, #weaponButtons)]
				OfferItemAnotherPlayer(info.entry.key, "Weapons")
			end
		elseif key == Keybinds.RemoveItem then
			RemoveItemAnotherPlayer()
		elseif key == Keybinds.RecentTrade then
			RecentTradeAction()
		elseif key == Keybinds.AcceptOffer then
			AcceptOfferAction()
		elseif key == Keybinds.HideUI then
			HideUIAction()
		elseif key == Keybinds.SpinWheel then
			task.spawn(openSpinWheel)
		end
	end
end
UserInputService.InputBegan:Connect(hotkeyHandler)

CreateSpace(setsFrame)
local resetKeybindsBtn = CreateButton(setsFrame, "Reset keybinds", function()
	Keybinds.ShowFakeRequest = Enum.KeyCode.E
	Keybinds.BlockPlayer = Enum.KeyCode.B
	Keybinds.AddRandomItem = Enum.KeyCode.R
	Keybinds.RemoveItem = Enum.KeyCode.Y
	Keybinds.RecentTrade = Enum.KeyCode.Q
	Keybinds.AcceptOffer = Enum.KeyCode.T
	Keybinds.HideUI = Enum.KeyCode.RightShift
	Keybinds.SpinWheel = Enum.KeyCode.U
	for action, btn in pairs(keybindButtons) do
		btn.Text = tostring(Keybinds[action]):gsub("Enum.KeyCode.", "")
		btn.BackgroundColor3 = Color3.fromRGB(200, 80, 160)
	end
end)
CreateSpace(setsFrame)

-- Экспорт функций
_G.AddFakePlayerToLeaderboard = AddFakePlayerToLeaderboard
_G.AddMultipleFakePlayers = AddMultipleFakePlayers
_G.RemoveFakePlayersFromLeaderboard = RemoveFakePlayersFromLeaderboard
_G.GetFakePlayersList = GetFakePlayersList
_G.SpawnFakePlayer = SpawnFakePlayer
_G.DeleteAllFakePlayers = DeleteAllFakePlayers

print("Script loaded successfully")
end