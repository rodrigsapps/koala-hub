--[[
	██╗  ██╗ ██████╗  █████╗ ██╗      █████╗     ██╗  ██╗██╗   ██╗██████╗
	██║ ██╔╝██╔═══██╗██╔══██╗██║     ██╔══██╗    ██║  ██║██║   ██║██╔══██╗
	█████╔╝ ██║   ██║███████║██║     ███████║    ███████║██║   ██║██████╔╝
	██╔═██╗ ██║   ██║██╔══██║██║     ██╔══██║    ██╔══██║██║   ██║██╔══██╗
	██║  ██╗╚██████╔╝██║  ██║███████╗██║  ██║    ██║  ██║╚██████╔╝██████╔╝
	╚═╝  ╚═╝ ╚═════╝ ╚═╝  ╚═╝╚══════╝╚═╝  ╚═╝    ╚═╝  ╚═╝ ╚═════╝ ╚═════╝

	KOALA HUB — Blade Ball / "Bola de Lamina" (PlaceId 13772394625)
	UI: WindUI (Footagesus)  |  Discord: https://discord.gg/ZRFffEgQQM

	Funcoes:
	  - Auto Parry (rebate a bola sozinho, com distancia configuravel)
	  - Auto Spam (modo troca de parry rapido)
	  - ESP da bola + linha ate voce
	  - Auto Ability (usa a habilidade quando a bola vem)
	  - Player: WalkSpeed, JumpPower, pulo infinito
	  - Misc: anti-AFK, FPS boost, rejoin, trocar de servidor

	AVISO: auto parry usa os remotes do jogo. Abusar (spam, distancia
	absurda) pode chamar atencao. Use com criterio.
]]

----------------------------------------------------------------------
-- BOOT / SINGLETON
----------------------------------------------------------------------
if _G.KOALA_HUB_LOADED and _G.KOALA_HUB_DESTROY then
	pcall(_G.KOALA_HUB_DESTROY)
end
_G.KOALA_HUB_LOADED = true

local function try(f, ...)
	local ok, r = pcall(f, ...)
	if ok then return r end
	return nil
end

----------------------------------------------------------------------
-- SERVICOS
----------------------------------------------------------------------
local cloneref = (cloneref or clonereference or function(i) return i end)

local Players           = cloneref(game:GetService("Players"))
local RunService        = cloneref(game:GetService("RunService"))
local ReplicatedStorage = cloneref(game:GetService("ReplicatedStorage"))
local Workspace         = cloneref(game:GetService("Workspace"))
local StarterGui        = cloneref(game:GetService("StarterGui"))
local VirtualInput      = cloneref(game:GetService("VirtualInputManager"))
local TeleportService   = cloneref(game:GetService("TeleportService"))
local HttpService       = cloneref(game:GetService("HttpService"))

local LP = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

local setclipboard = (setclipboard or toclipboard or function() end)

----------------------------------------------------------------------
-- CARREGAR WINDUI
----------------------------------------------------------------------
local WindUI
do
	local sources = {
		"https://github.com/Footagesus/WindUI/releases/latest/download/main.lua",
		"https://raw.githubusercontent.com/Footagesus/WindUI/main/dist/main.lua",
	}
	for _, url in ipairs(sources) do
		local src = try(function() return game:HttpGet(url) end)
		if type(src) == "string" and #src > 5000 then
			local fn = (loadstring or load)(src)
			if fn then
				local ok, lib = pcall(fn)
				if ok and type(lib) == "table" and lib.CreateWindow then
					WindUI = lib
					break
				end
			end
		end
	end
end

if not WindUI then
	try(function()
		StarterGui:SetCore("SendNotification", {
			Title = "KOALA HUB",
			Text = "Falha ao baixar a WindUI. Verifique sua internet.",
			Duration = 8,
		})
	end)
	warn("[KOALA HUB] WindUI nao carregou.")
	return
end

