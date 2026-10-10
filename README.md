-- =====================================================================
-- KYÔKA HUB v4.0 • PARTE 1/6 — CORE SYSTEMS
-- Project Slayers 2 • Standalone
-- =====================================================================
local Services = {
    Players = game:GetService("Players"),
    ReplicatedStorage = game:GetService("ReplicatedStorage"),
    RunService = game:GetService("RunService"),
    UserInputService = game:GetService("UserInputService"),
    TweenService = game:GetService("TweenService"),
    HttpService = game:GetService("HttpService"),
    StarterGui = game:GetService("StarterGui"),
    CoreGui = game:GetService("CoreGui"),
    VirtualInputManager = game:GetService("VirtualInputManager"),
    CollectionService = game:GetService("CollectionService"),
    SoundService = game:GetService("SoundService"),
    Lighting = game:GetService("Lighting"),
    GuiService = game:GetService("GuiService"),
    Workspace = workspace,
}

local LP = Services.Players.LocalPlayer
local RS = Services.ReplicatedStorage

-- LOGGER
local Logger = { enable = true, prefix = "[Kyôka]" }
function Logger.log(...) if Logger.enable then print(Logger.prefix, ...) end end
function Logger.warn(...) if Logger.enable then warn(Logger.prefix, ...) end end
function Logger.err(...) warn(Logger.prefix, "ERRO:", ...) end
function Logger.tagged(tag, ...) if Logger.enable then print(Logger.prefix .. "[" .. tag .. "]", ...) end end

-- SAFE REQUIRE
local function safeRequire(path)
    local n = RS
    for _, p in ipairs(path) do
        n = n and n:FindFirstChild(p)
        if not n then return nil, "missing: " .. p end
    end
    local ok, r = pcall(require, n)
    if not ok then return nil, tostring(r) end
    return r
end

-- MÓDULOS DO JOGO
local Game = {}
local MODULE_PATHS = {
    Utility = {"CAM","Global","Utility"},
    SignalEvent = {"Communication","ServerAndClient","Signals","SignalEvent"},
    SignalFunction = {"Communication","ServerAndClient","Signals","SignalFunction"},
    gameSettings = {"CAM","Global","gameSettings"},
    MinigameSettings = {"CAM","Global","MinigameSettings"},
    LiveConfig = {"CAM","Global","LiveConfig"},
    Items = {"CAM","Global","Collectibles","Items"},
    Rarities = {"CAM","Global","Rarities"},
    StatTypes = {"CAM","Global","Types","StatTypes"},
    Series = {"CAM","Global","Series"},
    Refinement = {"CAM","Global","Refinement"},
    Quests = {"CAM","Global","Subsets","Gameplay","Quests"},
    BossHunts = {"CAM","Global","Subsets","Gameplay","Quests","BossHunts"},
    Shop = {"CAM","Global","Shop"},
    Clans = {"CAM","Clans"},
    SpinBalance = {"CAM","Global","SpinBalance"},
    Regions = {"Regions"},
    DayAndNightHandler = {"CAM","Global","DayAndNightHandler"},
    Skill_Controller = {"CAM","Client","Controllers","Skill_Controller"},
    Skills_Provider = {"CAM","Client","Controllers","Skills_Provider"},
    Platform_Handler = {"CAM","Client","Controllers","Platform_Handler"},
    Skill_Info = {"CAM","Global","PlayerProfile","Skill_Info"},
    SkillStats = {"CAM","Global","SkillService","Stats"},
    manage_cd = {"CAM","Global","Subsets","Gameplay","manage_cd"},
    Menum = {"CAM","Global","Menum"},
}

setmetatable(Game, {
    __index = function(t, k)
        local path = MODULE_PATHS[k]
        if not path then return nil end
        local mod = safeRequire(path)
        rawset(t, k, mod)
        return mod
    end
})

local PortalEvent = RS:FindFirstChild("CAM")
    and RS.CAM:FindFirstChild("Global")
    and RS.CAM.Global:FindFirstChild("ServerClientPortal")
    and RS.CAM.Global.ServerClientPortal:FindFirstChild("Event")
Game.PortalEvent = PortalEvent

-- EVENT BUS
local EventBus = { listeners = {} }
function EventBus.on(event, fn)
    EventBus.listeners[event] = EventBus.listeners[event] or {}
    table.insert(EventBus.listeners[event], fn)
    return function()
        for i, f in ipairs(EventBus.listeners[event]) do
            if f == fn then table.remove(EventBus.listeners[event], i) return end
        end
    end
end
function EventBus.emit(event, ...)
    local list = EventBus.listeners[event]
    if not list then return end
    for _, fn in ipairs(list) do pcall(fn, ...) end
end

-- RATE LIMIT
local RateLimit = { last = {} }
function RateLimit.check(key, interval)
    local now = os.clock()
    if RateLimit.last[key] and (now - RateLimit.last[key]) < interval then return false end
    RateLimit.last[key] = now
    return true
end

-- HELPERS
local H = {}
function H.char()
    local c = LP.Character
    if not c then return nil, nil, nil end
    return c, c:FindFirstChild("HumanoidRootPart"), c:FindFirstChildOfClass("Humanoid")
end
function H.pos() local _, h = H.char() return h and h.Position or nil end
function H.cframe() local _, h = H.char() return h and h.CFrame or nil end
function H.alive() local _, _, hu = H.char() return hu ~= nil and hu.Health > 0 end
function H.health() local _, _, hu = H.char() return hu and hu.Health or 0 end
function H.maxHealth() local _, _, hu = H.char() return hu and hu.MaxHealth or 100 end
function H.healthPct()
    local _, _, hu = H.char()
    if not hu or hu.MaxHealth <= 0 then return 100 end
    return hu.Health / hu.MaxHealth * 100
end
function H.humanoid() local _, _, hu = H.char() return hu end
function H.root() local _, h = H.char() return h end
function H.tool() local c = H.char() return c and c:FindFirstChildOfClass("Tool") end

function H.slot()
    if not Game.Utility then return nil end
    local ok, s = pcall(Game.Utility.GetData, LP, true)
    return ok and s or nil
end
function H.level()
    local slot = H.slot()
    if not slot then return 0 end
    local goal = slot:FindFirstChild("Exp") and slot.Exp:FindFirstChild("Goal")
    local per = Game.gameSettings and Game.gameSettings.expPerLevel
    if not goal or not per or per == 0 then return 0 end
    return math.floor(goal.Value / per)
end
function H.exp()
    local slot = H.slot()
    if not slot then return 0, 0 end
    local exp = slot:FindFirstChild("Exp")
    local cur = exp and exp:FindFirstChild("Current")
    local goal = exp and exp:FindFirstChild("Goal")
    return (cur and cur.Value or 0), (goal and goal.Value or 0)
end
function H.value(k)
    local slot = H.slot()
    local v = slot and slot:FindFirstChild(k)
    return v and v.Value
end
function H.race() return H.value("Race") or "?" end
function H.clan() return H.value("Clan") or "?" end
function H.wen() return H.value("Wen") or 0 end
function H.reputation() return H.value("Reputation") or 0 end
function H.skillPoints() return H.value("SkillPoints") or 0 end

function H.inventory()
    local slot = H.slot()
    if not slot then return nil, nil end
    local inv = slot:FindFirstChild("Inventory")
    return inv and inv:FindFirstChild("Inventory"), inv and inv:FindFirstChild("Toolbar")
end
function H.count(name)
    local items = H.inventory()
    local item = items and items:FindFirstChild(name)
    if not item then return 0 end
    local a = item:FindFirstChild("Amount")
    return a and a.Value or 1
end
function H.allItems()
    local items = H.inventory()
    if not items then return {} end
    local list = {}
    for _, item in ipairs(items:GetChildren()) do
        local a = item:FindFirstChild("Amount")
        list[item.Name] = a and a.Value or 1
    end
    return list
end
function H.hotbarIdx(name)
    local items, bar = H.inventory()
    if not (items and bar) then return nil end
    local item = items:FindFirstChild(name)
    local id = item and item:FindFirstChild("Id")
    if not id then return nil end
    for i, k in ipairs({"One","Two","Three","Four","Five"}) do
        local e = bar:FindFirstChild(k)
        if e and e.Value == id.Value then return i end
    end
    return nil
end
function H.equip(idx)
    local cfg = LP:FindFirstChild("Items_Config")
    local eq = cfg and cfg:FindFirstChild("Equipped")
    if not eq then return false end
    if eq.Value == idx then return true end
    if eq.Value ~= 0 then eq.Value = 0 task.wait(0.4) end
    eq.Value = idx
    task.wait(0.4)
    return eq.Value == idx
end

