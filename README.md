if not game:WaitForChild("ReplicatedStorage", 9e9):WaitForChild("Loaded", 9e9).Value then
	game:WaitForChild("ReplicatedStorage", 9e9):WaitForChild("Loaded", 9e9).Changed:Wait()
end
if not game.Players.LocalPlayer:WaitForChild("dataInitialized", 9e9).Value then
	game.Players.LocalPlayer:WaitForChild("dataInitialized", 9e9).Changed:Wait()
end
local t1 = tick()
local client = game.Players.LocalPlayer
local hitbox = client.Character:WaitForChild("hitbox", 9e9)
local entities = workspace.placeFolders.entityManifestCollection
local network_module = require(game.ReplicatedStorage.modules.network)
local charactercontainer = network_module:invoke("getMyClientCharacterContainer")
local equipped = network_module:invoke("getCurrentlyEquippedForRenderCharacter", charactercontainer.entity)
local MarketplaceService = game:GetService("MarketplaceService")
local RunService = game:GetService("RunService")
local playerRequest = game.ReplicatedStorage.playerRequest
local signal = game.ReplicatedStorage.signal
local modules = game.ReplicatedStorage.modules
local ItemData = game.ReplicatedStorage.itemData
local CollectionService = game:GetService("CollectionService")
--[[
local func = filtergc("function", { Name = "flagPlayer" }, true)
if func then
	hookfunction(func, function()
		return
	end)
end
--]]
local Teleports = {}
local ActualTeleports = {}
-- Network
local network = {}
do
	local _network = require(modules.network)
	function network:fireServer(...)
		return signal:fireServer(...)
	end
	function network:invokeServer(...)
		return _network:InvokeServer(...)
	end
	function network:invoke(...)
		return _network:invoke(...)
	end
	function network:fire(...)
		return _network:fire(...)
	end
end
local Utilities = {}
do
	function Utilities.HittableEntities()
		local list = {}
		local hb = client.Character:FindFirstChild("hitbox")
		for _, mob in next, entities:GetChildren() do
			if
				mob.Name == "hitbox"
				or mob.ClassName == "Model"
				or mob:FindFirstChild("pet")
				or mob.Name == "Lobster"
			then
				continue
			end
			local health, part = mob:FindFirstChild("health"), mob
			if
				health
				and hb
				and health.Value > 0
				and (part.Position - hb.Position).Magnitude <= Options.KillauraRange.Value
			then
				table.insert(list, mob)
			end
		end
		return list
	end
	function Utilities.Attack(Entities)
		local hb = client.Character:FindFirstChild("hitbox")
		if not hb then
			return
		end
		--network:fireServer("fireEvent", "playerWillUseBasicAttack", client)
		local weptype = equipped["1"].baseData.equipmentType
		for i = 1, 3 do
			network:fireServer(
				"replicatePlayerAnimationSequence",
				weptype .. "Animations",
				"strike" .. tostring(i),
				{ ["attackSpeed"] = 0 }
			)
			for _, entity in next, Entities do
				local health = entity:FindFirstChild("health")
				if health and health.Value > 0 then
					network:fireServer("playerRequest_damageEntity_batch", { { entity, entity.Position, "equipment" } })
				end
			end
		end
	end
local args = {
    [1] = "playerWillUseBasicAttack",
    [2] = game:GetService("Players").LocalPlayer
}
game:GetService("ReplicatedStorage"):WaitForChild("network"):WaitForChild("RemoteEvent"):WaitForChild("fireEvent"):FireServer(unpack(args))
	function Utilities.AttackNormal(Entities)
		local hb = client.Character:FindFirstChild("hitbox")
		if not hb then
			return
		end
		local weptype = equipped["1"].baseData.equipmentType
		game.ReplicatedStorage.network.RemoteEvent.replicatePlayerAnimationSequence:FireServer(
			weptype .. "Animations",
			"strike" .. tostring(math.random(1, 3)),
			{}
		)
		for _, entity in next, Entities do
			local health = entity:FindFirstChild("health")
			if health and health.Value > 0 then
				game.ReplicatedStorage.network.RemoteEvent.playerRequest_damageEntity_batch:FireServer({
					{ entity, entity.Position, "equipment" },
				})
			end
		end
	end
	function Utilities.getClosestEntity()
		local closest, maxDist = nil, 9e9
		for _, mob in next, entities:GetChildren() do
			if
				mob.Name == "hitbox"
				or mob.ClassName == "Model"
				or mob:FindFirstChild("pet")
				or mob.Name == "Hitbox"
				or mob.Name == "Lobster"
			then
				continue
			end
			local health, part = mob:FindFirstChild("health"), mob
			if health and health.Value > 0 and (part.Position - hitbox.Position).Magnitude <= maxDist then
				closest = mob
				maxDist = (part.Position - hitbox.Position).Magnitude
			end
		end
		return closest
	end
	function Utilities.getDrops()
		local list = {}
		for _, v in next, workspace.placeFolders.items:GetChildren() do
			local owners = v:FindFirstChild("owners")
			if owners and owners:FindFirstChild(client.UserId) or v:GetAttribute("anyoneCanPickupItem") then
				table.insert(list, v)
			end
		end
		return list
	end
	function Utilities.isClientAlive()
		if client.Character then
			return (client.Character:FindFirstChild("hitbox") ~= nil)
		end
		return false
	end
	function Utilities.getAutofarmTargets()
		local closest, maxDist = nil, 9e9
		for _, v in next, entities:GetChildren() do
			if
				v.Name == "hitbox"
				or v.ClassName == "Model"
				or v:FindFirstChild("pet")
				or v.Name == "Hitbox"
				or v.Name == "Lobster"
			then
				continue
			end
			if v:FindFirstChild("specialName") and Options.AutofarmIgnore.Value[v.specialName.Value] then
				continue
			end
			if
				not v:FindFirstChild("specialName")
				and v:FindFirstChild("entityId")
				and Options.AutofarmIgnore.Value[v.entityId.Value]
			then
				continue
			end
			local health, part = v:FindFirstChild("health"), v
			if health and health.Value > 0 and (part.Position - hitbox.Position).Magnitude <= maxDist then
				closest = v
				maxDist = (part.Position - hitbox.Position).Magnitude
			end
		end
		return closest
	end