----------------------------------------------------------------------
-- CONFIG
----------------------------------------------------------------------
local CFG = {
	-- auto parry
	AutoParry    = false,
	ParryDist    = 20,     -- studs: rebate quando a bola chega nessa distancia
	ParryMethod  = "Remote", -- Remote | Bindable | Key | Virtual
	SpamMode     = false,  -- rebate toda hora (modo troca)
	SpamDelay    = 0.25,   -- s entre parrys no spam
	OnlyTargeted = true,   -- so rebate se a bola estiver vindo pra voce

	-- ability
	AutoAbility  = false,
	AbilityDist  = 25,

	-- esp
	BallESP      = false,
	ESPTracer    = true,

	-- player
	WalkSpeed    = 16,
	JumpPower    = 50,
	InfJump      = false,
	AntiAFK      = true,
}

----------------------------------------------------------------------
-- CORES
----------------------------------------------------------------------
local ACCENT = Color3.fromHex("#22D3EE")
local GREEN  = Color3.fromHex("#22C55E")
local RED    = Color3.fromHex("#EF4444")
local YELLOW = Color3.fromHex("#EAB308")
local PURPLE = Color3.fromHex("#A78BFA")
local GRAY   = Color3.fromHex("#9CA3AF")

----------------------------------------------------------------------
-- ESTADO
----------------------------------------------------------------------
local StatusFn = function() end
local lastParry = 0
local lastSpam = 0
local parryCount = 0
local espObj = nil
local conns = {}

----------------------------------------------------------------------
-- PERSONAGEM
----------------------------------------------------------------------
local function char()
	return LP.Character
end

local function hrp()
	local c = char()
	return c and c:FindFirstChild("HumanoidRootPart")
end

local function hum()
	local c = char()
	return c and c:FindFirstChildOfClass("Humanoid")
end

local function isAlive()
	local h = hum()
	return h and h.Health > 0
end

----------------------------------------------------------------------
-- BOLA
----------------------------------------------------------------------
-- Blade Ball guarda a bola em Workspace.Balls (modelo com atributo
-- "realBall"/"target") ou como parte solta com nome "Ball"/"BladeBall".
local function getBall()
	-- caminho 1: pasta Balls
	local balls = Workspace:FindFirstChild("Balls")
	if balls then
		for _, b in ipairs(balls:GetChildren()) do
			local real = b:GetAttribute("realBall")
			if real == true or real == nil then
				local part = b:IsA("BasePart") and b or b:FindFirstChildWhichIsA("BasePart", true)
				if part then return part, b end
			end
		end
	end
	-- caminho 2: procura solta no workspace
	for _, n in ipairs({ "Ball", "BladeBall", "Bola" }) do
		local b = Workspace:FindFirstChild(n)
		if b then
			local part = b:IsA("BasePart") and b or b:FindFirstChildWhichIsA("BasePart", true)
			if part then return part, b end
		end
	end
	return nil, nil
end

-- a bola esta vindo pra mim?
local function ballTargetedAtMe(ballModel)
	if not ballModel then return true end
	local target = ballModel:GetAttribute("target")
	if target == nil then return true end -- sem info: assume que sim
	return target == LP.Name
end

local function ballDistance()
	local root = hrp()
	local ball = getBall()
	if not root or not ball then return nil end
	return (ball.Position - root.Position).Magnitude
end

----------------------------------------------------------------------
-- PARRY
----------------------------------------------------------------------
local Remotes = ReplicatedStorage:FindFirstChild("Remotes")

local function doParry()
	local now = os.clock()
	if now - lastParry < 0.05 then return end -- debounce minimo
	lastParry = now
	parryCount = parryCount + 1

	local m = CFG.ParryMethod

	if m == "Remote" then
		-- caminho principal: remote do jogo
		if Remotes then
			local p = Remotes:FindFirstChild("ParryAttempt")
			if p then try(function() p:FireServer() end) end
			local pa = Remotes:FindFirstChild("ParryAttemptAll")
			if pa then try(function() pa:FireServer() end) end
		end
	elseif m == "Bindable" then
		-- caminho via bindable (cliente -> script do jogo)
		if Remotes then
			local b = Remotes:FindFirstChild("ParryButtonPress")
			if b then try(function() b:Fire() end) end
			local b2 = Remotes:FindFirstChild("ParryButton")
			if b2 then try(function() b2:Fire() end) end
		end
	elseif m == "Key" then
		-- simula a tecla F (parry padrao no PC)
		try(function()
			VirtualInput:SendKeyEvent(true, Enum.KeyCode.F, false, game)
			task.wait(0.03)
			VirtualInput:SendKeyEvent(false, Enum.KeyCode.F, false, game)
		end)
	elseif m == "Virtual" then
		-- toque virtual (mobile)
		try(function()
			VirtualInput:SendMouseButtonEvent(0, 0, 0, true, game, 1)
			task.wait(0.03)
			VirtualInput:SendMouseButtonEvent(0, 0, 0, false, game, 1)
		end)
	end