function H.mobs(range)
    local _, hrp = H.char()
    if not hrp then return {} end
    range = range or 100
    local list = {}
    local regions = workspace:FindFirstChild("Humanoids") and workspace.Humanoids:FindFirstChild("Regions")
    if not regions then return list end
    for _, reg in ipairs(regions:GetChildren()) do
        local active = reg:FindFirstChild("ActiveNpcs")
        for _, folder in ipairs(active and active:GetChildren() or {}) do
            local rig = folder:FindFirstChild(folder.Name)
            if rig and rig:GetAttribute("IsMob") == true then
                local hum = rig:FindFirstChildOfClass("Humanoid")
                local root = rig:FindFirstChild("HumanoidRootPart")
                if hum and hum.Health > 0 and root then
                    local d = (root.Position - hrp.Position).Magnitude
                    if d <= range then
                        list[#list+1] = { rig = rig, root = root, hum = hum, dist = d, health = hum.Health, maxHealth = hum.MaxHealth }
                    end
                end
            end
        end
    end
    table.sort(list, function(a, b) return a.dist < b.dist end)
    return list
end
function H.bosses(range)
    local mobs = H.mobs(range or 500)
    local list = {}
    for _, m in ipairs(mobs) do
        local parent = m.rig.Parent
        if parent and parent:FindFirstChild("BossInfo") then list[#list+1] = m end
    end
    return list
end
function H.npcSpawn(name)
    if not Game.Regions or not Game.Regions.GetNpcSpawn then return nil end
    local ok, pos = pcall(Game.Regions.GetNpcSpawn, name)
    if ok and typeof(pos) == "Vector3" then return pos end
    return nil
end
function H.npcModel(name)
    local d = workspace:FindFirstChild("Debree")
    local r = d and d:FindFirstChild("Regions")
    for _, reg in ipairs(r and r:GetChildren() or {}) do
        local s = reg:FindFirstChild("StationaryNpcs")
        local m = s and s:FindFirstChild(name)
        if m and m:IsA("Model") then return m end
    end
    return nil
end
function H.firePrompt(prompt)
    if not prompt then return false end
    if fireproximityprompt then if pcall(fireproximityprompt, prompt) then return true end end
    local hold, sight = prompt.HoldDuration, prompt.RequiresLineOfSight
    local ok = pcall(function()
        prompt.HoldDuration = 0
        prompt.RequiresLineOfSight = false
        task.wait(0.1)
        prompt:InputHoldBegin()
        task.wait(0.15)
        prompt:InputHoldEnd()
    end)
    pcall(function() prompt.HoldDuration, prompt.RequiresLineOfSight = hold, sight end)
    return ok
end
function H.interactNPC(model)
    if not model then return false end
    local prompt = model:FindFirstChildWhichIsA("ProximityPrompt", true)
    if prompt then return H.firePrompt(prompt) end
    local cd = model:FindFirstChildWhichIsA("ClickDetector", true)
    if cd and fireclickdetector then return pcall(fireclickdetector, cd) end
    return false
end

function H.attack(pos)
    if not Game.SignalEvent or not pos then return false end
    pcall(function() Game.SignalEvent.ToServer("Tool_Mouse", "Down", pos) end)
    task.wait(0.05)
    pcall(function() Game.SignalEvent.ToServer("Tool_Mouse", "Up", pos) end)
    return true
end
function H.attackAt(target)
    if type(target) == "Vector3" then return H.attack(target) end
    if target and target.Position then return H.attack(target.Position) end
    return false
end
function H.moveTo(pos)
    local _, hum = H.char()
    if not hum or not pos then return false end
    pcall(function() hum:MoveTo(pos) end)
    return true
end
function H.safeTP(pos)
    local _, hrp = H.char()
    if not hrp or not pos then return false end
    local d = (pos - hrp.Position).Magnitude
    if d > 50 then
        local dir = (pos - hrp.Position).Unit
        pcall(function() hrp.CFrame = CFrame.new(hrp.Position + dir * 30 + Vector3.new(0, 3, 0)) end)
        return false
    end
    pcall(function() hrp.CFrame = CFrame.new(pos + Vector3.new(0, 3, 0)) end)
    return true
end
function H.faceTarget(pos)
    local _, hrp = H.char()
    if not hrp or not pos then return end
    local cam = workspace.CurrentCamera
    if cam then pcall(function() cam.CFrame = CFrame.new(cam.CFrame.Position, pos) end) end
end
function H.isCasting()
    local char = LP.Character
    if not char then return false end
    local shc = char:FindFirstChild("SHC") or char:FindFirstChild("SHCS")
    if not shc then return false end
    return shc.Value ~= "" or shc:GetAttribute("en") == true
end
function H.skillBar()
    if not Game.Skills_Provider then return {} end
    local ok, keys = pcall(Game.Skills_Provider.get_current_keys)
    if not ok or type(keys) ~= "table" then return {} end
    local bar = {}
    for i = 1, 10 do
        local e = keys[i]
        if type(e) == "table" and e.Name and e.Name ~= "" then
            bar[#bar+1] = { name = e.Name, maxHold = tonumber(e.Max_Hold) or 0, index = i }
        end
    end
    return bar
end
function H.findWater(radius)
    local _, hrp = H.char()
    if not hrp then return nil end
    radius = radius or 32
    local p = RaycastParams.new()
    p.FilterType = Enum.RaycastFilterType.Exclude
    p.FilterDescendantsInstances = { LP.Character }
    for r = 8, radius, 4 do
        for i = 0, 15 do
            local a = i * math.pi / 8
            local o = hrp.Position + Vector3.new(math.cos(a)*r, 50, math.sin(a)*r)
            local h = workspace:Raycast(o, Vector3.new(0,-150,0), p)
            if h and h.Material == Enum.Material.Water then return h.Position end
        end
    end
    return nil
end
function H.chooseQuest()
    if not Game.Quests then return nil end
    local lvl = H.level()
    local best, bestExp
    for key, def in pairs(Game.Quests.Holder) do
        if type(def) == "table" and type(def.OfferNpc) == "string" then
            local req = def.Requirements
            local rl = type(req) == "table" and (tonumber(req.Level) or 0) or 0
            if rl <= lvl then
                local ok, allowed, code = pcall(function() return Game.Quests.CanAddQuest(LP, key, true) end)
                if ok and (allowed == true or (allowed == false and code == false)) then
                    local r = def.Rewards
                    local exp = type(r) == "table" and tonumber(r.Exp) or 0
                    if exp > (bestExp or 0) then
                        best, bestExp = { key = key, def = def, npc = def.OfferNpc, exp = exp }, exp
                    end
                end
            end
        end
    end
    return best
end
function H.currentTask(quest)
    if not quest or not Game.Quests then return nil end
    local slot = H.slot()
    local holder = slot and slot:FindFirstChild("Quests") and slot.Quests:FindFirstChild("Holder")
    if not holder then return nil end
    for _, q in ipairs(holder:GetChildren()) do
        local qs = q:FindFirstChild("QuestString")
        local def = qs and Game.Quests.Holder[qs.Value]
        if def == quest.def then
            local tasks = q:FindFirstChild("Tasks")
            if tasks then
                for _, t in ipairs(tasks:GetChildren()) do
                    local v, m = t:FindFirstChild("Value"), t:FindFirstChild("Max")
                    if v and m and v.Value < m.Value then
                        return { name = t.Name, value = v.Value, max = m.Value }
                    end
                end
                return nil
            end
        end
    end
    return nil
end

-- ANTI-CHEAT
local AntiCheat = { lastPos = nil, fails = 0, onFail = nil }
function AntiCheat.check()
    local _, hrp = H.char()
    if not hrp then return false end
    if AntiCheat.lastPos then
        local diff = (hrp.Position - AntiCheat.lastPos).Magnitude
        if diff > 100 then
            AntiCheat.lastPos = hrp.Position
            AntiCheat.fails = AntiCheat.fails + 1
            if AntiCheat.onFail then AntiCheat.onFail() end
            return true
        end
    end
    AntiCheat.lastPos = hrp.Position
    return false
end
function AntiCheat.reset() AntiCheat.fails = 0 AntiCheat.lastPos = nil end

-- CONFIG
local Config = { data = {}, file = "KyokaHub/config.json" }
function Config.set(key, value) Config.data[key] = value end
function Config.get(key, default) return Config.data[key] or default end
function Config.save()
    if not writefile then return false end
    return pcall(function() writefile(Config.file, Services.HttpService:JSONEncode(Config.data)) end)
end
function Config.load()
    if not readfile or not isfile then return false end
    if not isfile(Config.file) then return false end
    local ok, content = pcall(readfile, Config.file)
    if not ok then return false end
    local ok2, data = pcall(Services.HttpService.JSONDecode, Services.HttpService, content)
    if ok2 and type(data) == "table" then Config.data = data return true end
    return false
end

-- EXPORTA
_G.__KyokaHub = _G.__KyokaHub or {}
_G.__KyokaHub.Services = Services
_G.__KyokaHub.Game = Game
_G.__KyokaHub.H = H
_G.__KyokaHub.Logger = Logger
_G.__KyokaHub.EventBus = EventBus
_G.__KyokaHub.RateLimit = RateLimit
_G.__KyokaHub.AntiCheat = AntiCheat
_G.__KyokaHub.Config = Config

Logger.log("Parte 1/6 (Core) carregada")
Logger.log("Pronto para a Parte 2/6")-- =====================================================================
-- KYÔKA HUB v4.0 • PARTE 2/6 — UI FRAMEWORK
-- Project Slayers 2 • Standalone
-- =====================================================================

local Core = _G.__KyokaHub
if not Core then warn("[Kyôka] Rode a Parte 1 primeiro!") return end

local Services = Core.Services
local Logger = Core.Logger
local H = Core.H
local EventBus = Core.EventBus

local LP = Services.Players.LocalPlayer
local CoreGui = Services.CoreGui
local UIS = Services.UserInputService
local TweenService = Services.TweenService

Logger.log("Iniciando Parte 2/6 (UI)")

-- FLUENT LIBRARY
local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()
local SaveManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/SaveManager.lua"))()
local InterfaceManager = loadstring(game:HttpGet("https://raw.githubusercontent.com/dawid-scripts/Fluent/master/Addons/InterfaceManager.lua"))()

Core.Fluent = Fluent
Core.SaveManager = SaveManager
Core.InterfaceManager = InterfaceManager

-- SISTEMA DE SOM
local Sound = {
    enabled = true,
    volume = 0.35,
    source = "rbxasset://sounds/electronicpingshort.wav",
    _pool = {},
}

function Sound.play(name)
    if not Sound.enabled then return end
    local pitch = ({
        click = 1.1, toggle_on = 1.4, toggle_off = 0.8,
        open = 1.6, close = 0.7, notify = 1.2,
        error = 0.5, success = 1.5,
    })[name] or 1

    local s = Sound._pool[name]
    if not s or not s.Parent then
        s = Instance.new("Sound")
        s.Name = "Kyoka_Sound_" .. tostring(name)
        s.Parent = Services.SoundService
        Sound._pool[name] = s
    end
    s.SoundId = Sound.source
    s.Volume = Sound.volume
    s.PlaybackSpeed = pitch
    pcall(function() s:Play() end)
end

-- JANELA PRINCIPAL
local Window = Fluent:CreateWindow({
    Title = "Kyôka Hub",
    SubTitle = "• Project Slayers 2",
    TabWidth = 165,
    Size = UDim2.fromOffset(650, 520),
    Acrylic = true,
    Theme = "Dark",
    MinimizeKey = Enum.KeyCode.LeftControl,
})

-- ABAS
local Tabs = {
    Farm      = Window:AddTab({ Title = "Farm",       Icon = "boxes" }),
    Combat    = Window:AddTab({ Title = "Combat",     Icon = "sword" }),
    Skills    = Window:AddTab({ Title = "Skills",     Icon = "sparkles" }),
    Fishing   = Window:AddTab({ Title = "Fishing",    Icon = "fish" }),
    Progress  = Window:AddTab({ Title = "Progress",   Icon = "trending-up" }),
    Player    = Window:AddTab({ Title = "Player",     Icon = "user" }),
    Teleport  = Window:AddTab({ Title = "Teleport",   Icon = "map-pin" }),
    Settings  = Window:AddTab({ Title = "Settings",   Icon = "settings" }),
}

local Options = Fluent.Options

-- NOTIFICAÇÕES
local Notify = {}

function Notify.info(title, content, dur)
    Fluent:Notify({ Title = title, Content = content, Duration = dur or 4 })
    Sound.play("notify")
end

function Notify.success(title, content, dur)
    Fluent:Notify({ Title = "OK " .. title, Content = content, Duration = dur or 4 })
    Sound.play("success")
end

function Notify.error(title, content, dur)
    Fluent:Notify({ Title = "X " .. title, Content = content, Duration = dur or 6 })
    Sound.play("error")
end

function Notify.warn(title, content, dur)
    Fluent:Notify({ Title = "! " .. title, Content = content, Duration = dur or 5 })
end

-- STATUS PANELS
local Panels = {}

function Panels.create(tab, title, initial)
    local p = tab:AddParagraph({
        Title = title or "Status",
        Content = initial or "-",
    })
    return {
        _p = p,
        set_title = function(self, t) pcall(function() p:SetTitle(t) end) end,
        set_desc = function(self, d) pcall(function() p:SetDesc(d) end) end,
        update = function(self, t, d)
            pcall(function()
                if t then p:SetTitle(t) end
                if d then p:SetDesc(d) end
            end)
        end,
    }
end

-- TOGGLE COM SOM
local function addToggleWithSound(tab, id, config)
    local orig = config.Callback
    config.Callback = function(v)
        Sound.play(v and "toggle_on" or "toggle_off")
        if orig then pcall(orig, v) end
        EventBus.emit("toggle:" .. id, v)
    end
    return tab:AddToggle(id, config)
end

-- TEMA
local Theme = {
    current = "Dark",
    themes = {
        Dark = "Dark",
        Light = "Light",
        Rose = "Rose",
        AMOLED = "AMOLED",
    },
}

function Theme.set(name)
    if not Theme.themes[name] then return false end
    Theme.current = name
    pcall(function() Fluent:SetTheme(name) end)
    Notify.info("Tema", "Alterado para " .. name, 3)
    return true
end

-- BOTÃO FLUTUANTE DO GATO
local CatButton = {}

function CatButton.create()
    if CoreGui:FindFirstChild("KyokaHub_Cat") then
        CoreGui.KyokaHub_Cat:Destroy()
    end

    local gui = Instance.new("ScreenGui")
    gui.Name = "KyokaHub_Cat"
    gui.ResetOnSpawn = false
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = CoreGui

    local holder = Instance.new("Frame")
    holder.Size = UDim2.fromOffset(60, 60)
    holder.Position = UDim2.new(0, 20, 0.5, -30)
    holder.BackgroundColor3 = Color3.fromRGB(220, 40, 60)
    holder.BackgroundTransparency = 0.85
    holder.BorderSizePixel = 0
    holder.Parent = gui

    local hc = Instance.new("UICorner")
    hc.CornerRadius = UDim.new(1, 0)
    hc.Parent = holder

    local btn = Instance.new("ImageButton")
    btn.Size = UDim2.fromOffset(55, 55)
    btn.Position = UDim2.new(0.5, -27.5, 0.5, -27.5)
    btn.BackgroundColor3 = Color3.fromRGB(15, 15, 18)
    btn.BorderSizePixel = 0
    btn.AutoButtonColor = false
    btn.ScaleType = Enum.ScaleType.Crop
    btn.Parent = holder

    local bc = Instance.new("UICorner")
    bc.CornerRadius = UDim.new(1, 0)
    bc.Parent = btn

    local bs = Instance.new("UIStroke")
    bs.Color = Color3.fromRGB(220, 40, 60)
    bs.Thickness = 2
    bs.Transparency = 0.15
    bs.Parent = btn

    task.spawn(function()
        pcall(function()
            local url = "https://i.supaimg.com/96505f3a-6d22-4d4b-9d36-5bfc5e566443/09bb24b4-ee02-4d6f-a5b5-9f08f91c07ac.jpg"
            if writefile and getcustomasset then
                if not isfile or not isfile("KyokaCat.jpg") then
                    writefile("KyokaCat.jpg", game:HttpGet(url))
                end
                btn.Image = getcustomasset("KyokaCat.jpg")
            else
                btn.Image = url
            end
        end)
    end)

    task.spawn(function()
        while gui.Parent do
            pcall(function()
                local t = TweenService:Create(holder,
                    TweenInfo.new(1.5, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut),
                    { BackgroundTransparency = 0.95 })
                t:Play()
                t.Completed:Wait()
                local t2 = TweenService:Create(holder,
                    TweenInfo.new(1.5, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut),
                    { BackgroundTransparency = 0.75 })
                t2:Play()
                t2.Completed:Wait()
            end)
            task.wait(0.1)
        end
    end)

    local dragging = false
    local dragStart, startPos

    btn.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStart = input.Position
            startPos = holder.Position
        end
    end)

    UIS.InputChanged:Connect(function(input)
        if not dragging or not dragStart then return end
        if input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch then
            local delta = input.Position - dragStart
            holder.Position = UDim2.new(
                startPos.X.Scale, startPos.X.Offset + delta.X,
                startPos.Y.Scale, startPos.Y.Offset + delta.Y
            )
        end
    end)

    UIS.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then
            dragging = false
        end
    end)

    btn.MouseEnter:Connect(function()
        TweenService:Create(btn, TweenInfo.new(0.2), {
            Size = UDim2.fromOffset(60, 60),
            Position = UDim2.new(0.5, -30, 0.5, -30),
        }):Play()
    end)
    btn.MouseLeave:Connect(function()
        TweenService:Create(btn, TweenInfo.new(0.2), {
            Size = UDim2.fromOffset(55, 55),
            Position = UDim2.new(0.5, -27.5, 0.5, -27.5),
        }):Play()
    end)

    local hubOpen = true
    local function setHubOpen(state)
        hubOpen = state
        pcall(function() Window.Root.Visible = state end)
        Sound.play(state and "open" or "close")
    end

    btn.MouseButton1Click:Connect(function()
        setHubOpen(not hubOpen)
    end)

    CatButton.gui = gui
    CatButton.button = btn
    CatButton.holder = holder
    CatButton.setVisible = function(v) gui.Enabled = v end
    CatButton.isOpen = function() return hubOpen end
    CatButton.setOpen = setHubOpen

    return CatButton
end

CatButton.create()

-- KEYBIND GLOBAL
UIS.InputBegan:Connect(function(input, gpe)
    if gpe then return end
    if input.KeyCode == Enum.KeyCode.RightShift then
        CatButton.setOpen(not CatButton.isOpen())
    end
end)

-- HUB STATUS PANEL
local HubStatus = { gui = nil, label = nil }

function HubStatus.create()
    local gui = Instance.new("ScreenGui")
    gui.Name = "KyokaHub_Status"
    gui.ResetOnSpawn = false
    gui.DisplayOrder = 5
    gui.Parent = CoreGui

    local frame = Instance.new("Frame")
    frame.Size = UDim2.fromOffset(220, 30)
    frame.Position = UDim2.new(1, -240, 0, 15)
    frame.BackgroundColor3 = Color3.fromRGB(18, 18, 22)
    frame.BackgroundTransparency = 0.25
    frame.BorderSizePixel = 0
    frame.Parent = gui

    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, 8)
    c.Parent = frame

    local s = Instance.new("UIStroke")
    s.Color = Color3.fromRGB(220, 40, 60)
    s.Thickness = 1
    s.Transparency = 0.5
    s.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -16, 1, 0)
    label.Position = UDim2.fromOffset(8, 0)
    label.BackgroundTransparency = 1
    label.Text = "Kyôka Hub • v4.0"
    label.TextColor3 = Color3.fromRGB(240, 240, 245)
    label.TextSize = 13
    label.Font = Enum.Font.GothamBold
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    HubStatus.gui = gui
    HubStatus.label = label