end
for _, v in next, workspace:GetDescendants() do
	if v.Name == "teleportDestination" then
		if v then
			pcall(function()
				local Name = MarketplaceService:GetProductInfo(v.Value).Name
				if not table.find(Teleports, Name) then
					table.insert(Teleports, Name)
					ActualTeleports[Name] = v.Parent
				end
			end)
		end
	end
end
local LoadFromGithub, LoadFromFile
do
	local function LoadFromUrl(url)
		local succ, res = pcall(game.HttpGet, game, url)
		if not succ then
			return false, string.format("HttpError: %*", res)
		end
		local fn, err = loadstring(res, "@" .. url)
		if not fn then
			return false, string.format("LoadError: %*", err)
		end
		local results = table.pack(pcall(fn))
		if results[1] == false then
			return false, string.format("InitError: %*", results[2])
		end
		return true, results
	end
	function LoadFromGithub(props)
		local url = table.concat({
			"https://raw.githubusercontent.com",
			props.owner,
			props.repo,
			props.branch or "main",
			props.path,
		}, "/")
		local loaded, results = LoadFromUrl(url)
		if not loaded then
			return error(string.format("Failed to load %* with err: %*", url, results))
		end
		return unpack(results, 2, results.n)
	end
	function LoadFromFile(path)
		local closure, err = loadstring(readfile(path), "@" .. path)
		assert(closure, err)
		return closure()
	end
end
local repo = "https://raw.githubusercontent.com/wally-rblx/LinoriaLib/main/"
local Library = LoadFromGithub({ owner = "violin-suzutsuki", repo = "LinoriaLib", path = "Library.lua" })
local ThemeManager =
	LoadFromGithub({ owner = "violin-suzutsuki", repo = "LinoriaLib", path = "addons/ThemeManager.lua" })