end

local function useAbility()
	if Remotes then
		local a = Remotes:FindFirstChild("ActivateAbility")
		if a then try(function() a:FireServer() end) end
		local r = Remotes:FindFirstChild("RequestAbilityUse")
		if r then try(function() r:FireServer() end) end
	end
end

----------------------------------------------------------------------
-- LOOP PRINCIPAL (auto parry / spam / ability)
----------------------------------------------------------------------
local mainConn
local function startMainLoop()
	if mainConn then return end
	mainConn = RunService.Heartbeat:Connect(function()
		if not isAlive() then return end

		local dist = ballDistance()
		local ball, ballModel = getBall()

		-- auto parry
		if CFG.AutoParry and dist then
			local targeted = ballTargetedAtMe(ballModel)
			if (not CFG.OnlyTargeted or targeted) and dist <= CFG.ParryDist then
				doParry()
			end
		end

		-- spam mode
		if CFG.SpamMode then
			local now = os.clock()
			if now - lastSpam >= CFG.SpamDelay then
				lastSpam = now
				doParry()
			end
		end

		-- auto ability
		if CFG.AutoAbility and dist and dist <= CFG.AbilityDist then
			local targeted = ballTargetedAtMe(ballModel)
			if not CFG.OnlyTargeted or targeted then
				useAbility()
			end
		end
	end)
	table.insert(conns, mainConn)
end

----------------------------------------------------------------------
-- ESP DA BOLA
----------------------------------------------------------------------
local espConn
local function clearESP()
	if espObj then
		try(function()
			if espObj.hl then espObj.hl:Destroy() end
			if espObj.bb then espObj.bb:Destroy() end
			if espObj.line then espObj.line:Remove() end
		end)
		espObj = nil
	end
end

local function updateESP()
	clearESP()
	if not CFG.BallESP then return end
	local ball = getBall()
	if not ball then return end

	local hl = try(function()
		local h = Instance.new("Highlight")
		h.FillColor = Color3.fromRGB(255, 60, 60)
		h.OutlineColor = Color3.fromRGB(255, 255, 255)
		h.FillTransparency = 0.4
		h.Adornee = ball:IsA("Model") and ball or ball.Parent
		h.Parent = ball:IsA("Model") and ball or ball.Parent
		return h
	end)

	local bb = try(function()
		local g = Instance.new("BillboardGui")
		g.Size = UDim2.fromOffset(120, 40)
		g.StudsOffset = Vector3.new(0, 3, 0)
		g.AlwaysOnTop = true
		g.Adornee = ball
		g.Parent = ball
		local t = Instance.new("TextLabel")
		t.Size = UDim2.fromScale(1, 1)
		t.BackgroundTransparency = 1
		t.TextColor3 = Color3.fromRGB(255, 80, 80)
		t.TextStrokeTransparency = 0
		t.Font = Enum.Font.GothamBold
		t.TextSize = 14
		t.Text = "BOLA"
		t.Parent = g
		return g
	end)

	local line
	if CFG.ESPTracer and Drawing then
		line = try(function()
			local l = Drawing.new("Line")
			l.Thickness = 2
			l.Color = Color3.fromRGB(255, 60, 60)
			l.Transparency = 0.8
			l.Visible = false
			return l
		end)
	end

	espObj = { hl = hl, bb = bb, line = line }
end