end

HubStatus.create()

function HubStatus.set(text)
    if HubStatus.label then
        pcall(function() HubStatus.label.Text = text end)
    end
end

-- TUTORIAL
local function showTutorial()
    task.spawn(function()
        Notify.info("Kyôka Hub", "Bem-vindo! Carregando em 3 segundos...", 3)
        task.wait(3)
        Notify.info("Dica 1", "RightShift abre/fecha o hub", 5)
        task.wait(5)
        Notify.info("Dica 2", "O gato no canto esquerdo tambem abre", 5)
        task.wait(5)
        Notify.info("Dica 3", "Use em conta alternativa!", 5)
    end)
end

-- MANAGERS
SaveManager:SetLibrary(Fluent)
InterfaceManager:SetLibrary(Fluent)
SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({ "MenuKeybind" })
InterfaceManager:SetFolder("KyokaHub")
SaveManager:SetFolder("KyokaHub/Slayers2")

-- UI HELPERS
local UIHelper = {}

function UIHelper.makeStatus(tab, title, initial)
    return Panels.create(tab, title, initial)
end

function UIHelper.addToggle(tab, id, config)
    return addToggleWithSound(tab, id, config)
end

-- EXPORTA
Core.Window = Window
Core.Tabs = Tabs
Core.Options = Options
Core.Notify = Notify
Core.Sound = Sound
Core.Theme = Theme
Core.Panels = Panels
Core.CatButton = CatButton
Core.HubStatus = HubStatus
Core.UIHelper = UIHelper
Core.addToggleWithSound = addToggleWithSound

-- BOOT
Notify.info("Kyôka Hub", "UI carregada! Configurando...", 4)
HubStatus.set("Kyôka Hub • v4.0 • Inicializando")

showTutorial()

Logger.log("Parte 2/6 (UI) carregada — 8 abas prontas")
Logger.log("Pronto para a Parte 3/6")-- =====================================================================
-- KYÔKA HUB v4.0 • PARTE 3/6 — FARM & COMBAT
-- Project Slayers 2 • Standalone
-- =====================================================================