local SaveManager = LoadFromGithub({ owner = "violin-suzutsuki", repo = "LinoriaLib", path = "addons/SaveManager.lua" })
do
	local Window = Library:CreateWindow({ Title = "Vesteria", Center = true, AutoShow = true })
	local Tabs = {}
	local Groups = {}
	local SubTabs = {}
	Tabs.Main = Window:AddTab("Main")
	Tabs.Visuals = Window:AddTab("Visuals")
	Tabs["UI Settings"] = Window:AddTab("UI Settings")
	-- Main tab
	Groups.Autofarm = Tabs.Main:AddLeftGroupbox("Autofarm")
	Groups.Killaura = Tabs.Main:AddLeftGroupbox("Killaura")
	Groups.AutoCollect = Tabs.Main:AddLeftGroupbox("Auto Collect")
	Groups.Godmode = Tabs.Main:AddLeftGroupbox("Godmode")
	Groups.Teleports = Tabs.Main:AddLeftGroupbox("Teleports")
	Groups.Misc = Tabs.Main:AddLeftGroupbox("Misc")
	Groups.AutoStats = Tabs.Main:AddRightGroupbox("Auto Stats")
	Groups.AutoSell = Tabs.Main:AddRightGroupbox("Auto Sell")
	Groups.AutoSell = Tabs.Main:AddRightGroupbox("Auto Sell")
	if game.PlaceId == 3207211233 then
		Groups.AutoSQR = Tabs.Main:AddRightGroupbox("SQR")
	end
	-- Ui Settings
	Groups["UI Settings"] = Tabs["UI Settings"]:AddRightGroupbox("UI Settings")
	Groups["Theme Settings"] = Tabs["UI Settings"]:AddLeftGroupbox("Theme Settings")
	-- Auto farm tab
	Groups.Autofarm
		:AddToggle("Autofarm", { Text = "Enabled" })
		:AddKeyPicker("AutofarmBind", { Text = "Autofarm", Default = "None", Mode = "Toggle", SyncToggleState = true })
	Groups.Autofarm:AddDropdown("AutofarmIgnore", { Text = "Ignore Mobs", Values = {}, AllowNull = true, Multi = true })
	Groups.Autofarm:AddSlider(
		"AutofarmOffset", -- round it like 1 2 3 4 5 6 7 8 9 10
		{ Text = "Offset", Min = 0, Max = 15, Default = 5, Rounding = 0, Compact = true }
	)
	-- Main toggles
	Groups.Killaura
		:AddToggle("Killaura", { Text = "Enabled" })
		:AddKeyPicker("KillauraBind", { Text = "Killaura", Default = "None", Mode = "Toggle", SyncToggleState = true })
	Groups.Killaura:AddSlider(
		"KillauraRange",
		{ Text = "Range", Suffix = " studs", Min = 1, Max = 15, Default = 15, Rounding = 0, Compact = true }
	)
	Groups.Killaura:AddDropdown("KillauraMode", { Text = "Mode", Values = { "Normal", "Wally" }, Default = 1 })
	-- Godmode Button
	Groups.Godmode:AddButton({
   	 Text = 'Activate God Mode',
   	 Func = function()
     	   local REPLICATED_STORAGE = cloneref(game:GetService("ReplicatedStorage"))
      	  local remote = REPLICATED_STORAGE.network.RemoteEvent.playerRequest_damageEntity
       	 remote:Destroy()
  	  end,
   	 Tooltip = 'Click to activate God Mode'
	})
	-- Auto collect
	Groups.AutoCollect:AddToggle("AutoCollect", { Text = "Enabled" })
	-- Teleports
	Groups.Teleports:AddDropdown("TeleportSelected", { Text = "Selected Place", Values = Teleports, AllowNull = true })
	Groups.Teleports:AddButton("Teleport", function()
		client.Character:PivotTo(ActualTeleports[Options.TeleportSelected.Value].CFrame)
		game.ReplicatedStorage.network.RemoteFunction.playerRequest_useTeleporter:InvokeServer(
			ActualTeleports[Options.TeleportSelected.Value]
		)
	end)
	Groups.AutoStats:AddToggle("AutoSTR", { Text = "STR" })
	Groups.AutoStats:AddToggle("AutoVIT", { Text = "VIT" })
	Groups.AutoStats:AddToggle("AutoINT", { Text = "INT" })
	Groups.AutoStats:AddToggle("AutoDEX", { Text = "DEX" })
	-- Auto SQR
	if game.PlaceId == 3207211233 then
		Groups.AutoSQR:AddToggle("AutoSQR", { Text = "Enabled" })
	end
	-- Ui Settings
	Groups["UI Settings"]:AddButton("Unload", function()
		Library:Unload()
	end)
	Groups["UI Settings"]
		:AddLabel("Menu bind")
		:AddKeyPicker("MenuBind", { Default = "RightShift", NoUI = true, Text = "Menu bind" })
	Library.ToggleKeybind = Options.MenuBind
	ThemeManager:SetLibrary(Library)
	SaveManager:SetLibrary(Library)
	SaveManager:IgnoreThemeSettings()
	SaveManager:SetIgnoreIndexes({ "MenuBind" })
	ThemeManager:SetFolder("VesteriaOHYEYE")
	SaveManager:SetFolder("VesteriaOHYEYE/Settings")
	SaveManager:LoadAutoloadConfig()
	SaveManager:BuildConfigSection(Tabs["UI Settings"])
	ThemeManager:ApplyToTab(Tabs["UI Settings"])
end
local function getTotalAmount()
	local total = 0
	for _, entity in next, entities:GetChildren() do
		if CollectionService:HasTag(entity, "monster") then
			local pet, hb = entity:FindFirstChild("pet"), entity:FindFirstChild("hitbox")
			if pet or hb then
				continue
			end
			local health = entity:FindFirstChild("health") and entity.health.Value
			if health and health <= 0 then
				continue
			end
			total += 1
		end
	end
	return total
end
local function repairBridge()
	for _, v in next, workspace:GetChildren() do
		local interact, rare = v:FindFirstChild("interactScript"), v:FindFirstChild("Rare")
		if interact and rare then
			game.Players.LocalPlayer.Character:PivotTo(v.CFrame)
			task.wait(0.5)
			network:invokeServer("playerRequest_pickupWheel", v)
			task.wait(0.5)
		end
	end
	client.Character:PivotTo(workspace.BridgeWheel.CFrame)
	keyclick(Enum.KeyCode.C)
end
local function nearestEgg()
	local closest, maxDist = nil, 9e9
	for _, v in next, entities:GetChildren() do
		local name = v:FindFirstChild("specialName") and v.specialName.Value
		if name == "Spider Egg Pile" then
			local health = v:FindFirstChild("health")
			if health and health.Value > 0 then
				local dist = (v.Position - client.Character.PrimaryPart.Position).magnitude
				if dist < maxDist then
					maxDist = dist
					closest = v
				end
			end
		end
	end
	return closest
end
local function nearestHealing()
	local nearest, maxDist = nil, 9e9
	for _, v in next, workspace.placeFolders.entityManifestCollection:GetChildren() do
		if v:FindFirstChild("specialName") and v.specialName.Value == "Shield Generator" then
			if v:FindFirstChild("health") and v.health.Value > 0 then
				local dist = (v.Position - client.Character.PrimaryPart.Position).magnitude
				if dist < maxDist then
					maxDist = dist
					nearest = v
				end
			end
		end
	end
	return nearest
