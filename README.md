-- TTD DAVID PRIME HUB v1.1: макрос + авто-голос (Easy/Medium/Hard/Nightmare) + AUTO PLAY AGAIN + CONFIG | Toilet Tower Defense
-- Ремоут dataRemoteEvent: FireServer({ {аргументы}, "код" })
-- v8: новый метод воспроизведения "по экономике с проверкой" (без таймингов и режимов)

local Players = game:GetService("Players")
local CoreGui = game:GetService("CoreGui")
local UIS = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local lp = Players.LocalPlayer

pcall(function() -- убрать старые окна
    local roots = {CoreGui}
    if gethui then table.insert(roots, gethui()) end
    local pg = lp:FindFirstChild("PlayerGui")
    if pg then table.insert(roots, pg) end
    for _, r in ipairs(roots) do
        for _, n in ipairs({"TTDv8", "TTDv7", "TTDv6", "TTDv5", "TTDv4", "TTDAll", "TTDMacro", "TTDAutoVote"}) do
            local o = r:FindFirstChild(n)
            if o then o:Destroy() end
        end
    end
end)

local G = getgenv and getgenv() or _G
G.__TTD_V10 = G.__TTD_V10 or {installed = false}
local H = G.__TTD_V10
H.cb, H.cash = nil, nil

---------------------------------------------------------------- настройки
local REMOTE = "ReplicatedStorage.dataRemoteEvent"
local REMOTE_NAME = "dataRemoteEvent"
local MODES = {"Easy", "Medium", "Hard", "Nightmare"}
local FOLDER, CFG, SETTINGS = "TTDMacros", "TTD_v4_Config.json", "TTD_Settings.json"
local skipCode = {}                     -- коды, которые НЕ пишем в макрос (тап по записи в логе)
local AGAIN_WORDS = {"play again", "playagain", "replay", "try again", "retry"} -- текст кнопки конца матча
-- Если деньги только в GUI и не находятся сами, задай геттер:
-- local CASH_GETTER = function() return tonumber((lp.PlayerGui.Main.Cash.Text:gsub("%D", ""))) end
local CASH_GETTER = nil
local CASH_NAMES = {"Cash", "Money", "Coins", "Gold", "$"}
pcall(function() if makefolder and not isfolder(FOLDER) then makefolder(FOLDER) end end)

local recording, playing, loop, recVote = false, false, false, false
local speed, loopDelay, cashTimeout = 1, 1, 40
local macro, currentName, startT, stopT = {}, nil, 0, 0
local voteAuto, votes, voteDelay, voteRetry = false, 0, 1, 15
local voteMode, voteRecs = "Nightmare", {}   -- voteRecs[режим] = запомненный вызов
local voteTrigger, voteMethod = "Окно", "Ремоут"
local cachedEl, cachedBtn, seenAt, lastSeen, votedAt, voted = nil, nil, nil, 0, 0, false
local againAuto, againCount, againDelay, againRetry = false, 0, 2, 10
local againMethod, againRec, againMacro, againMacroDelay = "GUI-клик", nil, false, 10
local aEl, aBtn, aSeen, aLast, aDoneAt, aDone, lastAScan = nil, nil, nil, 0, 0, false, 0
local autoload, autostart = true, false
local learn = nil
local SC, DG = {}, {} -- SC: коды действий ЭТОЙ сессии (place / upg / vote); DG: вспомогательные функции

local METHOD_INFO = "Метод: по экономике. Шаги идут строго по порядку: ждёт деньги → шлёт → проверяет, что юнит поставился / апгрейд прошёл → если нет, повторяет. Тайминги не нужны."

local C = {
    bg = Color3.fromRGB(16, 24, 18), panel = Color3.fromRGB(26, 38, 29),
    accent = Color3.fromRGB(60, 190, 90), dark = Color3.fromRGB(40, 70, 48),
    red = Color3.fromRGB(190, 60, 60), text = Color3.fromRGB(235, 245, 235),
    dim = Color3.fromRGB(150, 175, 155), sel = Color3.fromRGB(40, 110, 60),
}

local function saveCfg() -- запомненные вызовы (голос по режимам / play again)
    pcall(function() writefile(CFG, HttpService:JSONEncode({vote = voteRecs, again = againRec})) end)
end
local function loadCfg()
    local ok, d = pcall(function() if isfile(CFG) then return HttpService:JSONDecode(readfile(CFG)) end end)
    if ok and type(d) == "table" then
        if type(d.vote) == "table" then
            voteRecs = d.vote.remote and {Nightmare = d.vote} or d.vote -- старый формат = Nightmare
        end
        againRec = d.again
    end
end

---------------------------------------------------------------- деньги (автопоиск)
local cashObj
local CX = {gui = nil, scan = 0, src = "?"}

function CX.parse(t) -- "$1,250" / "1.5K" / "Cash: 500" -> число
    t = tostring(t):gsub(",", "")
    local sub = t:match("%$(.*)") or t
    local num, suf, nxt = sub:match("([%d%.]+)%s*([KkMmBbTt]?)(%a?)")
    num = num and tonumber(num)
    if not num then return nil end
    if nxt ~= "" then suf = "" end
    local mult = {k = 1e3, m = 1e6, b = 1e9, t = 1e12}
    return num * (mult[suf:lower()] or 1)
end

function CX.findGui()
    if CX.gui and CX.gui.Parent then return CX.gui end
    if os.clock() - CX.scan < 3 then return nil end
    CX.scan = os.clock()
    local pg = lp:FindFirstChild("PlayerGui")
    if not pg then return nil end
    local second
    for _, d in ipairs(pg:GetDescendants()) do
        if d:IsA("TextLabel") and not d:FindFirstAncestor("TTDv8") and #d.Text <= 24 and d.Text ~= "" then
            local nm = d.Name:lower()
            if (nm:find("cash", 1, true) or nm:find("money", 1, true) or nm:find("coin", 1, true) or nm:find("gold", 1, true))
                and CX.parse(d.Text) then
                CX.gui = d
                return d
            end
            if not second and d.Text:match("^%s*%$%s*[%d%.,]+%s*[KkMmBbTt]?%s*$") then second = d end
        end
    end
    CX.gui = second
    return second
end

local function getCash()
    if CASH_GETTER then
        local ok, v = pcall(CASH_GETTER)
        if ok and type(v) == "number" then CX.src = "CASH_GETTER" return v end
    end
    if not (cashObj and cashObj.Parent) then
        cashObj = nil
        local ls = lp:FindFirstChild("leaderstats")
        if ls then
            for _, n in ipairs(CASH_NAMES) do
                local o = ls:FindFirstChild(n)
                if o and (o:IsA("IntValue") or o:IsA("NumberValue")) then cashObj = o break end
            end
        end
    end
    if cashObj then CX.src = "leaderstats." .. cashObj.Name return cashObj.Value end
    for _, n in ipairs(CASH_NAMES) do
        local v = lp:GetAttribute(n)
        if type(v) == "number" then CX.src = "attribute " .. n return v end
    end
    local g = CX.findGui()
    if g then
        local gv = CX.parse(g.Text)
        if gv then CX.src = "GUI " .. g:GetFullName() return gv end
    end
    local ls = lp:FindFirstChild("leaderstats")
    if ls then
        for _, o in ipairs(ls:GetChildren()) do
            if o:IsA("IntValue") or o:IsA("NumberValue") then
                cashObj = o
                CX.src = "leaderstats." .. o.Name .. " (любое число)"
                return o.Value
            end
        end
    end
    return nil
end

local function cashFast() -- без сканирования (можно звать из хука)
    if CASH_GETTER then
        local ok, v = pcall(CASH_GETTER)
        if ok and type(v) == "number" then return v end
    end
    if cashObj and cashObj.Parent then return cashObj.Value end
    if CX.gui and CX.gui.Parent then return CX.parse(CX.gui.Text) end
    return H.cash
end
H.cashFast = cashFast

local function fm(x) return tostring(math.floor((x or 0) + 0.5)) end

---------------------------------------------------------------- сериализация и поиск юнитов
local REG = {} -- юниты, поставленные в текущем прогоне макроса

local function ser(v, d)
    d = d or 0
    local t = typeof(v)
    if t == "number" then
        if v ~= v or v == math.huge or v == -math.huge then return 0 end
        return v
    elseif t == "string" or t == "boolean" then return v
    elseif t == "nil" then return {__t = "nil"}
    elseif t == "Vector3" then return {__t = "V3", x = v.X, y = v.Y, z = v.Z}
    elseif t == "CFrame" then return {__t = "CF", c = {v:GetComponents()}}
    elseif t == "Instance" then
        local o = {__t = "Inst", p = v:GetFullName(), nm = v.Name}
        pcall(function()
            local pos = v:IsA("Model") and v:GetPivot().Position or (v:IsA("BasePart") and v.Position)
            if pos then o.x, o.y, o.z = pos.X, pos.Y, pos.Z end
        end)
        return o
    elseif t == "table" then
        if d > 6 then return "<deep>" end
        local o = {}
        for k, x in pairs(v) do o[tostring(k)] = ser(x, d + 1) end
        return o
    end
    return tostring(v)
end

-- снимок аргументов делается прямо в момент вызова (внутри хука), чтобы игра не успела их изменить
H.pack = function(args)
    local p = {}
    for i = 1, args.n do p[i] = ser(args[i]) end
    return p
end

local function resolve(p)
    local cur = game
    for part in string.gmatch(p, "[^%.]+") do
        if part ~= "game" then cur = cur and cur:FindFirstChild(part) end
    end
    return cur
end

local function xz(a, b)
    local dx, dz = a.X - b.X, a.Z - b.Z
    return math.sqrt(dx * dx + dz * dz)
end

local function instPos(i)
    local p
    pcall(function() p = i:IsA("Model") and i:GetPivot().Position or (i:IsA("BasePart") and i.Position) or nil end)
    return p
end

-- найти юнит из записи: сначала среди поставленных в этом прогоне, потом живой поиск рядом
local function findUnit(v, maxwait)
    local parentPath, pname = tostring(v.p):match("^(.*)%.([^%.]+)$")
    local name = (v.nm or pname or ""):lower()
    local pos = v.x and Vector3.new(v.x, v.y or 0, v.z)
    local deadline = os.clock() + (maxwait or 0)
    repeat
        if pos then
            local best, bd
            for _, e in ipairs(REG) do
                local u = e.inst
                if u and u.Parent and (name == "" or u.Name:lower() == name) then
                    local p = instPos(u)
                    if p then
                        local d = xz(p, pos)
                        if d < 5 and (not bd or d < bd) then best, bd = u, d end
                    end
                end
            end
            if best then return best end
            local parent = parentPath and resolve(parentPath)
            if parent then
                local b2, d2, b3, d3
                for _, c in ipairs(parent:GetChildren()) do
                    local p = instPos(c)
                    if p then
                        local d = xz(p, pos)
                        if c.Name:lower() == name then
                            if d < 6 and (not d2 or d < d2) then b2, d2 = c, d end
                        elseif d < 3 and (not d3 or d < d3) then
                            b3, d3 = c, d
                        end
                    end
                end
                if b2 or b3 then return b2 or b3 end
            end
        else
            local r = resolve(v.p)
            if r then return r end
        end
        if os.clock() >= deadline then return nil end
        task.wait(0.1)
    until false
end