local Core = _G.__KyokaHub
if not Core then warn("[Kyôka] Rode a Parte 1 primeiro!") return end

local Services = Core.Services
local Game = Core.Game
local H = Core.H
local Logger = Core.Logger
local Notify = Core.Notify
local Panels = Core.Panels
local Tabs = Core.Tabs
local Options = Core.Options
local Sound = Core.Sound
local RateLimit = Core.RateLimit
local AntiCheat = Core.AntiCheat
local addToggleWithSound = Core.addToggleWithSound
local EventBus = Core.EventBus

local LP = Services.Players.LocalPlayer

Logger.log("Iniciando Parte 3/6 (Farm & Combat)")

-- ESTADO GLOBAL
local S = {
    antiSun = false,
    antiDrown = false,
    autoHeal = false,
    healPct = 45,
    healPotion = "Health Potion",
    autoFarm = false,
    farmMode = "MoveTo",
    farmRange = 50,
    farmPriority = "Closest",
    farmDelay = 0.3,
    killAura = false,
    killAuraRange = 15,
    autoBoss = false,
    bossRange = 100,
    attackCount = 0,
    kills = 0,
    healCount = 0,
    lastTarget = "-",
    lastHit = 0,
}

-- PATCH: ANTI SUN
do
    if Game.MinigameSettings and Game.MinigameSettings.Get then
        local orig = Game.MinigameSettings.Get
        Game.MinigameSettings.Get = function(k, ...)
            if k == "NoSunDamage" and S.antiSun then return true end
            return orig(k, ...)
        end
    end
    if Game.SignalFunction and Game.SignalFunction.ToServer then
        local orig = Game.SignalFunction.ToServer
        Game.SignalFunction.ToServer = function(n, v, ...)
            if n == "SunDamage" and v == true and S.antiSun then
                return orig(n, false, ...)
            end
            return orig(n, v, ...)
        end
    end
end

-- PATCH: ANTI DROWN
do
    if Game.MinigameSettings and Game.MinigameSettings.Get then
        local orig = Game.MinigameSettings.Get
        Game.MinigameSettings.Get = function(k, ...)
            if k == "NoDrowning" and S.antiDrown then return true end
            return orig(k, ...)
        end
    end
    if Game.SignalEvent and Game.SignalEvent.ToServer then
        local orig = Game.SignalEvent.ToServer
        Game.SignalEvent.ToServer = function(n, a, ...)
            if n == "Swim" and a == "DrownDamage" and S.antiDrown then
                return nil
            end
            return orig(n, a, ...)
        end
    end
end

-- AUTO HEAL
task.spawn(function()
    local failUntil = 0
    while true do
        task.wait(0.5)
        if S.autoHeal and os.clock() > failUntil then
            if H.alive() and H.healthPct() <= S.healPct then
                if H.count(S.healPotion) > 0 then
                    local idx = H.hotbarIdx(S.healPotion)
                    if idx then
                        local before = H.count(S.healPotion)
                        H.equip(idx)
                        local _, hrp = H.char()
                        if hrp and Game.SignalEvent then
                            pcall(function()
                                Game.SignalEvent.ToServer("Tool_Mouse", "Down", hrp.Position)
                            end)
                            task.wait(2.1)
                            pcall(function()
                                Game.SignalEvent.ToServer("Tool_Mouse", "Up", hrp.Position)
                            end)
                            task.wait(0.5)
                        end
                        H.equip(0)
                        if H.count(S.healPotion) >= before then
                            failUntil = os.clock() + 5
                        else
                            S.healCount = S.healCount + 1
                        end
                    end
                end
            end
        end
    end
end)

-- ESCOLHA DE ALVO
local function pickTarget(mobs)
    if #mobs == 0 then return nil end
    if S.farmPriority == "Lowest HP" then
        table.sort(mobs, function(a, b) return a.health < b.health end)
    elseif S.farmPriority == "Highest HP" then
        table.sort(mobs, function(a, b) return a.health > b.health end)
    else
        table.sort(mobs, function(a, b) return a.dist < b.dist end)
    end
    return mobs[1]
end

-- AUTO FARM
task.spawn(function()
    while true do
        task.wait(S.farmDelay)
        if S.autoFarm and H.alive() then
            local mobs = H.mobs(S.farmRange)
            local t = pickTarget(mobs)
            if t then
                S.lastTarget = t.rig.Name
                if t.dist > 8 then
                    if S.farmMode == "Teleport" then
                        H.safeTP(t.root.Position)
                    else
                        H.moveTo(t.root.Position)
                    end
                else
                    H.faceTarget(t.root.Position)
                    H.attack(t.root.Position)
                    S.attackCount = S.attackCount + 1
                    if t.health <= 0 then
                        S.kills = S.kills + 1
                    end
                end
            end
        end
    end
end)

-- KILL AURA
task.spawn(function()
    while true do
        task.wait(0.15)
        if S.killAura and H.alive() then
            local _, hrp = H.char()
            if hrp then
                local mobs = H.mobs(S.killAuraRange)
                for _, m in ipairs(mobs) do
                    if m.hum.Health > 0 then
                        H.attack(m.root.Position)
                        S.attackCount = S.attackCount + 1
                    end
                end
            end
        end
    end
end)

-- AUTO BOSS
task.spawn(function()
    while true do
        task.wait(0.4)
        if S.autoBoss and H.alive() then
            local bosses = H.bosses(S.bossRange)
            local t = bosses[1]
            if t then
                S.lastTarget = "[BOSS] " .. t.rig.Name
                if t.dist > 10 then
                    if S.farmMode == "Teleport" then
                        H.safeTP(t.root.Position)
                    else
                        H.moveTo(t.root.Position)
                    end
                else
                    H.faceTarget(t.root.Position)
                    H.attack(t.root.Position)
                    S.attackCount = S.attackCount + 1
                end
            end
        end
    end
end)

-- ANTI-CHEAT
AntiCheat.onFail = function()
    Notify.warn("Anti-Cheat", "Teleporte detectado! Pausando 8s", 5)
    if S.autoFarm then S.autoFarm = false end
    if S.autoBoss then S.autoBoss = false end
    task.spawn(function()
        task.wait(8)
        AntiCheat.reset()
    end)
end

task.spawn(function()
    while true do
        task.wait(0.5)
        if S.autoFarm or S.killAura or S.autoBoss then
            AntiCheat.check()
        end
    end
end)

-- UI: ABA FARM
Tabs.Farm:AddSection("Anti Dano (Patch)")

addToggleWithSound(Tabs.Farm, "AntiSun", {
    Title = "Anti Sun Damage",
    Description = "Demonio nao toma dano de sol",
    Default = false,
    Callback = function(v) S.antiSun = v end,
})

addToggleWithSound(Tabs.Farm, "AntiDrown", {
    Title = "Anti Drown",
    Description = "Nao afoga embaixo d'agua",
    Default = false,
    Callback = function(v) S.antiDrown = v end,
})

Tabs.Farm:AddSection("Auto Heal")

addToggleWithSound(Tabs.Farm, "AutoHeal", {
    Title = "Auto Heal",
    Description = "Bebe pocao quando a vida baixa",
    Default = false,
    Callback = function(v) S.autoHeal = v end,
})

Tabs.Farm:AddSlider("HealPct", {
    Title = "Curar abaixo de",
    Description = "Porcentagem de vida pra beber pocao",
    Default = 45, Min = 10, Max = 95, Rounding = 0, Suffix = "%",
    Callback = function(v) S.healPct = v end,
})

Tabs.Farm:AddInput("HealPotion", {
    Title = "Pocao",
    Description = "Nome exato da pocao no inventario",
    Default = "Health Potion",
    Placeholder = "Health Potion",
    Callback = function(v) S.healPotion = v end,
})

Tabs.Farm:AddSection("Auto Farm")

addToggleWithSound(Tabs.Farm, "AutoFarm", {
    Title = "Auto Farm (Kill Mobs)",
    Description = "Ataca o mob mais proximo pra ganhar XP",
    Default = false,
    Callback = function(v) S.autoFarm = v end,
})

Tabs.Farm:AddDropdown("FarmMode", {
    Title = "Modo de Movimento",
    Description = "MoveTo = anda (seguro) | Teleport = 30 studs (risco)",
    Values = { "MoveTo", "Teleport" },
    Default = "MoveTo",
    Multi = false,
    Callback = function(v) S.farmMode = v end,
})

Tabs.Farm:AddDropdown("FarmPriority", {
    Title = "Prioridade de Alvo",
    Description = "Qual mob escolher primeiro",
    Values = { "Closest", "Lowest HP", "Highest HP" },
    Default = "Closest",
    Multi = false,
    Callback = function(v) S.farmPriority = v end,
})

Tabs.Farm:AddSlider("FarmRange", {
    Title = "Alcance de Busca",
    Description = "Raio em studs pra procurar mobs",
    Default = 50, Min = 10, Max = 200, Rounding = 0, Suffix = " studs",
    Callback = function(v) S.farmRange = v end,
})

Tabs.Farm:AddSlider("FarmDelay", {
    Title = "Delay entre ataques",
    Description = "Segundos entre ciclos (maior = mais seguro)",
    Default = 0.3, Min = 0.1, Max = 2, Rounding = 2, Suffix = "s",
    Callback = function(v) S.farmDelay = v end,
})

local farmStatus = Panels.create(Tabs.Farm, "Auto Farm: Off", "Attacks: 0 | Kills: 0")

-- UI: ABA COMBAT
Tabs.Combat:AddSection("Kill Aura")

addToggleWithSound(Tabs.Combat, "KillAura", {
    Title = "Kill Aura",
    Description = "Ataca TODOS os mobs no alcance ao mesmo tempo",
    Default = false,
    Callback = function(v) S.killAura = v end,
})

Tabs.Combat:AddSlider("KillAuraRange", {
    Title = "Alcance",
    Description = "Raio em studs pra atacar",
    Default = 15, Min = 5, Max = 50, Rounding = 0, Suffix = " studs",
    Callback = function(v) S.killAuraRange = v end,
})

Tabs.Combat:AddSection("Auto Boss")

addToggleWithSound(Tabs.Combat, "AutoBoss", {
    Title = "Auto Boss",
    Description = "Ataca bosses automaticamente",
    Default = false,
    Callback = function(v) S.autoBoss = v end,
})