end
-- auto sqr
if game.PlaceId == 3207211233 then
	task.spawn(function()
		while true do
			task.wait()
			if Toggles.AutoSQR and Toggles.AutoSQR.Value then
				pcall(function()
					if game.ReplicatedStorage.dungeon:GetAttribute("difficulty") == 1 then
						local currentRoom = game.ReplicatedStorage.currentRoom
						Toggles.Godmode:SetValue(true)
						if currentRoom.Value == 0 then
							client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
							repeat
								network:invokeServer(
									"playerRequest_advanceToNextRoom",
									workspace[tostring(currentRoom.Value)].Sender
								)
							until currentRoom.Value ~= 0 or not Toggles.AutoSQR.Value
						end
						if currentRoom.Value == 1 then
							Toggles.Autofarm:SetValue(true)
							Options.KillauraMode:SetValue("Wally")
							Toggles.Killaura:SetValue(true)
							repeat
								task.wait(1)
							until getTotalAmount() == 0 or not Toggles.AutoSQR.Value
							Toggles.Autofarm:SetValue(false)
							client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
							repeat
								task.wait(1)
								network:invokeServer(
									"playerRequest_advanceToNextRoom",
									workspace[tostring(currentRoom.Value)].Sender
								)
							until currentRoom.Value ~= 1 or not Toggles.AutoSQR.Value
						end
						if currentRoom.Value == 2 then
							Library:Notify("We builderman the bridge now", 5)
							repairBridge()
							keyclick(Enum.KeyCode.C)
							task.wait(6) -- wait for bridge to be repaired
							client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
							repeat
								task.wait(1)
								network:invokeServer(
									"playerRequest_advanceToNextRoom",
									workspace[tostring(currentRoom.Value)].Sender
								)
							until currentRoom.Value ~= 2 or not Toggles.AutoSQR.Value
						end
						if currentRoom.Value == 3 then
							Toggles.Autofarm:SetValue(true)
							repeat
								task.wait(1)
							until getTotalAmount() == 0 or not Toggles.AutoSQR.Value
							Toggles.Autofarm:SetValue(false)
							client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
							repeat
								task.wait(1)
								network:invokeServer(
									"playerRequest_advanceToNextRoom",
									workspace[tostring(currentRoom.Value)].Sender
								)
							until currentRoom.Value ~= 3 or not Toggles.AutoSQR.Value
						end
						if currentRoom.Value == 4 then
							client.Character:PivotTo(CFrame.new(121.238312, 301.576416, 414))
							task.wait(4)
							client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
							repeat
								task.wait(1)
								network:invokeServer(
									"playerRequest_advanceToNextRoom",
									workspace[tostring(currentRoom.Value)].Sender
								)
							until currentRoom.Value ~= 4 or not Toggles.AutoSQR.Value
						end
						if currentRoom.Value == 5 then
							Library:Notify("aw shii fuck this egg room", 5)
							network:invoke("setCharacterArrested", true)
							repeat
								task.wait()
								local egg = nearestEgg()
								if egg then
									client.Character:PivotTo(egg.CFrame * CFrame.new(0, 5, 0))
								end
							until egg == nil or not Toggles.AutoSQR.Value
							if nearestEgg() == nil then
								network:invoke("setCharacterArrested", false)
								client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
								repeat
									task.wait(1)
									network:invokeServer(
										"playerRequest_advanceToNextRoom",
										workspace[tostring(currentRoom.Value)].Sender
									)
								until currentRoom.Value ~= 5 or not Toggles.AutoSQR.Value
							end
						end
						if currentRoom.Value == 8 then
							repeat
								task.wait()
								network:invoke("setCharacterArrested", true)
								local heal = nearestHealing()
								if heal then
									client.Character.PrimaryPart.CanCollide = false
									client.Character.PrimaryPart.Velocity = Vector3.zero
									client.Character:PivotTo(heal.CFrame * CFrame.new(0, 8, 0))
								end
							until heal == nil or not Toggles.AutoSQR.Value
						end
						if nearestHealing() == nil then
							network:invoke("setCharacterArrested", false)
							Toggles.Autofarm:SetValue(true)
							repeat
								task.wait()
							until Utilities.getAutofarmTargets() == nil or not Toggles.AutoSQR.Value
							if Utilities.getAutofarmTargets() == nil then
								Toggles.Autofarm:SetValue(false)
								Toggles.AutoSQR:SetValue(false)
								client.Character.PrimaryPart.CanCollide = true
								network:invoke("setCharacterArrested", false)
								-- if the dungeon difficulty is 2, we wanna print ("not yet supported")
							end
						end
					else
						if game.ReplicatedStorage.dungeon:GetAttribute("difficulty") == 2 then
							local currentRoom = game.ReplicatedStorage.currentRoom
							Toggles.Godmode:SetValue(true)
							if currentRoom.Value == 0 then
								client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
								repeat
									network:invokeServer(
										"playerRequest_advanceToNextRoom",
										workspace[tostring(currentRoom.Value)].Sender
									)
								until currentRoom.Value ~= 0 or not Toggles.AutoSQR.Value
								shared.startTick = tick()
							end
							if currentRoom.Value == 1 then
								Toggles.Autofarm:SetValue(true)
								Options.KillauraMode:SetValue("Wally")
								Toggles.Killaura:SetValue(true)
								repeat
									task.wait(1)
								until getTotalAmount() == 0 or not Toggles.AutoSQR.Value
								Toggles.Autofarm:SetValue(false)
								client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
								repeat
									task.wait(1)
									network:invokeServer(
										"playerRequest_advanceToNextRoom",
										workspace[tostring(currentRoom.Value)].Sender
									)
								until currentRoom.Value ~= 1 or not Toggles.AutoSQR.Value
							end
							if currentRoom.Value == 2 then
								Library:Notify("We builderman the bridge now", 5)
								repairBridge()
								keyclick(Enum.KeyCode.C)
								task.wait(6) -- wait for bridge to be repaired
								client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
								repeat
									task.wait(1)
									network:invokeServer(
										"playerRequest_advanceToNextRoom",
										workspace[tostring(currentRoom.Value)].Sender
									)
								until currentRoom.Value ~= 2 or not Toggles.AutoSQR.Value
							end
							if currentRoom.Value == 3 then
								Toggles.Autofarm:SetValue(true)
								repeat
									task.wait(1)
								until getTotalAmount() == 0 or not Toggles.AutoSQR.Value
								Toggles.Autofarm:SetValue(false)
								client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
								repeat
									task.wait(1)
									network:invokeServer(
										"playerRequest_advanceToNextRoom",
										workspace[tostring(currentRoom.Value)].Sender
									)
								until currentRoom.Value ~= 3 or not Toggles.AutoSQR.Value
							end
							if currentRoom.Value == 4 then
								client.Character:PivotTo(CFrame.new(121.238312, 301.576416, 414))
								task.wait(4)
								client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
								repeat
									task.wait(1)
									network:invokeServer(
										"playerRequest_advanceToNextRoom",
										workspace[tostring(currentRoom.Value)].Sender
									)
								until currentRoom.Value ~= 4 or not Toggles.AutoSQR.Value
							end
							if currentRoom.Value == 5 then
								Library:Notify("aw shii fuck this egg room", 5)
								network:invoke("setCharacterArrested", true)
								repeat
									task.wait()
									local egg = nearestEgg()
									if egg then
										client.Character:PivotTo(egg.CFrame * CFrame.new(0, 5, 0))
									end
								until egg == nil or not Toggles.AutoSQR.Value
								if nearestEgg() == nil then
									network:invoke("setCharacterArrested", false)
									client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
									repeat
										task.wait(1)
										network:invokeServer(
											"playerRequest_advanceToNextRoom",
											workspace[tostring(currentRoom.Value)].Sender
										)
									until currentRoom.Value ~= 5 or not Toggles.AutoSQR.Value
								end
							end
							if currentRoom.Value == 6 then
								Toggles.Autofarm:SetValue(true)
								repeat
									task.wait(1)
								until getTotalAmount() == 0 or not Toggles.AutoSQR.Value
								Toggles.Autofarm:SetValue(false)
								client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
								repeat
									task.wait(1)
									network:invokeServer(
										"playerRequest_advanceToNextRoom",
										workspace[tostring(currentRoom.Value)].Sender
									)
								until currentRoom.Value ~= 6 or not Toggles.AutoSQR.Value
							end
							if currentRoom.Value == 7 then
								-- we need to exit asap
								client.Character:PivotTo(CFrame.new(-404.346, 220.082, 1158.29))
								task.wait(2)
								client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
								repeat
									task.wait(1)
									network:invokeServer(
										"playerRequest_advanceToNextRoom",
										workspace[tostring(currentRoom.Value)].Sender
									)
								until currentRoom.Value ~= 7 or not Toggles.AutoSQR.Value
							end
							if currentRoom.Value == 9 then
								task.wait(7.5)
								Toggles.Autofarm:SetValue(true)
								task.wait(5)
								repeat
									task.wait()
								until Utilities.getAutofarmTargets() == nil or not Toggles.AutoSQR.Value
								task.wait(10)
								if Utilities.getAutofarmTargets() == nil or getTotalAmount() <= 3 then
									Toggles.Autofarm:SetValue(false)
									network:invoke("setCharacterArrested", false)
									client.Character.PrimaryPart.CanCollide = true
									client.Character.PrimaryPart.Velocity = Vector3.zero
								end
							end
						else
							if game.ReplicatedStorage.dungeon:GetAttribute("difficulty") == 3 then
								local currentRoom = game.ReplicatedStorage.currentRoom
								Toggles.Godmode:SetValue(true)
								if currentRoom.Value == 0 then
									Library:Notify("Starting SQR", 5)
									client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
									repeat
										network:invokeServer(
											"playerRequest_advanceToNextRoom",
											workspace[tostring(currentRoom.Value)].Sender
										)
									until currentRoom.Value ~= 0 or not Toggles.AutoSQR.Value
									shared.startTick = tick()
								end
								if currentRoom.Value == 1 then
									Library:Notify("Room 1", 5)
									Toggles.Autofarm:SetValue(true)
									Options.KillauraMode:SetValue("Wally")
									Toggles.Killaura:SetValue(true)
									repeat
										task.wait(1)
									until getTotalAmount() == 0 or not Toggles.AutoSQR.Value
									Toggles.Autofarm:SetValue(false)
									client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
									repeat
										task.wait(1)
										network:invokeServer(
											"playerRequest_advanceToNextRoom",
											workspace[tostring(currentRoom.Value)].Sender
										)
									until currentRoom.Value ~= 1 or not Toggles.AutoSQR.Value
								end
								if currentRoom.Value == 2 then
									Library:Notify("We builderman the bridge now", 5)
									repairBridge()
									keyclick(Enum.KeyCode.C)
									task.wait(10) -- wait for bridge to be repaired
									client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
									repeat
										task.wait(1)
										network:invokeServer(
											"playerRequest_advanceToNextRoom",
											workspace[tostring(currentRoom.Value)].Sender
										)
									until currentRoom.Value ~= 2 or not Toggles.AutoSQR.Value
								end
								if currentRoom.Value == 3 then
									Library:Notify("Room 3", 5)
									Toggles.Autofarm:SetValue(true)
									repeat
										task.wait(1)
									until getTotalAmount() == 0 or not Toggles.AutoSQR.Value
									Toggles.Autofarm:SetValue(false)
									client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
									repeat
										task.wait(1)
										network:invokeServer(
											"playerRequest_advanceToNextRoom",
											workspace[tostring(currentRoom.Value)].Sender
										)
									until currentRoom.Value ~= 3 or not Toggles.AutoSQR.Value
								end
								if currentRoom.Value == 4 then
									Library:Notify("Room 4", 5)
									client.Character:PivotTo(CFrame.new(121.238312, 301.576416, 414))
									task.wait(4)
									client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
									repeat
										task.wait(1)
										network:invokeServer(
											"playerRequest_advanceToNextRoom",
											workspace[tostring(currentRoom.Value)].Sender
										)
									until currentRoom.Value ~= 4 or not Toggles.AutoSQR.Value
								end
								if currentRoom.Value == 5 then
									Library:Notify("Room 5", 5)
									Library:Notify("aw shii fuck this egg room", 5)
									network:invoke("setCharacterArrested", true)
									repeat
										task.wait()
										local egg = nearestEgg()
										if egg then
											client.Character:PivotTo(egg.CFrame * CFrame.new(0, 5, 0))
										end
									until egg == nil or not Toggles.AutoSQR.Value
									if nearestEgg() == nil then
										network:invoke("setCharacterArrested", false)
										client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
										repeat
											task.wait(1)
											network:invokeServer(
												"playerRequest_advanceToNextRoom",
												workspace[tostring(currentRoom.Value)].Sender
											)
										until currentRoom.Value ~= 5 or not Toggles.AutoSQR.Value
									end
								end
								if currentRoom.Value == 6 then
									Library:Notify("Room 6", 5)
									Toggles.Autofarm:SetValue(true)
									repeat
										task.wait(1)
									until getTotalAmount() == 0 or not Toggles.AutoSQR.Value
									Toggles.Autofarm:SetValue(false)
									client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
									repeat
										task.wait(1)
										network:invokeServer(
											"playerRequest_advanceToNextRoom",
											workspace[tostring(currentRoom.Value)].Sender
										)
									until currentRoom.Value ~= 6 or not Toggles.AutoSQR.Value
								end
								if currentRoom.Value == 7 then
									Library:Notify("Room 7", 5)
									-- we need to exit asap
									client.Character:PivotTo(CFrame.new(-404.346, 220.082, 1158.29))
									task.wait(2)
									client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
									repeat
										task.wait(1)
										network:invokeServer(
											"playerRequest_advanceToNextRoom",
											workspace[tostring(currentRoom.Value)].Sender
										)
									until currentRoom.Value ~= 7 or not Toggles.AutoSQR.Value
								end
								if currentRoom.Value == 8 then
									Library:Notify("Room 8", 5)
									repeat
										task.wait()
										network:invoke("setCharacterArrested", true)
										local heal = nearestHealing()
										if heal then
											client.Character.PrimaryPart.CanCollide = false
											client.Character.PrimaryPart.Velocity = Vector3.zero
											client.Character:PivotTo(heal.CFrame * CFrame.new(0, 8, 0))
										end
									until heal == nil or not Toggles.AutoSQR.Value
								end
								if nearestHealing() == nil then
									network:invoke("setCharacterArrested", false)
									Toggles.Autofarm:SetValue(true)
									repeat
										task.wait()
									until Utilities.getAutofarmTargets() == nil or not Toggles.AutoSQR.Value
									if Utilities.getAutofarmTargets() == nil then
										Toggles.Autofarm:SetValue(false)
										client.Character.PrimaryPart.CanCollide = true
										network:invoke("setCharacterArrested", false)
										task.wait(1)
										client.Character:PivotTo(workspace[tostring(currentRoom.Value)].Sender.CFrame)
										repeat
											task.wait(1)
											network:invokeServer(
												"playerRequest_advanceToNextRoom",
												workspace[tostring(currentRoom.Value)].Sender
											)
										until currentRoom.Value ~= 8 or not Toggles.AutoSQR.Value
									end
								end
								if currentRoom.Value == 9 then
									Library:Notify("Room 9", 5)
									task.wait(7.5)
									Toggles.Autofarm:SetValue(true)
									task.wait(5)
									repeat
										task.wait()
									until Utilities.getAutofarmTargets() == nil or not Toggles.AutoSQR.Value
								end
								if Utilities.getAutofarmTargets() == nil or getTotalAmount() <= 3 then
									network:invoke("setCharacterArrested", false)
									client.Character.PrimaryPart.CanCollide = true
									client.Character.PrimaryPart.Velocity = Vector3.zero
									task.wait(1)
									Toggles.Autofarm:SetValue(false)
									Toggles.AutoSQR:SetValue(false)
									Library:Notify("AutoSQR Finished", 5)
								end
							end
						end
					end
				end)
			end
		end
	end)