local function deser(v, st)
    if type(v) ~= "table" then return v end
    local t = v.__t
    if t == "nil" then return nil end
    if t == "V3" then return Vector3.new(v.x, v.y, v.z) end
    if t == "CF" then return CFrame.new(unpack(v.c)) end
    if t == "Inst" then
        local o = findUnit(v, st and st.wait or 0)
        if not o and st then st.missing = true end
        return o
    end
    local o = {}
    for k, x in pairs(v) do o[tonumber(k) or k] = deser(x, st) end
    return o
end

local function getRemote(a)
    local r = resolve(a.remote)
    if r and (r:IsA("RemoteEvent") or r:IsA("RemoteFunction")) then return r end
    local nm = tostring(a.remote):match("[^%.]+$")
    for _, d in ipairs(ReplicatedStorage:GetDescendants()) do
        if d.Name == nm and (d:IsA("RemoteEvent") or d:IsA("RemoteFunction")) then return d end
    end
end

local function buildArgs(a, wait)
    local st, args = {missing = false, wait = wait or 0}, {}
    for k = 1, a.n do args[k] = deser(a.args[k], st) end
    return args, st
end

local function callRemote(rem, a, args)
    return pcall(function()
        if rem:IsA("RemoteFunction") then
            task.spawn(function() pcall(function() rem:InvokeServer(unpack(args, 1, a.n)) end) end)
        else
            rem:FireServer(unpack(args, 1, a.n))
        end
    end)
end

-- простая отправка записанного вызова (голос / play again)
local function fireRec(a)
    local rem = getRemote(a)
    if not rem then return false, "нет ремоута" end
    local args, st = buildArgs(a, 0)
    if st.missing then return false, "юнит не найден" end
    local ok, err = callRemote(rem, a, args)
    return ok, err
end

local function defaultVote() -- {{режим}, код голоса этой сессии}
    return {remote = REMOTE, method = "FireServer", n = 1, args = {ser({{voteMode}, SC.vote or "9"})}}
end

function DG.freshVote(rec) -- подставить актуальный код голоса в сохранённый вызов
    if rec and SC.vote and type(rec.args) == "table" and type(rec.args[1]) == "table" and rec.args[1]["2"] ~= SC.vote then
        rec.args[1]["2"] = SC.vote
    end
    return rec
end

---------------------------------------------------------------- UI helpers
local syncers = {} -- функции обновления подписей (после загрузки конфига)

local function make(class, props, parent)
    local o = Instance.new(class)
    for k, v in pairs(props) do o[k] = v end
    o.Parent = parent
    return o
end
local function round(o, r) make("UICorner", {CornerRadius = UDim.new(0, r or 8)}, o) end

local function button(parent, text, pos, size, color, cb)
    local b = make("TextButton", {
        Text = text, Position = pos, Size = size, BackgroundColor3 = color or C.dark,
        TextColor3 = C.text, Font = Enum.Font.GothamBold, TextScaled = true,
    }, parent)
    make("UITextSizeConstraint", {MaxTextSize = 15, MinTextSize = 8}, b)
    round(b)
    if cb then b.MouseButton1Click:Connect(function() cb(b) end) end
    return b
end

local function label(parent, text, pos, size, color, align)
    local l = make("TextLabel", {
        Text = text, Position = pos, Size = size, BackgroundTransparency = 1,
        TextColor3 = color or C.text, Font = Enum.Font.Gotham, TextScaled = true,
        TextXAlignment = align or Enum.TextXAlignment.Center,
    }, parent)
    make("UITextSizeConstraint", {MaxTextSize = 15, MinTextSize = 8}, l)
    return l
end

local function stepper(parent, y, titleText, getv, setv, step, mn, mx)
    local row = make("Frame", {Position = UDim2.new(0, 0, 0, y), Size = UDim2.new(1, 0, 0, 30), BackgroundColor3 = C.panel}, parent)
    round(row)
    local lbl = label(row, "", UDim2.new(0, 8, 0, 3), UDim2.new(1, -92, 1, -6), C.text, Enum.TextXAlignment.Left)
    local function refresh() lbl.Text = titleText .. ": " .. tostring(getv()) end
    local function snap(v) return math.floor(v * 100 + 0.5) / 100 end
    button(row, "−", UDim2.new(1, -80, 0, 3), UDim2.new(0, 36, 1, -6), C.dark, function() setv(math.max(mn, snap(getv() - step))) refresh() end)
    button(row, "+", UDim2.new(1, -40, 0, 3), UDim2.new(0, 36, 1, -6), C.dark, function() setv(math.min(mx, snap(getv() + step))) refresh() end)
    refresh()
    table.insert(syncers, refresh)
end

---------------------------------------------------------------- окно
local gui = make("ScreenGui", {Name = "TTDv8", ResetOnSpawn = false, DisplayOrder = 999}, nil)
pcall(function() gui.Parent = (gethui and gethui()) or CoreGui end)
if not gui.Parent then gui.Parent = lp:WaitForChild("PlayerGui") end

task.spawn(function() while gui.Parent do H.cash = getCash() task.wait(0.1) end end)

local frame = make("Frame", {
    Size = UDim2.new(0, 330, 0, 410), Position = UDim2.new(0, 20, 0.12, 0),
    BackgroundColor3 = C.bg, BorderSizePixel = 0,
}, gui)
round(frame, 12)
make("UIStroke", {Color = C.accent, Thickness = 1.5}, frame)

local title = make("Frame", {Size = UDim2.new(1, 0, 0, 30), BackgroundColor3 = C.panel, BorderSizePixel = 0}, frame)
round(title, 12)
label(title, "TTD David Prime Hub v1.1", UDim2.new(0, 10, 0, 4), UDim2.new(1, -90, 1, -8), C.accent, Enum.TextXAlignment.Left)

local mini = make("TextButton", {
    Text = "M", Size = UDim2.new(0, 40, 0, 40), Position = frame.Position,
    BackgroundColor3 = C.accent, TextColor3 = Color3.new(0, 0, 0), Font = Enum.Font.GothamBold,
    TextSize = 20, Visible = false,
}, gui)
round(mini, 20)
mini.MouseButton1Click:Connect(function() mini.Visible = false frame.Visible = true end)

do -- перетаскивание за заголовок
    local dragging, dragStart, startPos
    title.InputBegan:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            dragging, dragStart, startPos = true, i.Position, frame.Position
        end
    end)
    UIS.InputChanged:Connect(function(i)
        if dragging and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
            local d = i.Position - dragStart
            frame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
        end
    end)
    UIS.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then dragging = false end
    end)
end

button(title, "—", UDim2.new(1, -62, 0, 3), UDim2.new(0, 28, 1, -6), C.dark, function()
    mini.Position = frame.Position
    frame.Visible, mini.Visible = false, true
end)
button(title, "✕", UDim2.new(1, -31, 0, 3), UDim2.new(0, 28, 1, -6), C.red, function()
    recording, playing, voteAuto, againAuto, learn, H.cb = false, false, false, false, nil, nil
    gui:Destroy()
end)

local status = label(frame, "Готов", UDim2.new(0, 8, 1, -28), UDim2.new(1, -16, 0, 22), C.accent)
local function setStatus(s) status.Text = s end

local refreshCfgSummary -- задаётся ниже
local pages, tabButtons = {}, {}
local tabs = {
    {"Макросы", "📚", "📚 Макросы: создать / выбрать"},
    {"Запуск", "🎮", "🎮 Запись и воспроизведение"},
    {"Голос", "🗳", "🗳 Автоголос за режим"},
    {"Реплей", "🔁", "🔁 Авто Play Again после матча"},
    {"AFK", "🌙", "🌙 Ночной режим: скип, анти-АФК, реконнект"},
    {"Настройки", "⚙", "⚙ Настройки"},
    {"Конфиг", "💾", "💾 Конфиг: сохранить / загрузить"},
    {"Лог", "📝", "📝 Лог событий"},
}
local function showPage(id)
    for n, p in pairs(pages) do p.Visible = (n == id) end
    for n, b in pairs(tabButtons) do b.BackgroundColor3 = (n == id) and C.accent or C.dark end
    for _, t in ipairs(tabs) do if t[1] == id then setStatus(t[3]) end end
    if id == "Конфиг" and refreshCfgSummary then refreshCfgSummary() end