Tabs.Combat:AddSlider("BossRange", {
    Title = "Alcance de Boss",
    Description = "Raio em studs pra procurar bosses",
    Default = 100, Min = 30, Max = 500, Rounding = 0, Suffix = " studs",
    Callback = function(v) S.bossRange = v end,
})

Tabs.Combat:AddSection("Info")
Tabs.Combat:AddParagraph({
    Title = "Sobre o Combate",
    Content = "Kill Aura ataca tudo no raio. Auto Boss foca bosses.\nCombine Kill Aura + Auto Heal pra farmar bosses.",
})

local combatStatus = Panels.create(Tabs.Combat, "Combat: Off", "Attacks: 0")

-- STATUS LOOP
task.spawn(function()
    while true do
        task.wait(1)
        pcall(function()
            farmStatus:update(
                "Auto Farm: " .. (S.autoFarm and "ON" or "Off"),
                string.format("Attacks: %d | Kills: %d | Target: %s",
                    S.attackCount, S.kills, S.lastTarget)
            )
            combatStatus:update(
                "Combat: " .. ((S.killAura or S.autoBoss) and "ON" or "Off"),
                string.format("Kill Aura: %s | Auto Boss: %s | Attacks: %d",
                    S.killAura and "OK" or "X",
                    S.autoBoss and "OK" or "X",
                    S.attackCount)
            )
        end)
    end
end)

-- EXPORTA
Core.Farm = S
Core.FarmPanels = {
    farm = farmStatus,
    combat = combatStatus,
}

-- BOOT
Notify.success("Farm & Combat", "Parte 3 carregada!", 4)
Logger.log("Parte 3/6 (Farm & Combat) carregada")
Logger.log("Pronto para a Parte 4/6")-- =====================================================================
-- KYÔKA HUB v4.0 • PARTE 4/6 — SKILLS & FISHING
-- Project Slayers 2 • Standalone
-- =====================================================================

local Core = _G.__KyokaHub
if not Core then warn("[Kyôka] Rode a Parte 1 primeiro!") return end

local Services = Core.Services
local Game = Core.Game
local H = Core.H
local Logger = Core.Logger
local Notify = Core.Notify
local Panels = Core.Panels
local Tabs = Core.Tabs
local Options = Core.Options
local Sound = Core.Sound
local EventBus = Core.EventBus
local addToggleWithSound = Core.addToggleWithSound

local LP = Services.Players.LocalPlayer
local CS = Services.CollectionService

Logger.log("Iniciando Parte 4/6 (Skills & Fishing)")

-- ESTADO GLOBAL
local S = {
    autoSkill = false,
    skillHold = 0.4,
    skillMode = "While fighting",
    skillFilter = {},
    autoSkillTree = false,
    autoFish = false,
    fishCaught = 0,
    fishStatus = "Off",
    autoLoot = false,
    lootRange = 100,
    lootCollected = 0,
    casts = 0,
    lastSkill = "-",
}

-- AIM HOOK
local aimAt = nil

do
    if Game.Platform_Handler and Game.Platform_Handler.mousepos then
        local orig = Game.Platform_Handler.mousepos
        Game.Platform_Handler.mousepos = function(...)
            if aimAt then
                local ok, pos = pcall(aimAt)
                if ok and typeof(pos) == "Vector3" then return pos end
            end
            return orig(...)
        end
        Logger.log("Aim hook instalado")
    else
        Logger.warn("Platform_Handler.mousepos nao disponivel")
    end
end

-- AUTO SKILL
local function castSkill(skill, rig)
    local C = Game.Skill_Controller
    if not C or not C.Attempt_Hold then return false end

    aimAt = function()
        if rig and rig.Parent then
            local r = rig:FindFirstChild("HumanoidRootPart")
            if r then return r.Position end
        end
        local _, hrp = H.char()
        return hrp and (hrp.Position + hrp.CFrame.LookVector * 20) or nil
    end

    local ok, pressed = pcall(C.Attempt_Hold, skill.name, nil)
    if not ok or pressed ~= true then
        aimAt = nil
        return false
    end

    C.CurrentMax = skill.maxHold
    C.HeldSkill = skill.name

    local char = LP.Character
    local shc = char and (char:FindFirstChild("SHC") or char:FindFirstChild("SHCS"))
    local hold = math.min(S.skillHold, skill.maxHold or 0)
    local dl = os.clock() + hold

    while shc and shc.Value == skill.name and os.clock() < dl do
        task.wait(0.05)
    end

    if shc and shc.Value == skill.name then
        C.UnHoldBoolean = true
        local rel = os.clock() + 1
        while shc.Value == skill.name and os.clock() < rel do
            task.wait(0.05)
        end
    end

    aimAt = nil
    S.casts = S.casts + 1
    S.lastSkill = skill.name
    return true
end

task.spawn(function()
    while true do
        task.wait(0.25)
        if S.autoSkill and H.alive() and not H.isCasting() then
            local mobs = H.mobs(40)
            local rig = mobs[1] and mobs[1].rig

            if S.skillMode == "While fighting" and not rig then
                -- Sem alvo, espera
            else
                for _, s in ipairs(H.skillBar()) do
                    if s.name ~= "Blocking" then
                        local allowed = next(S.skillFilter) == nil or S.skillFilter[s.name]
                        if allowed then
                            if castSkill(s, rig) then break end
                        end
                    end
                end
            end
        end
    end
end)

-- AUTO SKILL TREE
task.spawn(function()
    while true do
        task.wait(2)
        if S.autoSkillTree and H.alive() then
            local points = H.skillPoints()
            if points > 0 and Game.SignalFunction then
                local skillStats = Game.SkillStats
                if skillStats and skillStats.GetRequirements then
                    local ok, unlocked = pcall(function()
                        return skillStats.IsSkillUnlocked(LP, "Double Jump")
                    end)
                    if ok and not unlocked then
                        pcall(function()
                            Game.SignalFunction.ToServer("UnlockSkillTreeNode", "Double Jump")
                        end)
                    end
                end
            end
        end
    end
end)

-- AUTO FISH
local biteToken = nil
local PortalEvent = Game.PortalEvent

if PortalEvent then
    PortalEvent.OnClientEvent:Connect(function(ch, kind, token)
        if ch == "FishingRod" and kind == "Bite" then
            biteToken = token
        end
    end)
end

task.spawn(function()
    while true do
        task.wait(0.5)
        if S.autoFish and PortalEvent and Game.SignalEvent and H.alive() then
            local water = H.findWater(32)
            if not water then
                S.fishStatus = "Sem agua perto"
            else
                S.fishStatus = "Pescando..."
                biteToken = nil
                pcall(function()
                    Game.SignalEvent.ToServer("Tool_Mouse", "Up", water)
                end)

                local dl = os.clock() + 30
                while S.autoFish and not biteToken and os.clock() < dl do
                    task.wait(0.1)
                end

                if biteToken then
                    S.fishStatus = "Reeling..."
                    task.wait(4.7)
                    if S.autoFish and PortalEvent then
                        pcall(function()
                            PortalEvent:FireServer("FishingRod", biteToken, true)
                        end)
                        S.fishCaught = S.fishCaught + 1
                        S.fishStatus = "Pescou! Total: " .. S.fishCaught
                        task.wait(2)
                    end
                else
                    S.fishStatus = "Sem bite"
                end
            end
        else
            S.fishStatus = "Off"
        end
    end
end)

-- AUTO LOOT
local function mine(drop)
    local owner = drop:GetAttribute("DropOwnerUserId")
    if typeof(owner) == "number" and owner ~= LP.UserId then
        return false
    end
    local reserved = drop:GetAttribute("DropReservedFor")
    if typeof(reserved) == "string" then
        if not string.find(reserved, "," .. LP.UserId .. ",", 1, true) then
            return false
        end
    end
    return drop:GetAttribute("DropClaimedBy") == nil
end

task.spawn(function()
    while true do
        task.wait(0.5)
        if S.autoLoot and H.alive() then
            local _, hrp = H.char()
            if hrp then
                local drops = CS:GetTagged("LootDrop")
                for _, drop in ipairs(drops) do
                    if drop:IsA("BasePart") and drop.Parent and mine(drop) then
                        local d = (drop.Position - hrp.Position).Magnitude
                        if d <= S.lootRange then
                            pcall(function()
                                hrp.CFrame = CFrame.new(drop.Position + Vector3.new(0, 3, 0))
                            end)
                            S.lootCollected = S.lootCollected + 1
                            break
                        end
                    end
                end
            end
        end
    end
end)

-- UI: ABA SKILLS
Tabs.Skills:AddSection("Auto Skill")

addToggleWithSound(Tabs.Skills, "AutoSkill", {
    Title = "Auto Skill",
    Description = "Usa skills automaticamente no mob mais proximo",
    Default = false,
    Callback = function(v) S.autoSkill = v end,
})

Tabs.Skills:AddDropdown("SkillMode", {
    Title = "Modo de Uso",
    Description = "While fighting = so em combate | Always = sempre",
    Values = { "While fighting", "Always" },
    Default = "While fighting",
    Multi = false,
    Callback = function(v) S.skillMode = v end,
})

Tabs.Skills:AddSlider("SkillHold", {
    Title = "Hold Time (skills que carregam)",
    Description = "Quanto tempo segurar skills de carga",
    Default = 0.4, Min = 0, Max = 3, Rounding = 1, Suffix = "s",
    Callback = function(v) S.skillHold = v end,
})

local skillStatus = Panels.create(Tabs.Skills, "Auto Skill: Off", "Casts: 0 | Last: -")

Tabs.Skills:AddSection("Auto Skill Tree")

addToggleWithSound(Tabs.Skills, "AutoSkillTree", {
    Title = "Auto Skill Tree",
    Description = "Gasta skill points automaticamente",
    Default = false,
    Callback = function(v) S.autoSkillTree = v end,
})