-- loop do tracer + re-ESP quando a bola respawna
local espLoopConn
local function startESPLoop()
	if espLoopConn then return end
	espLoopConn = RunService.RenderStepped:Connect(function()
		if not CFG.BallESP then
			if espObj and espObj.line then espObj.line.Visible = false end
			return
		end
		local ball = getBall()
		if ball and (not espObj or not espObj.hl or not espObj.hl.Parent) then
			updateESP()
		end
		-- tracer
		if espObj and espObj.line and ball then
			local root = hrp()
			if root then
				local bp, bon = Camera:WorldToViewportPoint(ball.Position)
				local rp, ron = Camera:WorldToViewportPoint(root.Position)
				if bon and ron then
					espObj.line.From = Vector2.new(rp.X, rp.Y)
					espObj.line.To = Vector2.new(bp.X, bp.Y)
					espObj.line.Visible = CFG.ESPTracer
				else
					espObj.line.Visible = false
				end
			end
		end
	end)
	table.insert(conns, espLoopConn)
end

----------------------------------------------------------------------
-- PLAYER
----------------------------------------------------------------------
local function applySpeed()
	local h = hum()
	if h then try(function() h.WalkSpeed = CFG.WalkSpeed end) end
end

local function applyJump()
	local h = hum()
	if h then
		try(function()
			h.UseJumpPower = true
			h.JumpPower = CFG.JumpPower
		end)
	end
end

-- pulo infinito
local infConn
local function setInfJump(on)
	if infConn then pcall(function() infConn:Disconnect() end) infConn = nil end
	if on then
		infConn = game:GetService("UserInputService").JumpRequest:Connect(function()
			local h = hum()
			if h then try(function() h:ChangeState(Enum.HumanoidStateType.Jumping) end) end
		end)
		table.insert(conns, infConn)
	end
end

-- anti-AFK
local afkConn
local function setAntiAFK(on)
	if afkConn then pcall(function() afkConn:Disconnect() end) afkConn = nil end
	if on then
		afkConn = LP.Idled:Connect(function()
			try(function()
				VirtualInput:SendKeyEvent(true, Enum.KeyCode.Space, false, game)
				task.wait(0.05)
				VirtualInput:SendKeyEvent(false, Enum.KeyCode.Space, false, game)
			end)
		end)
		table.insert(conns, afkConn)
	end
end

-- FPS boost
local function fpsBoost()
	try(function()
		for _, v in ipairs(Workspace:GetDescendants()) do
			if v:IsA("ParticleEmitter") or v:IsA("Trail") or v:IsA("Smoke") or v:IsA("Fire") then
				v.Enabled = false
			elseif v:IsA("PostEffect") then
				v.Enabled = false
			end
		end
		local lighting = game:GetService("Lighting")
		for _, v in ipairs(lighting:GetChildren()) do
			if v:IsA("PostEffect") then v.Enabled = false end
		end
		lighting.GlobalShadows = false
		lighting.FogEnd = 9e9
		settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
	end)
end

----------------------------------------------------------------------
-- UI (WindUI)
----------------------------------------------------------------------
local Window = WindUI:CreateWindow({
	Title = "KOALA HUB",
	Icon = "swords",
	Author = "Blade Ball  •  Bola de Lamina",
	Folder = "KoalaHub",
	Size = UDim2.fromOffset(600, 420),
	Theme = "Dark",
	Transparent = true,
	NewElements = true,
	HideSearchBar = false,
	Resizable = true,
	SideBarWidth = 190,
	Background = "",
	OpenButton = {
		Title = "KOALA",
		Enabled = true,
		Draggable = true,
		OnlyMobile = false,
		CornerRadius = UDim.new(1, 0),
		StrokeThickness = 2,
		Color = ColorSequence.new(Color3.fromHex("#22D3EE"), Color3.fromHex("#A78BFA")),
	},
	Topbar = { Height = 44, ButtonsType = "Mac" },
})

try(function()
	Window:Tag({ Title = "v1.0", Icon = "github", Color = Color3.fromHex("#1c1c1c"), Border = true })
	Window:Tag({ Title = "AUTO PARRY", Icon = "swords", Color = ACCENT, Border = true })
end)