end
for i, t in ipairs(tabs) do
    tabButtons[t[1]] = button(frame, t[2], UDim2.new((i - 1) / #tabs, 2, 0, 34), UDim2.new(1 / #tabs, -4, 0, 28), C.dark, function() showPage(t[1]) end)
    pages[t[1]] = make("Frame", {
        Position = UDim2.new(0, 8, 0, 68), Size = UDim2.new(1, -16, 1, -100), BackgroundTransparency = 1,
    }, frame)
end

---------------------------------------------------------------- лог
local logScroll = make("ScrollingFrame", {
    Size = UDim2.new(1, 0, 1, -30), BackgroundColor3 = C.panel, BorderSizePixel = 0,
    ScrollBarThickness = 4, AutomaticCanvasSize = Enum.AutomaticSize.Y, CanvasSize = UDim2.new(),
}, pages["Лог"])
round(logScroll)
make("UIListLayout", {Padding = UDim.new(0, 2)}, logScroll)
label(pages["Лог"], "Тап по записи = больше не писать этот код", UDim2.new(0, 0, 1, -26), UDim2.new(1, 0, 0, 24), C.dim)

local logCount, logLines = 0, {}
local function addLog(text, code)
    logCount += 1
    local b = make("TextButton", {
        Text = text, Size = UDim2.new(1, -6, 0, 22), BackgroundColor3 = C.dark, TextColor3 = C.text,
        Font = Enum.Font.Code, TextSize = 12, TextXAlignment = Enum.TextXAlignment.Left, LayoutOrder = logCount,
        TextTruncate = Enum.TextTruncate.AtEnd,
    }, logScroll)
    round(b, 5)
    table.insert(logLines, b)
    if #logLines > 80 then table.remove(logLines, 1):Destroy() end
    if code then
        b.MouseButton1Click:Connect(function()
            skipCode[code] = true
            for i = #macro, 1, -1 do if macro[i].code == code then table.remove(macro, i) end end
            b.Text, b.BackgroundColor3 = "🚫 код " .. code .. " убран", C.red
        end)
    end
    logScroll.CanvasPosition = Vector2.new(0, 99999)
end

-- элементы страниц (объявлены заранее для замыканий)
local libScroll, emptyLbl, nameBox, curLabel, cntLabel, recBtn, playBtn
local loopBtn, recVoteBtn
local voteInfo, voteState, voteAutoBtn, voteStopBtn, voteTrigBtn, voteMethBtn
local voteModeBtns = {}
local againInfo, againState, againAutoBtn, againStopBtn, againMethBtn, againMacroBtn
local cfgInfo, cfgSummary, autoloadBtn, autostartBtn
local selectMacro, deleteMacro, refreshLibrary, updateControl

---------------------------------------------------------------- перехват вызовов (безопасный хук, снимок аргументов на месте)
local function dispatch() end -- переопределяется ниже

local function installHook()
    if H.installed then return true end
    local ok = pcall(function()
        local old
        old = hookmetamethod(game, "__namecall", newcclosure(function(self, ...)
            local cb, dg, ob = H.cb, H.diag, H.obs
            if cb or dg or ob then
                local m = getnamecallmethod()
                if (m == "FireServer" or m == "InvokeServer") and not checkcaller() then
                    local args = table.pack(...)
                    local wantOb = ob and self.Name == REMOTE_NAME
                    if cb or wantOb then
                        local t = os.clock()
                        local cash = H.cashFast and H.cashFast()
                        local packed
                        pcall(function() packed = H.pack(args) end)
                        if cb then task.defer(cb, self, m, args.n, packed, t, cash) end
                        if wantOb then task.defer(ob, self, packed, cash) end
                    end
                    if dg then
                        if m == "InvokeServer" then
                            local res = table.pack(old(self, ...))
                            task.defer(dg, self, m, args, res)
                            return unpack(res, 1, res.n)
                        end
                        task.defer(dg, self, m, args, nil)
                    end
                end
                if setnamecallmethod and m then setnamecallmethod(m) end
            end
            return old(self, ...)
        end))
    end)
    if ok then H.installed = true end
    return ok
end

local function syncHook() H.cb = (recording or learn) and dispatch or nil end

local function describe(self, packed) -- код действия, описание и класс (по структуре, а не по коду)
    local a = packed[1]
    if self.Name == REMOTE_NAME and type(a) == "table" and type(a["2"]) == "string" then
        local cls = DG.classOfPacked(packed)
        local f = type(a["1"]) == "table" and a["1"]["1"] or nil
        local nm = ""
        if type(f) == "table" then nm = f.nm or "" elseif type(f) == "string" then nm = f end
        local kind = (cls == "place" and "поставить") or (cls == "vote" and "голос") or (cls == "inst" and "действие") or ("код " .. a["2"])
        return a["2"], kind .. " " .. nm, cls
    end
    return nil, self.Name
end

local function capture(self, m, n, packed, t, before)
    if typeof(self) ~= "Instance" or not packed then return end
    if not (recording or t <= stopT) or t < startT then return end
    local code, desc, cls = describe(self, packed)
    if not code then return end -- пишем только действия из dataRemoteEvent
    if (skipCode[code] or cls == "vote") and not recVote then return end
    local rec = {t = t - startT, remote = self:GetFullName(), code = code, desc = desc, method = m, args = packed, n = n}
    table.insert(macro, rec)
    -- цена действия: на сколько упали деньги до следующего действия (максимум 0.6 с)
    task.spawn(function()
        local minC
        for _ = 1, 12 do
            task.wait(0.05)
            if macro[#macro] ~= rec then break end
            local c = H.cashFast and H.cashFast()
            if c then minC = math.min(minC or c, c) end
        end
        if before and minC and before - minC >= 1 then rec.cost = before - minC end
    end)
    setStatus("● REC: " .. #macro .. " | " .. desc)
    addLog(string.format("%.1fs %s", rec.t, desc), code)
end

local function learnCapture(self, m, n, packed)
    if not learn or typeof(self) ~= "Instance" or not packed then return end
    local code, _, cls = describe(self, packed)
    table.insert(learn.list, {remote = self:GetFullName(), method = m, args = packed, n = n, code = code, cls = cls})
end

dispatch = function(self, m, n, packed, t, before)
    local ok, err = pcall(function()
        capture(self, m, n, packed, t, before)
        learnCapture(self, m, n, packed)
    end)
    if not ok then addLog("⚠ " .. tostring(err):sub(1, 40)) end
end

-- запомнить вызов: нажми нужную кнопку в игре в течение 8 секунд
local function startLearn(infoLbl, done, prompt, prefCode)
    if learn or recording then setStatus("Сначала ■ Стоп") return end
    if not installHook() then addLog("⚠ перехват не поддерживается") return end
    local cur = {list = {}}
    learn = cur
    syncHook()
    task.spawn(function()
        for s = 8, 1, -1 do
            if learn ~= cur then return end
            infoLbl.Text = "🎓 " .. prompt .. "… " .. s
            task.wait(1)
        end
        task.wait(0.3)
        learn = nil
        syncHook()
        if #cur.list == 0 then addLog("🎓 ничего не поймано, попробуй ещё") done(nil) return end
        local best = cur.list[#cur.list]
        if prefCode then
            for i = #cur.list, 1, -1 do
                if cur.list[i].cls == prefCode then best = cur.list[i] break end
            end
        end
        addLog("✔ вызов запомнен (код " .. tostring(best.code) .. ")")
        done(best)
    end)
end

---------------------------------------------------------------- диагностика
function DG.classOfPacked(packed)
    local a = packed and packed[1]
    if type(a) ~= "table" then return nil end
    local f = a["1"]
    if type(f) ~= "table" then return nil end
    local first = f["1"]
    if type(first) == "string" then
        local sec = f["2"]
        if type(sec) == "table" and (sec.__t == "V3" or sec.__t == "CF") then return "place" end
        if sec == nil then
            for _, m in ipairs(MODES) do if first == m then return "vote" end end
        end
        return nil
    end
    if type(first) == "table" and first.__t == "Inst" and f["2"] == nil then return "inst" end
    return nil
end

function DG.classOf(a) -- place / upg / vote / inst / nil
    local c = DG.classOfPacked(a.args)
    if c == "inst" and a.cost and a.cost > 0 then return "upg" end
    return c
end

-- учим коды этой сессии по твоим РУЧНЫМ действиям (поставил юнита / улучшил / проголосовал)
function DG.observe(self, packed, before)
    local cls = DG.classOfPacked(packed)
    local a = packed and packed[1]
    local code = type(a) == "table" and a["2"]
    if not cls or type(code) ~= "string" then return end
    if cls == "place" or cls == "vote" then
        if SC[cls] ~= code then
            SC[cls] = code
            addLog("🔑 код «" .. (cls == "place" and "поставить" or "голос") .. "» в этой сессии: " .. code)
        end
    elseif cls == "inst" then
        task.spawn(function()
            local minC
            for _ = 1, 12 do
                task.wait(0.05)
                local c = cashFast()
                if c then minC = math.min(minC or c, c) end
            end
            if before and minC and before - minC >= 1 and SC.upg ~= code then
                SC.upg = code
                addLog("🔑 код «улучшить» в этой сессии: " .. code)
            end
        end)
    end
end

-- поиск таблицы кодов в памяти игры (если она там есть)
function DG.scan(silent)
    if not getgc then if not silent then addLog("⚠ getgc недоступен") end return end
    local ok, gc = pcall(getgc, true)
    if not ok or type(gc) ~= "table" then return end
    local KEYS = {place = "place", summon = "place", upgrade = "upg", upg = "upg", vote = "vote"}
    local hits, i = 0, 0
    for _, t in ipairs(gc) do
        i += 1
        if i % 4000 == 0 then task.wait() end
        if type(t) == "table" and getmetatable(t) == nil then
            pcall(function()
                local n, short = 0, 0
                for k, v in pairs(t) do
                    n += 1
                    if n > 80 then return end
                    if type(k) == "string" and type(v) == "string" and (#v <= 2 or #k <= 2) then short += 1 end
                end
                if n < 4 or short < n * 0.7 then return end
                local found = {}
                for k, v in pairs(t) do
                    if type(k) == "string" and type(v) == "string" then
                        local name, code = k, v
                        if #v > 2 then name, code = v, k end
                        local lk = name:lower()
                        for kw, cls in pairs(KEYS) do
                            if lk:find(kw, 1, true) then found[cls] = code end
                        end
                    end
                end
                if next(found) then
                    hits += 1
                    local parts = {}
                    for cls, code in pairs(found) do
                        if not SC[cls] then SC[cls] = code end
                        parts[#parts + 1] = cls .. "=" .. code
                    end
                    addLog("🔎 таблица кодов: " .. table.concat(parts, " "))
                end
            end)
            if hits >= 3 then break end
        end
    end
    if hits == 0 and not silent then addLog("🔎 таблицы кодов нет — поставь/улучши/проголосуй вручную один раз") end
end

function DG.tstr(v, d) -- читаемый текст значения
    d = d or 0
    local t = typeof(v)
    if t == "string" then return '"' .. v:sub(1, 40) .. '"'
    elseif t == "number" then return tostring(math.floor(v * 100 + 0.5) / 100)
    elseif t == "CFrame" then
        local p = v.Position
        return string.format("CF(%.1f,%.1f,%.1f)", p.X, p.Y, p.Z)
    elseif t == "Vector3" then return string.format("V3(%.1f,%.1f,%.1f)", v.X, v.Y, v.Z)
    elseif t == "Instance" then return "<" .. v.ClassName .. " " .. v:GetFullName() .. ">"
    elseif t == "table" then
        if d > 4 then return "{...}" end
        local parts, n = {}, 0
        for k, x in pairs(v) do
            n += 1
            if n > 14 then parts[#parts + 1] = "..." break end
            parts[#parts + 1] = (type(k) == "number" and "" or (tostring(k) .. "=")) .. DG.tstr(x, d + 1)
        end
        return "{" .. table.concat(parts, ", ") .. "}"
    end
    return tostring(v)
end

function DG.clip(txt, file)
    pcall(setclipboard, txt)
    pcall(writefile, file, txt)
end

-- 20 секунд пишет ВСЕ вызовы на сервер (с ответами InvokeServer): сделай нужное действие вручную
function DG.run()
    if H.diag or recording or learn then setStatus("Сначала ■ Стоп / дождись конца") return end
    if not installHook() then addLog("⚠ перехват не поддерживается") return end
    local lines, seen, t0 = {}, {}, os.clock()
    H.diag = function(self, m, args, res)
        local ok, line = pcall(function()
            local nm = self.Name
            seen[nm] = (seen[nm] or 0) + 1
            if seen[nm] > 12 then return nil end
            local parts = {}
            for i = 1, args.n do parts[i] = DG.tstr(args[i]) end
            local out = string.format("%.2fs %s %s(%s)", os.clock() - t0, m, self:GetFullName(), table.concat(parts, ", "))
            if res then
                local rp = {}
                for i = 1, res.n do rp[i] = DG.tstr(res[i]) end
                out = out .. "  => " .. table.concat(rp, ", ")
            end
            return out
        end)
        if ok and line then
            table.insert(lines, line)
            if #lines <= 40 then addLog("🔬 " .. line) end
        end
    end
    addLog("🔬 20 сек: ПОСТАВЬ ЮНИТА ВРУЧНУЮ (и улучши)")
    task.spawn(function()
        for s = 20, 1, -1 do
            setStatus("🔬 диагностика… " .. s .. " с — делай действия вручную")
            task.wait(1)
        end
        H.diag = nil
        local txt = table.concat(lines, "\n")
        DG.clip(txt, "TTD_diag.txt")
        addLog("🔬 готово: " .. #lines .. " вызовов, скопировано в буфер")
        setStatus("🔬 скопировано в буфер — вставь и пришли")
    end)
end

function DG.dump()
    if #macro == 0 then addLog("📋 макрос пуст") return end
    local codes, priced = {}, 0
    for _, a in ipairs(macro) do
        local k = tostring(a.code)
        codes[k] = (codes[k] or 0) + 1
        if a.cost and a.cost > 0 then priced += 1 end
    end
    local cp = {}
    for k, v in pairs(codes) do cp[#cp + 1] = k .. "=" .. v end
    local out = {string.format("macro=%s actions=%d priced=%d codes: %s", tostring(currentName), #macro, priced, table.concat(cp, " "))}
    for i = 1, math.min(8, #macro) do
        local ok, js = pcall(function() return HttpService:JSONEncode(macro[i]) end)
        out[#out + 1] = i .. ": " .. (ok and js or "?")
    end
    local txt = table.concat(out, "\n")
    DG.clip(txt, "TTD_macro_dump.txt")
    addLog("📋 дамп макроса скопирован (" .. #txt .. " симв.)")
    showPage("Лог")
end

---------------------------------------------------------------- макросы: файлы
local function path(n) return FOLDER .. "/" .. n .. ".json" end

local function clean(s)
    s = tostring(s):gsub('[/\\:%*%?"<>|%.]', "")
    s = s:gsub("^%s+", ""):gsub("%s+$", "")
    return s:sub(1, 30)
end

local function listMacros()
    local names = {}
    local ok, list = pcall(listfiles, FOLDER)
    if ok then
        for _, p in ipairs(list) do
            local n = p:match("([^/\\]+)%.json$")
            if n then table.insert(names, n) end
        end
    end
    table.sort(names, function(a, b) return a:lower() < b:lower() end)
    return names
end

local function countActions(n)
    local ok, data = pcall(function() return HttpService:JSONDecode(readfile(path(n))) end)
    return (ok and type(data) == "table") and #data or 0
end

local function saveTo(name, data)
    if not name then return false end
    return (pcall(function() writefile(path(name), HttpService:JSONEncode(data)) end))
end

updateControl = function()
    curLabel.Text = currentName and ("Макрос: " .. currentName) or "Макрос не выбран"
    cntLabel.Text = "Действий в записи: " .. #macro
    recBtn.Text = recording and "● Идёт запись…" or "● Записать"
    recBtn.BackgroundColor3 = recording and C.red or Color3.fromRGB(120, 45, 45)
    playBtn.Text = playing and "⏹ Остановить" or "▶ Играть"
    playBtn.BackgroundColor3 = playing and C.red or C.accent
end

selectMacro = function(n)
    if recording or playing then setStatus("Сначала нажми ■ Стоп") return end
    local ok, data = pcall(function() return HttpService:JSONDecode(readfile(path(n))) end)
    macro = (ok and type(data) == "table") and data or {}
    currentName = n
    updateControl()
    refreshLibrary()
    setStatus("Выбран: " .. n)
end

deleteMacro = function(n)
    if recording or playing then setStatus("Сначала нажми ■ Стоп") return end
    pcall(delfile, path(n))
    if currentName == n then currentName, macro = nil, {} end
    updateControl()
    refreshLibrary()
    setStatus("Удалён: " .. n)
end

refreshLibrary = function()
    for _, c in ipairs(libScroll:GetChildren()) do if c:IsA("Frame") then c:Destroy() end end
    local names = listMacros()
    emptyLbl.Visible = (#names == 0)
    for i, n in ipairs(names) do
        local selected = (n == currentName)
        local row = make("Frame", {
            Size = UDim2.new(1, -6, 0, 36), LayoutOrder = i, BackgroundColor3 = selected and C.sel or C.dark,
        }, libScroll)
        round(row, 6)
        local count = selected and #macro or countActions(n)
        local pick = make("TextButton", {
            Size = UDim2.new(1, -46, 1, 0), BackgroundTransparency = 1,
            Text = (selected and "✔ " or "") .. n .. "  (" .. count .. ")",
            TextColor3 = C.text, Font = Enum.Font.GothamBold, TextSize = 14, TextXAlignment = Enum.TextXAlignment.Left,
        }, row)
        make("UIPadding", {PaddingLeft = UDim.new(0, 8)}, pick)
        pick.MouseButton1Click:Connect(function() selectMacro(n) end)
        local del = button(row, "🗑", UDim2.new(1, -40, 0, 3), UDim2.new(0, 36, 1, -6), C.red)
        local armed = false
        del.MouseButton1Click:Connect(function()
            if not armed then
                armed, del.Text = true, "?"
                setStatus("Тапни 🗑 ещё раз — удалить «" .. n .. "»")
                task.delay(3, function() if armed and del.Parent then armed, del.Text = false, "🗑" end end)
            else
                deleteMacro(n)
            end
        end)
    end
end

local function createMacro()
    local n = clean(nameBox.Text)
    if n == "" then setStatus("Впиши название макроса в поле сверху") return end
    if isfile(path(n)) then setStatus("Макрос с таким именем уже есть") return end
    if recording or playing then setStatus("Сначала нажми ■ Стоп") return end
    if not saveTo(n, {}) then setStatus("⚠ Не удалось создать файл") return end
    nameBox.Text = ""
    selectMacro(n)
    showPage("Запуск")
    setStatus("Создан «" .. n .. "». Нажми ● Записать")
end

local function renameMacro()
    if recording or playing then setStatus("Сначала нажми ■ Стоп") return end
    if not currentName then setStatus("Сначала выбери макрос в списке") return end
    local n = clean(nameBox.Text)
    if n == "" then setStatus("Впиши новое название в поле сверху") return end
    if isfile(path(n)) then setStatus("Это имя уже занято") return end
    if saveTo(n, macro) then
        pcall(delfile, path(currentName))
        currentName, nameBox.Text = n, ""
        updateControl()
        refreshLibrary()
        setStatus("Переименован: " .. n)
    end
end

local function copyMacro()
    if recording or playing then setStatus("Сначала нажми ■ Стоп") return end
    if not currentName then setStatus("Сначала выбери макрос в списке") return end
    local base, n, i = currentName .. "_копия", currentName .. "_копия", 1
    while isfile(path(n)) do i += 1 n = base .. i end
    if saveTo(n, macro) then selectMacro(n) setStatus("Создана копия: " .. n) end
end

---------------------------------------------------------------- воспроизведение: "по экономике с проверкой"
local play
do -- (блок воспроизведения; локалы здесь не считаются в лимит 200)
local WAIT_UNIT = 12                     -- сколько ждать появления юнита для апгрейда, сек
local MAXTRY, VERIFY, ACTION_MAX = 2, 2.5, 150 -- попыток на действие / окно проверки / лимит на действие
local exCache = setmetatable({}, {__mode = "k"})

-- позиция и имена из аргументов действия (для проверки постановки юнита)
local function extract(a)
    if exCache[a] then return exCache[a] end
    local ex = {names = {}}
    local function walk(v, d)
        if d > 5 then return end
        local t = type(v)
        if t == "string" then
            if #v <= 40 then ex.names[v:lower()] = true end
        elseif t == "table" then
            if v.__t == "CF" then
                if not ex.pos and v.c then ex.pos = Vector3.new(v.c[1], v.c[2], v.c[3]) end
            elseif v.__t == "V3" then
                if not ex.pos then ex.pos = Vector3.new(v.x, v.y, v.z) end
            elseif v.__t == "Inst" then
                ex.hasInst = true
                if v.nm then ex.names[v.nm:lower()] = true end
            else
                for _, x in pairs(v) do walk(x, d + 1) end
            end
        end
    end
    for i = 1, a.n do walk(a.args[i], 0) end
    exCache[a] = ex
    return ex
end

-- следим за появлением нового юнита рядом с точкой установки
local function watchNew(ex, entry)
    local w = {}
    local troops = workspace:FindFirstChild("Troops")
    w.conn = (troops and troops.ChildAdded or workspace.DescendantAdded):Connect(function(d)
        if w.strong then return end
        if not (d:IsA("Model") or d:IsA("BasePart")) then return end
        local top = d
        while top.Parent and top.Parent:IsA("Model") do top = top.Parent end
        local p = instPos(top)
        if not p or xz(p, ex.pos) > 10 then return end
        if ex.names[top.Name:lower()] then
            w.strong = true
            entry.inst = top
        elseif not w.weak then
            w.weak = true
            entry.inst = entry.inst or top
        end
    end)
    return w
end

local function runOnce(data)
    REG = {}
    -- старые макросы (v7) содержат чужие ремоуты без кода: их пропускаем
    local hasCoded = false
    for _, a in ipairs(data) do if a.code then hasCoded = true break end end
    if hasCoded then
        local f = {}
        for _, a in ipairs(data) do if a.code then f[#f + 1] = a end end
        if #f < #data then addLog("пропущено чужих ремоутов: " .. (#data - #f)) end
        data = f
    end
    local dumped = false
    local xlog = {}
    -- коды меняются между сессиями: если код постановки в записи ≠ коду этой сессии, переводим
    local rotated = false
    if SC.place then
        for _, a in ipairs(data) do
            if DG.classOf(a) == "place" and a.code ~= SC.place then rotated = true break end
        end
    end
    if rotated then
        local plan, skipped = {}, 0
        for _, a in ipairs(data) do
            local c = DG.classOf(a)
            if (c == "place" or c == "upg") and SC[c] then plan[#plan + 1] = a else skipped += 1 end
        end
        addLog("🔑 коды записи ≠ коды сессии: перевожу, пропущено " .. skipped)
        data = plan
    end
    local n = #data
    local sent, failed, prevT, lastFire = 0, 0, 0, os.clock()
    for i, a in ipairs(data) do
        if not playing then break end
        local cashOk = getCash() ~= nil
        local hasCost = cashOk and a.cost ~= nil and a.cost > 0
        local label_ = a.desc or a.code or "?"

        -- нет цены/денег: держим порядок по паузам записи (с ограничением)
        if not hasCost then
            local gap = (a.t - prevT) / speed
            if i == 1 then gap = math.min(gap, 3) end
            gap = math.min(gap, cashOk and 1.5 or 30)
            while playing and os.clock() < lastFire + gap do task.wait(0.05) end
        end
        prevT = a.t

        local ex = extract(a)
        local cls = DG.classOf(a)
        local isPlace = (cls == "place")
        local entry = isPlace and {pos = ex.pos} or nil
        if entry then table.insert(REG, entry) end

        local done, tries, a0 = false, 0, os.clock()
        local lastArgs
        while playing and tries < MAXTRY and os.clock() - a0 < ACTION_MAX do
            -- 1) ждём деньги
            if hasCost then
                local g0 = os.clock()
                while playing and (getCash() or math.huge) < a.cost and os.clock() - g0 < cashTimeout do
                    setStatus(string.format("⏳ %d/%d %s: деньги %s/%s", i, n, label_, fm(getCash()), fm(a.cost)))
                    task.wait(0.1)
                end
                if not playing then break end
            end
            -- 2) ремоут и аргументы (ждём появления юнита для апгрейда)
            local rem = getRemote(a)
            if not rem then addLog("⚠ " .. i .. " нет ремоута") break end
            setStatus(string.format("▶ %d/%d %s", i, n, label_))
            local args, st = buildArgs(a, WAIT_UNIT)
            lastArgs = args
            if st.missing then addLog("⚠ " .. i .. " " .. label_ .. ": юнит не найден, пропуск") break end
            tries += 1
            local want = cls and SC[cls]
            local useCode = (tries == 1 and want) or a.code -- 2-я попытка: записанный код
            if useCode and type(args[1]) == "table" and args[1][2] ~= useCode then
                args[1][2] = useCode
                local kk = tostring(a.code) .. ">" .. tostring(useCode)
                if not xlog[kk] then xlog[kk] = true addLog("🔑 код " .. tostring(a.code) .. " → " .. tostring(useCode)) end
            end
            -- 3) отправка
            local pre = getCash()
            local w = (isPlace and ex.pos) and watchNew(ex, entry) or nil
            local ok, err = callRemote(rem, a, args)
            lastFire = os.clock()
            if not ok then addLog("✖ " .. i .. ": " .. tostring(err):sub(1, 60)) end
            -- 4) проверка: деньги списались или юнит появился
            local canVerify = (hasCost and pre ~= nil) or (w ~= nil)
            local verified = false
            if canVerify then
                local minC, t1 = pre, os.clock()
                while playing and os.clock() - t1 < VERIFY do
                    task.wait(0.05)
                    local c = getCash()
                    if c and minC then minC = math.min(minC, c) end
                    if hasCost and pre and minC and pre - minC >= a.cost * 0.25 then verified = true break end
                    if w and (w.strong or (w.weak and not hasCost)) then verified = true break end
                end
            else
                task.wait(0.15)
                verified = true
            end
            if w then
                if verified then
                    task.delay(2, function() w.conn:Disconnect() end)
                else
                    w.conn:Disconnect()
                end
            end
            if verified then done = true break end
            addLog(string.format("↻ %d/%d %s: не подтверждено (попытка %d)", i, n, label_, tries))
        end
        if done then
            sent += 1
        else
            failed += 1
            if playing then
                addLog("✖ " .. i .. " " .. label_ .. ": не удалось")
                if not dumped then
                    dumped = true
                    local parts = {}
                    if lastArgs then for k = 1, a.n do parts[k] = DG.tstr(lastArgs[k]) end end
                    local okj, js = pcall(function() return HttpService:JSONEncode(a) end)
                    local txt = string.format("FAIL #%d code=%s desc=%s remote=%s cost=%s cash=%s\nsent: %s\nrecorded: %s",
                        i, tostring(a.code), label_, tostring(a.remote), tostring(a.cost), tostring(getCash()),
                        table.concat(parts, ", "), okj and js or "?")
                    DG.clip(txt, "TTD_fail.txt")
                    addLog("📋 данные первой ошибки скопированы в буфер")
                end
            end
        end
    end
    return sent, failed
end

play = function()
    if playing or recording then return end
    if not currentName then setStatus("Сначала выбери макрос во вкладке 📚") showPage("Макросы") return end
    if #macro == 0 then setStatus("Макрос пуст — сначала ● Записать") return end
    playing = true
    updateControl()
    local data = macro
    task.spawn(function()
        local priced = 0
        for _, a in ipairs(data) do if a.cost and a.cost > 0 then priced += 1 end end
        local cash = getCash()
        addLog(string.format("▶ действий %d, с ценой %d, деньги: %s (%s)", #data, priced, cash and fm(cash) or "НЕ найдены", cash and CX.src or "—"))
        addLog("🔑 коды сессии: поставить=" .. tostring(SC.place) .. " улучшить=" .. tostring(SC.upg) .. " голос=" .. tostring(SC.vote))
        if not cash or priced == 0 then
            addLog("⚠ без денег/цен идёт по паузам записи. Проверь ⚙ 💰 и перезапиши макрос")
        end
        local sent, failed = 0, 0
        repeat
            sent, failed = runOnce(data)
            if playing and loop then task.wait(loopDelay) end
        until not playing or not loop
        playing = false
        updateControl()
        setStatus(string.format("Готово: ✔%d ✖%d", sent, failed))
    end)
end

end -- do play

local startRecording
do
local recArmed, armToken = false, 0

startRecording = function()
    if recording then return end
    if not currentName then setStatus("Сначала создай или выбери макрос") showPage("Макросы") return end
    if #macro > 0 and not recArmed then
        recArmed, armToken = true, armToken + 1
        local my = armToken
        recBtn.Text = "Перезаписать? Тапни ещё"
        setStatus("В макросе уже есть записи — они будут стёрты")
        task.delay(3, function() if armToken == my and recArmed then recArmed = false updateControl() end end)
        return
    end
    recArmed, armToken = false, armToken + 1
    if not installHook() then setStatus("⚠ hookmetamethod не поддерживается") return end
    playing = false
    macro, startT, recording = {}, os.clock(), true
    syncHook()
    updateControl()
    setStatus("● Запись пошла. Играй матч, потом ■ Стоп")
    addLog("— запись: " .. currentName .. " —")
    if getCash() == nil then addLog("⚠ деньги не найдены — цены не запишутся (⚙ 💰)") end
end

end -- do rec

local function stopAll()
    local wasRec = recording
    recording, playing = false, false
    stopT = os.clock()
    updateControl()
    if wasRec then
        local name, data = currentName, macro
        setStatus("Сохраняю…")
        task.delay(0.8, function()
            syncHook()
            if saveTo(name, data) then
                setStatus("✔ Сохранено: " .. name .. " (" .. #data .. ")")
                addLog("— сохранено: " .. name .. ", действий " .. #data .. " —")
            else
                setStatus("⚠ Не удалось сохранить")
            end
            if data == macro then updateControl() end
            refreshLibrary()
        end)
    else
        setStatus("Остановлено")
    end
end

---------------------------------------------------------------- поиск кнопок на экране и клики
local function visibleChain(o)
    local cur = o
    while cur and cur ~= game do
        if cur:IsA("GuiObject") and not cur.Visible then return false end
        if cur:IsA("ScreenGui") and not cur.Enabled then return false end
        cur = cur.Parent
    end
    return o:IsA("GuiObject") and o.AbsoluteSize.X > 0 and o.AbsoluteSize.Y > 0
end

local function clickableFor(d, pg)
    if d:IsA("GuiButton") then return d end
    local cur = d.Parent
    for _ = 1, 4 do
        if not cur or cur == pg then break end
        if cur:IsA("GuiButton") then return cur end
        cur = cur.Parent
    end
    local cont = d.Parent
    if cont then
        local found = {}
        for _, x in ipairs(cont:GetDescendants()) do if x:IsA("GuiButton") then table.insert(found, x) end end
        if #found == 1 then return found[1] end
    end
end

local function scanGui(words) -- видимый текст с одним из слов → элемент, кнопка
    local pg = lp:FindFirstChild("PlayerGui")
    if not pg then return nil end
    for _, d in ipairs(pg:GetDescendants()) do
        if (d:IsA("TextButton") or d:IsA("TextLabel")) and not d:IsDescendantOf(gui) and d.Text ~= "" then
            local tl = d.Text:lower()
            for _, w in ipairs(words) do
                if tl:find(w, 1, true) and visibleChain(d) then return d, clickableFor(d, pg) end
            end
        end
    end
end

local function fireConns(sig)
    if getconnections then
        local ok, conns = pcall(getconnections, sig)
        if ok and conns and #conns > 0 then
            for _, c in ipairs(conns) do pcall(function() c:Fire() end) end
            return true
        end
    end
    return false
end

local function clickButton(b)
    for _, n in ipairs({"MouseButton1Click", "Activated", "MouseButton1Down", "MouseButton1Up"}) do
        if fireConns(b[n]) then return true, n end
    end
    if firesignal then
        for _, n in ipairs({"MouseButton1Click", "Activated"}) do
            if pcall(firesignal, b[n]) then return true, n end
        end
    end
    return false
end

---------------------------------------------------------------- авто-голос (Easy / Medium / Hard / Nightmare)
local lastScan = 0
local function findVote()
    if cachedEl and cachedEl.Parent and visibleChain(cachedEl) then return cachedEl, cachedBtn end
    if os.clock() - lastScan < 1 then return nil end
    lastScan = os.clock()
    cachedEl, cachedBtn = scanGui({voteMode:lower()})
    return cachedEl, cachedBtn
end

local vsText, vsColor = "ВЫКЛ", C.dim
local function redrawVoteState()
    voteState.Text, voteState.TextColor3 = "🗳 " .. voteMode .. ": " .. vsText, vsColor
end
local function setVoteState(text, color)
    vsText, vsColor = text, color or C.accent
    redrawVoteState()
end
local function refreshVoteInfo()
    voteInfo.Text = (voteRecs[voteMode] and "Голос: свой (запомнен)" or "Голос: по умолчанию") .. "  |  Голосов: " .. votes
end
local function refreshModeBtns()
    for m, b in pairs(voteModeBtns) do b.BackgroundColor3 = (m == voteMode) and C.accent or C.dark end
end

local function doVote(reason)
    if voteMethod == "GUI-клик" and cachedBtn and cachedBtn.Parent then
        local ok, sig = clickButton(cachedBtn)
        if ok then
            votes += 1
            addLog(string.format("🗳 Голос #%d %s: GUI-клик (%s, %s)", votes, voteMode, sig, reason))
            refreshVoteInfo()
            return true
        end
    end
    local ok, err = fireRec(DG.freshVote(voteRecs[voteMode]) or defaultVote())
    if ok then
        votes += 1
        addLog(string.format("🗳 Голос #%d %s: ремоут (%s)", votes, voteMode, reason))
        refreshVoteInfo()
        return true
    end
    addLog("⚠ голос не ушёл: " .. tostring(err))
    return false
end

local function setVoteAuto(on)
    voteAuto, seenAt, voted, votedAt = on, nil, false, 0
    voteAutoBtn.BackgroundColor3 = on and C.accent or C.dark
    voteStopBtn.BackgroundColor3 = on and C.dark or C.red
    if on then
        setVoteState(voteTrigger == "Окно" and "жду окно…" or "по таймеру")
        addLog("▶ Auto Vote (" .. voteMode .. "): ВКЛ")
    else
        setVoteState("ВЫКЛ", C.dim)
        addLog("■ Auto Vote: ВЫКЛ")
    end
end

task.spawn(function()
    while gui.Parent do
        if voteAuto and not learn then
            if voteTrigger == "Окно" then
                local el = findVote()
                if el then
                    lastSeen = os.clock()
                    if not seenAt then
                        seenAt, voted = os.clock(), false
                        addLog("🗳 окно голосования найдено")
                        setVoteState("окно открыто")
                    end
                    if not voted and os.clock() - seenAt >= voteDelay then
                        if doVote("окно") then voted, votedAt = true, os.clock() setVoteState("✔ проголосовано") end
                    elseif voted and voteRetry > 0 and os.clock() - votedAt >= voteRetry then
                        doVote("повтор")
                        votedAt = os.clock()
                    end
                elseif seenAt and os.clock() - lastSeen > 3 then
                    seenAt, voted = nil, false
                    setVoteState("жду окно…")
                end
            elseif os.clock() - votedAt >= math.max(voteRetry, 1) then
                findVote()
                if doVote("таймер") then setVoteState("✔ голос в " .. os.date("%H:%M:%S")) end
                votedAt = os.clock()
            end
        end
        task.wait(0.5)
    end
end)

---------------------------------------------------------------- авто Play Again после матча
local function findAgain()
    if aEl and aEl.Parent and visibleChain(aEl) then return aEl, aBtn end
    if os.clock() - lastAScan < 1 then return nil end
    lastAScan = os.clock()
    aEl, aBtn = scanGui(AGAIN_WORDS)
    return aEl, aBtn
end

local function refreshAgainInfo()
    againInfo.Text = (againRec and "Вызов: запомнен" or "Вызов: нет (только GUI)") .. "  |  Перезапусков: " .. againCount
end
local function setAgainState(text, color)
    againState.Text, againState.TextColor3 = "🔁 Play Again: " .. text, color or C.accent
end

local function doAgain(reason)
    if againMethod == "GUI-клик" and aBtn and aBtn.Parent then
        local ok, sig = clickButton(aBtn)
        if ok then
            againCount += 1
            addLog(string.format("🔁 Перезапуск #%d: GUI-клик (%s, %s)", againCount, sig, reason))
            refreshAgainInfo()
            return true
        end
    end
    if againRec then
        local ok, err = fireRec(againRec)
        if ok then
            againCount += 1
            addLog(string.format("🔁 Перезапуск #%d: ремоут (%s)", againCount, reason))
            refreshAgainInfo()
            return true
        end
        addLog("⚠ перезапуск не ушёл: " .. tostring(err))
        return false
    end
    addLog("⚠ нечем перезапускать: жми 🎓 и тапни Play Again в игре")
    return false
end

local function scheduleMacro() -- запуск выбранного макроса через N секунд после перезапуска
    if not againMacro or not currentName then return end
    task.delay(againMacroDelay, function()
        if againAuto and not playing and not recording and #macro > 0 then
            addLog("🔁 запускаю макрос «" .. currentName .. "»")
            play()
        end
    end)
end

local function setAgainAuto(on)
    againAuto, aSeen, aDone, aDoneAt = on, nil, false, 0
    againAutoBtn.BackgroundColor3 = on and C.accent or C.dark
    againStopBtn.BackgroundColor3 = on and C.dark or C.red
    if on then
        setAgainState("жду конец матча…")
        addLog("▶ Auto Play Again: ВКЛ")
    else
        setAgainState("ВЫКЛ", C.dim)
        addLog("■ Auto Play Again: ВЫКЛ")
    end
end

task.spawn(function()
    while gui.Parent do
        if againAuto and not learn then
            local el = findAgain()
            if el then
                aLast = os.clock()
                if not aSeen then
                    aSeen, aDone = os.clock(), false
                    addLog("🔁 экран конца матча найден" .. (aBtn and "" or " (кнопка не определена)"))
                    setAgainState("конец матча")
                    if playing then playing = false updateControl() end
                end
                if not aDone and os.clock() - aSeen >= againDelay then
                    if doAgain("конец матча") then
                        aDone, aDoneAt = true, os.clock()
                        setAgainState("✔ перезапуск")
                        scheduleMacro()
                    end
                elseif aDone and againRetry > 0 and os.clock() - aDoneAt >= againRetry then
                    doAgain("повтор")
                    aDoneAt = os.clock()
                end
            elseif aSeen and os.clock() - aLast > 3 then
                aSeen, aDone = nil, false
                setAgainState("жду конец матча…")
            end
        end
        task.wait(0.5)
    end
end)

---------------------------------------------------------------- AFK: анти-АФК, авто-скип волн, реконнект
local AF = {skip = true, skips = 0, anti = true, rejoin = true, lastSkip = 0, lastBtn = 0, t0 = os.clock()}

do -- анти-АФК: Roblox не кикает за бездействие
    local VU = game:GetService("VirtualUser")
    lp.Idled:Connect(function()
        if AF.anti and gui.Parent then
            pcall(function()
                VU:CaptureController()
                VU:ClickButton2(Vector2.new())
            end)
        end
    end)
end

do -- реконнект при вылете (после него скрипт нужно запустить снова: положи его в autoexec)
    local TS = game:GetService("TeleportService")
    task.spawn(function()
        pcall(function()
            local po = CoreGui:WaitForChild("RobloxPromptGui", 30):WaitForChild("promptOverlay", 30)
            po.ChildAdded:Connect(function(c)
                if gui.Parent and AF.rejoin and c.Name == "ErrorPrompt" then
                    addLog("🔌 вылет — захожу заново")
                    task.wait(2)
                    pcall(function() TS:Teleport(game.PlaceId, lp) end)
                end
            end)
        end)
    end)
end

function AF.colorOf(o)
    if o.BackgroundTransparency < 0.9 then return o.BackgroundColor3 end
    if o:IsA("ImageLabel") or o:IsA("ImageButton") then return o.ImageColor3 end
    return o.BackgroundColor3
end

function AF.state(o) -- красный = выключен, зелёный = включён
    local c = AF.colorOf(o)
    if c.R > 0.5 and c.R > c.G * 1.6 then return "off" end
    if c.G > 0.4 and c.G > c.R * 1.2 then return "on" end
    return "?"
end

-- переключатель «Auto Skip» рядом с надписью (приоритет у объектов красного/зелёного цвета)
function AF.findToggle()
    local pg = lp:FindFirstChild("PlayerGui")
    if not pg then return nil end
    for _, d in ipairs(pg:GetDescendants()) do
        if d:IsA("TextLabel") and not d:IsDescendantOf(gui) and d.Text:lower():find("auto skip", 1, true) and visibleChain(d) then
            local lc = d.AbsolutePosition + d.AbsoluteSize / 2
            local root = (d.Parent and d.Parent.Parent) or pg
            local best, bd, cbest, cbd
            for _, o in ipairs(root:GetDescendants()) do
                if o ~= d and o:IsA("GuiObject") and not d:IsDescendantOf(o) and visibleChain(o) then
                    local sz = o.AbsoluteSize
                    if sz.X >= 8 and sz.Y >= 8 and sz.X <= 90 and sz.Y <= 90 then
                        local off = (o.AbsolutePosition + sz / 2) - lc
                        if math.abs(off.X) < 250 and math.abs(off.Y) < 40 then
                            local dist = off.Magnitude
                            if not bd or dist < bd then best, bd = o, dist end
                            if AF.state(o) ~= "?" and (not cbd or dist < cbd) then cbest, cbd = o, dist end
                        end
                    end
                end
            end
            local pick = cbest or best
            if pick then return pick, pg, d end
        end
    end
end

function AF.findSkipBtn()
    local pg = lp:FindFirstChild("PlayerGui")
    if not pg then return nil end
    for _, d in ipairs(pg:GetDescendants()) do
        if d:IsA("TextButton") and not d:IsDescendantOf(gui) then
            local t = d.Text:lower()
            if t:find("skip", 1, true) and not t:find("auto", 1, true) and visibleChain(d) then return d end
        end
    end
end

function AF.vim(o, withInset) -- настоящий клик мышью по центру объекта
    return (pcall(function()
        local VIM = game:GetService("VirtualInputManager")
        local p, sz = o.AbsolutePosition, o.AbsoluteSize
        local iy = withInset and game:GetService("GuiService"):GetGuiInset().Y or 0
        local x, y = p.X + sz.X / 2, p.Y + sz.Y / 2 + iy
        VIM:SendMouseButtonEvent(x, y, 0, true, game, 0)
        task.wait(0.05)
        VIM:SendMouseButtonEvent(x, y, 0, false, game, 0)
    end))
end

function AF.click(o, pg)
    local b = o:IsA("GuiButton") and o or clickableFor(o, pg)
    if b and clickButton(b) then return true end
    return AF.vim(o, true)
end

-- включить переключатель: пробуем способы по очереди, пока он не станет зелёным
function AF.press(tg, pg)
    local methods = {
        function() local b = tg:IsA("GuiButton") and tg or clickableFor(tg, pg) return b and clickButton(b) end,
        function() return AF.vim(tg, true) end,
        function() return AF.vim(tg, false) end,
    }
    local order = {1, 2, 3}
    if AF.method then order = {AF.method, 1, 2, 3} end
    for _, i in ipairs(order) do
        pcall(methods[i])
        task.wait(0.7)
        if AF.state(tg) == "on" then AF.method = i return true, i end
        if AF.state(tg) == "?" then return false end -- цвет не читается: дальше не жмём, чтобы не щёлкать вслепую
    end
    return false
end

-- тест: что нашёл, какого цвета, получилось ли включить; список объектов рядом уходит в буфер
function AF.test()
    local tg, pg, lblObj = AF.findToggle()
    if not tg then
        addLog("🧪 «Auto Skip» не найден на экране")
        showPage("Лог")
        return
    end
    local c = AF.colorOf(tg)
    addLog("🧪 переключатель: " .. tg.ClassName .. " " .. tg.Name)
    addLog(string.format("🧪 цвет %.2f %.2f %.2f → %s", c.R, c.G, c.B, AF.state(tg)))
    local lines = {"label: " .. lblObj:GetFullName(), "pick: " .. tg:GetFullName() .. " [" .. tg.ClassName .. "]"}
    local lc = lblObj.AbsolutePosition + lblObj.AbsoluteSize / 2
    local root = (lblObj.Parent and lblObj.Parent.Parent) or pg
    for _, o in ipairs(root:GetDescendants()) do
        if o:IsA("GuiObject") and visibleChain(o) then
            local off = (o.AbsolutePosition + o.AbsoluteSize / 2) - lc
            if math.abs(off.X) < 300 and math.abs(off.Y) < 60 then
                local col = AF.colorOf(o)
                lines[#lines + 1] = string.format("%s | %s | %dx%d | %.2f,%.2f,%.2f | %s", o.ClassName, o:GetFullName(),
                    o.AbsoluteSize.X, o.AbsoluteSize.Y, col.R, col.G, col.B, o:IsA("TextLabel") and o.Text or "")
            end
        end
    end
    DG.clip(table.concat(lines, "\n"), "TTD_skip_dump.txt")
    addLog("🧪 объекты рядом скопированы в буфер")
    if AF.state(tg) == "off" then
        local ok, m = AF.press(tg, pg)
        addLog(ok and ("🧪 включил (способ " .. tostring(m) .. ")") or "🧪 включить не получилось")
    elseif AF.state(tg) == "on" then
        addLog("🧪 уже включён")
    end
    showPage("Лог")
end

task.spawn(function()
    while gui.Parent do
        pcall(function()
            if AF.skip then
                if os.clock() - AF.lastSkip > 3 then
                    local tg, pg = AF.findToggle()
                    if not tg then
                        if os.clock() - (AF.lastMiss or 0) > 60 then
                            AF.lastMiss = os.clock()
                            addLog("⏭ «Auto Skip» на экране не найден")
                        end
                    elseif AF.state(tg) == "off" then
                        local ok, m = AF.press(tg, pg)
                        if ok then
                            AF.skips += 1
                            AF.lastSkip = os.clock()
                            addLog("⏭ Auto Skip включён (способ " .. tostring(m) .. ")")
                        else
                            AF.lastSkip = os.clock() + 30
                            addLog("⏭ не удалось включить Auto Skip (🧪 в AFK)")
                        end
                    end
                end
                if os.clock() - AF.lastBtn > 5 then
                    local b = AF.findSkipBtn()
                    if b and AF.click(b, b) then
                        AF.skips += 1
                        AF.lastBtn = os.clock()
                    end
                end
            end
            if AF.stat then
                local t = math.floor(os.clock() - AF.t0)
                AF.stat.Text = string.format("AFK %dч %02dм | голосов %d | реплеев %d | скипов %d", t // 3600, (t % 3600) // 60, votes, againCount, AF.skips)
            end
        end)
        task.wait(1.5)
    end
end)

---------------------------------------------------------------- конфиг: сохранить / загрузить
local D = { -- значения по умолчанию (для сброса)
    speed = 1, loopDelay = 1, cashTimeout = 40, loop = false, recVote = false,
    voteMode = "Nightmare", voteDelay = 1, voteRetry = 15, voteTrigger = "Окно", voteMethod = "Ремоут",
    againDelay = 2, againRetry = 10, againMethod = "GUI-клик", againMacro = false, againMacroDelay = 10,
    afk = {anti = true, rejoin = true},
}

local function num(v, d, mn, mx)
    v = tonumber(v)
    if not v then return d end
    return math.max(mn, math.min(mx, v))
end
local function pick(v, list, d)
    for _, x in ipairs(list) do if x == v then return v end end
    return d
end
local function onoff(b) return b and "вкл" or "выкл" end

refreshCfgSummary = function()
    cfgSummary.Text = string.format(
        "Макрос: %s\nМетод: по экономике · пауза x%s · повтор макроса: %s\nГолос: %s · %s / %s · задержка %sс · повтор %sс\nPlay Again: %s · задержка %sс · макрос после: %s (%sс)\nСейчас включено: голос %s · реплей %s",
        currentName or "—", speed, onoff(loop), voteMode, voteTrigger, voteMethod, voteDelay, voteRetry,
        againMethod, againDelay, onoff(againMacro), againMacroDelay, onoff(voteAuto), onoff(againAuto))
end

local function applyUI() -- подписи кнопок под текущие значения
    loopBtn.Text = loop and "Повтор: ВКЛ" or "Повтор: выкл"
    loopBtn.BackgroundColor3 = loop and C.accent or C.dark
    voteTrigBtn.Text = "Триггер: " .. voteTrigger
    voteMethBtn.Text = "Метод: " .. voteMethod
    refreshModeBtns()
    refreshVoteInfo()
    redrawVoteState()
    againMethBtn.Text = "Метод: " .. againMethod
    againMacroBtn.Text = "Макрос после: " .. (againMacro and "да" or "нет")
    againMacroBtn.BackgroundColor3 = againMacro and C.accent or C.dark
    recVoteBtn.Text = "Писать голос в макрос: " .. (recVote and "да" or "нет")
    autoloadBtn.Text = "Автозагрузка при запуске: " .. (autoload and "да" or "нет")
    autoloadBtn.BackgroundColor3 = autoload and C.accent or C.dark
    autostartBtn.Text = "Автостарт функций: " .. (autostart and "да" or "нет")
    autostartBtn.BackgroundColor3 = autostart and C.accent or C.dark
    for _, f in ipairs(syncers) do f() end
    refreshCfgSummary()
end

local function applySettings(d)
    speed = num(d.speed, speed, 0.25, 4)
    loopDelay = num(d.loopDelay, loopDelay, 0, 60)
    cashTimeout = num(d.cashTimeout, cashTimeout, 20, 300)
    loop, recVote = d.loop == true, d.recVote == true
    voteMode = pick(d.voteMode, MODES, voteMode)
    voteDelay = num(d.voteDelay, voteDelay, 0, 30)
    voteRetry = num(d.voteRetry, voteRetry, 0, 120)
    voteTrigger = pick(d.voteTrigger, {"Окно", "Таймер"}, voteTrigger)
    voteMethod = pick(d.voteMethod, {"Ремоут", "GUI-клик"}, voteMethod)
    againDelay = num(d.againDelay, againDelay, 0, 60)
    againRetry = num(d.againRetry, againRetry, 0, 120)
    againMethod = pick(d.againMethod, {"GUI-клик", "Ремоут"}, againMethod)
    againMacro = d.againMacro == true
    againMacroDelay = num(d.againMacroDelay, againMacroDelay, 0, 120)
    autoload, autostart = d.autoload ~= false, d.autostart == true
    if type(d.afk) == "table" then AF.anti, AF.rejoin = d.afk.anti ~= false, d.afk.rejoin ~= false end
    cachedEl, cachedBtn, seenAt, voted = nil, nil, nil, false
end

local function collect()
    return {
        v = 2, macro = currentName, speed = speed, loopDelay = loopDelay, cashTimeout = cashTimeout,
        loop = loop, recVote = recVote,
        voteMode = voteMode, voteDelay = voteDelay, voteRetry = voteRetry, voteTrigger = voteTrigger, voteMethod = voteMethod,
        againDelay = againDelay, againRetry = againRetry, againMethod = againMethod,
        againMacro = againMacro, againMacroDelay = againMacroDelay,
        autoload = autoload, autostart = autostart, autos = {vote = voteAuto, again = againAuto, skip = AF.skip},
        afk = {anti = AF.anti, rejoin = AF.rejoin},
    }
end

local function readSettings()
    local ok, d = pcall(function() if isfile(SETTINGS) then return HttpService:JSONDecode(readfile(SETTINGS)) end end)
    return (ok and type(d) == "table") and d or nil
end

local function saveSettings()
    saveCfg()
    local ok = pcall(function() writefile(SETTINGS, HttpService:JSONEncode(collect())) end)
    cfgInfo.Text = ok and ("💾 Сохранено в " .. os.date("%H:%M:%S")) or "⚠ Не удалось сохранить"
    setStatus(cfgInfo.Text)
    addLog(ok and "💾 конфиг сохранён" or "⚠ конфиг не сохранён")
    refreshCfgSummary()
end

local function loadSettings(silent)
    local d = readSettings()
    if not d then
        if not silent then setStatus("Конфиг не найден — сначала 💾 Save Config") end
        return false
    end
    applySettings(d)
    applyUI()
    if d.macro and not (recording or playing) and isfile(path(d.macro)) then selectMacro(d.macro) end
    if autostart and type(d.autos) == "table" then
        if d.autos.vote and not voteAuto then setVoteAuto(true) end
        if d.autos.again and not againAuto then setAgainAuto(true) end
        if d.autos.skip then AF.skip = true applyUI() end
    end
    refreshCfgSummary()
    cfgInfo.Text = "📂 Загружено в " .. os.date("%H:%M:%S")
    if not silent then setStatus(cfgInfo.Text) end
    addLog("📂 конфиг загружен")
    return true
end

---------------------------------------------------------------- вкладка 📚 Макросы
do
    local L = pages["Макросы"]
    nameBox = make("TextBox", {
        Text = "", PlaceholderText = "Название (для создания / переименования)", PlaceholderColor3 = C.dim,
        Size = UDim2.new(1, 0, 0, 30), BackgroundColor3 = C.panel, TextColor3 = C.text,
        Font = Enum.Font.Gotham, TextSize = 14, ClearTextOnFocus = false,
    }, L)
    round(nameBox)
    button(L, "➕ Создать", UDim2.new(0, 0, 0, 34), UDim2.new(0.4, -3, 0, 32), C.accent, createMacro)
    button(L, "✏ Переим.", UDim2.new(0.4, 0, 0, 34), UDim2.new(0.3, -3, 0, 32), C.dark, renameMacro)
    button(L, "📋 Копия", UDim2.new(0.7, 0, 0, 34), UDim2.new(0.3, 0, 0, 32), C.dark, copyMacro)
    label(L, "Твои макросы · тап = выбрать · 🗑 = удалить", UDim2.new(0, 0, 0, 70), UDim2.new(1, 0, 0, 18), C.dim)
    libScroll = make("ScrollingFrame", {
        Position = UDim2.new(0, 0, 0, 92), Size = UDim2.new(1, 0, 1, -92), BackgroundColor3 = C.panel,
        BorderSizePixel = 0, ScrollBarThickness = 4, AutomaticCanvasSize = Enum.AutomaticSize.Y, CanvasSize = UDim2.new(),
    }, L)
    round(libScroll)
    make("UIListLayout", {Padding = UDim.new(0, 3)}, libScroll)
    emptyLbl = label(libScroll, "Пока нет макросов.\nВпиши название и нажми ➕ Создать", UDim2.new(0, 0, 0, 0), UDim2.new(1, -6, 0, 70), C.dim)
    emptyLbl.TextWrapped, emptyLbl.LayoutOrder = true, 0
end

---------------------------------------------------------------- вкладка 🎮 Запуск
do
    local P = pages["Запуск"]
    curLabel = label(P, "", UDim2.new(0, 0, 0, 0), UDim2.new(1, 0, 0, 24), C.accent, Enum.TextXAlignment.Left)
    cntLabel = label(P, "", UDim2.new(0, 0, 0, 24), UDim2.new(1, 0, 0, 18), C.dim, Enum.TextXAlignment.Left)
    recBtn = button(P, "● Записать", UDim2.new(0, 0, 0, 48), UDim2.new(0.5, -3, 0, 36), Color3.fromRGB(120, 45, 45), startRecording)
    button(P, "■ Стоп", UDim2.new(0.5, 3, 0, 48), UDim2.new(0.5, -3, 0, 36), C.dark, stopAll)
    playBtn = button(P, "▶ Играть", UDim2.new(0, 0, 0, 90), UDim2.new(1, 0, 0, 40), C.accent, function()
        if playing then playing = false else play() end
    end)
    loopBtn = button(P, "Повтор: выкл", UDim2.new(0, 0, 0, 136), UDim2.new(1, 0, 0, 30), C.dark, function()
        loop = not loop
        applyUI()
    end)
    local info = label(P, METHOD_INFO, UDim2.new(0, 0, 0, 172), UDim2.new(1, 0, 0, 56), C.dim, Enum.TextXAlignment.Left)
    info.TextWrapped = true
    local g = label(P, "Запись и ▶ Играть начинай в один и тот же момент матча (сразу после голосования). Нужен макрос, записанный в v8: старые записи работают хуже.",
        UDim2.new(0, 0, 0, 234), UDim2.new(1, 0, 0, 60), C.text, Enum.TextXAlignment.Left)
    g.TextWrapped, g.TextYAlignment, g.BackgroundTransparency, g.BackgroundColor3 = true, Enum.TextYAlignment.Top, 0, C.panel
    round(g)
end

---------------------------------------------------------------- вкладка 🗳 Голос
do
    local P = pages["Голос"]
    voteState = label(P, "", UDim2.new(0, 0, 0, 0), UDim2.new(1, 0, 0, 34), C.dim)
    voteState.Font = Enum.Font.GothamBold
    voteInfo = label(P, "", UDim2.new(0, 0, 0, 36), UDim2.new(1, 0, 0, 18), C.dim)
    for i, m in ipairs(MODES) do -- выбор режима: Easy / Medium / Hard / Nightmare
        voteModeBtns[m] = button(P, m, UDim2.new((i - 1) / 4, 2, 0, 58), UDim2.new(1 / 4, -4, 0, 30), C.dark, function()
            if learn then setStatus("Подожди окончания 🎓") return end
            voteMode = m
            cachedEl, cachedBtn, seenAt, voted = nil, nil, nil, false
            applyUI()
        end)
    end
    voteAutoBtn = button(P, "▶ Auto Vote", UDim2.new(0, 0, 0, 92), UDim2.new(0.5, -3, 0, 36), C.dark, function() setVoteAuto(true) end)
    voteStopBtn = button(P, "■ Stop Auto Vote", UDim2.new(0.5, 3, 0, 92), UDim2.new(0.5, -3, 0, 36), C.red, function() setVoteAuto(false) end)
    button(P, "🎓 Запомнить голос", UDim2.new(0, 0, 0, 132), UDim2.new(0.5, -3, 0, 28), C.dark, function()
        local m = voteMode
        startLearn(voteInfo, function(best)
            if best then voteRecs[m] = best saveCfg() end
            refreshVoteInfo()
        end, "Нажми голос за " .. m, "vote")
    end)
    button(P, "🧪 Тест голоса", UDim2.new(0.5, 3, 0, 132), UDim2.new(0.5, -3, 0, 28), C.dark, function()
        findVote()
        doVote("тест")
    end)
    voteTrigBtn = button(P, "Триггер: " .. voteTrigger, UDim2.new(0, 0, 0, 164), UDim2.new(0.5, -3, 0, 28), C.dark, function()
        voteTrigger = (voteTrigger == "Окно") and "Таймер" or "Окно"
        seenAt, voted = nil, false
        applyUI()
    end)
    voteMethBtn = button(P, "Метод: " .. voteMethod, UDim2.new(0.5, 3, 0, 164), UDim2.new(0.5, -3, 0, 28), C.dark, function()
        voteMethod = (voteMethod == "Ремоут") and "GUI-клик" or "Ремоут"
        applyUI()
    end)
    stepper(P, 196, "Задержка, с", function() return voteDelay end, function(v) voteDelay = v end, 0.5, 0, 30)
    stepper(P, 230, "Повтор, с (0=1 раз)", function() return voteRetry end, function(v) voteRetry = v end, 5, 0, 120)
end

---------------------------------------------------------------- вкладка 🔁 Реплей
do
    local P = pages["Реплей"]
    againState = label(P, "", UDim2.new(0, 0, 0, 0), UDim2.new(1, 0, 0, 40), C.dim)
    againState.Font = Enum.Font.GothamBold
    againInfo = label(P, "", UDim2.new(0, 0, 0, 42), UDim2.new(1, 0, 0, 20), C.dim)
    againAutoBtn = button(P, "▶ Auto Play Again", UDim2.new(0, 0, 0, 66), UDim2.new(0.5, -3, 0, 38), C.dark, function() setAgainAuto(true) end)
    againStopBtn = button(P, "■ Stop Auto Again", UDim2.new(0.5, 3, 0, 66), UDim2.new(0.5, -3, 0, 38), C.red, function() setAgainAuto(false) end)
    button(P, "🎓 Запомнить кнопку", UDim2.new(0, 0, 0, 108), UDim2.new(0.5, -3, 0, 30), C.dark, function()
        startLearn(againInfo, function(best)
            if best then againRec = best saveCfg() end
            refreshAgainInfo()
        end, "Нажми Play Again в игре", nil)
    end)
    button(P, "🧪 Тест", UDim2.new(0.5, 3, 0, 108), UDim2.new(0.5, -3, 0, 30), C.dark, function()
        findAgain()
        doAgain("тест")
    end)
    againMethBtn = button(P, "Метод: " .. againMethod, UDim2.new(0, 0, 0, 142), UDim2.new(0.5, -3, 0, 30), C.dark, function()
        againMethod = (againMethod == "GUI-клик") and "Ремоут" or "GUI-клик"
        applyUI()
    end)
    againMacroBtn = button(P, "Макрос после: нет", UDim2.new(0.5, 3, 0, 142), UDim2.new(0.5, -3, 0, 30), C.dark, function()
        againMacro = not againMacro
        applyUI()
    end)
    stepper(P, 176, "Задержка, с", function() return againDelay end, function(v) againDelay = v end, 1, 0, 60)
    stepper(P, 210, "Повтор, с (0=1 раз)", function() return againRetry end, function(v) againRetry = v end, 5, 0, 120)
    stepper(P, 244, "Макрос через, с", function() return againMacroDelay end, function(v) againMacroDelay = v end, 5, 0, 120)
end

---------------------------------------------------------------- вкладка ⚙ Настройки
do
    local S = pages["Настройки"]
    stepper(S, 0, "Пауза без денег x", function() return speed end, function(v) speed = v end, 0.25, 0.25, 4)
    stepper(S, 36, "Пауза повтора, с", function() return loopDelay end, function(v) loopDelay = v end, 0.5, 0, 60)
    stepper(S, 72, "Ждать деньги, с", function() return cashTimeout end, function(v) cashTimeout = v end, 10, 20, 300)
    recVoteBtn = button(S, "Писать голос в макрос: нет", UDim2.new(0, 0, 0, 112), UDim2.new(1, 0, 0, 32), C.dark, function()
        recVote = not recVote
        applyUI()
    end)
    button(S, "🧹 Очистить лог", UDim2.new(0, 0, 0, 150), UDim2.new(1, 0, 0, 32), C.dark, function()
        for _, b in ipairs(logLines) do b:Destroy() end
        logLines = {}
        setStatus("Лог очищен")
    end)
    button(S, "💰 Деньги", UDim2.new(0, 0, 0, 188), UDim2.new(0.5, -3, 0, 32), C.dark, function()
        CX.gui, CX.scan, cashObj = nil, 0, nil
        local v = getCash()
        if v then
            addLog("💰 деньги = " .. fm(v) .. " | " .. tostring(CX.src))
        else
            addLog("💰 деньги НЕ найдены → задай CASH_GETTER в начале скрипта")
        end
        showPage("Лог")
    end)
    button(S, "🔑 Коды сессии", UDim2.new(0.5, 3, 0, 188), UDim2.new(0.5, -3, 0, 32), C.dark, function()
        DG.scan(false)
        addLog("🔑 сейчас: поставить=" .. tostring(SC.place) .. " улучшить=" .. tostring(SC.upg) .. " голос=" .. tostring(SC.vote))
        showPage("Лог")
    end)
    button(S, "🔬 Диагностика 20с (ставь юнита сам)", UDim2.new(0, 0, 0, 224), UDim2.new(1, 0, 0, 32), C.dark, function()
        DG.run()
        showPage("Лог")
    end)
    button(S, "📋 Копия макроса в буфер", UDim2.new(0, 0, 0, 260), UDim2.new(1, 0, 0, 32), C.dark, DG.dump)
end

---------------------------------------------------------------- вкладка 🌙 AFK
do
    local P = pages["AFK"]
    AF.stat = label(P, "AFK", UDim2.new(0, 0, 0, 0), UDim2.new(1, 0, 0, 34), C.accent)
    AF.stat.Font = Enum.Font.GothamBold
    local nightBtn = button(P, "🌙 Включить ночной режим", UDim2.new(0, 0, 0, 40), UDim2.new(1, 0, 0, 44), C.dark, function()
        local on = not (voteAuto and againAuto and AF.skip)
        if on then
            voteMethod, againMethod = "GUI-клик", "GUI-клик" -- по кнопкам на экране: не зависит от кодов игры
            voteTrigger = "Окно"
            AF.anti, AF.rejoin, AF.skip = true, true, true
            if not voteAuto then setVoteAuto(true) end
            if not againAuto then setAgainAuto(true) end
            addLog("🌙 ночной режим: ВКЛ (" .. voteMode .. ")")
        else
            if voteAuto then setVoteAuto(false) end
            if againAuto then setAgainAuto(false) end
            AF.skip = false
            addLog("🌙 ночной режим: ВЫКЛ")
        end
        applyUI()
    end)
    local antiBtn = button(P, "", UDim2.new(0, 0, 0, 90), UDim2.new(0.5, -3, 0, 30), C.dark, function()
        AF.anti = not AF.anti
        applyUI()
    end)
    local rejoinBtn = button(P, "", UDim2.new(0.5, 3, 0, 90), UDim2.new(0.5, -3, 0, 30), C.dark, function()
        AF.rejoin = not AF.rejoin
        applyUI()
    end)
    local skipBtn = button(P, "", UDim2.new(0, 0, 0, 124), UDim2.new(1, 0, 0, 30), C.dark, function()
        AF.skip = not AF.skip
        applyUI()
    end)
    button(P, "🧪 Тест Auto Skip (отчёт в 📝 Лог)", UDim2.new(0, 0, 0, 158), UDim2.new(1, 0, 0, 30), C.dark, AF.test)
    local hint = label(P, "Ночной режим: голос за выбранный режим (вкладка 🗳), Play Again, скип волн, анти-АФК и реконнект. Всё по кнопкам на экране. После реконнекта скрипт нужно запустить снова: положи его в autoexec.",
        UDim2.new(0, 0, 0, 194), UDim2.new(1, 0, 0, 84), C.dim, Enum.TextXAlignment.Left)
    hint.TextWrapped, hint.TextYAlignment = true, Enum.TextYAlignment.Top
    table.insert(syncers, function()
        local night = voteAuto and againAuto and AF.skip
        nightBtn.Text = night and "🌙 Ночной режим: ВКЛ (тап = выкл)" or "🌙 Включить ночной режим"
        nightBtn.BackgroundColor3 = night and C.accent or C.dark
        antiBtn.Text = "Анти-АФК: " .. (AF.anti and "да" or "нет")
        antiBtn.BackgroundColor3 = AF.anti and C.accent or C.dark
        rejoinBtn.Text = "Реконнект: " .. (AF.rejoin and "да" or "нет")
        rejoinBtn.BackgroundColor3 = AF.rejoin and C.accent or C.dark
        skipBtn.Text = "Авто-скип волн: " .. (AF.skip and "да" or "нет")
        skipBtn.BackgroundColor3 = AF.skip and C.accent or C.dark
    end)
end

---------------------------------------------------------------- вкладка 💾 Конфиг
do
    local P = pages["Конфиг"]
    cfgInfo = label(P, isfile(SETTINGS) and "Файл конфига найден" or "Конфиг ещё не сохранён", UDim2.new(0, 0, 0, 0), UDim2.new(1, 0, 0, 20), C.dim)
    cfgSummary = label(P, "", UDim2.new(0, 0, 0, 24), UDim2.new(1, 0, 0, 112), C.text, Enum.TextXAlignment.Left)
    cfgSummary.TextWrapped, cfgSummary.TextYAlignment, cfgSummary.BackgroundTransparency, cfgSummary.BackgroundColor3 = true, Enum.TextYAlignment.Top, 0, C.panel
    round(cfgSummary)
    button(P, "💾 Save Config", UDim2.new(0, 0, 0, 142), UDim2.new(0.5, -3, 0, 36), C.accent, saveSettings)
    button(P, "📂 Load Config", UDim2.new(0.5, 3, 0, 142), UDim2.new(0.5, -3, 0, 36), C.dark, function() loadSettings(false) end)
    local resetArmed = false
    button(P, "♻ Сбросить настройки", UDim2.new(0, 0, 0, 184), UDim2.new(1, 0, 0, 30), C.dark, function(b)
        if not resetArmed then
            resetArmed, b.Text = true, "Точно? Тапни ещё"
            task.delay(3, function() if resetArmed then resetArmed, b.Text = false, "♻ Сбросить настройки" end end)
            return
        end
        resetArmed, b.Text = false, "♻ Сбросить настройки"
        local r = {}
        for k, v in pairs(D) do r[k] = v end
        r.autoload, r.autostart = autoload, autostart
        applySettings(r)
        applyUI()
        setStatus("Сброшено. Нажми 💾, чтобы перезаписать файл")
    end)
    autoloadBtn = button(P, "Автозагрузка при запуске: да", UDim2.new(0, 0, 0, 220), UDim2.new(1, 0, 0, 30), C.accent, function()
        autoload = not autoload
        applyUI()
        saveSettings()
    end)
    autostartBtn = button(P, "Автостарт функций: нет", UDim2.new(0, 0, 0, 254), UDim2.new(1, 0, 0, 30), C.dark, function()
        autostart = not autostart
        applyUI()
        saveSettings()
    end)
end

---------------------------------------------------------------- старт
loadCfg()
updateControl()
refreshLibrary()
refreshVoteInfo()
refreshAgainInfo()
setVoteState("ВЫКЛ", C.dim)
setAgainState("ВЫКЛ", C.dim)
applyUI()
showPage("Макросы")
addLog("TTD David Prime Hub v1.1: готов. Создай макрос во вкладке 📚.")
installHook()
H.obs = DG.observe
task.spawn(function() task.wait(3) DG.scan(true) end)
local d0 = readSettings()
if d0 and d0.autoload ~= false then loadSettings(true) end
task.delay(1.5, function()
    local v = getCash()
    addLog(v and ("💰 деньги найдены: " .. fm(v) .. " | " .. tostring(CX.src)) or "⚠ деньги не найдены (⚙ 💰) — цены не запишутся")
end)
