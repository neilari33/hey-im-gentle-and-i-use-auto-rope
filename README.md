--[[
    Auto-Rope + Webhook (standalone) — Hit Edition
    - Sem toggle
    - So cordeia quando VOCE da hit no alvo (nao repoe sozinho, sem polling)
    - Manda webhook no Discord com quem executou
]]

local Players = game:GetService("Players")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer

-- ================== CONFIG ==================
local WEBHOOK_URL =
	"https://discord.com/api/webhooks/1560064635598544947/1THcsUZgjHqHhJ1ZiNEiNDd1n_-Ku4uDERaj0BzjxZAszaVMmOT-BsWBcHitoW2tKHLG"

local CONFIG = {
	SelectedPart = "Torso",   -- parte do corpo pra cordear
	RandomizePart = false,    -- true = escolhe uma parte aleatoria

	-- Hit
	HitWindow = 0.8,          -- (s) apos seu hit: se o alvo ragdollar nesse tempo, a corda vai nele
	MaxDistance = 18,         -- alcance (studs) pra achar quem voce acertou
	InFrontOnly = false,      -- true = so considera alvo na sua frente
	UseActivatedFallback = true, -- tambem detecta o swing pelo Tool.Activated
	Debug = false,            -- true = printa as acoes detectadas

	-- Retentativas (so logo apos o hit)
	RetryCount = 3,           -- coloque 1 pra disparar UMA vez so
	RetryDelay = 0.1,
	RopeCooldown = 0.4,       -- evita cordear o mesmo alvo varias vezes seguidas
}

local PARTS = { "Head", "Torso", "Left Arm", "Right Arm", "Left Leg", "Right Leg" }

local PART_PRIORITY = {
	"Torso", "UpperTorso", "LowerTorso", "HumanoidRootPart", "Head",
	"Left Arm", "Right Arm", "Left Leg", "Right Leg",
	"LeftUpperArm", "RightUpperArm", "LeftLowerArm", "RightLowerArm",
	"LeftUpperLeg", "RightUpperLeg", "LeftLowerLeg", "RightLowerLeg",
	"LeftHand", "RightHand", "LeftFoot", "RightFoot",
}

-- ================== ESTADO ==================
local lastHit = 0
local LastRoped = {}

local function Log(...)
	if CONFIG.Debug then
		print("[AutoRope]", ...)
	end
end

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
		if not plr then return end

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

-- ================== HELPERS ==================
local function IsValidPart(p)
	return typeof(p) == "Instance" and p:IsA("BasePart") and p.Parent ~= nil
end

local function FindRopePart(char)
	if not char then return nil end

	for _, name in ipairs(PART_PRIORITY) do
		local p = char:FindFirstChild(name)
		if IsValidPart(p) then return p end
	end

	local best
	for _, d in ipairs(char:GetDescendants()) do
		if IsValidPart(d) and not d:IsA("Accessory") and d.Name ~= "Handle" then
			if not best or (d.Mass or 0) > (best.Mass or 0) then
				best = d
			end
		end
	end
	return best
end

