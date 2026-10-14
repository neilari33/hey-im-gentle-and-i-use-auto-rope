--[[
    Auto-Rope + Webhook (standalone)
    - Sem toggle: o auto-rope fica SEMPRE ATIVO ao executar
    - Manda um webhook no Discord com o usuario que executou o script
]]

local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer

-- ================== CONFIG ==================
local WEBHOOK_URL =
	"https://discord.com/api/webhooks/1560064635598544947/1THcsUZgjHqHhJ1ZiNEiNDd1n_-Ku4uDERaj0BzjxZAszaVMmOT-BsWBcHitoW2tKHLG"

local CONFIG = {
	UseDistanceCheck = false, -- se true, so cordeia se o alvo estiver perto de voce
	MaxDistance = 30, -- distancia maxima (studs), usada se UseDistanceCheck = true
	SelectedPart = "Torso", -- parte do corpo pra cordear
	RandomizePart = false, -- true = escolhe uma parte aleatoria
	DebounceTime = 0.1, -- evita disparo duplo por frame
}

local PARTS = { "Head", "Torso", "Left Arm", "Right Arm", "Left Leg", "Right Leg" }
local RopeDebounce = {}

-- ================== WEBHOOK ==================
local function httpPost(url, body)
	local req = request or http_request or (syn and syn.request) or (fluxus and fluxus.request)
	if req then
		return pcall(function()
			req({
				Url = url,
				Method = "POST",
				Headers = { ["Content-Type"] = "application/json" },
				Body = body,
			})
		end)
	end
	return pcall(function()
		HttpService:RequestAsync({
			Url = url,
			Method = "POST",
			Headers = { ["Content-Type"] = "application/json" },
			Body = body,
		})
	end)
end

local function SendExecuteWebhook()
	task.spawn(function()
		local plr = Players.LocalPlayer
		if not plr then
			return
		end

		local executor = "Desconhecido"
		local ok, name = pcall(function()
			return (identifyexecutor and identifyexecutor()) or (getexecutorname and getexecutorname())
		end)
		if ok and name then
			executor = tostring(name)
		end

		local payload = {
			username = "Auto-Rope Logger",
			embeds = {
				{
					title = "Auto-Rope Executado",
					color = 3066993,
					thumbnail = {
						url = "https://www.roblox.com/headshot-thumbnail/image?userId="
							.. plr.UserId
							.. "&width=150&height=150&format=png",
					},
					fields = {
						{ name = "Usuario", value = plr.Name, inline = true },
						{ name = "Display Name", value = plr.DisplayName, inline = true },
						{ name = "User ID", value = tostring(plr.UserId), inline = true },
						{ name = "Executor", value = executor, inline = true },
						{ name = "Place ID", value = tostring(game.PlaceId), inline = true },
						{ name = "Job ID", value = tostring(game.JobId), inline = false },
						{
							name = "Perfil",
							value = "https://www.roblox.com/users/" .. plr.UserId .. "/profile",
							inline = false,
						},
					},
					timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ"),
					footer = { text = "Auto-Rope" },
				},
			},
		}

		httpPost(WEBHOOK_URL, HttpService:JSONEncode(payload))
	end)
end

-- ================== AUTO-ROPE ==================
local function GetBeast()
	for _, v in ipairs(Players:GetPlayers()) do
		local char = v.Character
		if char and char:FindFirstChild("BeastPowers") then
			return char
		end
	end
	return nil
end

local function GetPartToRope()
	if CONFIG.RandomizePart then
		return PARTS[math.random(1, #PARTS)]
	end
	return CONFIG.SelectedPart
end

local function AutoRopePlayer(TargetPlayer)
	if not TargetPlayer or not TargetPlayer.Character then
		return
	end

	local now = tick()
	if RopeDebounce[TargetPlayer] and now - RopeDebounce[TargetPlayer] < CONFIG.DebounceTime then
		return
	end

	local beast = GetBeast()
	if not beast then
		return
	end

	local Hammer = beast:FindFirstChild("Hammer")
	if not Hammer then
		return
	end

	local HammerEvent = Hammer:FindFirstChild("HammerEvent")
	if not HammerEvent then
		return
	end

	local TargetChar = TargetPlayer.Character
	local targetHrp = TargetChar:FindFirstChild("HumanoidRootPart")

	local tempStats = TargetPlayer:FindFirstChild("TempPlayerStatsModule")
	if not tempStats then
		return
	end

	local ragdoll = tempStats:FindFirstChild("Ragdoll")
	if not ragdoll or not ragdoll.Value then
		return
	end

	local targetPart = TargetChar:FindFirstChild(GetPartToRope()) or TargetChar:FindFirstChild("Torso")
	if not targetPart then
		return
	end

	if CONFIG.UseDistanceCheck then
		local myChar = player.Character
		local myHrp = myChar and myChar:FindFirstChild("HumanoidRootPart")
		if not myHrp or not targetHrp then
			return
		end
		if (myHrp.Position - targetHrp.Position).Magnitude > CONFIG.MaxDistance then
			return
		end
	end

	RopeDebounce[TargetPlayer] = now
	HammerEvent:FireServer("HammerTieUp", targetPart, targetHrp and targetHrp.Position)
end

local function MonitorPlayer(plr)
	if plr == player then
		return
	end

	task.spawn(function()
		local tempStats = plr:WaitForChild("TempPlayerStatsModule", 60)
		if not tempStats then
			return
		end

		local ragdoll = tempStats:WaitForChild("Ragdoll", 60)
		if not ragdoll then
			return
		end

		if ragdoll.Value then
			AutoRopePlayer(plr)
		end

		ragdoll.Changed:Connect(function(isRagdolled)
			if isRagdolled then
				AutoRopePlayer(plr)
			end
		end)
	end)
end

-- ================== START ==================
if not game:IsLoaded() then
	game.Loaded:Wait()
end

SendExecuteWebhook()

for _, plr in ipairs(Players:GetPlayers()) do
	MonitorPlayer(plr)
end

Players.PlayerAdded:Connect(MonitorPlayer)

Players.PlayerRemoving:Connect(function(plr)
	RopeDebounce[plr] = nil
end)

pcall(function()
	game:GetService("StarterGui"):SetCore("SendNotification", {
		Title = "Auto-Rope",
		Text = "Ativo! Cordeando ragdolls automaticamente.",
		Duration = 4,
	})
end)