local SecMain   = Window:Section({ Title = "Combate" })
local SecPlayer = Window:Section({ Title = "Player" })
local SecMisc   = Window:Section({ Title = "Misc" })

----------------------------------------------------------------------
-- TAB: AUTO PARRY
----------------------------------------------------------------------
local TabParry = SecMain:Tab({
	Title = "Auto Parry",
	Desc = "Rebate a bola sozinho",
	Icon = "swords",
	IconColor = ACCENT,
	IconShape = "Square",
	Border = true,
})

local StatusPar = TabParry:Paragraph({
	Title = "Status",
	Desc = "Auto parry desligado",
	Icon = "info",
})

StatusFn = function(text, color)
	try(function() StatusPar:SetDesc(text) end)
end

TabParry:Toggle({
	Title = "Auto Parry",
	Desc = "Rebate a bola automaticamente quando ela chega perto",
	Value = CFG.AutoParry,
	Callback = function(v)
		CFG.AutoParry = v
		StatusFn(v and "Auto parry LIGADO" or "Auto parry desligado", v and GREEN or GRAY)
	end,
})

TabParry:Slider({
	Title = "Distancia do parry",
	Desc = "Rebate quando a bola chega nessa distancia (studs)",
	Value = { Min = 5, Max = 60, Default = CFG.ParryDist },
	Step = 1,
	Callback = function(v) CFG.ParryDist = v end,
})

TabParry:Dropdown({
	Title = "Metodo do parry",
	Desc = "Remote = mais confiavel. Key/Virtual = simula input (mobile)",
	Values = { "Remote", "Bindable", "Key", "Virtual" },
	Value = CFG.ParryMethod,
	Callback = function(v) CFG.ParryMethod = v end,
})

TabParry:Toggle({
	Title = "So quando mirado em mim",
	Desc = "So rebate se a bola estiver vindo na sua direcao (recomendado)",
	Value = CFG.OnlyTargeted,
	Callback = function(v) CFG.OnlyTargeted = v end,
})

TabParry:Toggle({
	Title = "Modo Spam",
	Desc = "Rebate sem parar (modo troca de parry). CUIDADO: chamativo",
	Value = CFG.SpamMode,
	Callback = function(v)
		CFG.SpamMode = v
		StatusFn(v and "Spam LIGADO" or "Spam desligado", v and RED or GRAY)
	end,
})

TabParry:Slider({
	Title = "Delay do spam",
	Desc = "Segundos entre cada parry no modo spam",
	Value = { Min = 0.1, Max = 2, Default = CFG.SpamDelay },
	Step = 0.05,
	Callback = function(v) CFG.SpamDelay = v end,
})

TabParry:Button({
	Title = "Parry manual",
	Desc = "Rebate uma vez agora (teste)",
	Icon = "zap",
	Callback = function()
		doParry()
		StatusFn("Parry disparado (" .. parryCount .. ")", GREEN)
	end,
})

----------------------------------------------------------------------
-- TAB: ABILITY
----------------------------------------------------------------------
local TabAbility = SecMain:Tab({
	Title = "Ability",
	Desc = "Habilidade automatica",
	Icon = "sparkles",
	IconColor = PURPLE,
	IconShape = "Square",
	Border = true,
})

TabAbility:Toggle({
	Title = "Auto Ability",
	Desc = "Usa a habilidade quando a bola chega perto",
	Value = CFG.AutoAbility,
	Callback = function(v) CFG.AutoAbility = v end,
})

TabAbility:Slider({
	Title = "Distancia da ability",
	Desc = "Usa a habilidade quando a bola chega nessa distancia",
	Value = { Min = 5, Max = 60, Default = CFG.AbilityDist },
	Step = 1,
	Callback = function(v) CFG.AbilityDist = v end,
})

TabAbility:Button({
	Title = "Usar ability agora",
	Desc = "Dispara a habilidade uma vez",
	Icon = "zap",
	Callback = function() useAbility() end,
})