end
local a = Options.AutofarmIgnore
a.Values = {}
--[[
table.insert(a.Values, "None")
a:SetValues()
a:Display()
--]]
for _, v in next, workspace.placeFolders.entityManifestCollection:GetChildren() do
	local specialName = v:FindFirstChild("specialName")
	if
		specialName
		and not table.find(a.Values, specialName.Value)
		and not v:FindFirstChild("pet")
		and not v:FindFirstChild("tombstone")
	then
		table.insert(a.Values, specialName.Value)
		a:SetValues()
	else
		local entityId = v:FindFirstChild("entityId")
		if
			entityId
			and not table.find(a.Values, entityId.Value)
			and not v:FindFirstChild("pet")
			and not v:FindFirstChild("tombstone")
		then
			table.insert(a.Values, entityId.Value)
			a:SetValues()
		end
	end
end
entities.ChildAdded:Connect(function(child)
	if CollectionService:HasTag(child, "giantEnemy") then
		Library:Notify("yo.. Giant " .. child.Name .. " is here *gulp*", 5)
	end
end)
entities.ChildAdded:Connect(function(child)
	if CollectionService:HasTag(child, "monster") then
		if child.Name == "Hitbox" or child.Name == "hitbox" or child.Name:lower():find("pet") then
			return
		end
		if not child:FindFirstChild("pet") and not child:FindFirstChild("tombstone") then
			local name = child:FindFirstChild("specialName") and child.specialName.Value or child.Name
			if name and not table.find(Options.AutofarmIgnore.Values, name) then
				Library:Notify(name .. " has spawned!", 5)
				table.insert(Options.AutofarmIgnore.Values, name)
				Options.AutofarmIgnore:SetValues()
			else
				local id = child:FindFirstChild("entityId") and child.entityId.Value or child.Name
				if id and not table.find(Options.AutofarmIgnore.Values, id) then
					Library:Notify(id .. " has spawned!", 5)
					table.insert(Options.AutofarmIgnore.Values, id)
					Options.AutofarmIgnore:SetValues()
				end
			end
		end
	end
end)
Options.KillauraMode:OnChanged(function()
	if Options.KillauraMode.Value == "Wally" then
		Library:Notify("wally mode can lag you out", 5)
	end
end)
--[[
local function AutoSellItem(ItemName)
	local Item = ItemData:FindFirstChild(ItemName)
	local ItemModule = Item and require(Item)
	if not ItemModule then
		return
	end
	for _, v in next, network:invoke("getCacheValueByNameTag", "inventory") do
		if
			Options.CommonSell.Value[ItemModule.name]
			or Options.RareSell.Value[ItemModule.name]
			or Options.LegendarySell.Value[ItemModule.name]
		then
			local invId, id = v.id, ItemModule.id
			if invId == id then
				local amount
				-- if theres more than 1 stack, plus them all together
				if ItemModule.canStack then
					amount = v.stacks
				else
					for _, duplicates in next, network:invoke("getCacheValueByNameTag", "inventory") do
						if duplicates.id == id then
							amount = (amount or 0) + 1
						end
					end
				end
				Library:Notify(
					string.format(
						"Sold: %s x%s for %s",
						ItemModule.name,
						amount,
						addSuffix(ItemModule.sellValue * amount)
					),
					5
				)
				game.ReplicatedStorage.playerRequest:invokeServer(
					"playerRequest_sellItemToShop",
					{ ["id"] = ItemModule.id, ["inventorySlotDataPosition"] = v.position },
					v.stacks
				)
			end
		end
	end
end
--]]
local Functions = {}
do
	function Functions.Killaura()
		while true do
			task.wait()
			if (Toggles.Killaura and Toggles.Killaura.Value) and Options.KillauraBind:GetState() then
				local alive = Utilities.isClientAlive()
				local targets = nil
				if alive and client.Character:FindFirstChild("hitbox") then
					targets = Utilities.HittableEntities()
				end
				-- Check if the hitbox exists before executing the rest of the code
				if client.Character:FindFirstChild("hitbox") then
					if #targets == 0 then
						continue
					end
					if Options.KillauraMode.Value == "Normal" then
						Utilities.AttackNormal(targets)
					else
						Utilities.Attack(targets)
					end
				end
			end
		end
	end
	function Functions.AutoLoot()
		while true do
			task.wait()
			if Toggles.AutoCollect and Toggles.AutoCollect.Value then
				local drops = Utilities.getDrops()
				if #drops == 0 then
					continue
				end
				for _, target in drops do
					game.ReplicatedStorage.network.RemoteFunction.pickUpItemRequest:InvokeServer(target)
				end
			end
		end
	end
	function Functions.Autofarm()
		while true do
			task.wait()
			if (Toggles.Autofarm and Toggles.Autofarm.Value) and Options.AutofarmBind:GetState() then
				local closest = Utilities.getAutofarmTargets()
				if closest then
					client.Character:PivotTo(closest.CFrame * CFrame.new(0, Options.AutofarmOffset.Value, 0))
				end
			end
		end
	end
	local net = require(game:GetService("ReplicatedStorage").modules.network)
	function Functions.AutoStats()
		while true do
			task.wait()
			if Toggles.AutoSTR and Toggles.AutoSTR.Value then
				net:InvokeServer("playerRequest_incrementPlayerStatPointsByStatName", "str", 1)
			end
			if Toggles.AutoVIT and Toggles.AutoVIT.Value then
				net:InvokeServer("playerRequest_incrementPlayerStatPointsByStatName", "vit", 1)
			end
			if Toggles.AutoINT and Toggles.AutoINT.Value then
				net:InvokeServer("playerRequest_incrementPlayerStatPointsByStatName", "int", 1)
			end
			if Toggles.AutoDEX and Toggles.AutoDEX.Value then
				net:InvokeServer("playerRequest_incrementPlayerStatPointsByStatName", "dex", 1)
			end
		end
	end