local function ResolveTargetPart(char)
	if CONFIG.RandomizePart then
		local p = char:FindFirstChild(PARTS[math.random(1, #PARTS)])
		if IsValidPart(p) then return p end
		return FindRopePart(char)
	end
	local p = char:FindFirstChild(CONFIG.SelectedPart)
	if IsValidPart(p) then return p end
	return FindRopePart(char)
end

local function GetStatsModule(plr)
	local m = plr:FindFirstChild("TempPlayerStatsModule")
	if m then return m end
	local char = plr.Character
	return char and char:FindFirstChild("TempPlayerStatsModule")
end

local function IsRagdolled(plr)
	if not plr then return false end

	local stats = GetStatsModule(plr)
	if stats then
		local rag = stats:FindFirstChild("Ragdoll")
		if rag and rag.Value then return true end
	end

	local char = plr.Character
	if char then
		local hum = char:FindFirstChildOfClass("Humanoid")
		if hum then
			local st = hum:GetState()
			if st == Enum.HumanoidStateType.FallingDown
				or st == Enum.HumanoidStateType.Physics then
				return true
			end
			if Enum.HumanoidStateType.Ragdoll and st == Enum.HumanoidStateType.Ragdoll then
				return true
			end
		end
	end

	return false
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

local function FireRope(TargetPlayer)
	local beast = GetBeast()
	if not beast then return false end

	local Hammer = beast:FindFirstChild("Hammer")
	if not Hammer then return false end

	local HammerEvent = Hammer:FindFirstChild("HammerEvent")
	if not HammerEvent then return false end

	local char = TargetPlayer.Character
	if not char then return false end

	local targetPart = ResolveTargetPart(char)
	if not targetPart then return false end

	local hrp = char:FindFirstChild("HumanoidRootPart")
	local pos = targetPart.Position

	HammerEvent:FireServer("HammerTieUp", targetPart, pos)
	return true
end

local function CanRope(plr)
	local t = LastRoped[plr]
	return not t or (tick() - t) >= CONFIG.RopeCooldown
end

local function RopeTarget(plr)
	if not plr or not plr.Character then return end
	if not CanRope(plr) then return end

	LastRoped[plr] = tick()
	Log("Cordeando", plr.Name)

	task.spawn(function()
		for _ = 1, math.max(1, CONFIG.RetryCount) do
			if not plr or not plr.Character then return end
			FireRope(plr)
			task.wait(CONFIG.RetryDelay)
		end
	end)
end

-- ================== DETECCAO DE HIT ==================
local function IsInFront(myHrp, targetPos)
	local dir = targetPos - myHrp.Position
	if dir.Magnitude < 0.01 then return true end
	return dir.Unit:Dot(myHrp.CFrame.LookVector) > 0.25
end

-- Acha o alvo que voce acabou de acertar (caido e perto de voce)
local function FindHitTarget()
	local myChar = player.Character
	local myHrp = myChar and myChar:FindFirstChild("HumanoidRootPart")

	local best, bestDist
	for _, plr in ipairs(Players:GetPlayers()) do
		if plr ~= player and IsRagdolled(plr) then
			local char = plr.Character
			local hrp = char and char:FindFirstChild("HumanoidRootPart")
			if hrp then
				local dist = myHrp and (myHrp.Position - hrp.Position).Magnitude or 0
				local ok = dist <= CONFIG.MaxDistance
				if ok and CONFIG.InFrontOnly and myHrp then
					ok = IsInFront(myHrp, hrp.Position)
				end
				if ok and (not bestDist or dist < bestDist) then
					best, bestDist = plr, dist
				end
			end
		end
	end
	return best
end

local function RegisterSwing(targetHint)
	lastHit = tick()

	if targetHint then
		-- se o alvo ja esta caido, cordeia agora; senao o evento de ragdoll cuida
		if IsRagdolled(targetHint) then
			RopeTarget(targetHint)
		end
	else
		-- sem alvo nos args: se tem alguem caido perto, considera como acerto
		local t = FindHitTarget()
		if t then
			RopeTarget(t)
		end
	end
end

local function OnRemote(args)
	local action = args[1]
	if type(action) == "string" then
		Log("acao:", action)
		if action:lower():find("tie") then
			return -- nosso proprio tie
		end
	end

	-- tenta achar um Player/character nos args (alguns jogos mandam o alvo)
	local targetHint
	for _, a in ipairs(args) do
		if typeof(a) == "Instance" then
			local p = Players:GetPlayerFromCharacter(a)
			if not p and a:IsA("Player") then p = a end
			if p and p ~= player then
				targetHint = p
				break
			end
		end
	end

	RegisterSwing(targetHint)
end

-- ================== MONITOR ==================
local function MonitorPlayer(plr)
	if plr == player then return end

	local humConns = {}

	local function bindChar(char)
		for _, c in ipairs(humConns) do c:Disconnect() end
		humConns = {}

		local hum = char:WaitForChild("Humanoid", 30)
		if not hum then return end

		table.insert(humConns, hum.StateChanged:Connect(function(_, new)
			if new == Enum.HumanoidStateType.FallingDown
				or new == Enum.HumanoidStateType.Physics
				or (Enum.HumanoidStateType.Ragdoll and new == Enum.HumanoidStateType.Ragdoll) then
				if (tick() - lastHit) <= CONFIG.HitWindow then
					RopeTarget(plr)
				end
			end
		end))
	end

	task.spawn(function()
		local stats = plr:WaitForChild("TempPlayerStatsModule", 60)
		if not stats then return end

		local ragdoll = stats:WaitForChild("Ragdoll", 60)
		if not ragdoll then return end

		ragdoll.Changed:Connect(function(isRagdolled)
			if isRagdolled and (tick() - lastHit) <= CONFIG.HitWindow then
				RopeTarget(plr)
			end
		end)
	end)

	if plr.Character then bindChar(plr.Character) end
	plr.CharacterAdded:Connect(function(char)
		task.wait(0.3)
		bindChar(char)
	end)
end

-- ================== HOOK DO HAMMER (SEU HIT) ==================
local function BindHammer(char)
	local hammer = char:WaitForChild("Hammer", 20)
	if not hammer then return end

	local remote = hammer:WaitForChild("HammerEvent", 20)
	if remote then
		remote.OnClientEvent:Connect(function(...)
			OnRemote({ ... })
		end)
	end

	if CONFIG.UseActivatedFallback and hammer:IsA("Tool") then
		hammer.Activated:Connect(function()
			RegisterSwing(nil)
		end)
	end
end

local function SetupNamecallHook()
	if not (getrawmetatable and setreadonly and hookmetamethod and getnamecallmethod and newcclosure) then
		Log("sem hookmetamethod, usando apenas fallbacks")
		return
	end
	pcall(function()
		local mt = getrawmetatable(game)
		local old = mt.__namecall
		setreadonly(mt, false)
		mt.__namecall = newcclosure(function(self, ...)
			if getnamecallmethod() == "FireServer"
				and typeof(self) == "Instance"
				and self.Name == "HammerEvent" then
				OnRemote({ ... })
			end
			return old(self, ...)
		end)
		setreadonly(mt, true)
	end)
end

-- ================== START ==================
if not game:IsLoaded() then
	game.Loaded:Wait()
end

SendExecuteWebhook()
SetupNamecallHook()

for _, plr in ipairs(Players:GetPlayers()) do
	MonitorPlayer(plr)
end

Players.PlayerAdded:Connect(MonitorPlayer)

Players.PlayerRemoving:Connect(function(plr)
	LastRoped[plr] = nil
end)

local function onChar(char)
	BindHammer(char)
end

if player.Character then
	onChar(player.Character)
end
player.CharacterAdded:Connect(onChar)

pcall(function()
	game:GetService("StarterGui"):SetCore("SendNotification", {
		Title = "Auto-Rope",
		Text = "Ativo! Cordeia so quando voce da hit.",
		Duration = 4,
	})
end)