----------------------------------------------------------------------
-- TAB: ESP
----------------------------------------------------------------------
local TabESP = SecMain:Tab({
	Title = "ESP",
	Desc = "Visual da bola",
	Icon = "eye",
	IconColor = YELLOW,
	IconShape = "Square",
	Border = true,
})

TabESP:Toggle({
	Title = "ESP da bola",
	Desc = "Destaca a bola (Highlight + nome)",
	Value = CFG.BallESP,
	Callback = function(v)
		CFG.BallESP = v
		if v then updateESP() else clearESP() end
	end,
})

TabESP:Toggle({
	Title = "Linha ate a bola",
	Desc = "Desenha uma linha de voce ate a bola (precisa de Drawing)",
	Value = CFG.ESPTracer,
	Callback = function(v) CFG.ESPTracer = v end,
})

----------------------------------------------------------------------
-- TAB: PLAYER
----------------------------------------------------------------------
local TabPlayer = SecPlayer:Tab({
	Title = "Player",
	Desc = "Velocidade e pulo",
	Icon = "footprints",
	IconColor = GREEN,
	IconShape = "Square",
	Border = true,
})

TabPlayer:Slider({
	Title = "WalkSpeed",
	Desc = "Velocidade de movimento",
	Value = { Min = 16, Max = 200, Default = CFG.WalkSpeed },
	Step = 1,
	Callback = function(v)
		CFG.WalkSpeed = v
		applySpeed()
	end,
})

TabPlayer:Slider({
	Title = "JumpPower",
	Desc = "Forca do pulo",
	Value = { Min = 50, Max = 300, Default = CFG.JumpPower },
	Step = 5,
	Callback = function(v)
		CFG.JumpPower = v
		applyJump()
	end,
})

TabPlayer:Toggle({
	Title = "Pulo infinito",
	Desc = "Pula no ar quantas vezes quiser",
	Value = CFG.InfJump,
	Callback = function(v)
		CFG.InfJump = v
		setInfJump(v)
	end,
})

TabPlayer:Button({
	Title = "Reaplicar speed/jump",
	Desc = "Reaplica os valores (caso o jogo resete)",
	Icon = "refresh-cw",
	Callback = function()
		applySpeed()
		applyJump()
		StatusFn("Speed/Jump reaplicados", GREEN)
	end,
})

----------------------------------------------------------------------
-- TAB: MISC
----------------------------------------------------------------------
local TabMisc = SecMisc:Tab({
	Title = "Misc",
	Desc = "Utilidades",
	Icon = "wrench",
	IconColor = GRAY,
	IconShape = "Square",
	Border = true,
})

TabMisc:Toggle({
	Title = "Anti-AFK",
	Desc = "Evita kick por inatividade",
	Value = CFG.AntiAFK,
	Callback = function(v)
		CFG.AntiAFK = v
		setAntiAFK(v)
	end,
})

TabMisc:Button({
	Title = "FPS Boost",
	Desc = "Desliga efeitos e sombras pra ganhar FPS",
	Icon = "gauge",
	Callback = function()
		fpsBoost()
		StatusFn("FPS boost aplicado", GREEN)
		try(function()
			WindUI:Notify({ Title = "KOALA HUB", Content = "FPS boost aplicado.", Icon = "gauge", Duration = 3 })
		end)
	end,
})

TabMisc:Button({
	Title = "Rejoin",
	Desc = "Volta pro mesmo servidor",
	Icon = "log-in",
	Callback = function()
		try(function()
			TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, LP)
		end)
	end,
})

TabMisc:Button({
	Title = "Trocar de servidor",
	Desc = "Vai pra um servidor diferente",
	Icon = "shuffle",
	Callback = function()
		StatusFn("Procurando servidor...", YELLOW)
		task.spawn(function()
			local ok = pcall(function()
				local url = ("https://games.roblox.com/v1/games/%d/servers/Public?sortOrder=Asc&limit=100"):format(game.PlaceId)
				local res = game:HttpGet(url)
				local data = HttpService:JSONDecode(res)
				for _, s in ipairs(data.data) do
					if s.playing < s.maxPlayers and s.id ~= game.JobId then
						TeleportService:TeleportToPlaceInstance(game.PlaceId, s.id, LP)
						return
					end
				end
			end)
			if not ok then StatusFn("Falha ao trocar de servidor", RED) end
		end)
	end,
})