end
RunService.Stepped:Connect(function()
	if Toggles.Autofarm and Toggles.Autofarm.Value then
		pcall(function()
			local closest = Utilities.getClosestEntity()
			if closest then
				network:invoke("setCharacterArrested", true)
				client.Character.PrimaryPart.CanCollide = false
				client.Character.PrimaryPart.Velocity = Vector3.zero
				client.Character.hitbox:FindFirstChild("hitboxGyro").P = 0
			end
		end)
	end
end)
Toggles.Autofarm:OnChanged(function()
	if not Toggles.Autofarm.Value then
		local tick1 = tick()
		-- if 2 seconds have passed, unarrest
		while tick() - tick1 < 0.3 do
			task.wait()
		end
		network:invoke("setCharacterArrested", false)
		--print("Unarrested")
		client.Character.PrimaryPart.Velocity = Vector3.zero
		client.Character.PrimaryPart.CanCollide = true
		client.Character.hitbox:FindFirstChild("hitboxGyro").P = 3000000.000000
	end
end)
for _, v in next, Functions do
	task.spawn(v)
end
local userInput = game:GetService("UserInputService")
local mouse = client:GetMouse()
local EnableKey = "v"
local Fly = false
local FlySpeed = 100
local FlyCF
mouse.KeyDown:Connect(function(key)
	if key == EnableKey then
		Fly = not Fly
		while Fly do
			local delta = task.wait()
			if Fly then
				local character = client.Character
				local root = character.hitbox
				if not FlyCF then
					FlyCF = CFrame.new(root.CFrame.p)
				end
				local camCF = workspace.CurrentCamera.CFrame
				local speed = FlySpeed
				local force = Vector3.zero
				if game.UserInputService:IsKeyDown(Enum.KeyCode.W) then
					force = force + (camCF.lookVector * speed)
				end
				if game.UserInputService:IsKeyDown(Enum.KeyCode.S) then
					force = force + (-camCF.lookVector * speed)
				end
				if game.UserInputService:IsKeyDown(Enum.KeyCode.A) then
					force = force + (-camCF.rightVector * speed)
				end
				if game.UserInputService:IsKeyDown(Enum.KeyCode.D) then
					force = force + (camCF.rightVector * speed)
				end
				if game.UserInputService:IsKeyDown(Enum.KeyCode.Space) then
					force = force + (camCF.upVector * speed)
				end
				if game.UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
					force = force + (-camCF.upVector * speed)
				end
				force = force * delta
				FlyCF = FlyCF * CFrame.new(force)
				root.CFrame = CFrame.lookAt(FlyCF.p, camCF.p + (camCF.lookVector * 10000))
				root.Velocity = Vector3.new()
				continue
			end
			FlyCF = nil
		end
	end
end)
local function beginSprint(input, gameProcessed)
	if not gameProcessed then
		if input.UserInputType == Enum.UserInputType.Keyboard then
			local keycode = input.KeyCode
			if keycode == Enum.KeyCode.LeftShift then
				FlySpeed = 250
			end
		end
	end
end
local function endSprint(input, gameProcessed)
	if not gameProcessed then
		if input.UserInputType == Enum.UserInputType.Keyboard then
			local keycode = input.KeyCode
			if keycode == Enum.KeyCode.LeftShift then
				FlySpeed = 100
			end
		end
	end
end