Tabs.Skills:AddSection("Skill List")
local skillListPanel = Panels.create(Tabs.Skills, "Skills no Bar", "-")
Tabs.Skills:AddButton({
    Title = "Atualizar Lista de Skills",
    Description = "Recarrega as skills do seu bar atual",
    Callback = function()
        local bar = H.skillBar()
        local names = {}
        for _, s in ipairs(bar) do
            if s.name ~= "Blocking" then
                names[#names+1] = s.name
            end
        end
        skillListPanel:update("Skills no Bar (" .. #names .. ")",
            #names > 0 and table.concat(names, ", ") or "Nenhuma")
        Notify.info("Skills", #names .. " skills encontradas", 3)
    end,
})

-- UI: ABA FISHING
Tabs.Fishing:AddSection("Auto Fish")

addToggleWithSound(Tabs.Fishing, "AutoFish", {
    Title = "Auto Fishing",
    Description = "Acha agua, casta e recolhe automaticamente",
    Default = false,
    Callback = function(v) S.autoFish = v end,
})

local fishStatus = Panels.create(Tabs.Fishing, "Auto Fish: Off", "Caught: 0")

Tabs.Fishing:AddParagraph({
    Title = "Como Usar",
    Content = "Fique em pe perto de agua (dock, praia) antes de ligar.\nO hub vai castar, esperar o bite, e usar instant reel.",
})

Tabs.Fishing:AddSection("Auto Loot")

addToggleWithSound(Tabs.Fishing, "AutoLoot", {
    Title = "Auto Loot",
    Description = "Coleta drops no chao automaticamente",
    Default = false,
    Callback = function(v) S.autoLoot = v end,
})

Tabs.Fishing:AddSlider("LootRange", {
    Title = "Alcance de Coleta",
    Description = "Raio em studs pra coletar drops",
    Default = 100, Min = 20, Max = 500, Rounding = 0, Suffix = " studs",
    Callback = function(v) S.lootRange = v end,
})

local lootStatus = Panels.create(Tabs.Fishing, "Auto Loot: Off", "Collected: 0")

-- STATUS LOOP
task.spawn(function()
    while true do
        task.wait(1)
        pcall(function()
            skillStatus:update(
                "Auto Skill: " .. (S.autoSkill and "ON" or "Off"),
                string.format("Casts: %d | Last: %s", S.casts, S.lastSkill)
            )
            fishStatus:update(
                "Auto Fish: " .. (S.autoFish and "ON" or "Off"),
                "Caught: " .. S.fishCaught .. " | " .. S.fishStatus
            )
            lootStatus:update(
                "Auto Loot: " .. (S.autoLoot and "ON" or "Off"),
                "Collected: " .. S.lootCollected
            )
        end)
    end
end)

-- EXPORTA
Core.Skills = S
Core.SkillPanels = {
    skill = skillStatus,
    fish = fishStatus,
    loot = lootStatus,
}

-- BOOT
Notify.success("Skills & Fishing", "Parte 4 carregada!", 4)
Logger.log("Parte 4/6 (Skills & Fishing) carregada")
Logger.log("Pronto para a Parte 5/6")-- =====================================================================
-- KYÔKA HUB v4.0 • PARTE 5/6 — PLAYER & PROGRESS
-- Project Slayers 2 • Standalone
-- =====================================================================

local Core = _G.__KyokaHub
if not Core then warn("[Kyôka] Rode a Parte 1 primeiro!") return end

local Services = Core.Services
local Game = Core.Game
local H = Core.H
local Logger = Core.Logger
local Notify = Core.Notify
local Panels = Core.Panels
local Tabs = Core.Tabs
local Options = Core.Options
local Sound = Core.Sound
local addToggleWithSound = Core.addToggleWithSound
local AntiCheat = Core.AntiCheat

local LP = Services.Players.LocalPlayer

Logger.log("Iniciando Parte 5/6 (Player & Progress)")

-- ESTADO GLOBAL
local S = {
    showHUD = true,
    autoQuest = false,
    questTP = false,
    questStep = "idle",
    questMsg = "Idle",
    questsDone = 0,
    questFails = 0,
    autoAcc = false,
    autoSpin = false,
    spinDelay = 0.4,
    stopRarity = "Mythic",
    spinsDone = 0,
    autoCodes = false,
    codesRedeemed = 0,
}

-- AUTO QUEST
local questState = {
    step = "idle",
    quest = nil,
    npc = nil,
    npcPos = nil,
    lastPos = nil,
}

task.spawn(function()
    while true do
        task.wait(0.4)
        if S.autoQuest and H.alive() then
            local _, hrp, hum = H.char()
            if hrp and hum then
                if questState.lastPos and (hrp.Position - questState.lastPos).Magnitude > 100 then
                    S.questMsg = "AC detectado, pausando"
                    S.questFails = S.questFails + 1
                    if S.questFails >= 3 then
                        S.autoQuest = false
                        Notify.error("Auto Quest", "Parado apos 3 falhas de AC", 6)
                    end
                    task.wait(8)
                end
                questState.lastPos = hrp.Position

                if questState.step == "idle" then
                    local pick = H.chooseQuest()
                    if not pick then
                        S.questMsg = "Sem quest disponivel (lvl " .. H.level() .. ")"
                        task.wait(3)
                    else
                        questState.quest = pick
                        questState.npc = pick.npc
                        questState.npcPos = H.npcSpawn(pick.npc)
                        if not questState.npcPos then
                            S.questMsg = "NPC sem spawn"
                            task.wait(3)
                        else
                            questState.step = "goNPC"
                            S.questMsg = "-> " .. pick.npc
                        end
                    end

                elseif questState.step == "goNPC" then
                    local pos = H.npcSpawn(questState.npc) or questState.npcPos
                    local d = (hrp.Position - pos).Magnitude
                    if d > 8 then
                        if S.questTP then H.safeTP(pos) else H.moveTo(pos) end
                        S.questMsg = string.format("-> %s (%d studs)", questState.npc, math.floor(d))
                    else
                        questState.step = "accept"
                    end

                elseif questState.step == "accept" then
                    H.moveTo(questState.npcPos)
                    task.wait(0.4)
                    local model = H.npcModel(questState.npc)
                    if model then H.interactNPC(model) end
                    S.questMsg = "Aceitando quest..."
                    task.wait(2)
                    questState.step = "farm"

                elseif questState.step == "farm" then
                    local task = H.currentTask(questState.quest)
                    if not task then
                        S.questMsg = "Completo -> voltando"
                        questState.step = "return"
                    else
                        local mobs = H.mobs(150)
                        if #mobs > 0 then
                            local t = mobs[1]
                            if t.dist > 8 then
                                if S.questTP then H.safeTP(t.root.Position) else H.moveTo(t.root.Position) end
                                S.questMsg = string.format("X %s (%d studs)", t.rig.Name, math.floor(t.dist))
                            else
                                H.faceTarget(t.root.Position)
                                H.attack(t.root.Position)
                                S.questMsg = "X Atacando " .. t.rig.Name
                            end
                        else
                            S.questMsg = "Sem mobs perto"
                            task.wait(1)
                        end
                    end

                elseif questState.step == "return" then
                    local pos = H.npcSpawn(questState.npc) or questState.npcPos
                    local d = (hrp.Position - pos).Magnitude
                    if d > 8 then
                        if S.questTP then H.safeTP(pos) else H.moveTo(pos) end
                        S.questMsg = string.format("<- %s (%d studs)", questState.npc, math.floor(d))
                    else
                        questState.step = "complete"
                    end

                elseif questState.step == "complete" then
                    H.moveTo(questState.npcPos)
                    task.wait(0.4)
                    local model = H.npcModel(questState.npc)
                    if model then H.interactNPC(model) end
                    S.questMsg = "Entregando..."
                    task.wait(2)
                    S.questsDone = S.questsDone + 1
                    S.questFails = 0
                    questState.step = "idle"
                end
            else
                S.questMsg = "Aguardando respawn"
            end
        else
            if not S.autoQuest then
                questState.step = "idle"
                S.questMsg = "Off"
            end
        end
    end
end)

-- AUTO ACCESSORIES (placeholder)
task.spawn(function()
    while true do
        task.wait(5)
        if S.autoAcc and H.alive() then
            -- Logica a ser implementada
        end
    end
end)

-- CLAN AUTO SPIN
task.spawn(function()
    while true do
        task.wait(S.spinDelay)
        if S.autoSpin and Game.SignalFunction and H.alive() then
            local ok, rolled = pcall(function()
                return Game.SignalFunction.ToServer("ClanSpin")
            end)
            if ok and rolled then
                S.spinsDone = S.spinsDone + 1
                pcall(function()
                    Game.SignalEvent.ToServer("ClanSpinComplete")
                end)
            end
        end
    end
end)

-- AUTO CODES
task.spawn(function()
    task.wait(5)
    if not S.autoCodes then return end
    if not Game.SignalFunction then return end
    local ok, status = pcall(function()
        return Game.SignalFunction.ToServer("CodeStatus")
    end)
    if not ok or type(status) ~= "table" or type(status.codes) ~= "table" then
        Notify.warn("Codes", "Nao conseguiu carregar lista de codes", 5)
        return
    end
    for code, info in pairs(status.codes) do
        if not info.redeemed then
            local upper = string.upper(code)
            if not string.find(upper, "RESET", 1, true)
                and not string.find(upper, "WIPE", 1, true)
                and (string.find(upper, "SPIN", 1, true) or string.find(upper, "ROLL", 1, true)) then
                pcall(function()
                    Game.SignalFunction.ToServer("RedeemCode", code)
                end)
                S.codesRedeemed = S.codesRedeemed + 1
                task.wait(0.75)
            end
        end
    end
    if S.codesRedeemed > 0 then
        Notify.success("Codes", S.codesRedeemed .. " codes resgatados", 5)
    end
end)

-- HUD DE PLAYER
local PlayerHUD = { gui = nil, labels = {} }

function PlayerHUD.create()
    if Services.CoreGui:FindFirstChild("KyokaHub_PlayerHUD") then
        Services.CoreGui.KyokaHub_PlayerHUD:Destroy()
    end

    local gui = Instance.new("ScreenGui")
    gui.Name = "KyokaHub_PlayerHUD"
    gui.ResetOnSpawn = false
    gui.DisplayOrder = 4
    gui.Parent = Services.CoreGui

    local frame = Instance.new("Frame")
    frame.Size = UDim2.fromOffset(220, 140)
    frame.Position = UDim2.new(0, 15, 0, 15)
    frame.BackgroundColor3 = Color3.fromRGB(18, 18, 22)
    frame.BackgroundTransparency = 0.3
    frame.BorderSizePixel = 0
    frame.Parent = gui

    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, 10)
    c.Parent = frame

    local s = Instance.new("UIStroke")
    s.Color = Color3.fromRGB(220, 40, 60)
    s.Thickness = 1
    s.Transparency = 0.4
    s.Parent = frame

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -16, 0, 22)
    title.Position = UDim2.fromOffset(8, 6)
    title.BackgroundTransparency = 1
    title.Text = "Kyoka Hub • Player"
    title.TextColor3 = Color3.fromRGB(240, 240, 245)
    title.TextSize = 14
    title.Font = Enum.Font.GothamBold
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = frame

    local function line(y)
        local l = Instance.new("TextLabel")
        l.Size = UDim2.new(1, -16, 0, 18)
        l.Position = UDim2.fromOffset(8, y)
        l.BackgroundTransparency = 1
        l.Text = "-"
        l.TextColor3 = Color3.fromRGB(200, 200, 210)
        l.TextSize = 12
        l.Font = Enum.Font.Gotham
        l.TextXAlignment = Enum.TextXAlignment.Left
        l.Parent = frame
        return l
    end

    PlayerHUD.gui = gui
    PlayerHUD.labels.level = line(30)
    PlayerHUD.labels.race = line(50)
    PlayerHUD.labels.clan = line(70)
    PlayerHUD.labels.wen = line(90)
    PlayerHUD.labels.quest = line(110)
end

PlayerHUD.create()

task.spawn(function()
    while true do
        task.wait(1)
        if S.showHUD and PlayerHUD.gui then
            pcall(function()
                local lvl = H.level()
                local race = tostring(H.race())
                local clan = tostring(H.clan())
                local wen = H.wen()
                local cur, goal = H.exp()
                local pct = goal > 0 and math.floor(cur / goal * 100) or 0
                PlayerHUD.labels.level.Text = string.format("Level: %d (%d%%)", lvl, pct)
                PlayerHUD.labels.race.Text = "Raca: " .. race
                PlayerHUD.labels.clan.Text = "Cla: " .. clan
                PlayerHUD.labels.wen.Text = "Wen: " .. tostring(wen)
                PlayerHUD.labels.quest.Text = "Quest: " .. S.questMsg
            end)
        end
    end
end)

-- UI: ABA PLAYER
Tabs.Player:AddSection("Info do Personagem")

local playerPanel = Panels.create(Tabs.Player, "Level: ...", "Carregando...")

Tabs.Player:AddToggle("ShowHUD", {
    Title = "Mostrar HUD na tela",
    Description = "Painel com stats no canto superior esquerdo",
    Default = true,
    Callback = function(v)
        S.showHUD = v
        if PlayerHUD.gui then PlayerHUD.gui.Enabled = v end
    end,
})

Tabs.Player:AddSection("Auto Accessories")

addToggleWithSound(Tabs.Player, "AutoAcc", {
    Title = "Auto Accessories",
    Description = "Equipa os melhores acessorios automaticamente",
    Default = false,
    Callback = function(v) S.autoAcc = v end,
})

Tabs.Player:AddSection("Clan Auto Spin")

addToggleWithSound(Tabs.Player, "AutoSpin", {
    Title = "Auto Clan Spin",
    Description = "Gira o cla automaticamente ate a raridade alvo",
    Default = false,
    Callback = function(v) S.autoSpin = v end,
})

Tabs.Player:AddDropdown("StopRarity", {
    Title = "Parar em Raridade",
    Values = { "Common", "Uncommon", "Rare", "Epic", "Legendary", "Mythic" },
    Default = "Mythic",
    Multi = false,
    Callback = function(v) S.stopRarity = v end,
})

Tabs.Player:AddSlider("SpinDelay", {
    Title = "Delay de Spin",
    Default = 0.4, Min = 0.1, Max = 3, Rounding = 1, Suffix = "s",
    Callback = function(v) S.spinDelay = v end,
})

local spinPanel = Panels.create(Tabs.Player, "Clan Spin: Off", "Spins: 0")

-- UI: ABA PROGRESS
Tabs.Progress:AddSection("Auto Quest (Loop Completo)")

addToggleWithSound(Tabs.Progress, "AutoQuest", {
    Title = "Auto Quest",
    Description = "Le level -> aceita -> farma -> entrega -> repete",
    Default = false,
    Callback = function(v) S.autoQuest = v end,
})

addToggleWithSound(Tabs.Progress, "QuestTP", {
    Title = "Usar Teleporte",
    Description = "OFF = anda (seguro) | ON = teleporta 30 studs (risco)",
    Default = false,
    Callback = function(v) S.questTP = v end,
})

local questPanel = Panels.create(Tabs.Progress, "Auto Quest: Off", "Quests: 0")

Tabs.Progress:AddSection("Auto Codes")

addToggleWithSound(Tabs.Progress, "AutoCodes", {
    Title = "Auto Codes",
    Description = "Resgata codes de spin/reroll automaticamente",
    Default = false,
    Callback = function(v) S.autoCodes = v end,
})

local codesPanel = Panels.create(Tabs.Progress, "Codes: 0 resgatados", "-")

Tabs.Progress:AddSection("Quests Disponiveis")

local questListPanel = Panels.create(Tabs.Progress, "Quests", "Clique em atualizar")

Tabs.Progress:AddButton({
    Title = "Atualizar Lista de Quests",
    Description = "Mostra as quests disponiveis no seu level",
    Callback = function()
        if not Game.Quests then
            Notify.warn("Quests", "Modulo de quests indisponivel", 3)
            return
        end
        local lvl = H.level()
        local count = 0
        local list = {}
        for key, def in pairs(Game.Quests.Holder) do
            if type(def) == "table" and type(def.OfferNpc) == "string" then
                local req = def.Requirements
                local rl = type(req) == "table" and (tonumber(req.Level) or 0) or 0
                if rl <= lvl and rl >= lvl - 10 then
                    count = count + 1
                    if count <= 10 then
                        list[#list+1] = string.format("Lv%d: %s (%s)", rl, tostring(key), def.OfferNpc)
                    end
                end
            end
        end
        questListPanel:update("Quests no seu level (" .. count .. ")",
            #list > 0 and table.concat(list, "\n") or "Nenhuma")
    end,
})

-- STATUS LOOP
task.spawn(function()
    while true do
        task.wait(1)
        pcall(function()
            local lvl = H.level()
            local race = tostring(H.race())
            local clan = tostring(H.clan())
            local wen = H.wen()
            local cur, goal = H.exp()
            local pct = goal > 0 and math.floor(cur / goal * 100) or 0
            playerPanel:update(
                "Level: " .. lvl .. " (" .. pct .. "%)",
                string.format("Raca: %s\nCla: %s\nWen: %d\nXP: %d / %d",
                    race, clan, wen, cur, goal)
            )
            questPanel:update(
                "Auto Quest: " .. (S.autoQuest and S.questMsg or "Off"),
                string.format("Quests completas: %d | Fails: %d", S.questsDone, S.questFails)
            )
            spinPanel:update(
                "Clan Spin: " .. (S.autoSpin and "ON" or "Off"),
                "Spins: " .. S.spinsDone
            )
            codesPanel:update(
                "Codes: " .. S.codesRedeemed .. " resgatados",
                S.autoCodes and "Resgatando..." or "Off"
            )
        end)
    end
end)

-- EXPORTA
Core.Progress = S
Core.PlayerHUD = PlayerHUD
Core.ProgressPanels = {
    player = playerPanel,
    quest = questPanel,
    spin = spinPanel,
    codes = codesPanel,
    questList = questListPanel,
}

-- BOOT
Notify.success("Player & Progress", "Parte 5 carregada!", 4)
Logger.log("Parte 5/6 (Player & Progress) carregada")
Logger.log("Pronto para a Parte 6/6")-- =====================================================================
-- KYÔKA HUB v4.0 • PARTE 6/6 — SETTINGS, TELEPORT & FINALIZE
-- Project Slayers 2 • Standalone
-- =====================================================================

local Core = _G.__KyokaHub
if not Core then warn("[Kyôka] Rode a Parte 1 primeiro!") return end

local Services = Core.Services
local Game = Core.Game
local H = Core.H
local Logger = Core.Logger
local Notify = Core.Notify
local Panels = Core.Panels
local Tabs = Core.Tabs
local Options = Core.Options
local Sound = Core.Sound
local CatButton = Core.CatButton
local HubStatus = Core.HubStatus
local addToggleWithSound = Core.addToggleWithSound
local Config = Core.Config

local LP = Services.Players.LocalPlayer
local UIS = Services.UserInputService

Logger.log("Iniciando Parte 6/6 (Settings & Finalize)")

-- ESTADO GLOBAL
local S = {
    teleportMethod = "Instant",
    soundEnabled = true,
    soundVolume = 0.35,
    version = "4.0",
    startTime = os.clock(),
    loadTime = 0,
}

-- TEMA CUSTOMIZADO
local CustomTheme = {
    current = "Dark",
    available = { "Dark", "Light", "Rose", "AMOLED", "Aqua" },
    apply = function(name)
        pcall(function()
            if name == "Aqua" then
                if Core.Fluent and Core.Fluent.SetTheme then
                    Core.Fluent:SetTheme("Dark")
                end
            else
                if Core.Fluent and Core.Fluent.SetTheme then
                    Core.Fluent:SetTheme(name)
                end
            end
        end)
        CustomTheme.current = name
        Config.set("theme", name)
    end,
}

-- TELEPORT
local Teleport = {}

function Teleport.toNPC(name)
    local pos = H.npcSpawn(name)
    if not pos then return false, "NPC nao encontrado" end
    local _, hrp = H.char()
    if not hrp then return false, "Sem personagem" end
    if S.teleportMethod == "Instant" then
        hrp.CFrame = CFrame.new(pos + Vector3.new(0, 5, 0))
    else
        task.spawn(function()
            local start = hrp.CFrame
            local endCF = CFrame.new(pos + Vector3.new(0, 5, 0))
            for i = 0, 1, 0.05 do
                hrp.CFrame = start:Lerp(endCF, i)
                task.wait(0.02)
            end
        end)
    end
    return true
end

function Teleport.toCoords(x, y, z)
    local _, hrp = H.char()
    if not hrp then return false, "Sem personagem" end
    hrp.CFrame = CFrame.new(Vector3.new(x, y, z))
    return true
end

-- UI: ABA TELEPORT
Tabs.Teleport:AddSection("NPCs de Quest")

Tabs.Teleport:AddDropdown("TpNPC", {
    Title = "Escolher NPC",
    Description = "Teleporta pro NPC de quest correspondente",
    Values = { "Krue", "Tom", "Chaka", "Wagwan", "Jugg", "Goro", "Tomoi", "Mitsu", "KasugaiCrow" },
    Default = "Krue",
    Multi = false,
})

Tabs.Teleport:AddButton({
    Title = "Teleportar para NPC",
    Description = "Vai pra posicao do NPC selecionado",
    Callback = function()
        local n = Options.TpNPC.Value
        local ok, err = Teleport.toNPC(n)
        if ok then
            Notify.success("Teleport", "Foi pra " .. n, 3)
        else
            Notify.error("Teleport", tostring(err), 4)
        end
    end,
})

Tabs.Teleport:AddSection("Coordenadas Customizadas")

Tabs.Teleport:AddInput("TpX", { Title = "X", Default = "0", Placeholder = "0" })
Tabs.Teleport:AddInput("TpY", { Title = "Y", Default = "0", Placeholder = "0" })
Tabs.Teleport:AddInput("TpZ", { Title = "Z", Default = "0", Placeholder = "0" })

Tabs.Teleport:AddButton({
    Title = "Ir para Coordenadas",
    Description = "Teleporta pra posicao X, Y, Z",
    Callback = function()
        local x = tonumber(Options.TpX.Value) or 0
        local y = tonumber(Options.TpY.Value) or 0
        local z = tonumber(Options.TpZ.Value) or 0
        local ok = Teleport.toCoords(x, y, z)
        if ok then
            Notify.success("Teleport", string.format("Foi pra %.0f, %.0f, %.0f", x, y, z), 3)
        else
            Notify.error("Teleport", "Sem personagem", 3)
        end
    end,
})

Tabs.Teleport:AddSection("Coordenadas Rapidas")

local QUICK_PLACES = {
    { name = "Spawn", x = 0, y = 300, z = 0 },
    { name = "Final Selection", x = 500, y = 1000, z = 500 },
    { name = "Urokodaki House", x = -200, y = 300, z = 800 },
    { name = "Butterfly Mansion", x = 800, y = 300, z = -400 },
    { name = "Zahab Town", x = 1500, y = 300, z = 300 },
}

for _, place in ipairs(QUICK_PLACES) do
    Tabs.Teleport:AddButton({
        Title = "-> " .. place.name,
        Description = string.format("Ir para %s", place.name),
        Callback = function()
            Teleport.toCoords(place.x, place.y, place.z)
            Notify.success("Teleport", "Foi pra " .. place.name, 3)
        end,
    })
end

Tabs.Teleport:AddSection("Metodo de Teleport")

Tabs.Teleport:AddDropdown("TpMethod", {
    Title = "Metodo",
    Description = "Instant = teleporta direto | Tween = movimento suave",
    Values = { "Instant", "Tween" },
    Default = "Instant",
    Multi = false,
    Callback = function(v) S.teleportMethod = v end,
})

-- UI: ABA SETTINGS
Tabs.Settings:AddSection("Interface")

addToggleWithSound(Tabs.Settings, "HideCat", {
    Title = "Esconder Botao do Gato",
    Description = "Oculta o icone do gato (reabra com RightShift)",
    Default = false,
    Callback = function(v)
        if CatButton and CatButton.setVisible then
            CatButton.setVisible(not v)
        end
    end,
})

Tabs.Settings:AddKeybind("ToggleKey", {
    Title = "Abrir/Fechar Hub",
    Description = "Tecla pra abrir/fechar a janela do hub",
    Mode = "Toggle",
    Default = "RightShift",
})

Tabs.Settings:AddSection("Som")

addToggleWithSound(Tabs.Settings, "SoundEnabled", {
    Title = "Sons da Interface",
    Description = "Liga/desliga sons de click, toggle, notificacao",
    Default = true,
    Callback = function(v)
        S.soundEnabled = v
        if Sound then Sound.enabled = v end
    end,
})

Tabs.Settings:AddSlider("SoundVolume", {
    Title = "Volume",
    Description = "Volume dos sons",
    Default = 0.35, Min = 0, Max = 1, Rounding = 2,
    Callback = function(v)
        S.soundVolume = v
        if Sound then Sound.volume = v end
    end,
})

Tabs.Settings:AddSection("Tema")

Tabs.Settings:AddDropdown("ThemeDropdown", {
    Title = "Tema da Interface",
    Description = "Escolha o tema visual do hub",
    Values = CustomTheme.available,
    Default = "Dark",
    Multi = false,
    Callback = function(v)
        CustomTheme.apply(v)
        Notify.info("Tema", "Alterado para " .. v, 3)
    end,
})

Tabs.Settings:AddSection("Config")

Tabs.Settings:AddInput("ConfigName", {
    Title = "Nome do Config",
    Default = "default",
    Placeholder = "MeuConfig",
})

Tabs.Settings:AddButton({
    Title = "Salvar Config",
    Description = "Salva as configuracoes atuais",
    Callback = function()
        if Config.save() then
            Notify.success("Config", "Salvo com sucesso!", 3)
        else
            Notify.error("Config", "Falha ao salvar (writefile indisponivel)", 4)
        end
    end,
})

Tabs.Settings:AddButton({
    Title = "Carregar Config",
    Description = "Carrega as configuracoes salvas",
    Callback = function()
        if Config.load() then
            Notify.success("Config", "Carregado com sucesso!", 3)
        else
            Notify.error("Config", "Nenhum config salvo encontrado", 4)
        end
    end,
})

Tabs.Settings:AddSection("Informacoes")

local infoPanel = Panels.create(Tabs.Settings, "Kyoka Hub v" .. S.version, "Iniciando...")

Tabs.Settings:AddParagraph({
    Title = "Sobre",
    Content = "Kyoka Hub e um script standalone para Project Slayers 2.\n\n" ..
              "- Baseado nos remotes reais do jogo\n" ..
              "- Nao depende do Slopix\n" ..
              "- Use apenas em conta alternativa\n" ..
              "- Violacao dos Termos de Servico da Roblox",
})

Tabs.Settings:AddSection("Acoes")

Tabs.Settings:AddButton({
    Title = "Recarregar Hub",
    Description = "Recarrega o hub do zero",
    Callback = function()
        Notify.warn("Reload", "Recarregando hub...", 3)
        task.wait(1)
        pcall(function()
            if Core.Window then Core.Window:Destroy() end
        end)
        Notify.info("Reload", "Rode o loader novamente", 5)
    end,
})

Tabs.Settings:AddButton({
    Title = "Unload Hub",
    Description = "Fecha o hub completamente",
    Callback = function()
        Notify.warn("Unload", "Fechando hub...", 3)
        task.wait(1)
        pcall(function()
            if CatButton and CatButton.gui then CatButton.gui:Destroy() end
            if HubStatus and HubStatus.gui then HubStatus.gui:Destroy() end
            if Core.PlayerHUD and Core.PlayerHUD.gui then Core.PlayerHUD.gui:Destroy() end
            if Core.Window then Core.Window:Destroy() end
        end)
    end,
})

-- HOTKEYS GLOBAIS
UIS.InputBegan:Connect(function(input, gpe)
    if gpe then return end

    if input.KeyCode == Enum.KeyCode.RightShift then
        if CatButton and CatButton.isOpen and CatButton.setOpen then
            local cur = CatButton.isOpen()
            CatButton.setOpen(not cur)
        end
    end

    if input.KeyCode == Enum.KeyCode.RightControl then
        local farm = Core.Farm
        if farm then
            farm.autoFarm = false
            farm.killAura = false
            farm.autoBoss = false
        end
        local skills = Core.Skills
        if skills then
            skills.autoSkill = false
            skills.autoFish = false
        end
        local progress = Core.Progress
        if progress then
            progress.autoQuest = false
        end
        Notify.warn("Emergency Stop", "Toda automacao foi pausada!", 5)
    end
end)

-- LOOP DE INFO
task.spawn(function()
    while true do
        task.wait(3)
        pcall(function()
            S.loadTime = math.floor(os.clock() - S.startTime)
            local minutes = math.floor(S.loadTime / 60)
            local seconds = S.loadTime % 60
            local uptime = string.format("%d:%02d", minutes, seconds)
            infoPanel:update(
                "Kyoka Hub v" .. S.version,
                string.format(
                    "Uptime: %s\nExecutor: %s\nPlaceID: %d\nLevel: %d",
                    uptime,
                    (identifyexecutor and identifyexecutor()) or "Desconhecido",
                    game.PlaceId,
                    H.level()
                )
            )
            if HubStatus and HubStatus.set then
                HubStatus.set(string.format("Kyoka Hub v%s | Lvl %d", S.version, H.level()))
            end
        end)
    end
end)

-- MANAGERS FINAIS
local SaveManager = Core.SaveManager
local InterfaceManager = Core.InterfaceManager

if SaveManager and InterfaceManager then
    pcall(function()
        SaveManager:BuildConfigSection(Tabs.Settings)
        InterfaceManager:BuildInterfaceSection(Tabs.Settings)
    end)
end

-- BOOT FINAL
pcall(function()
    if Core.Window then Core.Window:SelectTab(1) end
end)

if Sound then Sound.play("open") end

task.spawn(function()
    task.wait(1)
    Notify.success("Kyoka Hub", "Carregado com sucesso!", 4)
    task.wait(2)
    Notify.info("Dica", "RightShift abre/fecha | RightCtrl pausa tudo", 6)
    task.wait(6)
    Notify.info("Pronto", "Explore as abas pra ver todas as funcoes", 5)
end)

Core.Teleport = Teleport
Core.CustomTheme = CustomTheme
Core.Settings = S

Logger.log("Parte 6/6 (Settings & Finalize) carregada")
Logger.log("KYOKA HUB v4.0 CARREGADO COM SUCESSO")
Logger.log("Total de partes: 6/6")
Logger.log("Total de linhas: ~3059")
Logger.log("Uptime comecou: " .. os.date("%H:%M:%S"))