TabMisc:Button({
	Title = "Copiar Discord",
	Desc = "Copia o convite do servidor oficial",
	Icon = "link",
	Callback = function()
		try(function() setclipboard("https://discord.gg/ZRFffEgQQM") end)
		StatusFn("Discord copiado!", GREEN)
		try(function()
			WindUI:Notify({ Title = "KOALA HUB", Content = "Convite do Discord copiado.", Icon = "link", Duration = 3 })
		end)
	end,
})

----------------------------------------------------------------------
-- TAB: SOBRE
----------------------------------------------------------------------
local TabAbout = SecMisc:Tab({
	Title = "Sobre",
	Desc = "Info e diagnostico",
	Icon = "info",
	IconColor = GRAY,
	IconShape = "Square",
	Border = true,
})

TabAbout:Paragraph({
	Title = "KOALA HUB v1.0",
	Desc = "Auto Parry + ESP + utilidades pra Blade Ball.\nDiscord: discord.gg/ZRFffEgQQM\n\nSe o parry nao funcionar, troque o 'Metodo do parry' na aba Auto Parry (Remote -> Key -> Virtual).",
	Icon = "swords",
})

TabAbout:Button({
	Title = "Diagnostico",
	Desc = "Confere remotes e bola (console F9)",
	Icon = "stethoscope",
	Callback = function()
		print("===== KOALA HUB — DIAGNOSTICO =====")
		print("Remotes folder:", Remotes ~= nil)
		if Remotes then
			print("  ParryAttempt:", Remotes:FindFirstChild("ParryAttempt") ~= nil)
			print("  ParryAttemptAll:", Remotes:FindFirstChild("ParryAttemptAll") ~= nil)
			print("  ParryButtonPress:", Remotes:FindFirstChild("ParryButtonPress") ~= nil)
			print("  ActivateAbility:", Remotes:FindFirstChild("ActivateAbility") ~= nil)
		end
		local ball, model = getBall()
		print("Bola encontrada:", ball ~= nil, model and model:GetFullName() or "")
		if model then print("  target:", model:GetAttribute("target")) end
		print("Distancia da bola:", ballDistance())
		print("Vivo:", isAlive())
		print("Parrys disparados:", parryCount)
		print("VirtualInputManager:", VirtualInput ~= nil)
		print("Drawing:", Drawing ~= nil)
		print("===================================")
		StatusFn("Diagnostico no console (F9)", ACCENT)
	end,
})

TabAbout:Button({
	Title = "Fechar KOALA HUB",
	Desc = "Remove a UI e desliga tudo",
	Icon = "power",
	Callback = function()
		if _G.KOALA_HUB_DESTROY then _G.KOALA_HUB_DESTROY() end
	end,
})

----------------------------------------------------------------------
-- DESTROY
----------------------------------------------------------------------
_G.KOALA_HUB_DESTROY = function()
	CFG.AutoParry = false
	CFG.SpamMode = false
	CFG.BallESP = false
	for _, c in ipairs(conns) do
		pcall(function() c:Disconnect() end)
	end
	clearESP()
	pcall(function() Window:Destroy() end)
	_G.KOALA_HUB_LOADED = false
	_G.KOALA_HUB_DESTROY = nil
end

----------------------------------------------------------------------
-- START
----------------------------------------------------------------------
startMainLoop()
startESPLoop()
setAntiAFK(CFG.AntiAFK)

-- reaplica speed/jump quando respawnar
table.insert(conns, LP.CharacterAdded:Connect(function()
	task.wait(1)
	applySpeed()
	applyJump()
end))

try(function()
	WindUI:Notify({
		Title = "KOALA HUB",
		Content = "Carregado. Ligue o Auto Parry na aba Auto Parry.",
		Icon = "swords",
		Duration = 5,
	})
end)

StatusFn("KOALA HUB pronto", GREEN)
