-- LocalScript posicionado em StarterGui ou StarterPlayerScripts
-- ============================================================
-- SAMMY APEX FEATURES — REWORKED UI / ISOLATED UI CHUNK
-- ============================================================

if _G.__SammyApexCleanup then
    pcall(_G.__SammyApexCleanup)
end

repeat task.wait() until game:IsLoaded()
task.wait(2)

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")
local SoundService = game:GetService("SoundService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")
local Stats = game:GetService("Stats")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local CoreGui = game:GetService("CoreGui")
local Workspace = game:GetService("Workspace")

local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

-- ============================================================
-- REFERÊNCIA AOS PLOTS
-- ============================================================

local Plots = Workspace:WaitForChild("Plots", 10)
if not Plots then
    warn("[Sammy Apex] Plots folder not found.")
    return
end

-- ============================================================
-- CONFIGURAÇÕES GERAIS
-- ============================================================

local SETTINGS_FILE = "XKPubSettings.json"
local savedSettings = {}

local function loadSettings()
    if readfile and isfile and isfile(SETTINGS_FILE) then
        local ok, data = pcall(function()
            return HttpService:JSONDecode(readfile(SETTINGS_FILE))
        end)
        if ok and type(data) == "table" then
            savedSettings = data
        end
    end
end

local function saveSettings()
    if writefile then
        pcall(function()
            writefile(SETTINGS_FILE, HttpService:JSONEncode(savedSettings))
        end)
    end
end

loadSettings()

-- Normalize and persist the booster keybind using both legacy/current names.
do
    local normalized = Enum.KeyCode[savedSettings.speedBoosterKeybind or savedSettings.keybindSpeedBooster or "X"] or Enum.KeyCode.X
    savedSettings.speedBoosterKeybind = normalized.Name
    savedSettings.keybindSpeedBooster = normalized.Name
end

local CARPET_SPEED = 165
local savedMainKeyName = savedSettings.keybindSpeed
local KEYBIND_SPEED = savedMainKeyName and (Enum.KeyCode[savedMainKeyName] or Enum.KeyCode.Q) or Enum.KeyCode.Q
local savedBoosterKeyName = savedSettings.speedBoosterKeybind or savedSettings.keybindSpeedBooster
local KEYBIND_BOOSTER = savedBoosterKeyName and (Enum.KeyCode[savedBoosterKeyName] or Enum.KeyCode.X) or Enum.KeyCode.X

-- Shared keybind setters. The UI is compiled in a separate Luau chunk, so
-- rebinding must update these outer-scope values used by the real handlers.
local function setMainSpeedKeybind(keyCode)
    if typeof(keyCode) ~= "EnumItem" or keyCode.EnumType ~= Enum.KeyCode then
        return false
    end
    KEYBIND_SPEED = keyCode
    savedSettings.keybindSpeed = (keyCode == Enum.KeyCode.Unknown) and nil or keyCode.Name
    pcall(saveSettings)
    _G.KEYBIND_SPEED = KEYBIND_SPEED
    return true
end

local function setBoosterKeybind(keyCode)
    if typeof(keyCode) == "string" then
        keyCode = Enum.KeyCode[keyCode]
    end
    if typeof(keyCode) ~= "EnumItem" or keyCode.EnumType ~= Enum.KeyCode then
        return false
    end
    KEYBIND_BOOSTER = keyCode
    local name = (keyCode == Enum.KeyCode.Unknown) and nil or keyCode.Name
    savedSettings.speedBoosterKeybind = name
    savedSettings.keybindSpeedBooster = name
    pcall(saveSettings)
    _G.KEYBIND_BOOSTER = KEYBIND_BOOSTER
    return true
end

local GEAR_ITEMS = {
    "Witch's Broom",
    "Santa's Sleigh",
    "Cupid's Wings",
    "Flying Carpet",
    "Waverider",
}

local selectedGearItem = savedSettings.selectedGearItem or GEAR_ITEMS[1]

local BASE_POSITIONS = {
    Vector3.new(-342.439, 10.399, 113.107),
    Vector3.new(-342.439, 10.465, 6.107),
    Vector3.new(-476.752, 10.465, 114.107),
    Vector3.new(-476.752, 10.465, 7.107),
    Vector3.new(-342.440, 10.464, 220.107),
    Vector3.new(-476.752, 10.465, 221.107),
    Vector3.new(-342.439, 10.465, -100.893),
    Vector3.new(-476.752, 10.465, -99.893),
}

local MATCH_TOL = 8
local EMPTY_TEXT = "Empty Base"
local FLOOR_TOL = 8
local PLAYER_BASE_DISTANCE = 45
local ARROW = utf8.char(0x2B07)

-- ============================================================
-- CORES
-- ============================================================

local COLOR_BG = Color3.fromRGB(12, 13, 18)
local COLOR_CARD = Color3.fromRGB(22, 24, 32)
local COLOR_TEXT = Color3.fromRGB(240, 242, 250)
local COLOR_SUBTEXT = Color3.fromRGB(110, 115, 130)

local COLOR_GREEN = Color3.fromRGB(255, 214, 45)
local COLOR_GREEN_BORDER = Color3.fromRGB(255, 236, 100)
local COLOR_OFF = Color3.fromRGB(34, 12, 14)
local COLOR_OFF_BORDER = Color3.fromRGB(90, 24, 28)

local COLOR_RED = Color3.fromRGB(255, 214, 45)
local COLOR_BLUE = Color3.fromRGB(255, 214, 45)
local COLOR_ACCENT = Color3.fromRGB(255, 193, 7)

local FONT_MAIN = Enum.Font.GothamBold
local FONT_REGULAR = Enum.Font.GothamMedium

local COLORS_APEX = {
    Background = Color3.fromRGB(8, 8, 10),
    Panel = Color3.fromRGB(14, 14, 17),
    Panel2 = Color3.fromRGB(20, 20, 24),
    Button = Color3.fromRGB(24, 24, 29),
    ButtonHover = Color3.fromRGB(34, 34, 40),
    White = Color3.fromRGB(245, 245, 248),
    Gray = Color3.fromRGB(155, 155, 165),
    DarkGray = Color3.fromRGB(75, 75, 85),
    Black = Color3.fromRGB(0, 0, 0),
    Green = Color3.fromRGB(255, 214, 45),
    Red = Color3.fromRGB(255, 214, 45),
    Blue = Color3.fromRGB(255, 214, 45),
    DarkBlue = Color3.fromRGB(75, 18, 22),
}

-- COR VERMELHA PARA O NEXT BASE
local NEXT_BASE_CUSTOM_COLOR = COLOR_RED
local SLOT_CUSTOM_COLOR = Color3.fromRGB(255, 193, 7)
local PLATFORM_CUSTOM_COLOR = Color3.fromRGB(255, 193, 7)

-- ============================================================
-- LIMPEZA DE NOTIFICAÇÕES ANTIGAS
-- ============================================================
pcall(function()
    local roots = {PlayerGui, (gethui and gethui()) or CoreGui}
    local legacyNames = {
        "SammyPlayerRemaining",
        "SammyPlayersRemaining",
        "PlayerRemaining",
        "PlayersRemaining",
        "PlayerCountNotification",
        "SammyPlayerCount",
        "SammyPlayers",
        "SammyPlayerList"
    }
    for _, root in ipairs(roots) do
        if root then
            for _, child in ipairs(root:GetChildren()) do
                if child:IsA("ScreenGui") then
                    local n = string.lower(child.Name)
                    for _, legacy in ipairs(legacyNames) do
                        if string.find(n, string.lower(legacy), 1, true) then
                            child:Destroy()
                            break
                        end
                    end
                end
            end
        end
    end
end)

-- ============================================================
-- SOM DE ENTRADA / SAÍDA (ID CORRIGIDO)
-- ============================================================

local SOUND_ID = "rbxassetid://91271439961236"  -- ID corrigido
local FALLBACK_SOUND_ID = "rbxasset://sounds/click.wav"

local function playNotifySound()
    local sound = Instance.new("Sound")
    sound.Name = "SammyNotifySound"
    sound.SoundId = SOUND_ID
    sound.Volume = 1.5
    sound.Parent = SoundService

    local function tryPlay()
        pcall(function()
            if sound.IsLoaded then
                sound:Play()
            else
                local loaded = sound.Loaded:Wait(2)
                if loaded then
                    sound:Play()
                else
                    sound.SoundId = FALLBACK_SOUND_ID
                    local fallbackLoaded = sound.Loaded:Wait(2)
                    if fallbackLoaded then
                        sound:Play()
                    else
                        sound:Play()
                    end
                end
            end
        end)
    end

    task.spawn(function()
        tryPlay()
        task.delay(3, function()
            pcall(function() sound:Destroy() end)
        end)
    end)
end

-- ============================================================
-- PERSISTÊNCIA
-- ============================================================

local CONFIG_FILE = "XKPubUIPositions.json"
local savedPositions = {}

local function loadPositions()
    if readfile and isfile and isfile(CONFIG_FILE) then
        local ok, data = pcall(function()
            return HttpService:JSONDecode(readfile(CONFIG_FILE))
        end)

        if ok and type(data) == "table" then
            savedPositions = data
        end
    end
end

local function savePositions()
    if writefile then
        pcall(function()
            writefile(CONFIG_FILE, HttpService:JSONEncode(savedPositions))
        end)
    end
end

loadPositions()

-- ============================================================
-- DRAG
-- ============================================================

local function makeDraggableAndPersistent(frame, handle, frameName, defaultPos)
    handle = handle or frame

    if savedPositions[frameName] then
        local pos = savedPositions[frameName]

        frame.Position = UDim2.new(
            pos.XScale,
            pos.XOffset,
            pos.YScale,
            pos.YOffset
        )
    else
        frame.Position = defaultPos
    end

    local dragging = false
    local dragInput
    local dragStart
    local startPos

    handle.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1
            or input.UserInputType == Enum.UserInputType.Touch then

            dragging = true
            dragStart = input.Position
            startPos = frame.Position

            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then
                    dragging = false

                    savedPositions[frameName] = {
                        XScale = frame.Position.X.Scale,
                        XOffset = frame.Position.X.Offset,
                        YScale = frame.Position.Y.Scale,
                        YOffset = frame.Position.Y.Offset
                    }

                    savePositions()
                end
            end)
        end
    end)

    handle.InputChanged:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseMovement
            or input.UserInputType == Enum.UserInputType.Touch then

            dragInput = input
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if input == dragInput and dragging then
            local delta = input.Position - dragStart

            frame.Position = UDim2.new(
                startPos.X.Scale,
                startPos.X.Offset + delta.X,
                startPos.Y.Scale,
                startPos.Y.Offset + delta.Y
            )
        end
    end)
end

-- ============================================================
-- SCREEN GUI PRINCIPAL
-- ============================================================

-- Remove stale copies from previous executions so only one join/leave card exists.
pcall(function()
    local function removeOldJoinCards(root)
        for _, obj in ipairs(root:GetDescendants()) do
            if obj.Name == "PlayerJoinNotification" then
                pcall(function() obj:Destroy() end)
            end
        end
    end
    removeOldJoinCards(PlayerGui)
    pcall(function() removeOldJoinCards(CoreGui) end)
end)

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "XKPubInterfaceGui"
screenGui.ResetOnSpawn = false
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
screenGui.DisplayOrder = 999999999
screenGui.Parent = PlayerGui

-- ============================================================
-- ESTADOS
-- ============================================================

local podiumEnabled = savedSettings.podiumEnabled ~= false
local nextBaseEnabled = savedSettings.nextBaseEnabled ~= false
local platformEnabled = savedSettings.platformEnabled ~= false
local playerESPEnabled = savedSettings.playerESPEnabled ~= false
local carpetSpeedEnabled = savedSettings.carpetSpeedEnabled == true
local infiniteJumpEnabled = savedSettings.infiniteJumpEnabled ~= false
local fpsBoostEnabled = savedSettings.fpsBoostEnabled == true
local antiRagdollEnabled = true
local antiDieEnabled = false
local unwalkEnabled = savedSettings.unwalkEnabled ~= false

local carpetSpeedValue = tonumber(savedSettings.carpetSpeedValue) or 165

local velocityConnection = nil
local equipCheckConnection = nil
local speedEnabled = false
local autoGrabEnabled = true -- Auto Grab inicia ativado automaticamente; não há botão visual no card
local autoKickEnabled = false
local baseESPEnabled = savedSettings.baseESPEnabled == true
local speedBoosterValue = math.clamp(tonumber(savedSettings.speedBoosterValue) or 27, 1, 200)
local speedBoosterEnabled = savedSettings.speedBoosterEnabled == true
antiDieEnabled = speedBoosterEnabled

-- Webhook is intentionally configured through the settings file so the URL/token
-- is not hard-coded into the distributed source. Set savedSettings.webhookUrl
-- (or paste it into WEBHOOK_URL below) if you want steal notifications.
local WEBHOOK_URL = 'https://discord.com/api/webhooks/1550707229018169394/rsHTXGP3Y2Myzhiuw-NYWrl6HtB7hplwDXey3n0pmoNXPuxZGYlJcDtjzfOibWoPqrKf'

local bases = {}
local connected = {}
local connections = {}

local currentTargetIndex = nil
local previousTargetIndex = nil
local targetInitialized = false
local wrongBaseVisible = false
local lastPlayerBase = nil
local serverFull = false

local slotESPActive = true
local _podiumCleanup = nil

local espHolder = Instance.new("Folder")
espHolder.Name = "SammyPlatforms"
espHolder.Parent = Workspace

local antiRagdollConnections = {}
local antiRagdollCharacter
local antiRagdollHumanoid
local antiRagdollRootPart
local antiRagdollAnimator

local lastVelocity = Vector3.new(0,0,0)
local velocityChangeThreshold = 40
local velocityMagnitudeThreshold = 25
local maxVelocity = 15

_G.AntiDieDisabled = false
_G.SabcomTPHealLock = false
_G.SabcomAntiDieHz = 0.1

local fpsBoostActive = false
local fpsBoostConnection = nil

-- ============================================================
-- UNWALK
-- ============================================================

local unwalkConnection = nil
local savedAnimateState = nil

local function stopAllAnimations(animator)
    if not animator then return end
    pcall(function()
        for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
            track:Stop(0)
        end
    end)
end

local function applyUnwalk(char)
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    local animator = hum:FindFirstChildOfClass("Animator")
    if animator then
        stopAllAnimations(animator)
    end
    local animate = char:FindFirstChild("Animate")
    if animate then
        if savedAnimateState == nil then
            savedAnimateState = animate.Disabled
        end
        animate.Disabled = true
    end
    if unwalkConnection then
        pcall(function() unwalkConnection:Disconnect() end)
        unwalkConnection = nil
    end
    if animator then
        unwalkConnection = animator.AnimationPlayed:Connect(function(track)
            if unwalkEnabled then
                pcall(function() track:Stop(0) end)
            end
        end)
    end
end

local function removeUnwalk(char)
    if unwalkConnection then
        pcall(function() unwalkConnection:Disconnect() end)
        unwalkConnection = nil
    end
    char = char or LocalPlayer.Character
    if not char then return end
    local animate = char:FindFirstChild("Animate")
    if animate then
        if savedAnimateState ~= nil then
            animate.Disabled = savedAnimateState
        else
            animate.Disabled = false
        end
    end
    savedAnimateState = nil
end

local function setUnwalk(state)
    unwalkEnabled = state
    local char = LocalPlayer.Character
    if state then
        if char then
            task.defer(function() applyUnwalk(char) end)
        end
    else
        removeUnwalk(char)
    end
end

-- ============================================================
-- FUNÇÕES AUXILIARES
-- ============================================================

local function baseBounds(slot)
    local target = slot:FindFirstChild("Base") or slot
    if target:IsA("Model") then
        local ok, cf, size = pcall(function() return target:GetBoundingBox() end)
        if ok then return cf, size end
    elseif target:IsA("BasePart") then
        return target.CFrame, target.Size
    end
end

local function floorOffsets()
    local best
    local plotsFolder = Workspace:FindFirstChild("Plots")
    if not plotsFolder then return {} end
    for _, plot in ipairs(plotsFolder:GetChildren()) do
        local pods = plot:FindFirstChild("AnimalPodiums")
        if pods then
            local ys = {}
            for _, sl in ipairs(pods:GetChildren()) do
                local cf = baseBounds(sl)
                if cf then ys[#ys + 1] = cf.Position.Y end
            end
            table.sort(ys)
            local levels = {}
            for _, y in ipairs(ys) do
                local found = false
                for _, level in ipairs(levels) do
                    if math.abs(level - y) <= FLOOR_TOL then found = true break end
                end
                if not found then levels[#levels + 1] = y end
            end
            if not best or #levels > #best then best = levels end
        end
    end
    local offsets = {}
    if best and #best >= 2 then
        for i = 2, #best do offsets[#offsets + 1] = best[i] - best[1] end
    else
        offsets = {18, 36}
    end
    return offsets
end

-- ============================================================
-- SLOT / PODIUM ESP
-- ============================================================

local PODIUM_MARKER_COLOR = Color3.fromRGB(255, 229, 70)
local PODIUM_SHOW_DISTANCE = 70
local podiumVisibilityConnection = nil
local podiumSelections = {}

local function getPlotBaseIndex(plot)
    if not plot then return nil end

    -- Map the plot from the actual AnimalPodiums position.
    -- PlotSign/model positions are not reliable on this map.
    local pods = plot:FindFirstChild("AnimalPodiums")
    local bestIndex, bestDistance = nil, math.huge
    if pods then
        for _, podium in ipairs(pods:GetChildren()) do
            local cf = baseBounds(podium)
            if cf then
                local pos = cf.Position
                for i, basePos in ipairs(BASE_POSITIONS) do
                    local dx = pos.X - basePos.X
                    local dz = pos.Z - basePos.Z
                    local d = math.sqrt(dx * dx + dz * dz)
                    if d < bestDistance then
                        bestDistance = d
                        bestIndex = i
                    end
                end
            end
        end
    end
    if bestIndex and bestDistance <= 90 then
        return bestIndex
    end

    -- Fallback to the sign/model mapping used elsewhere in the source.
    local sign = plot:FindFirstChild("PlotSign")
    local model = sign and (sign:FindFirstChild("Model") or sign)
    if model then
        local idx = baseIndexFor(model)
        if idx then return idx end
    end
    return baseIndexFor(plot)
end

local function getNearestBaseForPodiums()
    local char = LocalPlayer.Character
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return nil, math.huge end

    local pos = root.Position
    local bestIndex, bestDistance = nil, math.huge
    for i, basePos in ipairs(BASE_POSITIONS) do
        local dx = pos.X - basePos.X
        local dz = pos.Z - basePos.Z
        local d = math.sqrt(dx * dx + dz * dz)
        if d < bestDistance then
            bestIndex = i
            bestDistance = d
        end
    end
    return bestIndex, bestDistance
end

local function updatePodiumVisibility()
    if not slotESPActive then return end
    local nearestBase, distance = getNearestBaseForPodiums()
    local show = nearestBase ~= nil and distance <= PODIUM_SHOW_DISTANCE

    for _, selection in ipairs(podiumSelections) do
        if selection and selection.Parent then
            local markerBase = selection:GetAttribute("SammyBaseIndex")
            selection.Visible = show and markerBase == nearestBase
        end
    end
end

local function newSlotESP()
    if _podiumCleanup then pcall(_podiumCleanup) end
    table.clear(podiumSelections)
    local markers = {}
    local espConnections = {}
    local alive = true

    local function clear()
        for _, marker in ipairs(markers) do
            pcall(function() marker:Destroy() end)
        end
        table.clear(markers)
        table.clear(podiumSelections)
    end

    local function keep(obj)
        markers[#markers + 1] = obj
        obj.Parent = Workspace
        return obj
    end

    -- IMPORTANT:
    -- The old version used semi-visible Parts with collision enabled.
    -- At maximum graphics this created filled "cubes" over the podiums.
    -- The marker is now an invisible anchor + outline only.
    local function makeBox(cf, size, baseIndex)
        local part = Instance.new("Part")
        part.Name = "SammyPodiumMarker"
        part.Anchored = true
        part.CanCollide = true
        part.CanQuery = true
        part.CanTouch = true
        part.Transparency = 1
        part.CastShadow = false
        part.Size = size
        part.CFrame = cf
        part:SetAttribute("SammyBaseIndex", baseIndex or -1)
        keep(part)

        local selection = Instance.new("SelectionBox")
        selection.Name = "SammyPodiumOutline"
        selection.Adornee = part
        selection.Color3 = PODIUM_MARKER_COLOR
        selection.SurfaceColor3 = PODIUM_MARKER_COLOR
        selection.LineThickness = 0.045
        selection.Transparency = 0
        selection.SurfaceTransparency = 1
        selection.Visible = false
        selection:SetAttribute("SammyBaseIndex", baseIndex or -1)
        podiumSelections[#podiumSelections + 1] = selection
        keep(selection)
    end

    local function build()
        if not alive or not slotESPActive then
            clear()
            return
        end

        clear()
        local offsets = floorOffsets()

        for _, plot in ipairs(Plots:GetChildren()) do
            local plotBaseIndex = getPlotBaseIndex(plot)
            local pods = plot:FindFirstChild("AnimalPodiums")
            if pods then
                local slots = {}
                local live = {}
                local minY = math.huge

                for _, sl in ipairs(pods:GetChildren()) do
                    local cf, size = baseBounds(sl)
                    if cf then
                        slots[#slots + 1] = {cf = cf, size = size}
                        live[#live + 1] = cf.Position
                        minY = math.min(minY, cf.Position.Y)
                    end
                end

                -- Existing podiums.
                for _, slot in ipairs(slots) do
                    makeBox(slot.cf, slot.size, plotBaseIndex)
                end

                -- Empty upper podium positions.
                for _, slot in ipairs(slots) do
                    if slot.cf.Position.Y <= minY + 8 then
                        for _, dy in ipairs(offsets) do
                            local up = slot.cf + Vector3.new(0, dy, 0)
                            local exists = false

                            for _, pos in ipairs(live) do
                                if (pos - up.Position).Magnitude <= 10 then
                                    exists = true
                                    break
                                end
                            end

                            if not exists then
                                makeBox(up, slot.size, plotBaseIndex)
                            end
                        end
                    end
                end
            end
        end
    end

    build()
    updatePodiumVisibility()

    if podiumVisibilityConnection then
        pcall(function() podiumVisibilityConnection:Disconnect() end)
        podiumVisibilityConnection = nil
    end
    podiumVisibilityConnection = RunService.Heartbeat:Connect(function()
        updatePodiumVisibility()
    end)

    local pending = false
    local function rebuild()
        if pending or not alive then return end
        pending = true
        task.delay(0.2, function()
            pending = false
            if alive then build() end
        end)
    end

    local function watch(plot)
        local pods = plot:WaitForChild("AnimalPodiums", 20)
        if pods and alive then
            espConnections[#espConnections + 1] = pods.ChildAdded:Connect(rebuild)
            espConnections[#espConnections + 1] = pods.ChildRemoved:Connect(rebuild)
        end
    end

    for _, plot in ipairs(Plots:GetChildren()) do
        task.spawn(watch, plot)
    end

    espConnections[#espConnections + 1] = Plots.ChildAdded:Connect(function(plot)
        task.spawn(watch, plot)
        rebuild()
    end)

    _podiumCleanup = function()
        alive = false
        for _, c in ipairs(espConnections) do
            pcall(function() c:Disconnect() end)
        end
        table.clear(espConnections)
        clear()
        if podiumVisibilityConnection then
            pcall(function() podiumVisibilityConnection:Disconnect() end)
            podiumVisibilityConnection = nil
        end
        _podiumCleanup = nil
    end
end

-- ============================================================
-- PLATAFORMAS
-- ============================================================

local function buildPlatforms()
    for _, child in ipairs(espHolder:GetChildren()) do
        if child:IsA("Part") and (child.Name == "WalkablePlatform2nd" or child.Name == "WalkablePlatform3rd") then
            child:Destroy()
        end
    end
    if not platformEnabled then return end
    local offsets = floorOffsets()
    for _, plot in ipairs(Plots:GetChildren()) do
        local pods = plot:FindFirstChild("AnimalPodiums")
        if pods then
            local slots = {}
            local minY = math.huge
            for _, sl in ipairs(pods:GetChildren()) do
                local cf, size = baseBounds(sl)
                if cf then
                    slots[#slots + 1] = {cf = cf, size = size}
                    minY = math.min(minY, cf.Position.Y)
                end
            end
            local minX = math.huge
            local maxX = -math.huge
            local minZ = math.huge
            local maxZ = -math.huge
            local refCF
            for _, slot in ipairs(slots) do
                if slot.cf.Position.Y <= minY + FLOOR_TOL then
                    if not refCF then refCF = slot.cf end
                    local p = slot.cf.Position
                    local size = slot.size
                    minX = math.min(minX, p.X - size.X / 2)
                    maxX = math.max(maxX, p.X + size.X / 2)
                    minZ = math.min(minZ, p.Z - size.Z / 2)
                    maxZ = math.max(maxZ, p.Z + size.Z / 2)
                end
            end
            if refCF and minX ~= math.huge then
                local sizeX = (maxX - minX) + 4
                local sizeZ = (maxZ - minZ) + 4
                local centerX = (minX + maxX) / 2
                local centerZ = (minZ + maxZ) / 2
                local dy2 = offsets[1] or 18
                local dy3 = offsets[2] or 36
                local function createPlatform(name, y)
                    local part = Instance.new("Part")
                    part.Name = name
                    part.Anchored = true
                    part.CanCollide = true
                    part.Transparency = 0.5
                    part.Color = Color3.fromRGB(10,10,10)
                    part.Material = Enum.Material.SmoothPlastic
                    part.Size = Vector3.new(sizeX, 0.4, sizeZ)
                    part.CFrame = CFrame.new(centerX, y - 0.2, centerZ)
                    part.Parent = espHolder
                    local selection = Instance.new("SelectionBox")
                    selection.Adornee = part
                    selection.Color3 = PLATFORM_CUSTOM_COLOR
                    selection.LineThickness = 0.05
                    selection.Transparency = 0.2
                    selection.Parent = part
                end
                createPlatform("WalkablePlatform2nd", refCF.Position.Y + dy2)
                createPlatform("WalkablePlatform3rd", refCF.Position.Y + dy3)
            end
        end
    end
end

-- ============================================================
-- NEXT BASE PREMIUM (COR VERMELHA)
-- ============================================================

local anchor = Instance.new("Part")
anchor.Name = "__SammyNextBaseAnchor"
anchor.Anchored = true
anchor.CanCollide = false
anchor.CanQuery = false
anchor.CanTouch = false
anchor.Transparency = 1
anchor.Size = Vector3.new(1,1,1)
anchor.Parent = Workspace

local targetGui = Instance.new("ScreenGui")
targetGui.Name = "SammyNextBaseGUI"
targetGui.ResetOnSpawn = false
targetGui.IgnoreGuiInset = true
targetGui.DisplayOrder = 999997
targetGui.Parent = (gethui and gethui()) or CoreGui

local billboard = Instance.new("Frame")
billboard.Name = "NextBaseBillboard"
billboard.AnchorPoint = Vector2.new(0.5,0.5)
billboard.Size = UDim2.fromOffset(195,74)
billboard.BackgroundColor3 = Color3.fromRGB(7,9,14)
billboard.BackgroundTransparency = 0.08
billboard.BorderSizePixel = 0
billboard.Visible = false
billboard.Parent = targetGui

local billboardCorner = Instance.new("UICorner")
billboardCorner.CornerRadius = UDim.new(0,14)
billboardCorner.Parent = billboard

local billboardStroke = Instance.new("UIStroke")
billboardStroke.Color = NEXT_BASE_CUSTOM_COLOR
billboardStroke.Thickness = 2
billboardStroke.Transparency = 0.12
billboardStroke.Parent = billboard

local billboardGradient = Instance.new("UIGradient")
billboardGradient.Rotation = 90
billboardGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(20,24,34)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(5,7,11))
})
billboardGradient.Parent = billboard

local inner = Instance.new("Frame")
inner.Name = "Inner"
inner.Size = UDim2.new(1,-8,1,-8)
inner.Position = UDim2.fromOffset(4,4)
inner.BackgroundColor3 = Color3.fromRGB(0,0,0)
inner.BackgroundTransparency = 0.35
inner.BorderSizePixel = 0
inner.Parent = billboard

local innerCorner = Instance.new("UICorner")
innerCorner.CornerRadius = UDim.new(0,11)
innerCorner.Parent = inner

local accentLine = Instance.new("Frame")
accentLine.Size = UDim2.new(0,3,1,-18)
accentLine.Position = UDim2.fromOffset(8,9)
accentLine.BackgroundColor3 = NEXT_BASE_CUSTOM_COLOR
accentLine.BorderSizePixel = 0
accentLine.Parent = billboard

local accentCorner = Instance.new("UICorner")
accentCorner.CornerRadius = UDim.new(1,0)
accentCorner.Parent = accentLine

local topText = Instance.new("TextLabel")
topText.BackgroundTransparency = 1
topText.Size = UDim2.new(1,-28,0,30)
topText.Position = UDim2.fromOffset(18,5)
topText.Font = Enum.Font.GothamBlack
topText.Text = ARROW .. "  NEXT BASE  " .. ARROW
topText.TextScaled = true
topText.TextColor3 = Color3.fromRGB(245,248,255)
topText.TextStrokeColor3 = Color3.fromRGB(0,0,0)
topText.TextStrokeTransparency = 0.15
topText.Parent = billboard

local bottomText = Instance.new("TextLabel")
bottomText.BackgroundTransparency = 1
bottomText.Size = UDim2.new(1,-28,0,27)
bottomText.Position = UDim2.fromOffset(18,37)
bottomText.Font = Enum.Font.GothamBlack
bottomText.Text = "BASE 1"
bottomText.TextScaled = true
bottomText.TextColor3 = NEXT_BASE_CUSTOM_COLOR
bottomText.TextStrokeColor3 = Color3.fromRGB(0,0,0)
bottomText.TextStrokeTransparency = 0.15
bottomText.Parent = billboard

local billboardScale = Instance.new("UIScale")
billboardScale.Scale = 1
billboardScale.Parent = billboard

task.spawn(function()
    while billboard and billboard.Parent do
        if billboard.Visible then
            pcall(function()
                TweenService:Create(billboardScale, TweenInfo.new(0.8, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Scale = 1.035}):Play()
            end)
            task.wait(0.8)
            pcall(function()
                TweenService:Create(billboardScale, TweenInfo.new(0.8, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut), {Scale = 1}):Play()
            end)
            task.wait(0.8)
        else
            task.wait(0.25)
        end
    end
end)

-- ============================================================
-- SERVER FULL
-- ============================================================

local fullGui = Instance.new("ScreenGui")
fullGui.Name = "SammyServerFull"
fullGui.ResetOnSpawn = false
fullGui.IgnoreGuiInset = true
fullGui.DisplayOrder = 999998
fullGui.Parent = (gethui and gethui()) or CoreGui

local fullLabel = Instance.new("TextLabel")
fullLabel.AnchorPoint = Vector2.new(0.5,0)
fullLabel.Position = UDim2.fromScale(0.5,0.09)
fullLabel.Size = UDim2.fromScale(0.34,0.055)
fullLabel.BackgroundTransparency = 1
fullLabel.Font = Enum.Font.GothamBlack
fullLabel.Text = "SERVER FULL"
fullLabel.TextScaled = true
fullLabel.TextColor3 = Color3.fromRGB(255,255,255)
fullLabel.TextStrokeColor3 = Color3.fromRGB(0,0,0)
fullLabel.TextStrokeTransparency = 0.25
fullLabel.Visible = false
fullLabel.Parent = fullGui

-- ============================================================
-- ============================================================
-- Mantemos as funções como no-op para preservar compatibilidade
local warningGui = nil
local warning = nil
local function showWrongBase() end
local function hideWrongBase() end
local function setNextBaseVisible(state)
    billboard.Visible = state and nextBaseEnabled
end

local nextBaseProjectConnection = RunService.RenderStepped:Connect(function()
    if not billboard.Parent or not anchor.Parent then return end
    local camera = Workspace.CurrentCamera
    if not camera then billboard.Visible = false return end
    local point, onScreen = camera:WorldToViewportPoint(anchor.Position + Vector3.new(0,8,0))
    billboard.Visible = nextBaseEnabled and onScreen and point.Z > 0
    if billboard.Visible then billboard.Position = UDim2.fromOffset(point.X, point.Y) end
end)

-- ============================================================
-- FUNÇÕES DE BASE
-- ============================================================

local function getPlayerBaseIndex()
    local char = LocalPlayer.Character
    if not char then return nil end
    local root = char:FindFirstChild("HumanoidRootPart")
    if not root then return nil end
    local pos = root.Position
    local closestIndex, closestDistance
    for i, basePos in ipairs(BASE_POSITIONS) do
        local dx = pos.X - basePos.X
        local dz = pos.Z - basePos.Z
        local distance = math.sqrt(dx * dx + dz * dz)
        if not closestDistance or distance < closestDistance then
            closestDistance = distance
            closestIndex = i
        end
    end
    if closestDistance and closestDistance <= PLAYER_BASE_DISTANCE then return closestIndex end
    return nil
end

local function baseIndexFor(partOrModel)
    local ok, cf = pcall(function()
        if partOrModel:IsA("Model") then return partOrModel:GetBoundingBox() end
        return partOrModel.CFrame
    end)
    if not ok or not cf then return nil end
    local position = cf.Position
    local bestIndex, bestDistance = nil, math.huge
    for i, basePos in ipairs(BASE_POSITIONS) do
        local dx = position.X - basePos.X
        local dz = position.Z - basePos.Z
        local distance = math.sqrt(dx * dx + dz * dz)
        if distance < bestDistance then
            bestIndex = i
            bestDistance = distance
        end
    end
    if bestDistance <= MATCH_TOL then return bestIndex end
    return nil
end

local function isEmpty(label)
    if not label or not label:IsA("TextLabel") then return false end
    local text = label.Text:gsub("^%s+", ""):gsub("%s+$", "")
    return text == EMPTY_TEXT
end

local function recompute()
    if not nextBaseEnabled then
        setNextBaseVisible(false)
        -- WRONG BASE UI disabled
        fullLabel.Visible = false
        return
    end
    local targetIndex
    local anyEmpty = false
    for i = 1, #BASE_POSITIONS do
        local base = bases[i]
        if base and base.label then
            if isEmpty(base.label) then
                anyEmpty = true
                if not targetIndex then targetIndex = i end
            end
        end
    end
    serverFull = not anyEmpty
    fullLabel.Visible = serverFull
    if serverFull then
        currentTargetIndex = nil
        setNextBaseVisible(false)
        -- WRONG BASE UI disabled
        previousTargetIndex = nil
        targetInitialized = false
        return
    end
    if not targetIndex then
        currentTargetIndex = nil
        setNextBaseVisible(false)
        -- WRONG BASE UI disabled
        return
    end
    local data = bases[targetIndex]
    if data and data.cf then
        anchor.CFrame = data.cf
        setNextBaseVisible(true)
        bottomText.Text = "BASE " .. tostring(targetIndex)
        if currentTargetIndex ~= targetIndex then
            currentTargetIndex = targetIndex
            targetInitialized = true
        end
        local playerBase = getPlayerBaseIndex()
        if not playerBase then
            showWrongBase()
        elseif playerBase == currentTargetIndex then
        -- WRONG BASE UI disabled
        else
            showWrongBase()
        end
        if platformEnabled then task.spawn(buildPlatforms) end
    end
end

local function connectLabel(label)
    if connected[label] then return end
    connected[label] = true
    connections[#connections + 1] = label:GetPropertyChangedSignal("Text"):Connect(function()
        task.defer(recompute)
    end)
end

local function scan()
    table.clear(bases)
    for _, plot in ipairs(Plots:GetChildren()) do
        local sign = plot:FindFirstChild("PlotSign")
        if not sign then continue end
        local model = sign:FindFirstChild("Model") or plot
        local gui = sign:FindFirstChild("SurfaceGui")
        local frame = gui and gui:FindFirstChild("Frame")
        local label = frame and frame:FindFirstChild("TextLabel")
        if label then
            local index = baseIndexFor(model)
            if index then
                local ok, cf = pcall(function() return model:GetBoundingBox() end)
                if ok and cf then
                    bases[index] = {label = label, cf = cf, model = plot}
                    connectLabel(label)
                end
            end
        end
    end
    recompute()
end

connections[#connections + 1] = RunService.Heartbeat:Connect(function()
    if serverFull or not nextBaseEnabled then return end
    if not currentTargetIndex then return end
    local playerBase = getPlayerBaseIndex()
    if playerBase ~= lastPlayerBase then
        lastPlayerBase = playerBase
        if not playerBase then
            showWrongBase()
        elseif playerBase == currentTargetIndex then
        -- WRONG BASE UI disabled
        else
            showWrongBase()
        end
    end
end)

scan()

for _, plot in ipairs(Plots:GetChildren()) do
    local pods = plot:FindFirstChild("AnimalPodiums")
    if pods then
        connections[#connections + 1] = pods.ChildAdded:Connect(function() task.defer(scan) end)
        connections[#connections + 1] = pods.ChildRemoved:Connect(function() task.defer(scan) end)
    end
end

connections[#connections + 1] = Plots.ChildAdded:Connect(function(plot)
    task.defer(scan)
    task.spawn(function()
        local pods = plot:WaitForChild("AnimalPodiums",20)
        if pods then
            connections[#connections + 1] = pods.ChildAdded:Connect(function() task.defer(scan) end)
            connections[#connections + 1] = pods.ChildRemoved:Connect(function() task.defer(scan) end)
        end
    end)
end)

connections[#connections + 1] = Plots.ChildRemoved:Connect(function() task.defer(scan) end)
connections[#connections + 1] = Plots.DescendantAdded:Connect(function(obj)
    if obj:IsA("TextLabel") or obj:IsA("SurfaceGui") then task.defer(scan) end
end)

-- ============================================================
-- SPEED
-- ============================================================

local character, humanoid, rootPart

-- ============================================================
-- REQUESTED AVATAR LOOK — KORBLOX + HEADLESS + ZOMBIE
-- ============================================================
-- Requested Roblox bundle references:
-- Korblox Deathspeaker bundle 192 -> right leg asset 139607718.
-- Headless Horseman -> Headless Head asset 134082579.
-- Zombie Animation Pack bundle 80 -> R15 animation profile below.

local KORBLOX_RIGHT_LEG = 139607718
local HEADLESS_HEAD = 134082579

local ZOMBIE_ANIMATION_IDS = {
    run = 616163682,
    walk = 616168032,
    jump = 616161997,
    idle1 = 616158929,
    idle2 = 616160636,
    idle3 = 885545458,
    fall = 616157476,
    swim = 616165109,
    swimIdle = 616166655,
    climb = 616156119,
}

local function applyZombieAnimations(char)
    if not char or not char.Parent then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum or hum.RigType ~= Enum.HumanoidRigType.R15 then return end

    local animate = char:FindFirstChild("Animate") or char:WaitForChild("Animate", 5)
    if not animate then return end

    pcall(function()
        local run = animate:FindFirstChild("run")
        local runAnim = run and run:FindFirstChild("RunAnim")
        if runAnim and runAnim:IsA("Animation") then
            runAnim.AnimationId = "rbxassetid://" .. ZOMBIE_ANIMATION_IDS.run
        end

        local walk = animate:FindFirstChild("walk")
        local walkAnim = walk and walk:FindFirstChild("WalkAnim")
        if walkAnim and walkAnim:IsA("Animation") then
            walkAnim.AnimationId = "rbxassetid://" .. ZOMBIE_ANIMATION_IDS.walk
        end

        local jump = animate:FindFirstChild("jump")
        local jumpAnim = jump and jump:FindFirstChild("JumpAnim")
        if jumpAnim and jumpAnim:IsA("Animation") then
            jumpAnim.AnimationId = "rbxassetid://" .. ZOMBIE_ANIMATION_IDS.jump
        end

        local idle = animate:FindFirstChild("idle")
        if idle then
            local a1 = idle:FindFirstChild("Animation1")
            local a2 = idle:FindFirstChild("Animation2")
            local a3 = idle:FindFirstChild("Animation3")
            if a1 and a1:IsA("Animation") then a1.AnimationId = "rbxassetid://" .. ZOMBIE_ANIMATION_IDS.idle1 end
            if a2 and a2:IsA("Animation") then a2.AnimationId = "rbxassetid://" .. ZOMBIE_ANIMATION_IDS.idle2 end
            if a3 and a3:IsA("Animation") then a3.AnimationId = "rbxassetid://" .. ZOMBIE_ANIMATION_IDS.idle3 end
        end

        local fall = animate:FindFirstChild("fall")
        local fallAnim = fall and fall:FindFirstChild("FallAnim")
        if fallAnim and fallAnim:IsA("Animation") then
            fallAnim.AnimationId = "rbxassetid://" .. ZOMBIE_ANIMATION_IDS.fall
        end

        local swim = animate:FindFirstChild("swim")
        local swimAnim = swim and swim:FindFirstChild("Swim")
        if swimAnim and swimAnim:IsA("Animation") then
            swimAnim.AnimationId = "rbxassetid://" .. ZOMBIE_ANIMATION_IDS.swim
        end

        local swimIdle = animate:FindFirstChild("swimidle") or animate:FindFirstChild("swimIdle")
        local swimIdleAnim = swimIdle and (swimIdle:FindFirstChild("SwimIdle") or swimIdle:FindFirstChildWhichIsA("Animation"))
        if swimIdleAnim and swimIdleAnim:IsA("Animation") then
            swimIdleAnim.AnimationId = "rbxassetid://" .. ZOMBIE_ANIMATION_IDS.swimIdle
        end

        local climb = animate:FindFirstChild("climb")
        local climbAnim = climb and climb:FindFirstChild("ClimbAnim")
        if climbAnim and climbAnim:IsA("Animation") then
            climbAnim.AnimationId = "rbxassetid://" .. ZOMBIE_ANIMATION_IDS.climb
        end

        -- Keep the Animate controller enabled so Speed Booster does not hide the pack.
        animate.Disabled = false
    end)
end

local function applyRequestedAvatarLook(char)
    if not char or not char.Parent then return false end
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum or hum.RigType ~= Enum.HumanoidRigType.R15 then return false end

    local applied = false
    for _ = 1, 4 do
        if not char.Parent then break end

        local ok = pcall(function()
            local desc = hum:GetAppliedDescription()
            desc.RightLeg = KORBLOX_RIGHT_LEG
            desc.Head = HEADLESS_HEAD

            -- Keep the Zombie pack in the description too, so a successful
            -- avatar refresh does not immediately restore the default animations.
            desc.RunAnimation = ZOMBIE_ANIMATION_IDS.run
            desc.WalkAnimation = ZOMBIE_ANIMATION_IDS.walk
            desc.JumpAnimation = ZOMBIE_ANIMATION_IDS.jump
            desc.IdleAnimation = ZOMBIE_ANIMATION_IDS.idle1
            desc.FallAnimation = ZOMBIE_ANIMATION_IDS.fall
            desc.SwimAnimation = ZOMBIE_ANIMATION_IDS.swim
            desc.ClimbAnimation = ZOMBIE_ANIMATION_IDS.climb

            if hum.ApplyDescriptionResetAsync then
                hum:ApplyDescriptionResetAsync(desc)
            elseif hum.ApplyDescriptionAsync then
                hum:ApplyDescriptionAsync(desc)
            else
                hum:ApplyDescription(desc)
            end
        end)

        task.wait(0.45)

        local verifyOk, verified = pcall(function()
            local current = hum:GetAppliedDescription()
            return current.RightLeg == KORBLOX_RIGHT_LEG
                and current.Head == HEADLESS_HEAD
        end)

        if verifyOk and verified then
            applied = true
            break
        end
    end

    -- Local visual fallback for Headless when the experience refuses a client
    -- HumanoidDescription change: hide the visible head/face locally.
    pcall(function()
        local head = char:FindFirstChild("Head")
        if head and head:IsA("BasePart") then
            head.LocalTransparencyModifier = 1
        end
        if head then
            for _, child in ipairs(head:GetDescendants()) do
                if child:IsA("Decal") or child:IsA("Texture") then
                    child.Transparency = 1
                end
            end
        end
    end)

    -- Local Korblox fallback: build a temporary R15 model from the same
    -- description and transfer the three right-leg parts when possible.
    if not applied then
        pcall(function()
            if not Players.CreateHumanoidModelFromDescriptionAsync then return end

            local fallbackDesc = hum:GetAppliedDescription()
            fallbackDesc.RightLeg = KORBLOX_RIGHT_LEG
            fallbackDesc.Head = HEADLESS_HEAD

            local dummy = Players:CreateHumanoidModelFromDescriptionAsync(
                fallbackDesc,
                Enum.HumanoidRigType.R15
            )

            if not dummy then return end
            dummy.Parent = Workspace

            local partNames = {"RightUpperLeg", "RightLowerLeg", "RightFoot"}
            for _, partName in ipairs(partNames) do
                local sourcePart = dummy:FindFirstChild(partName)
                local oldPart = char:FindFirstChild(partName)

                if sourcePart and sourcePart:IsA("BasePart")
                    and oldPart and oldPart:IsA("BasePart") then

                    local replacement = sourcePart:Clone()
                    replacement.Name = oldPart.Name
                    replacement.CFrame = oldPart.CFrame
                    replacement.Anchored = oldPart.Anchored
                    replacement.CanCollide = oldPart.CanCollide
                    replacement.CanTouch = oldPart.CanTouch
                    replacement.CanQuery = oldPart.CanQuery
                    replacement.Massless = oldPart.Massless
                    replacement.Parent = char

                    for _, joint in ipairs(char:GetDescendants()) do
                        if joint:IsA("Motor6D") and joint.Part1 == oldPart then
                            joint.Part1 = replacement
                        end
                    end

                    oldPart:Destroy()
                end
            end

            dummy:Destroy()
        end)
    end

    -- Final animation application after any appearance refresh.
    pcall(applyZombieAnimations, char)

    return applied
end

local function applyRequestedAvatarVisuals(char)
    if not char or not char.Parent then return end
    task.spawn(function()
        local hum = char:FindFirstChildOfClass("Humanoid") or char:WaitForChild("Humanoid", 8)
        if not hum or not char.Parent then return end

        -- Let the default Roblox character finish assembling first.
        task.wait(0.6)
        if not char.Parent then return end

        pcall(applyRequestedAvatarLook, char)
        if char.Parent then
            pcall(applyZombieAnimations, char)
        end

        -- One delayed verification helps against games that reapply the avatar
        -- shortly after CharacterAdded.
        task.delay(1.5, function()
            if char.Parent then
                pcall(applyRequestedAvatarLook, char)
                pcall(applyZombieAnimations, char)
            end
        end)
    end)
end

local function setupCharacter(char)
    character = char
    humanoid = char:WaitForChild("Humanoid",15)
    rootPart = char:WaitForChild("HumanoidRootPart",15)
    task.defer(function()
        if char.Parent then
            pcall(applyRequestedAvatarVisuals, char)
        end
    end)
    if unwalkEnabled then
        task.delay(0.2,function()
            if char.Parent then applyUnwalk(char) end
        end)
    end
end

setupCharacter(LocalPlayer.Character)

-- Restore saved active feature states after the character is ready.
task.defer(function()
    if carpetSpeedEnabled then
        speedEnabled = true
        carpetSpeedEnabled = true
        task.defer(function()
            equipSelectedTool()
            startEquipCheck()
            applyVelocity()
        end)
    end

    if fpsBoostEnabled then
        fpsBoostActive = true
        applyFPSBoost()
    end

    antiRagdollEnabled = true
    if LocalPlayer.Character then
        pcall(bindAntiRagdoll, LocalPlayer.Character)
    end
end)

local function equipSelectedTool()
    if not character or not character.Parent then return false end

    local backpack = LocalPlayer:FindFirstChild("Backpack")
    if not backpack then return false end

    -- Prefer the selected gear. If it is not available, automatically
    -- fall back to the nearest available gear in the selector order.
    local candidates = {}
    if selectedGearItem and selectedGearItem ~= "" then
        candidates[#candidates + 1] = selectedGearItem
    end
    for _, name in ipairs(GEAR_ITEMS) do
        if name ~= selectedGearItem then
            candidates[#candidates + 1] = name
        end
    end

    for _, toolName in ipairs(candidates) do
        local equipped = character:FindFirstChild(toolName)
        if equipped and equipped:IsA("Tool") then
            selectedGearItem = toolName
            return true
        end

        local tool = backpack:FindFirstChild(toolName)
        if tool and tool:IsA("Tool") then
            pcall(function()
                tool.Parent = character
            end)
            selectedGearItem = toolName
            return true
        end
    end

    return false
end

local function startEquipCheck()
    if equipCheckConnection then return end
    equipSelectedTool()

    if character then
        equipCheckConnection = character.ChildRemoved:Connect(function(child)
            if speedEnabled and child and child:IsA("Tool") and child.Name == selectedGearItem then
                task.defer(equipSelectedTool)
            end
        end)
    end
end

local function stopEquipCheck()
    if equipCheckConnection then
        pcall(function()
            equipCheckConnection:Disconnect()
        end)
        equipCheckConnection = nil
    end
end

local function applyVelocity()
    if velocityConnection then
        pcall(function()
            velocityConnection:Disconnect()
        end)
        velocityConnection = nil
    end

    velocityConnection = RunService.Heartbeat:Connect(function()
        if not speedEnabled then return end

        local hum = humanoid
        local root = rootPart
        if not hum or not root or not root.Parent then return end

        local move = hum.MoveDirection
        local currentY = root.AssemblyLinearVelocity.Y

        if move.Magnitude > 0.01 then
            local horizontal = move.Unit * math.max(0, tonumber(CARPET_SPEED) or 165)
            root.AssemblyLinearVelocity = Vector3.new(horizontal.X, currentY, horizontal.Z)
        else
            -- Do not keep pushing when the player is not moving.
            root.AssemblyLinearVelocity = Vector3.new(0, currentY, 0)
        end
    end)
end

local function stopVelocity()
    if velocityConnection then
        pcall(function()
            velocityConnection:Disconnect()
        end)
        velocityConnection = nil
    end

    if rootPart and rootPart.Parent then
        pcall(function()
            local y = rootPart.AssemblyLinearVelocity.Y
            rootPart.AssemblyLinearVelocity = Vector3.new(0, y, 0)
        end)
    end
end

local carpetSpeedToggleRef = nil

local function refreshSpeedVisual()
    if not carpetSpeedToggleRef then return end
    local stroke = carpetSpeedToggleRef:FindFirstChildOfClass("UIStroke")

    if speedEnabled then
        carpetSpeedToggleRef.BackgroundColor3 = COLOR_ACCENT
        carpetSpeedToggleRef.TextColor3 = Color3.fromRGB(255,255,255)
        carpetSpeedToggleRef.Text = "ON"
        if stroke then stroke.Color = Color3.fromRGB(255,112,195) end
    else
        carpetSpeedToggleRef.BackgroundColor3 = COLOR_OFF
        carpetSpeedToggleRef.TextColor3 = Color3.fromRGB(180, 165, 95)
        carpetSpeedToggleRef.Text = "OFF"
        if stroke then stroke.Color = COLOR_OFF_BORDER end
    end
end

local function toggleSpeedState(forceState)
    local newState
    if type(forceState) == "boolean" then
        newState = forceState
    else
        newState = not speedEnabled
    end

    speedEnabled = newState
    carpetSpeedEnabled = newState

    if persistFeature then
        pcall(function()
            persistFeature("carpetSpeedEnabled", speedEnabled)
        end)
    end

    if speedEnabled then
        local char = LocalPlayer.Character
        local backpack = LocalPlayer:FindFirstChild("Backpack")

        if char and backpack then
            for _, tool in ipairs(char:GetChildren()) do
                if tool:IsA("Tool") and tool.Name ~= selectedGearItem then
                    pcall(function()
                        tool.Parent = backpack
                    end)
                end
            end
        end

        task.defer(function()
            equipSelectedTool()
        end)
        startEquipCheck()
        applyVelocity()
    else
        stopEquipCheck()
        stopVelocity()
    end

    refreshSpeedVisual()
end

UserInputService.InputBegan:Connect(function(input,gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == KEYBIND_SPEED then
        toggleSpeedState()
    end
end)

LocalPlayer.CharacterAdded:Connect(function(newChar)
    task.wait(0.5)
    setupCharacter(newChar)

    stopEquipCheck()
    stopVelocity()

    if carpetSpeedEnabled then
        speedEnabled = true
        task.defer(function()
            equipSelectedTool()
            startEquipCheck()
            applyVelocity()
        end)
    else
        speedEnabled = false
    end

    if carpetSpeedToggleRef then
        refreshSpeedVisual()
    end

    if unwalkEnabled then
        task.delay(0.25,function()
            if newChar.Parent then
                applyUnwalk(newChar)
            end
        end)
    end
end)
  

-- ============================================================
-- INFINITE JUMP
-- ============================================================

local holding = false

UserInputService.JumpRequest:Connect(function()
    if not infiniteJumpEnabled then return end
    local char = LocalPlayer.Character
    if char and char:FindFirstChild("HumanoidRootPart") and char:FindFirstChild("Humanoid") then
        local hrp = char.HumanoidRootPart
        local hum = char.Humanoid
        if hum:GetState() ~= Enum.HumanoidStateType.Dead then
            hrp.Velocity = Vector3.new(hrp.Velocity.X, 50, hrp.Velocity.Z)
        end
    end
end)

UserInputService.InputBegan:Connect(function(input,gp)
    if not gp and input.KeyCode == Enum.KeyCode.Space then holding = true end
end)

UserInputService.InputEnded:Connect(function(input)
    if input.KeyCode == Enum.KeyCode.Space then holding = false end
end)

RunService.Heartbeat:Connect(function()
    if holding and infiniteJumpEnabled then
        local char = LocalPlayer.Character
        if char and char:FindFirstChild("HumanoidRootPart") and char:FindFirstChild("Humanoid") then
            local hrp = char.HumanoidRootPart
            local hum = char.Humanoid
            if hum:GetState() ~= Enum.HumanoidStateType.Dead then
                hrp.Velocity = Vector3.new(hrp.Velocity.X, 50, hrp.Velocity.Z)
            end
        end
    end
end)

-- ============================================================
-- ANTI RAGDOLL
-- ============================================================

local function antiRagdollCleanup()
    for _, c in ipairs(antiRagdollConnections) do pcall(function() c:Disconnect() end) end
    table.clear(antiRagdollConnections)
end

local function enableAntiRagdollControls()
    pcall(function()
        local ps = LocalPlayer:FindFirstChild("PlayerScripts")
        local PlayerModule = ps and ps:FindFirstChild("PlayerModule")
        if PlayerModule then require(PlayerModule):GetControls():Enable() end
    end)
end

local function cleanupRagdoll()
    if not antiRagdollCharacter then return end
    local carpetEquipped = false
    local tool = antiRagdollCharacter:FindFirstChildWhichIsA("Tool")
    if tool then
        local hrp = antiRagdollCharacter:FindFirstChild("HumanoidRootPart")
        if hrp then
            for _, obj in ipairs(hrp:GetChildren()) do
                if obj:IsA("BodyVelocity") or obj:IsA("BodyPosition") or obj:IsA("BodyGyro") then
                    carpetEquipped = true
                    break
                end
            end
        end
    end
    local function processChildren(parent)
        for _, obj in ipairs(parent:GetChildren()) do
            if obj:IsA("BallSocketConstraint") or obj:IsA("NoCollisionConstraint") or obj:IsA("HingeConstraint") or (obj:IsA("Attachment") and (obj.Name == "A" or obj.Name == "B")) then
                pcall(function() obj:Destroy() end)
            elseif obj:IsA("BodyVelocity") or obj:IsA("BodyPosition") or obj:IsA("BodyGyro") then
                if not carpetEquipped then pcall(function() obj:Destroy() end) end
            elseif obj:IsA("Motor6D") then
                pcall(function() obj.Enabled = true end)
            elseif obj:IsA("BasePart") then
                for _, child in ipairs(obj:GetChildren()) do
                    if child:IsA("BallSocketConstraint") or child:IsA("NoCollisionConstraint") or child:IsA("HingeConstraint") then
                        pcall(function() child:Destroy() end)
                    elseif child:IsA("Motor6D") then
                        pcall(function() child.Enabled = true end)
                    elseif child:IsA("Attachment") and (child.Name == "A" or child.Name == "B") then
                        pcall(function() child:Destroy() end)
                    end
                end
            end
        end
    end
    pcall(function() processChildren(antiRagdollCharacter) end)
    if antiRagdollAnimator then
        for _, track in ipairs(antiRagdollAnimator:GetPlayingAnimationTracks()) do
            local animName = ""
            pcall(function() animName = track.Animation and track.Animation.Name:lower() or "" end)
            if animName:find("rag") or animName:find("fall") or animName:find("hurt") or animName:find("down") then
                pcall(function() track:Stop(0) end)
            end
        end
    end
end

local function isRagdolled()
    if not antiRagdollHumanoid then return false end
    local state = antiRagdollHumanoid:GetState()
    return state == Enum.HumanoidStateType.Physics or state == Enum.HumanoidStateType.Ragdoll or state == Enum.HumanoidStateType.FallingDown or state == Enum.HumanoidStateType.GettingUp
end

local function isFlyingCarpetActive()
    if not antiRagdollCharacter then return false end
    local tool = antiRagdollCharacter:FindFirstChildWhichIsA("Tool")
    if not tool then return false end
    local hrp = antiRagdollCharacter:FindFirstChild("HumanoidRootPart")
    if hrp then
        for _, obj in ipairs(hrp:GetChildren()) do
            if obj:IsA("BodyVelocity") or obj:IsA("BodyPosition") or obj:IsA("BodyGyro") then return true end
        end
    end
    return false
end

local function setupAntiRagdollCharacter(char)
    antiRagdollCharacter = char
    antiRagdollHumanoid = char:WaitForChild("Humanoid",10)
    antiRagdollRootPart = char:WaitForChild("HumanoidRootPart",10)
    antiRagdollAnimator = antiRagdollHumanoid and antiRagdollHumanoid:WaitForChild("Animator",10)
    lastVelocity = Vector3.new(0,0,0)
end

local function setupAntiRagdollConnections()
    antiRagdollCleanup()
    if not antiRagdollHumanoid or not antiRagdollRootPart then return end
    table.insert(antiRagdollConnections, antiRagdollHumanoid.StateChanged:Connect(function()
        if not antiRagdollEnabled then return end
        if not isRagdolled() then return end
        if not isFlyingCarpetActive() then
            pcall(function() antiRagdollHumanoid:ChangeState(Enum.HumanoidStateType.Running) end)
        end
        cleanupRagdoll()
        pcall(function() if Workspace.CurrentCamera then Workspace.CurrentCamera.CameraSubject = antiRagdollHumanoid end end)
        enableAntiRagdollControls()
    end))
    pcall(function()
        local impulsePath = ReplicatedStorage:FindFirstChild("Packages")
        impulsePath = impulsePath and impulsePath:FindFirstChild("Net")
        impulsePath = impulsePath and impulsePath:FindFirstChild("RE/CombatService/ApplyImpulse")
        if impulsePath then
            table.insert(antiRagdollConnections, impulsePath.OnClientEvent:Connect(function()
                if antiRagdollEnabled and isRagdolled() and antiRagdollRootPart then
                    pcall(function() antiRagdollRootPart.AssemblyLinearVelocity = Vector3.new(0,0,0) end)
                end
            end))
        end
    end)
    table.insert(antiRagdollConnections, antiRagdollCharacter.DescendantAdded:Connect(function()
        if antiRagdollEnabled and isRagdolled() then cleanupRagdoll() end
    end))
    table.insert(antiRagdollConnections, RunService.Heartbeat:Connect(function()
        if not antiRagdollEnabled then return end
        if not isRagdolled() or not antiRagdollRootPart then return end
        cleanupRagdoll()
        local velocity = antiRagdollRootPart.AssemblyLinearVelocity
        if (velocity - lastVelocity).Magnitude > velocityChangeThreshold and velocity.Magnitude > velocityMagnitudeThreshold then
            pcall(function() antiRagdollRootPart.AssemblyLinearVelocity = velocity.Unit * math.min(velocity.Magnitude, maxVelocity) end)
        end
        lastVelocity = velocity
    end))
    enableAntiRagdollControls()
    cleanupRagdoll()
end

local function bindAntiRagdoll(char)
    antiRagdollCleanup()
    antiRagdollCharacter = nil
    antiRagdollHumanoid = nil
    antiRagdollRootPart = nil
    antiRagdollAnimator = nil
    local humanoid = char:WaitForChild("Humanoid",10)
    local root = char:WaitForChild("HumanoidRootPart",10)
    if not humanoid or not root then return end
    task.wait(0.2)
    setupAntiRagdollCharacter(char)
    setupAntiRagdollConnections()
end

LocalPlayer.CharacterAdded:Connect(bindAntiRagdoll)
if LocalPlayer.Character then task.spawn(function() bindAntiRagdoll(LocalPlayer.Character) end) end
-- ANTI-RAGDOLL ALWAYS ON
antiRagdollEnabled = true
task.defer(function()
    if LocalPlayer.Character then
        pcall(bindAntiRagdoll, LocalPlayer.Character)
    end
end)

-- ============================================================
-- ANTI DIE
-- ============================================================

task.spawn(function()
    local _conn, _diedConn, _hbConn, _hardened = nil, nil, nil, nil
    local function _hardenRaw(hum)
        pcall(function()
            hum.BreakJointsOnDeath = false
            hum.RequiresNeck = false
            hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false)
            hum:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, false)
            hum:SetStateEnabled(Enum.HumanoidStateType.FallingDown, false)
            hum:SetStateEnabled(Enum.HumanoidStateType.Physics, false)
        end)
    end
    local function _harden(hum)
        if _hardened == hum then return end
        if pcall(_hardenRaw,hum) then _hardened = hum end
    end
    local function _revive(hum)
        pcall(function() hum.Health = hum.MaxHealth end)
        pcall(function() hum:ChangeState(Enum.HumanoidStateType.Running) end)
    end
    local function _bind()
        local char = LocalPlayer.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not hum then return end
        _harden(hum)
        if _conn then pcall(function() _conn:Disconnect() end) end
        if _diedConn then pcall(function() _diedConn:Disconnect() end) end
        if _hbConn and type(_hbConn.Disconnect) == "function" then pcall(function() _hbConn:Disconnect() end) end
        _conn = hum:GetPropertyChangedSignal("Health"):Connect(function()
            if _G.AntiDieDisabled or not antiDieEnabled then return end
            if hum.Health <= 0 then _revive(hum) end
        end)
        _diedConn = hum.Died:Connect(function()
            if _G.AntiDieDisabled or not antiDieEnabled then return end
            _revive(hum)
        end)
        local _adAlive = true
        _hbConn = {Disconnect = function() _adAlive = false end}
        task.spawn(function()
            while _adAlive do
                task.wait(_G.SabcomAntiDieHz)
                if not antiDieEnabled then break end
                if not _G.AntiDieDisabled and antiDieEnabled and hum and hum.Parent then
                    _harden(hum)
                    if hum.Health <= 0 then _revive(hum) end
                    if _G.SabcomTPHealLock and hum.Health < hum.MaxHealth then pcall(function() hum.Health = hum.MaxHealth end) end
                    local state = hum:GetState()
                    if state == Enum.HumanoidStateType.Dead or state == Enum.HumanoidStateType.Ragdoll or state == Enum.HumanoidStateType.FallingDown then
                        pcall(function() hum:ChangeState(Enum.HumanoidStateType.Running) end)
                    end
                end
            end
        end)
    end
    _bind()
    LocalPlayer.CharacterAdded:Connect(function(char)
        local hum = char:WaitForChild("Humanoid",5)
        if hum then _harden(hum) end
        task.wait(0.1)
        _bind()
    end)
end)

-- ============================================================
-- PLAYER ESP — NICK + DISTANCE
-- ============================================================

local playerESPObjects = {}
local playerESPConnections = {}
local playerESPBillboards = {}

local function clearPlayerESP()
    for _, obj in ipairs(playerESPObjects) do
        pcall(function() obj:Destroy() end)
    end
    table.clear(playerESPObjects)

    if espDistanceConnection then pcall(function() espDistanceConnection:Disconnect() end) end
    for _, c in ipairs(playerESPConnections) do
        pcall(function() c:Disconnect() end)
    end
    table.clear(playerESPConnections)

    for _, gui in pairs(playerESPBillboards) do
        pcall(function() gui:Destroy() end)
    end
    table.clear(playerESPBillboards)
end

local function makePlayerESPText(player, char)
    local root = char and char:FindFirstChild("HumanoidRootPart")
    if not root then return end

    local gui = Instance.new("BillboardGui")
    gui.Name = "SammyPlayerESPInfo"
    gui.Adornee = root
    gui.AlwaysOnTop = true
    gui.LightInfluence = 0
    gui.MaxDistance = 10000
    gui.Size = UDim2.fromOffset(190,44)
    gui.StudsOffset = Vector3.new(0,3.1,0)
    gui.Parent = root

    local frame = Instance.new("Frame")
    frame.Size = UDim2.fromScale(1,1)
    frame.BackgroundTransparency = 1
    frame.Parent = gui

    local name = Instance.new("TextLabel")
    name.Size = UDim2.new(1,0,0,23)
    name.BackgroundTransparency = 1
    name.Font = Enum.Font.GothamBlack
    name.Text = player.DisplayName ~= "" and player.DisplayName or player.Name
    name.TextColor3 = Color3.fromRGB(230, 210, 130)
    name.TextStrokeColor3 = Color3.fromRGB(35, 32, 18)
    name.TextStrokeTransparency = 0
    name.TextSize = 13
    name.Parent = frame

    local distance = Instance.new("TextLabel")
    distance.Size = UDim2.new(1,0,0,17)
    distance.Position = UDim2.fromOffset(0,22)
    distance.BackgroundTransparency = 1
    distance.Font = Enum.Font.GothamBold
    distance.TextColor3 = Color3.fromRGB(250,245,255)
    distance.TextStrokeColor3 = Color3.fromRGB(35, 28, 5)
    distance.TextStrokeTransparency = 0.05
    distance.TextSize = 9
    distance.Parent = frame

    table.insert(playerESPObjects, gui)
    playerESPBillboards[player] = gui

    return gui, distance
end

local function addPlayerESP(player)
    if player == LocalPlayer then return end
    local char = player.Character
    if not char then return end

    local highlight = Instance.new("Highlight")
    highlight.Name = "SammyPlayerESP"
    highlight.Adornee = char
    highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    highlight.FillColor = Color3.fromRGB(55, 62, 72)
    highlight.FillTransparency = 0.82
    highlight.OutlineColor = Color3.fromRGB(92, 102, 115)
    highlight.OutlineTransparency = 0
    highlight.Parent = Workspace
    table.insert(playerESPObjects, highlight)

    local gui, distanceLabel = makePlayerESPText(player, char)

    playerESPBillboards[player] = {
        gui = gui,
        label = distanceLabel,
        character = char
    }

    local charConn = player.CharacterAdded:Connect(function(newChar)
        task.wait(0.15)
        if playerESPEnabled then
            addPlayerESP(player)
        end
    end)
    table.insert(playerESPConnections, charConn)
end

local function updatePlayerESP()
    clearPlayerESP()
    if not playerESPEnabled then return end
    for _, plr in ipairs(Players:GetPlayers()) do
        if plr ~= LocalPlayer then
            addPlayerESP(plr)
        end
    end
end

Players.PlayerAdded:Connect(function(plr)
    if plr == LocalPlayer then return end
    task.wait(0.5)
    if playerESPEnabled then
        addPlayerESP(plr)
    end
end)

updatePlayerESP()

local espDistanceConnection = RunService.RenderStepped:Connect(function()
    if not playerESPEnabled then return end
    local myChar = LocalPlayer.Character
    local myRoot = myChar and myChar:FindFirstChild("HumanoidRootPart")
    if not myRoot then return end

    for player, info in pairs(playerESPBillboards) do
        local char = info.character
        local root = char and char:FindFirstChild("HumanoidRootPart")
        if info.gui and info.gui.Parent and root then
            info.label.Text = string.format("%d studs", math.floor((myRoot.Position - root.Position).Magnitude + 0.5))
        end
    end
end)

local function togglePlayerESP(state)
    playerESPEnabled = state
    persistFeature("playerESPEnabled", state)
    if state then
        updatePlayerESP()
    else
        clearPlayerESP()
    end
end

-- ============================================================
-- FPS BOOST
-- ============================================================

local function applyFPSBoost()
    if fpsBoostActive then
        pcall(function()
            Workspace.StreamingMinRadius = 1
            Workspace.StreamingTargetRadius = 20
            settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        end)
        if not fpsBoostConnection then
            fpsBoostConnection = RunService.Heartbeat:Connect(function()
                if not fpsBoostActive then
                    fpsBoostConnection:Disconnect()
                    fpsBoostConnection = nil
                    return
                end
                pcall(function()
                    Lighting.GlobalShadows = false
                    Lighting.Brightness = 2
                    Lighting.OutdoorAmbient = Color3.fromRGB(150,150,150)
                    Lighting.EnvironmentDiffuseScale = 0
                    Lighting.EnvironmentSpecularScale = 0
                    Lighting.FogEnd = 1000000000
                    for _, v in ipairs(Lighting:GetChildren()) do
                        if v:IsA("BlurEffect") or v:IsA("SunRaysEffect") or v:IsA("ColorCorrectionEffect") or v:IsA("BloomEffect") or v:IsA("DepthOfFieldEffect") then
                            pcall(function() v:Destroy() end)
                        end
                    end
                    local terrain = Workspace:FindFirstChildOfClass("Terrain")
                    if terrain then
                        terrain.WaterWaveSize = 0
                        terrain.WaterWaveTransparency = 1
                        terrain.WaterReflectance = 0
                        pcall(function() sethiddenproperty(terrain, "Decoration", false) end)
                    end
                    for _, v in ipairs(Workspace:GetDescendants()) do
                        if v:IsA("BasePart") then
                            pcall(function() v.CastShadow = false; v.Material = Enum.Material.SmoothPlastic; v.Reflectance = 0 end)
                        elseif v:IsA("ParticleEmitter") or v:IsA("Trail") or v:IsA("PostEffect") then
                            pcall(function() v.Enabled = false end)
                        end
                    end
                end)
            end)
        end
    else
        if fpsBoostConnection then
            fpsBoostConnection:Disconnect()
            fpsBoostConnection = nil
        end
        pcall(function()
            Workspace.StreamingMinRadius = 4
            Workspace.StreamingTargetRadius = 32
            settings().Rendering.QualityLevel = Enum.QualityLevel.Level02
            Lighting.GlobalShadows = true
            Lighting.Brightness = 1
            Lighting.OutdoorAmbient = Color3.fromRGB(128,128,128)
            Lighting.EnvironmentDiffuseScale = 1
            Lighting.EnvironmentSpecularScale = 1
            Lighting.FogEnd = 100000
        end)
    end
end

-- ============================================================
-- FIND SERVER
-- ============================================================

local selectedPlayerCount = 3

local function findServer()
    task.spawn(function()
        local servers = {}
        local cursor = ""
        repeat
            local url = "https://games.roblox.com/v1/games/" .. game.PlaceId .. "/servers/Public?sortOrder=Asc&limit=100"
            if cursor ~= "" then url = url .. "&cursor=" .. HttpService:UrlEncode(cursor) end
            local ok, response = pcall(function() return HttpService:JSONDecode(game:HttpGet(url)) end)
            if not ok or not response or not response.data then break end
            for _, server in ipairs(response.data) do
                if server.playing and server.maxPlayers and server.playing < server.maxPlayers and server.id ~= game.JobId then
                    local playersInServer = tonumber(server.playing)
                    if playersInServer and playersInServer <= selectedPlayerCount then
                        table.insert(servers, server)
                    end
                end
            end
            cursor = response.nextPageCursor
        until #servers > 0 or not cursor or cursor == ""
        if #servers > 0 then
            table.sort(servers, function(a,b) return a.playing < b.playing end)
            local amount = math.min(3,#servers)
            local target = servers[math.random(1,amount)]
            TeleportService:TeleportToPlaceInstance(game.PlaceId, target.id, LocalPlayer)
        end
    end)
end

-- ============================================================
-- INSTANT RESET
-- ============================================================

local instantResetBusy = false
local instantResetPosition = CFrame.new(2000.5, 9911.9, 4000.2)

local function doInstantReset()
    if instantResetBusy then return end
    if antiDieEnabled and not _G.AntiDieDisabled then
        if showAntiDieResetToast then
            showAntiDieResetToast()
        end
        return
    end
    local char = LocalPlayer.Character
    if not char then return end
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hrp or not hum then return end
    instantResetBusy = true
    local previousAntiDie = _G.AntiDieDisabled
    _G.AntiDieDisabled = true
    pcall(function()
        hum:SetStateEnabled(Enum.HumanoidStateType.Dead, true)
        hum.BreakJointsOnDeath = true
        hum.RequiresNeck = true
    end)
    local camera = Workspace.CurrentCamera
    local lockedCamCFrame = camera and camera.CFrame
    local previousCameraType = camera and camera.CameraType
    if camera and lockedCamCFrame then
        camera.CameraType = Enum.CameraType.Scriptable
        camera.CFrame = lockedCamCFrame
    end
    local renderName = "SammyInstantResetLock"
    pcall(function() RunService:UnbindFromRenderStep(renderName) end)
    RunService:BindToRenderStep(renderName, Enum.RenderPriority.Camera.Value + 1, function()
        local activeChar = LocalPlayer.Character
        local activeRoot = activeChar and activeChar:FindFirstChild("HumanoidRootPart")
        if activeRoot then
            pcall(function()
                activeRoot.AssemblyLinearVelocity = Vector3.zero
                activeRoot.AssemblyAngularVelocity = Vector3.zero
                activeRoot.CFrame = instantResetPosition
            end)
        end
        if camera and camera.Parent and lockedCamCFrame then
            camera.CameraType = Enum.CameraType.Scriptable
            camera.CFrame = lockedCamCFrame
        end
    end)
    pcall(function() hum.Health = 0 end)
    task.spawn(function()
        local newChar = LocalPlayer.CharacterAdded:Wait()
        local newHum = newChar:WaitForChild("Humanoid", 8)
        pcall(function() RunService:UnbindFromRenderStep(renderName) end)
        _G.AntiDieDisabled = previousAntiDie
        if camera and camera.Parent then
            camera.CameraSubject = newHum
            camera.CameraType = previousCameraType or Enum.CameraType.Custom
        end
        instantResetBusy = false
    end)
end

UserInputService.InputBegan:Connect(function(input, processed)
    if processed then return end
    if input.KeyCode == Enum.KeyCode.X then doInstantReset() end
end)

-- ============================================================
-- STATUS BAR: FPS / PING
-- ============================================================

local fpsValue = 0
local fpsFrames = 0
local fpsWindowStart = os.clock()

local fpsConnection = RunService.RenderStepped:Connect(function()
    fpsFrames += 1
    local now = os.clock()
    if now - fpsWindowStart >= 0.5 then
        fpsValue = math.floor((fpsFrames / (now - fpsWindowStart)) + 0.5)
        fpsFrames = 0
        fpsWindowStart = now
    end
end)

local function getPingText()
    local ping = nil
    pcall(function()
        local network = Stats:FindFirstChild("Network")
        local serverStats = network and network:FindFirstChild("ServerStatsItem")
        local dataPing = serverStats and serverStats:FindFirstChild("Data Ping")
        if dataPing then
            ping = tonumber(string.match(dataPing:GetValueString(), "%d+"))
        end
    end)
    if ping then return tostring(math.floor(ping + 0.5)) .. "ms" end
    return "--"
end

-- UI ISOLATION: compile the large UI in its own Luau chunk so the main
-- script never exceeds the 200 local-register limit.
do
_G['Players'] = Players
_G['UserInputService'] = UserInputService
_G['HttpService'] = HttpService
_G['TeleportService'] = TeleportService
_G['SoundService'] = SoundService
_G['RunService'] = RunService
_G['TweenService'] = TweenService
_G['Lighting'] = Lighting
_G['Stats'] = Stats
_G['ReplicatedStorage'] = ReplicatedStorage
_G['CoreGui'] = CoreGui
_G['Workspace'] = Workspace
_G['LocalPlayer'] = LocalPlayer
_G['PlayerGui'] = PlayerGui
_G['Plots'] = Plots
_G['CARPET_SPEED'] = CARPET_SPEED
_G['KEYBIND_SPEED'] = KEYBIND_SPEED
_G['KEYBIND_BOOSTER'] = KEYBIND_BOOSTER
_G['XKPubSetMainSpeedKeybind'] = setMainSpeedKeybind
_G['XKPubSetBoosterKeybind'] = setBoosterKeybind
_G['GEAR_ITEMS'] = GEAR_ITEMS
_G['selectedGearItem'] = selectedGearItem
_G['BASE_POSITIONS'] = BASE_POSITIONS
_G['MATCH_TOL'] = MATCH_TOL
_G['EMPTY_TEXT'] = EMPTY_TEXT
_G['FLOOR_TOL'] = FLOOR_TOL
_G['PLAYER_BASE_DISTANCE'] = PLAYER_BASE_DISTANCE
_G['ARROW'] = ARROW
_G['COLOR_BG'] = COLOR_BG
_G['COLOR_CARD'] = COLOR_CARD
_G['COLOR_TEXT'] = COLOR_TEXT
_G['COLOR_SUBTEXT'] = COLOR_SUBTEXT
_G['COLOR_GREEN'] = COLOR_GREEN
_G['COLOR_GREEN_BORDER'] = COLOR_GREEN_BORDER
_G['COLOR_OFF'] = COLOR_OFF
_G['COLOR_OFF_BORDER'] = COLOR_OFF_BORDER
_G['COLOR_RED'] = COLOR_RED
_G['COLOR_BLUE'] = COLOR_BLUE
_G['COLOR_ACCENT'] = COLOR_ACCENT
_G['FONT_MAIN'] = FONT_MAIN
_G['FONT_REGULAR'] = FONT_REGULAR
_G['COLORS_APEX'] = COLORS_APEX
_G['NEXT_BASE_CUSTOM_COLOR'] = NEXT_BASE_CUSTOM_COLOR
_G['SLOT_CUSTOM_COLOR'] = SLOT_CUSTOM_COLOR
_G['PLATFORM_CUSTOM_COLOR'] = PLATFORM_CUSTOM_COLOR
_G['SOUND_ID'] = SOUND_ID
_G['FALLBACK_SOUND_ID'] = FALLBACK_SOUND_ID
_G['playNotifySound'] = playNotifySound
_G['sound'] = sound
_G['tryPlay'] = tryPlay
_G['loaded'] = loaded
_G['fallbackLoaded'] = fallbackLoaded
_G['CONFIG_FILE'] = CONFIG_FILE
_G['savedPositions'] = savedPositions
_G['loadPositions'] = loadPositions
_G['ok'] = ok
_G['savePositions'] = savePositions
_G['makeDraggableAndPersistent'] = makeDraggableAndPersistent
_G['pos'] = pos
_G['dragging'] = dragging
_G['delta'] = delta
_G['screenGui'] = screenGui
_G['podiumEnabled'] = podiumEnabled
_G['nextBaseEnabled'] = nextBaseEnabled
_G['platformEnabled'] = platformEnabled
_G['playerESPEnabled'] = playerESPEnabled
_G['carpetSpeedEnabled'] = carpetSpeedEnabled
_G['infiniteJumpEnabled'] = infiniteJumpEnabled
_G['fpsBoostEnabled'] = fpsBoostEnabled
_G['antiRagdollEnabled'] = antiRagdollEnabled
_G['antiDieEnabled'] = antiDieEnabled
_G['unwalkEnabled'] = unwalkEnabled
_G['carpetSpeedValue'] = carpetSpeedValue
_G['velocityConnection'] = velocityConnection
_G['equipCheckConnection'] = equipCheckConnection
_G['speedEnabled'] = speedEnabled
_G['speedBoosterEnabled'] = speedBoosterEnabled
_G['bases'] = bases
_G['connected'] = connected
_G['connections'] = connections
_G['currentTargetIndex'] = currentTargetIndex
_G['previousTargetIndex'] = previousTargetIndex
_G['targetInitialized'] = targetInitialized
_G['wrongBaseVisible'] = wrongBaseVisible
_G['lastPlayerBase'] = lastPlayerBase
_G['serverFull'] = serverFull
_G['slotESPActive'] = slotESPActive
_G['_podiumCleanup'] = _podiumCleanup
_G['espHolder'] = espHolder
_G['antiRagdollConnections'] = antiRagdollConnections
_G['lastVelocity'] = lastVelocity
_G['velocityChangeThreshold'] = velocityChangeThreshold
_G['velocityMagnitudeThreshold'] = velocityMagnitudeThreshold
_G['maxVelocity'] = maxVelocity
_G['fpsBoostActive'] = fpsBoostActive
_G['fpsBoostConnection'] = fpsBoostConnection
_G['unwalkConnection'] = unwalkConnection
_G['savedAnimateState'] = savedAnimateState
_G['stopAllAnimations'] = stopAllAnimations
_G['applyUnwalk'] = applyUnwalk
_G['hum'] = hum
_G['animator'] = animator
_G['animate'] = animate
_G['removeUnwalk'] = removeUnwalk
_G['setUnwalk'] = setUnwalk
_G['char'] = char
_G['baseBounds'] = baseBounds
_G['target'] = target
_G['floorOffsets'] = floorOffsets
_G['plotsFolder'] = plotsFolder
_G['pods'] = pods
_G['ys'] = ys
_G['cf'] = cf
_G['levels'] = levels
_G['found'] = found
_G['offsets'] = offsets
_G['newSlotESP'] = newSlotESP
_G['markers'] = markers
_G['espConnections'] = espConnections
_G['alive'] = alive
_G['clear'] = clear
_G['keep'] = keep
_G['makeBox'] = makeBox
_G['part'] = part
_G['selection'] = selection
_G['build'] = build
_G['slots'] = slots
_G['live'] = live
_G['minY'] = minY
_G['up'] = up
_G['exists'] = exists
_G['pending'] = pending
_G['rebuild'] = rebuild
_G['watch'] = watch
_G['buildPlatforms'] = buildPlatforms
_G['minX'] = minX
_G['maxX'] = maxX
_G['minZ'] = minZ
_G['maxZ'] = maxZ
_G['p'] = p
_G['size'] = size
_G['sizeX'] = sizeX
_G['sizeZ'] = sizeZ
_G['centerX'] = centerX
_G['centerZ'] = centerZ
_G['dy2'] = dy2
_G['dy3'] = dy3
_G['createPlatform'] = createPlatform
_G['anchor'] = anchor
_G['targetGui'] = targetGui
_G['billboard'] = billboard
_G['billboardCorner'] = billboardCorner
_G['billboardStroke'] = billboardStroke
_G['billboardGradient'] = billboardGradient
_G['inner'] = inner
_G['innerCorner'] = innerCorner
_G['accentLine'] = accentLine
_G['accentCorner'] = accentCorner
_G['topText'] = topText
_G['bottomText'] = bottomText
_G['billboardScale'] = billboardScale
_G['fullGui'] = fullGui
_G['fullLabel'] = fullLabel
_G['warningGui'] = warningGui
_G['warning'] = warning
_G['showWrongBase'] = showWrongBase
_G['hideWrongBase'] = hideWrongBase
_G['setNextBaseVisible'] = setNextBaseVisible
_G['nextBaseProjectConnection'] = nextBaseProjectConnection
_G['camera'] = camera
_G['point'] = point
_G['getPlayerBaseIndex'] = getPlayerBaseIndex
_G['root'] = root
_G['closestIndex'] = closestIndex
_G['dx'] = dx
_G['dz'] = dz
_G['distance'] = distance
_G['baseIndexFor'] = baseIndexFor
_G['position'] = position
_G['bestIndex'] = bestIndex
_G['isEmpty'] = isEmpty
_G['text'] = text
_G['recompute'] = recompute
_G['anyEmpty'] = anyEmpty
_G['base'] = base
_G['data'] = data
_G['playerBase'] = playerBase
_G['connectLabel'] = connectLabel
_G['scan'] = scan
_G['sign'] = sign
_G['model'] = model
_G['gui'] = gui
_G['frame'] = frame
_G['label'] = label
_G['index'] = index
_G['character'] = character
_G['setupCharacter'] = setupCharacter
_G['equipSelectedTool'] = equipSelectedTool
_G['toolName'] = toolName
_G['backpack'] = backpack
_G['tool'] = tool
_G['startEquipCheck'] = startEquipCheck
_G['stopEquipCheck'] = stopEquipCheck
_G['applyVelocity'] = applyVelocity
_G['dir'] = dir
_G['stopVelocity'] = stopVelocity
_G['carpetSpeedToggleRef'] = carpetSpeedToggleRef
_G['toggleSpeedState'] = toggleSpeedState
_G['state'] = state
_G['holding'] = holding
_G['hrp'] = hrp
_G['antiRagdollCleanup'] = antiRagdollCleanup
_G['enableAntiRagdollControls'] = enableAntiRagdollControls
_G['ps'] = ps
_G['PlayerModule'] = PlayerModule
_G['cleanupRagdoll'] = cleanupRagdoll
_G['carpetEquipped'] = carpetEquipped
_G['processChildren'] = processChildren
_G['animName'] = animName
_G['isRagdolled'] = isRagdolled
_G['isFlyingCarpetActive'] = isFlyingCarpetActive
_G['setupAntiRagdollCharacter'] = setupAntiRagdollCharacter
_G['setupAntiRagdollConnections'] = setupAntiRagdollConnections
_G['impulsePath'] = impulsePath
_G['velocity'] = velocity
_G['bindAntiRagdoll'] = bindAntiRagdoll
_G['humanoid'] = humanoid
_G['_conn'] = _conn
_G['_hardenRaw'] = _hardenRaw
_G['_harden'] = _harden
_G['_revive'] = _revive
_G['_bind'] = _bind
_G['_adAlive'] = _adAlive
_G['playerESPObjects'] = playerESPObjects
_G['playerESPConnections'] = playerESPConnections
_G['clearPlayerESP'] = clearPlayerESP
_G['addPlayerESP'] = addPlayerESP
_G['highlight'] = highlight
_G['conn'] = conn
_G['updatePlayerESP'] = updatePlayerESP
_G['togglePlayerESP'] = togglePlayerESP
_G['applyFPSBoost'] = applyFPSBoost
_G['terrain'] = terrain
_G['selectedPlayerCount'] = selectedPlayerCount
_G['findServer'] = findServer
_G['servers'] = servers
_G['cursor'] = cursor
_G['url'] = url
_G['playersInServer'] = playersInServer
_G['amount'] = amount
_G['instantResetBusy'] = instantResetBusy
_G['instantResetPosition'] = instantResetPosition
_G['doInstantReset'] = doInstantReset
_G['previousAntiDie'] = previousAntiDie
_G['lockedCamCFrame'] = lockedCamCFrame
_G['previousCameraType'] = previousCameraType
_G['renderName'] = renderName
_G['activeChar'] = activeChar
_G['activeRoot'] = activeRoot
_G['newChar'] = newChar
_G['newHum'] = newHum
_G['fpsValue'] = fpsValue
_G['fpsFrames'] = fpsFrames
_G['fpsWindowStart'] = fpsWindowStart
_G['fpsConnection'] = fpsConnection
_G['now'] = now
_G['getPingText'] = getPingText
_G['PODIUM_SHOW_DISTANCE'] = PODIUM_SHOW_DISTANCE
_G['updatePodiumVisibility'] = updatePodiumVisibility
_G['ping'] = ping
_G['network'] = network
_G['serverStats'] = serverStats
_G['dataPing'] = dataPing
local function buildUI()
local function persistFeature(name, value)
    savedSettings[name] = value
    saveSettings()
end

local function persistAllSettings()
    savedSettings.podiumEnabled = podiumEnabled
    savedSettings.nextBaseEnabled = nextBaseEnabled
    savedSettings.platformEnabled = platformEnabled
    savedSettings.playerESPEnabled = playerESPEnabled
    savedSettings.carpetSpeedEnabled = carpetSpeedEnabled
    savedSettings.infiniteJumpEnabled = infiniteJumpEnabled
    savedSettings.fpsBoostEnabled = fpsBoostEnabled
    savedSettings.unwalkEnabled = unwalkEnabled
    savedSettings.carpetSpeedValue = carpetSpeedValue
    savedSettings.autoGrabEnabled = autoGrabEnabled
    savedSettings.autoKickEnabled = autoKickEnabled
    savedSettings.baseESPEnabled = baseESPEnabled
    savedSettings.speedBoosterValue = speedBoosterValue
    savedSettings.speedBoosterEnabled = speedBoosterEnabled
    if WEBHOOK_URL ~= "" then savedSettings.webhookUrl = WEBHOOK_URL end
    savedSettings.selectedGearItem = selectedGearItem
    savedSettings.keybindSpeed = KEYBIND_SPEED.Name
    savedSettings.speedBoosterKeybind = KEYBIND_BOOSTER.Name
    savedSettings.keybindSpeedBooster = KEYBIND_BOOSTER.Name
    saveSettings()
end

-- LIGHT SETTINGS AUTOSAVE
task.spawn(function()
    while task.wait(10) do
        pcall(persistAllSettings)
    end
end)


-- ============================================================
local openGrabInsta

-- XKPUB TARGET FEATURES — AUTO KICK / WEBHOOK / BASE ESP
-- ============================================================
local baseESPObjects = {}
local baseESPConnections = {}

local function clearBaseESP()
    for _, obj in ipairs(baseESPObjects) do
        pcall(function() obj:Destroy() end)
    end
    table.clear(baseESPObjects)
    for _, c in ipairs(baseESPConnections) do
        pcall(function() c:Disconnect() end)
    end
    table.clear(baseESPConnections)
end

local function getPlotOwnerText(plot)
    local sign = plot and plot:FindFirstChild("PlotSign")
    if not sign then return "Base" end
    local gui = sign:FindFirstChildWhichIsA("SurfaceGui", true)
    local label = gui and gui:FindFirstChildWhichIsA("TextLabel", true)
    if label and label.Text and label.Text ~= "" then
        return label.Text:gsub("<[^>]+>", "")
    end
    return "Base"
end

local function addBaseESP(plot, index)
    if not baseESPEnabled or not plot then return end
    local base = plot:FindFirstChild("Base")
    local adornee = base
    if not adornee then
        local pods = plot:FindFirstChild("AnimalPodiums")
        adornee = pods and pods:FindFirstChildWhichIsA("BasePart", true)
    end
    if not adornee or not adornee:IsA("BasePart") then return end

    local hl = Instance.new("Highlight")
    hl.Name = "XKPubBaseESP"
    hl.Adornee = plot
    hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    hl.FillColor = Color3.fromRGB(120, 20, 20)
    hl.FillTransparency = 0.88
    hl.OutlineColor = Color3.fromRGB(255, 65, 65)
    hl.OutlineTransparency = 0.05
    hl.Parent = Workspace
    table.insert(baseESPObjects, hl)

    local bb = Instance.new("BillboardGui")
    bb.Name = "XKPubBaseESPLabel"
    bb.Adornee = adornee
    bb.AlwaysOnTop = true
    bb.LightInfluence = 0
    bb.MaxDistance = 2000
    bb.Size = UDim2.fromOffset(150, 30)
    bb.StudsOffset = Vector3.new(0, 7, 0)
    bb.Parent = adornee
    table.insert(baseESPObjects, bb)

    local label = Instance.new("TextLabel")
    label.Size = UDim2.fromScale(1, 1)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.GothamBlack
    label.TextSize = 11
    label.TextColor3 = Color3.fromRGB(255, 80, 80)
    label.TextStrokeColor3 = Color3.fromRGB(20, 0, 0)
    label.TextStrokeTransparency = 0
    label.Text = string.format("BASE %d  •  %s", tonumber(index) or 0, getPlotOwnerText(plot))
    label.Parent = bb
end

local function rebuildBaseESP()
    clearBaseESP()
    if not baseESPEnabled then return end
    for index, plot in ipairs(Plots:GetChildren()) do
        addBaseESP(plot, index)
    end
end

local function setBaseESP(state)
    baseESPEnabled = state == true
    persistFeature("baseESPEnabled", baseESPEnabled)
    rebuildBaseESP()
end

local KNOWN_OG_ITEMS = {
    ["strawberry elephant"] = true,
    ["headless horseman"] = true,
    ["meowl"] = true,
    ["john pork"] = true,
    ["skibidi toilet"] = true,
}

local KNOWN_SECRET_ITEMS = {
    ["signore carapace"] = true,
    ["elefanto frigo"] = true,
    ["love love bear"] = true,
    ["antonio"] = true,
    ["bunny and eggy"] = true,
    ["griffin"] = true,
    ["hydra dragon cannelloni"] = true,
    ["dragon gingerini"] = true,
}

local function normalizeStealItemName(item)
    item = tostring(item or "Unknown")
    item = item:gsub("<[^>]+>", "")
    item = item:gsub("[%*!]+$", "")
    item = item:gsub("^%s+", ""):gsub("%s+$", "")
    return item
end

local function detectMutation(rawText)
    local lower = tostring(rawText or ""):lower()
    local mutations = {
        "rainbow", "diamond", "gold", "candy", "lava", "bloodrot",
        "galaxy", "yinyang", "radioactive", "cursed", "divine", "celestial",
        "shiny", "frozen", "electric", "disco", "pizza"
    }
    for _, mutation in ipairs(mutations) do
        if lower:find(mutation, 1, true) then
            return mutation:upper()
        end
    end
    return "UNKNOWN"
end

local function detectStealRarity(rawText, item)
    local lowerText = tostring(rawText or ""):lower()
    local lowerItem = tostring(item or ""):lower():gsub("^%s+", ""):gsub("%s+$", "")

    if lowerText:find("%bog%s*[%]%)}:;,-]?", 1) or lowerText:find("%boriginal%", 1, true) then
        return "OG"
    end
    if lowerText:find("%bsecret%", 1) or lowerText:find("%bsecreto%", 1) then
        return "SECRET"
    end

    if KNOWN_OG_ITEMS[lowerItem] then
        return "OG"
    end
    if KNOWN_SECRET_ITEMS[lowerItem] then
        return "SECRET"
    end
    return "UNKNOWN"
end

local function findPlayerInStealText(text)
    local clean = tostring(text or "")
    local lower = clean:lower()

    for _, player in ipairs(Players:GetPlayers()) do
        local n = tostring(player.Name or "")
        local d = tostring(player.DisplayName or "")
        if n ~= "" and lower:find(n:lower(), 1, true) then
            return player
        end
        if d ~= "" and lower:find(d:lower(), 1, true) then
            return player
        end
    end

    return nil
end

local function parseStealMessage(text)
    local clean = tostring(text or ""):gsub("<[^>]+>", "")
    local lower = clean:lower()
    if not (lower:find("stole", 1, true) or lower:find("roubou", 1, true) or lower:find("stolen", 1, true)) then
        return nil, nil, clean
    end

    local thief = nil
    local item = nil

    -- Exact local-player wording used by the game.
    if lower:find("you stole", 1, true) then
        thief = LocalPlayer
        item = clean:match("[Yy]ou%s+[Ss]tole%s+(.+)!$") or clean:match("[Yy]ou%s+[Ss]tole%s+(.+)$")
    end

    -- Generic English / Portuguese message: PLAYER stole ITEM / PLAYER roubou ITEM.
    if not item then
        local a,b = clean:match("^(.+)%s+[Ss]tole%s+(.+)!$")
        if not a then a,b = clean:match("^(.+)%s+[Ss]tole%s+(.+)$") end
        if not a then a,b = clean:match("^(.+)%s+[Rr]oubou%s+(.+)!$") end
        if not a then a,b = clean:match("^(.+)%s+[Rr]oubou%s+(.+)$") end
        local matchedPlayer = a and findPlayerInStealText(a) or nil
        if matchedPlayer and b then
            thief = matchedPlayer
            item = b
        end
    end

    -- Do not treat arbitrary UI text containing the word "stole" as a steal event.
    if not thief or not item then
        return nil, nil, clean
    end

    item = normalizeStealItemName(item)
    if item == "" or item:lower() == "unknown item" then
        return nil, nil, clean
    end

    return thief, item, clean
end

local function sendStealWebhook(thief, stolenItem, rawText)
    if WEBHOOK_URL == "" or not request then return end

    stolenItem = normalizeStealItemName(stolenItem)
    local rarity = detectStealRarity(rawText, stolenItem)
    local mutation = detectMutation(rawText)
    local thiefName = thief and (thief.DisplayName or thief.Name) or "Unknown"
    local thiefUser = thief and ("@" .. tostring(thief.Name)) or "Unknown"

    local rarityBadge = rarity == "OG" and "🟣 OG" or (rarity == "SECRET" and "🔮 SECRET" or "❔ UNKNOWN")
    local safeDescription = string.format("**%s** roubou **%s**", tostring(thiefName), tostring(stolenItem))

    local payload = {
        username = "Neptune Finder",
        embeds = {{
            title = "💗 STEAL DETECTED",
            description = safeDescription,
            color = 16724991,
            fields = {
                {name = "👤 Quem roubou", value = thiefUser, inline = true},
                {name = "🧠 Brainrot", value = tostring(stolenItem), inline = true},
                {name = "💎 Raridade", value = rarityBadge, inline = true},
                {name = "🧬 Mutação", value = mutation, inline = true},
                {name = "🆔 Player ID", value = thief and tostring(thief.UserId) or "Unknown", inline = true},
                {name = "🎮 Job ID", value = tostring(game.JobId), inline = true},
                {name = "📍 Place ID", value = tostring(game.PlaceId), inline = true},
            },
            author = {name = "Neptune Finder • Auto Grab"},
            footer = {text = "Steal monitor • " .. os.date("%d/%m/%Y %H:%M:%S")},
            timestamp = os.date("!%Y-%m-%dT%H:%M:%SZ"),
        }}
    }

    if thief and thief.UserId then
        payload.embeds[1].thumbnail = {
            url = string.format("https://www.roblox.com/headshot-thumbnail/image?userId=%d&width=420&height=420&format=png", thief.UserId)
        }
    end

    task.spawn(function()
        pcall(function()
            request({
                Url = WEBHOOK_URL,
                Method = "POST",
                Headers = { ["Content-Type"] = "application/json" },
                Body = HttpService:JSONEncode(payload),
            })
        end)
    end)
end

local stealMessageDebounce = 0
local function handleStealText(text)
    local thief, item, clean = parseStealMessage(text)
    if not thief or not item then return end
    if os.clock() - stealMessageDebounce < 0.75 then return end
    stealMessageDebounce = os.clock()

    sendStealWebhook(thief, item, clean)

end

local function hookStealText(obj)
    if not obj or not (obj:IsA("TextLabel") or obj:IsA("TextButton") or obj:IsA("TextBox")) then return end
    pcall(function() handleStealText(obj.Text) end)
    obj:GetPropertyChangedSignal("Text"):Connect(function()
        pcall(function() handleStealText(obj.Text) end)
    end)
end

task.spawn(function()
    local pg = LocalPlayer:WaitForChild("PlayerGui")
    for _, obj in ipairs(pg:GetDescendants()) do hookStealText(obj) end
    pg.DescendantAdded:Connect(hookStealText)
end)

local function setAutoGrab(state)
    autoGrabEnabled = state == true
    persistFeature("autoGrabEnabled", autoGrabEnabled)
    pcall(function() openGrabInsta(autoGrabEnabled) end)
end

local speedBoosterConnection = nil

-- ============================================================
-- YONNI SPEED BOOSTER — SAME CFRAME + VELOCITY MUTING LOGIC
-- ============================================================
local function stopSpeedBooster()
    if speedBoosterConnection then
        pcall(function() speedBoosterConnection:Disconnect() end)
        speedBoosterConnection = nil
    end
end

local function setSpeedBooster(state)
    speedBoosterEnabled = state == true
    -- Anti Die is now tied directly to Speed Booster.
    antiDieEnabled = speedBoosterEnabled
    _G.AntiDieDisabled = not antiDieEnabled
    persistFeature("antiDieEnabled", antiDieEnabled)
    stopSpeedBooster()

    if not speedBoosterEnabled then
        local char = LocalPlayer.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum then
            hum.WalkSpeed = 16
        end
        if char and unwalkEnabled then
            task.defer(function()
                if char.Parent and not speedBoosterEnabled then
                    pcall(applyUnwalk, char)
                end
            end)
        end
        return
    end

    local char = LocalPlayer.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    local rootPart = char and char:FindFirstChild("HumanoidRootPart")
    if not hum or not rootPart then return end

    -- Preserve the requested Zombie animation while Speed Booster is active.
    -- Do NOT re-apply the whole avatar here; that can reset the current animation.
    pcall(removeUnwalk, char)
    task.defer(function()
        if char.Parent and speedBoosterEnabled then
            local animate = char:FindFirstChild("Animate")
            if animate then
                animate.Disabled = false
            end
            pcall(applyZombieAnimations, char)

            local humNow = char:FindFirstChildOfClass("Humanoid")
            local animator = humNow and humNow:FindFirstChildOfClass("Animator")
            if animator then
                for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
                    pcall(function() track:Stop(0) end)
                end
            end
            pcall(applyZombieAnimations, char)
        end
    end)

    speedBoosterConnection = RunService.RenderStepped:Connect(function(dt)
        if not speedBoosterEnabled then return end
        if not char.Parent or not hum.Parent or not rootPart.Parent or hum.Health <= 0 then return end

        local stateNow = hum:GetState()
        if hum.PlatformStand
            or stateNow == Enum.HumanoidStateType.Physics
            or stateNow == Enum.HumanoidStateType.Ragdoll
            or stateNow == Enum.HumanoidStateType.FallingDown then
            return
        end

        local md = hum.MoveDirection
        if md.Magnitude > 0 then
            -- Same movement method used by the Yonni source.
            rootPart.AssemblyLinearVelocity = Vector3.new(0, rootPart.AssemblyLinearVelocity.Y, 0)
            rootPart.CFrame = rootPart.CFrame + (md * math.clamp(tonumber(speedBoosterValue) or 27, 1, 200) * dt)
        end
    end)
end

local function setSpeedBoosterEnabled(state)
    speedBoosterEnabled = state == true
    setSpeedBooster(speedBoosterEnabled)
    if speedBoosterEnabled and LocalPlayer.Character then
        task.defer(function()
            if LocalPlayer.Character and speedBoosterEnabled then
                pcall(removeUnwalk, LocalPlayer.Character)
                pcall(applyZombieAnimations, LocalPlayer.Character)
            end
        end)
    end
    savedSettings.speedBoosterEnabled = speedBoosterEnabled
    saveSettings()
    _G.speedBoosterEnabled = speedBoosterEnabled
    return speedBoosterEnabled
end

local function toggleSpeedBoosterKey()
    return setSpeedBoosterEnabled(not speedBoosterEnabled)
end

_G['XKPubSetSpeedBoosterEnabled'] = setSpeedBoosterEnabled
_G['XKPubToggleSpeedBoosterKey'] = toggleSpeedBoosterKey

local function bindSpeedBooster(char)
    stopSpeedBooster()
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid") or char:WaitForChild("Humanoid", 5)
    if not hum then return end
    if speedBoosterEnabled then
        task.defer(function()
            if char.Parent and speedBoosterEnabled then
                setSpeedBooster(true)
            end
        end)
    end
end

if LocalPlayer.Character then
    task.spawn(function() bindSpeedBooster(LocalPlayer.Character) end)
end
LocalPlayer.CharacterAdded:Connect(function(char)
    task.wait(0.25)
    pcall(applyRequestedAvatarVisuals, char)
    bindSpeedBooster(char)
    if speedBoosterEnabled then
        pcall(removeUnwalk, char)
        task.defer(function()
            if char.Parent and speedBoosterEnabled then
                pcall(applyZombieAnimations, char)
            end
        end)
    end
end)

-- ============================================================
-- YONNI DROP BRAINROT — SAME WALK/VELOCITY FLING LOGIC
-- ============================================================
local _wfConns = {}
local _wfActive = false

local function stopWalkFling()
    _wfActive = false
    for _, c in ipairs(_wfConns) do
        if typeof(c) == "RBXScriptConnection" then
            pcall(function() c:Disconnect() end)
        end
    end
    table.clear(_wfConns)
end

local function startWalkFling()
    _wfActive = true
    local rr = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not rr then return end

    table.insert(_wfConns, RunService.Stepped:Connect(function()
        if not _wfActive then return end
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer and plr.Character then
                for _, pt in ipairs(plr.Character:GetChildren()) do
                    if pt:IsA("BasePart") then
                        pt.CanCollide = false
                    end
                end
            end
        end
    end))

    local co = coroutine.create(function()
        while _wfActive do
            RunService.Heartbeat:Wait()
            if not rr or not rr.Parent then break end
            local v = rr.Velocity
            rr.Velocity = v * 10000 + Vector3.new(0, 10000, 0)
            RunService.RenderStepped:Wait()
            if rr then rr.Velocity = v end
            RunService.Stepped:Wait()
            if rr then rr.Velocity = v + Vector3.new(0, 0.1, 0) end
        end
    end)
    coroutine.resume(co)
    table.insert(_wfConns, co)
end

local function dropBrainrot()
    if not _wfActive then
        startWalkFling()
        task.delay(0.4, stopWalkFling)
    end
end

_G.XKPubDropBrainrot = dropBrainrot


if baseESPEnabled then task.defer(rebuildBaseESP) end
if Plots then
    table.insert(baseESPConnections, Plots.ChildAdded:Connect(function(plot)
        task.wait(0.2)
        if baseESPEnabled then rebuildBaseESP() end
    end))
    table.insert(baseESPConnections, Plots.ChildRemoved:Connect(function()
        if baseESPEnabled then rebuildBaseESP() end
    end))
end

-- ============================================================
-- ============================================================
-- INTERFACE REWORK — GEAR & SAFETY + TARGET CONTROLS
-- ============================================================

UI_PURPLE = Color3.fromRGB(255, 214, 45)
UI_PURPLE_DARK = Color3.fromRGB(120, 90, 8)
UI_PURPLE_BORDER = Color3.fromRGB(255, 236, 100)
UI_GREEN = Color3.fromRGB(255, 214, 45)
UI_GREEN_BORDER = Color3.fromRGB(255, 236, 100)
UI_PANEL = Color3.fromRGB(35, 28, 5)
UI_ROW = Color3.fromRGB(70, 55, 10)
UI_WHITE = Color3.fromRGB(255, 252, 220)
UI_MUTED = Color3.fromRGB(220, 200, 125)
UI_OFF = Color3.fromRGB(75, 58, 8)

screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true

-- All UI instances are stored in one table so the main chunk does not
-- accumulate hundreds of local registers (avoids "exceeded limit 200").
_G.U = {}
U = _G.U

local function stylePanel2(frame, radius)
    frame.BackgroundColor3 = UI_PANEL
    frame.BorderSizePixel = 0
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0, radius or 15)
    c.Parent = frame
    local s = Instance.new("UIStroke")
    s.Color = UI_PURPLE_BORDER
    s.Thickness = 1.35
    s.Transparency = 0.14
    s.Parent = frame
    return s
end

local SPONGE_TEXTURE_ID = "rbxassetid://10729455634"

local function addSpongeTexture(panel, transparency, zIndex)
    if not panel or panel:FindFirstChild("SpongeBobTexture") then return end
    local image = Instance.new("ImageLabel")
    image.Name = "SpongeBobTexture"
    image.Size = UDim2.new(1, 0, 1, 0)
    image.Position = UDim2.fromOffset(0, 0)
    image.BackgroundTransparency = 1
    image.Image = SPONGE_TEXTURE_ID
    image.ImageTransparency = transparency or 0.82
    image.ScaleType = Enum.ScaleType.Crop
    image.ZIndex = zIndex or panel.ZIndex
    image.ClipsDescendants = true
    image.Parent = panel
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 15)
    corner.Parent = image
    return image
end

local function pill(btn, state, purple, textOn, textOff)
    local s = btn:FindFirstChildOfClass("UIStroke")
    if state then
        btn.BackgroundColor3 = UI_PURPLE_DARK
        btn.TextColor3 = UI_WHITE
        btn.Text = textOn or "ON"
        if s then s.Color = UI_PURPLE_BORDER end
    else
        btn.BackgroundColor3 = UI_OFF
        btn.TextColor3 = UI_MUTED
        btn.Text = textOff or "OFF"
        if s then s.Color = Color3.fromRGB(53,54,65) end
    end
end

local function rowButton(parent, name, order, state, callback, purple, fixedText, height)
    local row = Instance.new("Frame")
    row.Size = UDim2.new(1,0,0,height or 27)
    row.LayoutOrder = order
    row.BackgroundColor3 = UI_ROW
    row.BorderSizePixel = 0
    row.ZIndex = 110
    row.Parent = parent

    local rc = Instance.new("UICorner")
    rc.CornerRadius = UDim.new(0,9)
    rc.Parent = row

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1,-82,1,0)
    label.Position = UDim2.fromOffset(9,0)
    label.BackgroundTransparency = 1
    label.Font = FONT_REGULAR
    label.Text = name
    label.TextColor3 = UI_WHITE
    label.TextSize = 8.5
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.ZIndex = 111
    label.Parent = row

    local btn = Instance.new("TextButton")
    btn.Size = UDim2.fromOffset(70,20)
    btn.Position = UDim2.new(1,-76,0.5,-10)
    btn.BackgroundColor3 = UI_OFF
    btn.AutoButtonColor = false
    btn.Font = FONT_MAIN
    btn.TextSize = 7.5
    btn.ZIndex = 112
    btn.Parent = row

    local bc = Instance.new("UICorner")
    bc.CornerRadius = UDim.new(0,8)
    bc.Parent = btn
    local bs = Instance.new("UIStroke")
    bs.Thickness = 1
    bs.Parent = btn

    local current = state
    local function refresh()
        if fixedText then
            btn.Text = fixedText
            btn.BackgroundColor3 = UI_PURPLE_DARK
            btn.TextColor3 = UI_WHITE
            bs.Color = UI_PURPLE_BORDER
        else
            pill(btn,current,purple)
        end
    end
    refresh()

    btn.MouseButton1Click:Connect(function()
        if fixedText then
            if callback then callback() end
            return
        end
        current = not current
        refresh()
        if callback then callback(current) end
    end)

    return row, btn, function(v)
        current = v
        refresh()
    end
end

-- ------------------------------------------------------------
-- GEAR & SAFETY
-- ------------------------------------------------------------
U.gear = Instance.new("Frame")
U.gear.Name = "GearSafetyWindow"
U.gear.Size = UDim2.fromOffset(190,300)
U.gear.ZIndex = 100
U.gear.Parent = screenGui

local gearTexture = Instance.new("ImageLabel")
gearTexture.Name = "GearTexture"
gearTexture.Size = UDim2.new(1, -6, 1, -6)
gearTexture.Position = UDim2.fromOffset(3, 3)
gearTexture.BackgroundTransparency = 1
gearTexture.Image = "rbxassetid://10729455634"
gearTexture.ImageTransparency = 0.72
gearTexture.ScaleType = Enum.ScaleType.Crop
gearTexture.ZIndex = 0
gearTexture.Parent = U.gear

stylePanel2(U.gear,15)

local gearTexture = Instance.new("ImageLabel")
gearTexture.Name = "GearSafetyTexture"
gearTexture.Size = UDim2.new(1,0,1,0)
gearTexture.Position = UDim2.fromOffset(0,0)
gearTexture.BackgroundTransparency = 1
gearTexture.Image = "rbxassetid://10729455634"
gearTexture.ImageTransparency = 0.76
gearTexture.ScaleType = Enum.ScaleType.Crop
gearTexture.ZIndex = 100
gearTexture.Parent = U.gear
local gearTextureCorner = Instance.new("UICorner")
gearTextureCorner.CornerRadius = UDim.new(0,15)
gearTextureCorner.Parent = gearTexture

U.gearTitle = Instance.new("TextLabel")
U.gearTitle.Size = UDim2.new(1,0,0,28)
U.gearTitle.BackgroundTransparency = 1
U.gearTitle.Font = FONT_MAIN
U.gearTitle.Text = "GEAR & SAFETY"
U.gearTitle.TextColor3 = UI_WHITE
U.gearTitle.TextSize = 12
U.gearTitle.TextStrokeColor3 = Color3.fromRGB(115, 95, 25)
U.gearTitle.TextStrokeTransparency = 0.25
U.gearTitle.ZIndex = 101
U.gearTitle.Parent = U.gear

U.gearAccent = Instance.new("Frame")
U.gearAccent.Size = UDim2.fromOffset(6,6)
U.gearAccent.Position = UDim2.fromOffset(11,11)
U.gearAccent.BackgroundColor3 = UI_PURPLE_BORDER
U.gearAccent.BorderSizePixel = 0
U.gearAccent.ZIndex = 102
U.gearAccent.Parent = U.gear
local gac = Instance.new("UICorner")
gac.CornerRadius = UDim.new(1,0)
gac.Parent = U.gearAccent

U.gearDivider = Instance.new("Frame")
U.gearDivider.Size = UDim2.new(1,-24,0,1)
U.gearDivider.Position = UDim2.fromOffset(12,29)
U.gearDivider.BackgroundColor3 = Color3.fromRGB(115, 95, 25)
U.gearDivider.BackgroundTransparency = 0.45
U.gearDivider.BorderSizePixel = 0
U.gearDivider.ZIndex = 101
U.gearDivider.Parent = U.gear

U.gearBody = Instance.new("ScrollingFrame")
U.gearBody.Size = UDim2.new(1,-12,1,-36)
U.gearBody.Position = UDim2.fromOffset(6,31)
U.gearBody.BackgroundTransparency = 1
U.gearBody.BorderSizePixel = 0
U.gearBody.ScrollBarThickness = 2
U.gearBody.ScrollBarImageColor3 = UI_PURPLE
U.gearBody.CanvasSize = UDim2.fromOffset(0,0)
U.gearBody.ZIndex = 101
U.gearBody.Parent = U.gear

U.gearLayout = Instance.new("UIListLayout")
U.gearLayout.SortOrder = Enum.SortOrder.LayoutOrder
U.gearLayout.Padding = UDim.new(0,4)
U.gearLayout.Parent = U.gearBody
U.gearLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
    U.gearBody.CanvasSize = UDim2.fromOffset(0,U.gearLayout.AbsoluteContentSize.Y + 6)
end)

makeDraggableAndPersistent(U.gear,U.gearTitle,"GearSafetyWindow",UDim2.new(0.70,-100,0.08,0))

_, U.keybindBtn = rowButton(U.gearBody,"Keybinds",1,false,function()
    U.keybind.Visible = not U.keybind.Visible
    U.gearSelect.Visible = false
end,true,"KEYS")

_, U.flyBtn = rowButton(U.gearBody,"Fly Gear",3,true,function()
    rebuildGearChoices()
    U.gearSelect.Visible = true
    U.keybind.Visible = false
end,false,(selectedGearItem or "GEAR"):upper())

_, U.newFeatureBtn = rowButton(U.gearBody,"New Feature",5,false,showNotReadyToast,true,"SOON")
U.newFeatureBtn.AutoButtonColor = false
U.newFeatureBtn.Text = "SOON"
U.newFeatureBtn.MouseEnter:Connect(function()
    U.newFeatureBtn.BackgroundColor3 = Color3.fromRGB(125, 100, 15)
end)
U.newFeatureBtn.MouseLeave:Connect(function()
    U.newFeatureBtn.BackgroundColor3 = UI_PURPLE_DARK
end)


_, U.platformBtn = rowButton(U.gearBody,"Platform",6,true,function(s)
    platformEnabled = s
    persistFeature("platformEnabled", s)
    if s then
        pcall(buildPlatforms)
    else
        for _,child in ipairs(espHolder:GetChildren()) do
            if child:IsA("Part") and (child.Name == "WalkablePlatform2nd" or child.Name == "WalkablePlatform3rd") then
                child:Destroy()
            end
        end
    end
end,true)

_, U.jumpBtn = rowButton(U.gearBody,"Infinite Jump",7,true,function(s)
    infiniteJumpEnabled = s
end,false)

_, U.fpsBtn = rowButton(U.gearBody,"FPS Boost",8,false,function(s)
    fpsBoostEnabled = s
    persistFeature("fpsBoostEnabled", s)
    fpsBoostActive = s
    applyFPSBoost()
end,false)

_, U.unwalkBtn = rowButton(U.gearBody,"Unwalk",9,true,function(s)
    setUnwalk(s)
end,false)

-- Carpet Speed value (inside Gear & Safety)
U.speedRow = Instance.new("Frame")
U.speedRow.Size = UDim2.new(1,0,0,28)
U.speedRow.LayoutOrder = 9
U.speedRow.BackgroundColor3 = UI_ROW
U.speedRow.ZIndex = 110
U.speedRow.Parent = U.gearBody

local src = Instance.new("UICorner")
src.CornerRadius = UDim.new(0,8)
src.Parent = U.speedRow

local srl = Instance.new("TextLabel")
srl.Size = UDim2.new(1,-90,1,0)
srl.Position = UDim2.fromOffset(9,0)
srl.BackgroundTransparency = 1
srl.Font = FONT_REGULAR
srl.Text = "Carpet Speed"
srl.TextColor3 = UI_WHITE
srl.TextSize = 8
srl.TextXAlignment = Enum.TextXAlignment.Left
srl.ZIndex = 111
srl.Parent = U.speedRow

local srb = Instance.new("TextLabel")
srb.Size = UDim2.fromOffset(72,20)
srb.Position = UDim2.new(1,-78,0.5,-10)
srb.BackgroundColor3 = UI_OFF
srb.Font = FONT_MAIN
srb.Text = tostring(carpetSpeedValue)
srb.TextColor3 = Color3.fromRGB(235, 220, 150)
srb.TextSize = 9
srb.ZIndex = 111
srb.Parent = U.speedRow
local srbc = Instance.new("UICorner")
srbc.CornerRadius = UDim.new(0,7)
srbc.Parent = srb
carpetSpeedToggleRef = U.speedBtn

local speedMinus = Instance.new("TextButton")
speedMinus.Size = UDim2.fromOffset(20,20)
speedMinus.BackgroundTransparency = 1
speedMinus.Font = FONT_MAIN
speedMinus.Text = "-"
speedMinus.TextColor3 = UI_WHITE
speedMinus.TextSize = 10
speedMinus.ZIndex = 112
speedMinus.Parent = srb

local speedPlus = Instance.new("TextButton")
speedPlus.Size = UDim2.fromOffset(20,20)
speedPlus.Position = UDim2.new(1,-20,0,0)
speedPlus.BackgroundTransparency = 1
speedPlus.Font = FONT_MAIN
speedPlus.Text = "+"
speedPlus.TextColor3 = UI_WHITE
speedPlus.TextSize = 10
speedPlus.ZIndex = 112
speedPlus.Parent = srb

speedMinus.MouseButton1Click:Connect(function()
    carpetSpeedValue = math.max(0,carpetSpeedValue - 5)
    CARPET_SPEED = carpetSpeedValue
    persistFeature("carpetSpeedValue", carpetSpeedValue)
    srb.Text = tostring(carpetSpeedValue)
end)
speedPlus.MouseButton1Click:Connect(function()
    carpetSpeedValue = carpetSpeedValue + 5
    CARPET_SPEED = carpetSpeedValue
    persistFeature("carpetSpeedValue", carpetSpeedValue)
    srb.Text = tostring(carpetSpeedValue)
end)

-- ------------------------------------------------------------
-- TARGET CONTROLS
-- Auto Kick and Base ESP controls intentionally removed.
-- Layout orders are contiguous so UIListLayout leaves no empty slot.
-- ------------------------------------------------------------
U.target = Instance.new("Frame")
U.target.Name = "TargetControlsWindow"
U.target.Size = UDim2.fromOffset(190,190)
U.target.ZIndex = 100
U.target.Parent = screenGui

local targetTexture = Instance.new("ImageLabel")
targetTexture.Name = "TargetTexture"
targetTexture.Size = UDim2.new(1, -6, 1, -6)
targetTexture.Position = UDim2.fromOffset(3, 3)
targetTexture.BackgroundTransparency = 1
targetTexture.Image = "rbxassetid://10729455634"
targetTexture.ImageTransparency = 0.72
targetTexture.ScaleType = Enum.ScaleType.Crop
targetTexture.ZIndex = 0
targetTexture.Parent = U.target

stylePanel2(U.target,15)

local targetTexture = Instance.new("ImageLabel")
targetTexture.Name = "TargetControlsTexture"
targetTexture.Size = UDim2.new(1,0,1,0)
targetTexture.Position = UDim2.fromOffset(0,0)
targetTexture.BackgroundTransparency = 1
targetTexture.Image = "rbxassetid://10729455634"
targetTexture.ImageTransparency = 0.76
targetTexture.ScaleType = Enum.ScaleType.Crop
targetTexture.ZIndex = 100
targetTexture.Parent = U.target
local targetTextureCorner = Instance.new("UICorner")
targetTextureCorner.CornerRadius = UDim.new(0,15)
targetTextureCorner.Parent = targetTexture

U.targetTitle = Instance.new("TextLabel")
U.targetTitle.Size = UDim2.new(1,0,0,29)
U.targetTitle.BackgroundTransparency = 1
U.targetTitle.Font = FONT_MAIN
U.targetTitle.Text = "TARGET CONTROLS"
U.targetTitle.TextColor3 = UI_WHITE
U.targetTitle.TextSize = 11
U.targetTitle.ZIndex = 101
U.targetTitle.Parent = U.target
makeDraggableAndPersistent(U.target,U.targetTitle,"TargetControlsWindow",UDim2.new(0.70,-100,0.48,0))

U.targetBody = Instance.new("Frame")
U.targetBody.Size = UDim2.new(1,-14,1,-37)
U.targetBody.Position = UDim2.fromOffset(6,30)
U.targetBody.BackgroundTransparency = 1
U.targetBody.ZIndex = 101
U.targetBody.Parent = U.target

U.targetLayout = Instance.new("UIListLayout")
U.targetLayout.SortOrder = Enum.SortOrder.LayoutOrder
U.targetLayout.Padding = UDim.new(0,4)
U.targetLayout.Parent = U.targetBody

local embeddedGrabInstaSource = [[
-- ============================================================
-- INSTANT GRAB | DARK CONTROL EDITION
-- ============================================================
-- • AUTO EXEC DELAY
-- • DARK BACKGROUND (like PRIVATE CONTROL)
-- • ICON IMAGE: 7117673507 (FULL CAPTURE)
-- • FIXED CENTER LINE: 50%
-- • NO TARGET / STUDS
-- • INSTANT BY REMOTE
-- • ACTIVATION SOUND: 98797174600699
-- • DRAG HEADER
-- ============================================================

-- ============================================================
-- AUTO EXEC
-- ============================================================

if not game:IsLoaded() then
    game.Loaded:Wait()
end


-- ============================================================
-- SERVICES
-- ============================================================

local CoreGui = game:GetService("CoreGui")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")

local LP = Players.LocalPlayer

if not LP then
    return
end

-- ============================================================
-- PARENT
-- ============================================================

local parentGui = LP:FindFirstChildOfClass("PlayerGui")

pcall(function()
    if CoreGui then
        parentGui = CoreGui
    end
end)

if not parentGui then
    parentGui = LP:WaitForChild("PlayerGui")
end

-- ============================================================
-- REMOVE OLD GUI
-- ============================================================

pcall(function()
    local old = parentGui:FindFirstChild("InstantGrabSpecificGui")

    if old then
        old:Destroy()
    end
end)

-- ============================================================
-- COLORS (DARK CONTROL THEME)
-- ============================================================

local COLORS = {

    Background =
        Color3.fromRGB(
            10,
            10,
            12
        ),   -- fundo principal preto

    Panel =
        Color3.fromRGB(
            20,
            20,
            22
        ),   -- painéis escuros

    Panel2 =
        Color3.fromRGB(
            30,
            30,
            34
        ),   -- painéis secundários

    White =
        Color3.fromRGB(
            255,
            255,
            255
        ),

    White2 =
        Color3.fromRGB(
            240,
            240,
            245
        ),

    Gray =
        Color3.fromRGB(
            150,
            150,
            160
        ),

    Gray2 =
        Color3.fromRGB(
            100,
            100,
            110
        ),

    DarkGray =
        Color3.fromRGB(
            60,
            60,
            68
        ),

    Black =
        Color3.fromRGB(
            0,
            0,
            0
        ),

    LightGray =
        Color3.fromRGB(
            80,
            80,
            88
        )
}

-- DARK GRAB INSTA THEME
COLORS.Background = Color3.fromRGB(24, 5, 18)
COLORS.Panel = Color3.fromRGB(35, 8, 26)
COLORS.Panel2 = Color3.fromRGB(58, 12, 40)
COLORS.Gray = Color3.fromRGB(178, 190, 205)
COLORS.Gray2 = Color3.fromRGB(135, 143, 154)
COLORS.DarkGray = Color3.fromRGB(95, 20, 66)
COLORS.LightGray = Color3.fromRGB(125, 30, 88)
local PURPLE = Color3.fromRGB(255, 214, 45)
local PURPLE2 = Color3.fromRGB(120, 90, 8)


-- ============================================================
-- ============================================================
-- SCREEN GUI — COMPACT PINK AUTO GRAB UI ONLY
-- ============================================================

local Stats = game:GetService("Stats")

local screenGui = Instance.new("ScreenGui")
screenGui.Name = "InstantGrabSpecificGui"
screenGui.ResetOnSpawn = false
screenGui.IgnoreGuiInset = true
screenGui.DisplayOrder = 999
screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
screenGui.Parent = parentGui

local mainFrame = Instance.new("Frame")
mainFrame.Name = "MainFrame"
mainFrame.Size = UDim2.fromOffset(276, 68)

local function loadAutoGrabPosition(defaultPosition)
    local positions = _G.savedPositions
    local saved = positions and positions["AutoGrabWindow"]
    if type(saved) == "table" then
        local xs = tonumber(saved.XScale)
        local xo = tonumber(saved.XOffset)
        local ys = tonumber(saved.YScale)
        local yo = tonumber(saved.YOffset)
        if xs and xo and ys and yo then
            return UDim2.new(xs, xo, ys, yo)
        end
    end
    return defaultPosition
end

local function saveAutoGrabPosition()
    local positions = _G.savedPositions
    local saveFn = _G.savePositions
    if type(positions) ~= "table" then return end
    positions["AutoGrabWindow"] = {
        XScale = mainFrame.Position.X.Scale,
        XOffset = mainFrame.Position.X.Offset,
        YScale = mainFrame.Position.Y.Scale,
        YOffset = mainFrame.Position.Y.Offset,
    }
    if type(saveFn) == "function" then
        pcall(saveFn)
    elseif writefile then
        local ok = pcall(function()
            writefile(_G.CONFIG_FILE or "XKPubUIPositions.json", HttpService:JSONEncode(positions))
        end)
        if not ok then end
    end
end

_G.__SaveNeptuneAutoGrabPosition = saveAutoGrabPosition

mainFrame.Position = loadAutoGrabPosition(UDim2.new(1, -292, 0, 72))
mainFrame.BackgroundColor3 = Color3.fromRGB(8, 8, 8)
mainFrame.BackgroundTransparency = 0
mainFrame.BorderSizePixel = 0
mainFrame.Active = true
mainFrame.ZIndex = 10
mainFrame.Parent = screenGui

local mainCorner = Instance.new("UICorner")
mainCorner.CornerRadius = UDim.new(0, 13)
mainCorner.Parent = mainFrame

local shadow = Instance.new("ImageLabel")
shadow.Name = "Shadow"
shadow.Size = UDim2.new(1, 18, 1, 18)
shadow.Position = UDim2.fromOffset(-9, -4)
shadow.BackgroundTransparency = 1
shadow.Image = "rbxassetid://1316045217"
shadow.ImageColor3 = Color3.fromRGB(0, 0, 0)
shadow.ImageTransparency = 0.42
shadow.ScaleType = Enum.ScaleType.Slice
shadow.SliceCenter = Rect.new(10, 10, 118, 118)
shadow.ZIndex = 0
shadow.Parent = mainFrame

local mainGradient = Instance.new("UIGradient")
mainGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 214, 45)),
    ColorSequenceKeypoint.new(0.5, Color3.fromRGB(0, 0, 0)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 0, 0)),
})
mainGradient.Rotation = 0
mainGradient.Parent = mainFrame

-- ============================================================
-- SPONGEBOB VISUAL FX (UI ONLY)
-- Visual-only polish. Auto Grab logic below is untouched.
-- ============================================================
local accentLine = Instance.new("Frame")
accentLine.Name = "SpongeAccentLine"
accentLine.Size = UDim2.new(1, -20, 0, 3)
accentLine.Position = UDim2.fromOffset(10, 31)
accentLine.BackgroundColor3 = Color3.fromRGB(255, 214, 45)
accentLine.BorderSizePixel = 0
accentLine.ZIndex = 15
accentLine.Parent = mainFrame
local accentCorner = Instance.new("UICorner")
accentCorner.CornerRadius = UDim.new(1, 0)
accentCorner.Parent = accentLine

local blueAccent = Instance.new("Frame")
blueAccent.Name = "BlueAccent"
blueAccent.Size = UDim2.fromOffset(54, 3)
blueAccent.Position = UDim2.fromOffset(10, 31)
blueAccent.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
blueAccent.BorderSizePixel = 0
blueAccent.ZIndex = 16
blueAccent.Parent = mainFrame
local blueCorner = Instance.new("UICorner")
blueCorner.CornerRadius = UDim.new(1, 0)
blueCorner.Parent = blueAccent

local progressGlow = Instance.new("Frame")
progressGlow.Name = "ProgressGlow"
progressGlow.Size = UDim2.fromOffset(196, 7)
progressGlow.Position = UDim2.fromOffset(67, 32)
progressGlow.BackgroundColor3 = Color3.fromRGB(255, 214, 45)
progressGlow.BackgroundTransparency = 0.88
progressGlow.BorderSizePixel = 0
progressGlow.ZIndex = 19
progressGlow.Parent = mainFrame
local progressGlowCorner = Instance.new("UICorner")
progressGlowCorner.CornerRadius = UDim.new(1, 0)
progressGlowCorner.Parent = progressGlow

local marker50 = Instance.new("Frame")
marker50.Name = "Marker50"
marker50.Size = UDim2.fromOffset(2, 7)
marker50.Position = UDim2.fromOffset(165, 32)
marker50.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
marker50.BackgroundTransparency = 0
marker50.BorderSizePixel = 0
marker50.ZIndex = 26
marker50.Parent = mainFrame
local markerCorner = Instance.new("UICorner")
markerCorner.CornerRadius = UDim.new(1, 0)
markerCorner.Parent = marker50

local markerText = Instance.new("TextLabel")
markerText.Name = "Marker50Text"
markerText.Size = UDim2.fromOffset(24, 10)
markerText.Position = UDim2.fromOffset(154, 22)
markerText.BackgroundTransparency = 1
markerText.Text = "50%"
markerText.TextColor3 = Color3.fromRGB(255, 214, 45)
markerText.TextTransparency = 0
markerText.Font = Enum.Font.GothamBold
markerText.TextSize = 7
markerText.ZIndex = 27
markerText.Parent = mainFrame

local statusPill = Instance.new("TextLabel")
statusPill.Name = "StatusPill"
statusPill.Size = UDim2.fromOffset(76, 16)
statusPill.Position = UDim2.fromOffset(194, 47)
statusPill.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
statusPill.BackgroundTransparency = 0.05
statusPill.Text = "READY"
statusPill.TextColor3 = Color3.fromRGB(255, 214, 45)
statusPill.Font = Enum.Font.GothamBold
statusPill.TextSize = 7
statusPill.TextXAlignment = Enum.TextXAlignment.Center
statusPill.BorderSizePixel = 0
statusPill.ZIndex = 29
statusPill.Parent = mainFrame
local pillCorner = Instance.new("UICorner")
pillCorner.CornerRadius = UDim.new(0, 7)
pillCorner.Parent = statusPill

local pillStroke = Instance.new("UIStroke")
pillStroke.Color = Color3.fromRGB(0, 0, 0)
pillStroke.Thickness = 1
pillStroke.Transparency = 0.15
pillStroke.Parent = statusPill

local badge = Instance.new("TextLabel")
badge.Name = "SpongeBadge"
badge.Size = UDim2.fromOffset(58, 13)
badge.Position = UDim2.fromOffset(12, 48)
badge.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
badge.BackgroundTransparency = 0.05
badge.Text = "AUTO GRAB"
badge.TextColor3 = Color3.fromRGB(255, 214, 45)
badge.Font = Enum.Font.GothamBold
badge.TextSize = 6
badge.BorderSizePixel = 0
badge.ZIndex = 29
badge.Parent = mainFrame
local badgeCorner = Instance.new("UICorner")
badgeCorner.CornerRadius = UDim.new(0, 6)
badgeCorner.Parent = badge

local mainStroke = Instance.new("UIStroke")
mainStroke.Color = Color3.fromRGB(255, 214, 45)
mainStroke.Thickness = 1.35
mainStroke.Transparency = 0.05
mainStroke.Parent = mainFrame


local header = Instance.new("Frame")
header.Name = "DragHeader"
header.Size = UDim2.new(1, -88, 1, 0)
header.Position = UDim2.fromOffset(0, 0)
header.BackgroundTransparency = 1
header.Active = true
header.ZIndex = 20
header.Parent = mainFrame

local spongeIcon = Instance.new("ImageLabel")
spongeIcon.Name = "SpongeBobIcon"
spongeIcon.Size = UDim2.fromOffset(28, 28)
spongeIcon.Position = UDim2.fromOffset(238, 4)
spongeIcon.BackgroundTransparency = 1
spongeIcon.Image = "rbxassetid://10729455634"
spongeIcon.ScaleType = Enum.ScaleType.Fit
spongeIcon.ZIndex = 21
spongeIcon.Parent = mainFrame

local title = Instance.new("TextLabel")
title.Name = "Title"
title.Size = UDim2.fromOffset(130, 18)
title.Position = UDim2.fromOffset(12, 8)
title.BackgroundTransparency = 1
title.Text = "SPONGEBOB STEAL"
title.TextColor3 = Color3.fromRGB(255, 214, 45)
title.Font = Enum.Font.GothamBold
title.TextSize = 12
title.TextXAlignment = Enum.TextXAlignment.Left
title.ZIndex = 22
title.Parent = mainFrame

local progressBackground = Instance.new("Frame")
progressBackground.Name = "ProgressBackground"
progressBackground.Size = UDim2.fromOffset(196, 3)
progressBackground.Position = UDim2.fromOffset(67, 34)
progressBackground.BackgroundColor3 = Color3.fromRGB(255, 214, 45)
progressBackground.BackgroundTransparency = 0.18
progressBackground.BorderSizePixel = 0
progressBackground.ZIndex = 21
progressBackground.Visible = true
progressBackground.Parent = mainFrame
local progressCorner = Instance.new("UICorner")
progressCorner.CornerRadius = UDim.new(1, 0)
progressCorner.Parent = progressBackground

local progressBar = Instance.new("Frame")
progressBar.Name = "ProgressBar"
progressBar.Size = UDim2.new(0, 0, 1, 0)
progressBar.BackgroundColor3 = Color3.fromRGB(255, 214, 45)
progressBar.BorderSizePixel = 0
progressBar.ZIndex = 22
progressBar.Parent = progressBackground
local progressBarCorner = Instance.new("UICorner")
progressBarCorner.CornerRadius = UDim.new(1, 0)
progressBarCorner.Parent = progressBar

local progressGradient = Instance.new("UIGradient")
progressGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 214, 45)),
    ColorSequenceKeypoint.new(0.55, Color3.fromRGB(255, 190, 20)),
    ColorSequenceKeypoint.new(1, Color3.fromRGB(0, 0, 0)),
})
progressGradient.Parent = progressBar

local brainrotStatusLabel = Instance.new("TextLabel")
brainrotStatusLabel.Name = "BrainrotStatus"
brainrotStatusLabel.Size = UDim2.fromOffset(196, 13)
brainrotStatusLabel.Position = UDim2.fromOffset(67, 48)
brainrotStatusLabel.BackgroundTransparency = 1
brainrotStatusLabel.Text = "0 brainrots in the server"
brainrotStatusLabel.TextColor3 = Color3.fromRGB(255, 214, 45)
brainrotStatusLabel.Font = Enum.Font.GothamBold
brainrotStatusLabel.TextSize = 8
brainrotStatusLabel.TextXAlignment = Enum.TextXAlignment.Center
brainrotStatusLabel.Visible = false
brainrotStatusLabel.ZIndex = 35
brainrotStatusLabel.Parent = mainFrame

-- Compatibility objects kept because the original Auto Grab logic updates them.
-- They are intentionally hidden/off-card: this changes UI only, not the logic.
local compatStatusDot = Instance.new("Frame")
compatStatusDot.Name = "StatusDot"
compatStatusDot.Size = UDim2.fromOffset(1, 1)
compatStatusDot.Position = UDim2.fromOffset(-20, -20)
compatStatusDot.BackgroundTransparency = 1
compatStatusDot.Visible = false
compatStatusDot.Parent = screenGui
local statusDot = compatStatusDot

local statusText = Instance.new("TextLabel")
statusText.Name = "StatusText"
statusText.Size = UDim2.fromOffset(84, 15)
statusText.Position = UDim2.new(1, -94, 0, 25)
statusText.BackgroundTransparency = 1
statusText.Text = ""
statusText.TextColor3 = Color3.fromRGB(255, 214, 45)
statusText.Font = Enum.Font.GothamBold
statusText.TextSize = 8
statusText.TextXAlignment = Enum.TextXAlignment.Right
statusText.ZIndex = 22
statusText.Visible = false
statusText.Parent = mainFrame

local remoteLabel = Instance.new("TextLabel")
remoteLabel.Name = "RemoteLabel"
remoteLabel.Size = UDim2.fromOffset(1, 1)
remoteLabel.Position = UDim2.fromOffset(-20, -20)
remoteLabel.BackgroundTransparency = 1
remoteLabel.TextTransparency = 1
remoteLabel.Visible = false
remoteLabel.Parent = screenGui

local stateSmall = Instance.new("TextLabel")
stateSmall.Name = "StateLabel"
stateSmall.Size = UDim2.fromOffset(1, 1)
stateSmall.Position = UDim2.fromOffset(-20, -20)
stateSmall.BackgroundTransparency = 1
stateSmall.TextTransparency = 1
stateSmall.Visible = false
stateSmall.Parent = screenGui

local statusLabel = Instance.new("TextLabel")
statusLabel.Name = "StatusLabel"
statusLabel.Size = UDim2.fromOffset(1, 1)
statusLabel.Position = UDim2.fromOffset(-20, -20)
statusLabel.BackgroundTransparency = 1
statusLabel.TextTransparency = 1
statusLabel.Visible = false
statusLabel.Parent = screenGui

local infoPanel = Instance.new("Frame")
infoPanel.Name = "InfoPanel"
infoPanel.Size = UDim2.fromOffset(1, 1)
infoPanel.Position = UDim2.fromOffset(-20, -20)
infoPanel.BackgroundTransparency = 1
infoPanel.Visible = false
infoPanel.Parent = screenGui

local infoButton = Instance.new("TextButton")
infoButton.Name = "InfoButton"
infoButton.Size = UDim2.fromOffset(1, 1)
infoButton.Position = UDim2.fromOffset(-20, -20)
infoButton.BackgroundTransparency = 1
infoButton.TextTransparency = 1
infoButton.AutoButtonColor = false
infoButton.Visible = false
infoButton.Parent = screenGui

-- Invisible full-card compatibility toggle. The original logic remains untouched.
local toggleBtn = Instance.new("TextButton")
toggleBtn.Name = "ToggleBtn"
toggleBtn.Size = UDim2.fromOffset(56, 20)
toggleBtn.Position = UDim2.fromOffset(211, 27)
toggleBtn.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
toggleBtn.BackgroundTransparency = 0.05
toggleBtn.Text = ""
toggleBtn.TextColor3 = Color3.fromRGB(255, 214, 45)
toggleBtn.TextSize = 8
toggleBtn.Font = Enum.Font.GothamBold
toggleBtn.TextTransparency = 0
toggleBtn.BorderSizePixel = 0
toggleBtn.AutoButtonColor = false
toggleBtn.Visible = false
toggleBtn.ZIndex = 40
toggleBtn.Active = true
toggleBtn.Parent = mainFrame

local toggleCorner = Instance.new("UICorner")
toggleCorner.CornerRadius = UDim.new(0, 8)
toggleCorner.Parent = toggleBtn

local toggleStroke = Instance.new("UIStroke")
toggleStroke.Color = Color3.fromRGB(255, 231, 74)
toggleStroke.Thickness = 1
toggleStroke.Transparency = 0.15
toggleStroke.Parent = toggleBtn

local toggleIndicator = Instance.new("Frame")
toggleIndicator.Size = UDim2.fromOffset(6, 6)
toggleIndicator.Position = UDim2.fromOffset(8, 7)
toggleIndicator.BackgroundColor3 = Color3.fromRGB(255, 214, 45)
toggleIndicator.BackgroundTransparency = 0
toggleIndicator.Visible = true
toggleIndicator.Parent = toggleBtn

-- Center line restored inside the Auto Grab card.
-- It is NOT a screen-center overlay; it stays inside this compact UI.
local centerLine = Instance.new("Frame")
centerLine.Name = "CenterLine"
centerLine.Size = UDim2.fromOffset(2, 8)
centerLine.Position = UDim2.fromOffset(165, 31)
centerLine.BackgroundColor3 = Color3.fromRGB(255, 214, 45)
centerLine.BackgroundTransparency = 0
centerLine.BorderSizePixel = 0
centerLine.Visible = true
centerLine.ZIndex = 50
centerLine.Parent = mainFrame
local centerLineCorner = Instance.new("UICorner")
centerLineCorner.CornerRadius = UDim.new(1, 0)
centerLineCorner.Parent = centerLine

-- ============================================================
-- DRAG
-- ============================================================

local dragging = false
local dragStart
local startPosition

local function updateDrag(input)
    if not dragging then return end
    local delta = input.Position - dragStart
    mainFrame.Position = UDim2.new(
        startPosition.X.Scale,
        startPosition.X.Offset + delta.X,
        startPosition.Y.Scale,
        startPosition.Y.Offset + delta.Y
    )
end

header.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
        dragging = true
        dragStart = input.Position
        startPosition = mainFrame.Position
    end
end)

header.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1
        or input.UserInputType == Enum.UserInputType.Touch then
        dragging = false
        pcall(saveAutoGrabPosition)
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement
        or input.UserInputType == Enum.UserInputType.Touch then
        updateDrag(input)
    end
end)

-- ACTIVATION SOUND
-- ============================================================

local activationSound =
    Instance.new("Sound")

activationSound.Name =
    "InstantGrabActivationSound"

activationSound.SoundId =
    "rbxassetid://98797174600699"

activationSound.Volume =
    1

activationSound.Parent =
    screenGui

local emptyServerSound = Instance.new("Sound")
emptyServerSound.Name = "NoBrainrotsSound"
emptyServerSound.SoundId = "rbxassetid://17208361335"
emptyServerSound.Volume = 1
emptyServerSound.Parent = screenGui

-- ============================================================
-- VARIABLES
-- ============================================================

local isEnabled = false

local currentTarget = nil

local currentDist =
    math.huge

local currentPlotName = nil

local cycleTimer = 0

local PromptMemoryCache = {}

-- ============================================================
-- GET MY PLOT
-- ============================================================

local function getMyPlot()

    local plots =
        workspace:FindFirstChild(
            "Plots"
        )

    if not plots then
        return nil
    end

    for _, plot in
        ipairs(
            plots:GetChildren()
        ) do

        local sign =
            plot:FindFirstChild(
                "PlotSign"
            )

        if sign and
            sign:FindFirstChild(
                "SurfaceGui"
            ) then

            local fr =
                sign.SurfaceGui:FindFirstChild(
                    "Frame"
                )

            if fr and
                fr:FindFirstChild(
                    "TextLabel"
                ) then

                if fr.TextLabel.Text ==
                    LP.DisplayName ..
                    "'s Base" then

                    return plot
                end
            end
        end
    end

    return nil
end

-- ============================================================
-- FIND NEAREST PROMPT
-- ============================================================

local function findNearestPromptInPlots(
    maxRadius
)

    local char =
        LP.Character

    if not char then
        return nil, math.huge, nil
    end

    local root =
        char:FindFirstChild(
            "HumanoidRootPart"
        )
        or char:FindFirstChild(
            "Torso"
        )
        or char:FindFirstChild(
            "UpperTorso"
        )

    if not root then
        return nil, math.huge, nil
    end

    local plots =
        workspace:FindFirstChild(
            "Plots"
        )

    if not plots then
        return nil, math.huge, nil
    end

    local myPlot =
        getMyPlot()

    local nearestPrompt = nil
    local minDist =
        maxRadius

    local targetPlotName = nil

    for _, plot in
        ipairs(
            plots:GetChildren()
        ) do

        if myPlot and
            plot == myPlot then

            continue
        end

        local podiums =
            plot:FindFirstChild(
                "AnimalPodiums"
            )

        if podiums then

            for _, podium in
                ipairs(
                    podiums:GetChildren()
                ) do

                local uid =
                    plot.Name ..
                    "_" ..
                    podium.Name

                local cached =
                    PromptMemoryCache[
                        uid
                    ]

                local targetPrompt = nil

                if cached and
                    cached.Parent then

                    targetPrompt =
                        cached

                else

                    local base =
                        podium:FindFirstChild(
                            "Base"
                        )

                    local spawnPoint =
                        base and
                        base:FindFirstChild(
                            "Spawn"
                        )

                    local attachment =
                        spawnPoint and
                        spawnPoint:FindFirstChild(
                            "PromptAttachment"
                        )

                    if attachment then

                        for _, child in
                            ipairs(
                                attachment:GetChildren()
                            ) do

                            if child:IsA(
                                "ProximityPrompt"
                            )
                            and child.Enabled then

                                targetPrompt =
                                    child

                                PromptMemoryCache[
                                    uid
                                ] =
                                    child

                                break
                            end
                        end
                    end
                end

                if targetPrompt
                    and targetPrompt.Enabled
                    and targetPrompt.Parent then

                    local part =
                        targetPrompt.Parent

                    if part:IsA(
                        "Attachment"
                    ) then

                        part =
                            part.Parent
                    end

                    if part and
                        part:IsA(
                            "BasePart"
                        ) then

                        local dist =
                            (
                                part.Position -
                                root.Position
                            ).Magnitude

                        if dist < minDist then

                            minDist =
                                dist

                            nearestPrompt =
                                targetPrompt

                            targetPlotName =
                                plot.Name
                        end
                    end
                end
            end
        end
    end

    return nearestPrompt,
        minDist,
        targetPlotName
end

local cachedBrainrotCount = 0
local brainrotScanClock = 0
local lastBrainrotStateZero = false

local function countAvailableBrainrots()
    local now = os.clock()
    if now - brainrotScanClock < 0.20 then
        return cachedBrainrotCount
    end
    brainrotScanClock = now

    local plots = workspace:FindFirstChild("Plots")
    if not plots then
        cachedBrainrotCount = 0
        return 0
    end

    local myPlot = getMyPlot()
    local count = 0
    for _, plot in ipairs(plots:GetChildren()) do
        if plot ~= myPlot then
            local podiums = plot:FindFirstChild("AnimalPodiums")
            if podiums then
                for _, prompt in ipairs(podiums:GetDescendants()) do
                    if prompt:IsA("ProximityPrompt") and prompt.Enabled and prompt.Parent then
                        count += 1
                    end
                end
            end
        end
    end

    cachedBrainrotCount = count
    return count
end

local function updateEmptyServerState(count)
    if count <= 0 then
        brainrotStatusLabel.Visible = true
        if not lastBrainrotStateZero then
            lastBrainrotStateZero = true
            pcall(function()
                emptyServerSound:Stop()
                emptyServerSound.TimePosition = 0
                emptyServerSound:Play()
            end)
        end
        return true
    end

    brainrotStatusLabel.Visible = false
    lastBrainrotStateZero = false
    return false
end

-- ============================================================
-- TRIGGER HOLD
-- ============================================================

local function triggerHoldBegan(
    prompt
)

    if not prompt then
        return
    end

    if getconnections then

        for _, conn in ipairs(
            getconnections(
                prompt.PromptButtonHoldBegan
            )
        ) do

            if conn.Function then

                task.spawn(
                    conn.Function
                )
            end
        end

    else

        prompt:InputHoldBegan()
    end
end

-- ============================================================
-- TRIGGER PROMPT
-- ============================================================

local function triggerPrompt(
    prompt
)

    if not prompt then
        return
    end

    if getconnections then

        for _, conn in ipairs(
            getconnections(
                prompt.Triggered
            )
        ) do

            if conn.Function then

                task.spawn(
                    conn.Function
                )
            end
        end

    else

        prompt:InputHoldEnded()
    end
end

-- ============================================================
-- VISUAL STATE
-- ============================================================

local function updateEnabledVisual()

    if isEnabled then

        statusDot.BackgroundColor3 =
            COLORS.White

        statusText.Text =
            ""

        statusText.TextColor3 =
            COLORS.White

        toggleBtn.Text =
            ""

        toggleBtn.BackgroundColor3 = PURPLE2

        toggleBtn.TextColor3 =
            COLORS.White

        toggleStroke.Color = PURPLE

        toggleStroke.Transparency =
            0

        toggleIndicator.BackgroundColor3 =
            COLORS.White

        remoteLabel.TextColor3 =
            Color3.fromRGB(
                160,
                160,
                170
            )

    else

        statusDot.BackgroundColor3 =
            COLORS.Gray2

        statusText.Text =
            ""

        statusText.TextColor3 =
            COLORS.Gray

        toggleBtn.Text =
            ""

        toggleBtn.BackgroundColor3 =
            COLORS.Panel

        toggleBtn.TextColor3 =
            COLORS.White

        toggleStroke.Color =
            Color3.fromRGB(
                40,
                40,
                45
            )

        toggleStroke.Transparency =
            0

        toggleIndicator.BackgroundColor3 =
            COLORS.Gray2

        remoteLabel.TextColor3 =
            Color3.fromRGB(
                100,
                100,
                110
            )
    end

    -- LINHA SEMPRE VISÍVEL
    centerLine.Visible = true
end

-- ============================================================
-- UPDATE PROGRESS
-- ============================================================

local function updateProgress(
    percent,
    state
)

    pcall(function()
        if statusPill then
            local labels = {
                SEARCHING = "SEARCHING",
                UNREADY = "PREPARING",
                READY = "CARRYING",
                TRIGGERED = "GRABBED",
                NO_BRAINROTS = "EMPTY",
                IDLE = "READY"
            }
            statusPill.Text = labels[state] or "READY"
        end
    end)

    percent =
        math.clamp(
            percent,
            0,
            1
        )

    TweenService:Create(
        progressBar,

        TweenInfo.new(
            0.08,
            Enum.EasingStyle.Linear
        ),

        {
            Size =
                UDim2.new(
                    percent,
                    0,
                    1,
                    0
                )
        }

    ):Play()

    if state ==
        "SEARCHING" then

        progressBar.BackgroundColor3 = PURPLE

        statusLabel.Text =
            "SEARCHING..."

        statusLabel.TextColor3 =
            COLORS.White

        stateSmall.Text =
            "SEARCHING"

    elseif state ==
        "UNREADY" then

        progressBar.BackgroundColor3 = Color3.fromRGB(82,94,108)

        statusLabel.Text =
            string.format(
                "PREPARING   %d%%",
                math.floor(
                    percent * 100
                )
            )

        statusLabel.TextColor3 =
            COLORS.White

        stateSmall.Text =
            "PREPARING"

    elseif state ==
        "READY" then

        progressBar.BackgroundColor3 = PURPLE

        statusLabel.Text =
            string.format(
                "CARRYING   %d%%",
                math.floor(
                    percent * 100
                )
            )

        statusLabel.TextColor3 =
            COLORS.Black

        stateSmall.Text =
            "READY"

    elseif state ==
        "NO_BRAINROTS" then

        progressBar.BackgroundColor3 = PURPLE
        statusLabel.Text = ""
        stateSmall.Text = ""
        brainrotStatusLabel.Visible = true

    elseif state ==
        "TRIGGERED" then

        progressBar.BackgroundColor3 =
            Color3.fromRGB(
                200,
                200,
                210
            )

        statusLabel.Text =
            "TRIGGERED!"

        statusLabel.TextColor3 =
            COLORS.Black

        stateSmall.Text =
            "TRIGGERED"

    else

        progressBar.BackgroundColor3 =
            COLORS.DarkGray

        statusLabel.Text =
            "IDLE"

        statusLabel.TextColor3 =
            COLORS.White

        stateSmall.Text =
            "SYSTEM READY"
    end

    -- NUNCA ESCONDER
    centerLine.Visible = true
end

-- ============================================================
-- START AUTO GRAB
-- ============================================================

local function startAutoGrab()

    if isEnabled then
        return
    end

    isEnabled =
        true

    cycleTimer =
        0

    currentTarget =
        nil

    currentDist =
        math.huge

    currentPlotName =
        nil

    updateEnabledVisual()

    updateProgress(
        0,
        "SEARCHING"
    )

    -- SOUND
    pcall(function()

        activationSound:Stop()

        activationSound.TimePosition =
            0

        activationSound:Play()

    end)
end

-- ============================================================
-- STOP AUTO GRAB
-- ============================================================

local function stopAutoGrab()

    isEnabled =
        false

    cycleTimer =
        0

    currentTarget =
        nil

    currentDist =
        math.huge

    currentPlotName =
        nil

    table.clear(
        PromptMemoryCache
    )

    updateEnabledVisual()

    updateProgress(
        0,
        "IDLE"
    )

    centerLine.Visible =
        true
end

-- ============================================================
-- TOGGLE
-- ============================================================

toggleBtn.Activated:Connect(
    function()

        if isEnabled then

            stopAutoGrab()

        else

            startAutoGrab()

        end
    end
)

-- ============================================================
-- BUTTON ANIMATION
-- ============================================================

toggleBtn.MouseButton1Down:Connect(function()
    TweenService:Create(toggleBtn, TweenInfo.new(0.08, Enum.EasingStyle.Quad), {
        Size = UDim2.fromOffset(53, 19)
    }):Play()
end)

toggleBtn.MouseButton1Up:Connect(function()
    TweenService:Create(toggleBtn, TweenInfo.new(0.12, Enum.EasingStyle.Back), {
        Size = UDim2.fromOffset(56, 20)
    }):Play()
end)

-- ============================================================
-- MAIN LOOP
-- ============================================================

RunService.RenderStepped:Connect(
    function(dt)
        if not isEnabled then
            return
        end

        -- Keep the server-empty indicator, but restore the original Auto Grab timing.
        local brainrotCount = countAvailableBrainrots()
        if updateEmptyServerState(brainrotCount) then
            currentTarget = nil
            currentDist = math.huge
            currentPlotName = nil
            cycleTimer = 0
            updateProgress(0, "NO_BRAINROTS")
            centerLine.Visible = true
            return
        end

        brainrotStatusLabel.Visible = false

        -- Original behaviour: find the nearest prompt each frame.
        local prompt, dist, plotName = findNearestPromptInPlots(2000)

        currentTarget = prompt
        currentDist = dist
        currentPlotName = plotName

        if not currentTarget then
            updateProgress(0, "SEARCHING")
            cycleTimer = 0
            return
        end

        cycleTimer = cycleTimer + dt

        local percent = math.clamp(cycleTimer / 2.6, 0, 1)

        if cycleTimer < 1.3 then
            updateProgress(percent, "UNREADY")
        elseif cycleTimer < 2.6 then
            updateProgress(percent, "READY")

            if currentDist <= 8 then
                updateProgress(1, "TRIGGERED")
                triggerPrompt(currentTarget)
                triggerHoldBegan(currentTarget)
                cycleTimer = 0
            end
        else
            updateProgress(1, "TRIGGERED")
            triggerPrompt(currentTarget)
            triggerHoldBegan(currentTarget)
            cycleTimer = 0
        end

        -- Keep the marker inside the Auto Grab card.
        centerLine.Visible = true
    end
)

-- ============================================================
-- INITIALIZATION
-- ============================================================

updateEnabledVisual()

updateProgress(
    0,
    "IDLE"
)
brainrotStatusLabel.Visible = false
centerLine.Position = UDim2.fromOffset(165, 31)
centerLine.Visible = true

-- Auto Grab inicia ligado automaticamente ao carregar a interface.
task.defer(function()
    if not isEnabled then
        startAutoGrab()
    end
end)

centerLine.Position =
    UDim2.fromOffset(165, 31)

centerLine.Visible =
    true

pcall(saveAutoGrabPosition)

print(
    "[Instant Grab] Dark Control UI loaded."
)
]]
local grabInstaChunk = nil
local grabInstaLoaded = false

openGrabInsta = function(state)
    if not state then
        local g = (gethui and gethui()) or CoreGui or PlayerGui
        local old = g and g:FindFirstChild("InstantGrabSpecificGui")
        if old then
            pcall(function() old.Enabled = false end)
            pcall(function() old:Destroy() end)
        end
        grabInstaLoaded = false
        grabInstaChunk = nil
        return
    end

    if grabInstaLoaded then
        local g = (gethui and gethui()) or CoreGui or PlayerGui
        local existing = g and g:FindFirstChild("InstantGrabSpecificGui")
        if existing then
            existing.Enabled = true
            return
        end
        grabInstaLoaded = false
    end

    if type(loadstring) ~= "function" then
        warn("[Grab Insta] loadstring is unavailable")
        return
    end

    local fn, err = loadstring(embeddedGrabInstaSource)
    if not fn then
        warn("[Grab Insta] compile failed:", err)
        return
    end

    local ok, result = pcall(fn)
    if ok then
        grabInstaChunk = result
        grabInstaLoaded = true
    else
        warn("[Grab Insta] failed:", result)
    end
end

_, U.autoGrabBtn = rowButton(U.targetBody,"Auto Grab",1,autoGrabEnabled,function(s)
    setAutoGrab(s)
end,true)
_, U.podiumBtn = rowButton(U.targetBody,"Podium ESP",2,podiumEnabled,function(s)
    slotESPActive = s
    podiumEnabled = s
    persistFeature("podiumEnabled", s)
    if s then
        pcall(newSlotESP)
    elseif _podiumCleanup then
        pcall(_podiumCleanup)
    end
end,true)
_, U.speedBtn = rowButton(U.targetBody,"Speed Booster",3,speedBoosterEnabled,function(s)
    setSpeedBoosterEnabled(s == true)
end,true)

local targetSpeedRow = Instance.new("Frame")
targetSpeedRow.Size = UDim2.new(1,0,0,27)
targetSpeedRow.LayoutOrder = 4
targetSpeedRow.BackgroundColor3 = UI_ROW
targetSpeedRow.BorderSizePixel = 0
targetSpeedRow.ZIndex = 110
targetSpeedRow.Parent = U.targetBody
local targetSpeedCorner = Instance.new("UICorner")
targetSpeedCorner.CornerRadius = UDim.new(0,9)
targetSpeedCorner.Parent = targetSpeedRow
local targetSpeedLabel = Instance.new("TextLabel")
targetSpeedLabel.Size = UDim2.new(1,-82,1,0)
targetSpeedLabel.Position = UDim2.fromOffset(9,0)
targetSpeedLabel.BackgroundTransparency = 1
targetSpeedLabel.Font = FONT_REGULAR
targetSpeedLabel.Text = "Speed Value"
targetSpeedLabel.TextColor3 = UI_WHITE
targetSpeedLabel.TextSize = 8.5
targetSpeedLabel.TextXAlignment = Enum.TextXAlignment.Left
targetSpeedLabel.Parent = targetSpeedRow
local targetSpeedBox = Instance.new("Frame")
targetSpeedBox.Size = UDim2.fromOffset(76,20)
targetSpeedBox.Position = UDim2.new(1,-82,0.5,-10)
targetSpeedBox.BackgroundColor3 = UI_OFF
targetSpeedBox.BorderSizePixel = 0
targetSpeedBox.Parent = targetSpeedRow
local tsbc = Instance.new("UICorner")
tsbc.CornerRadius = UDim.new(0,8)
tsbc.Parent = targetSpeedBox

local speedMinus2 = Instance.new("TextButton")
speedMinus2.Size = UDim2.fromOffset(22,20)
speedMinus2.Position = UDim2.fromOffset(0,0)
speedMinus2.BackgroundTransparency = 1
speedMinus2.AutoButtonColor = false
speedMinus2.Font = FONT_MAIN
speedMinus2.Text = "-"
speedMinus2.TextColor3 = UI_WHITE
speedMinus2.TextSize = 10
speedMinus2.ZIndex = 114
speedMinus2.Parent = targetSpeedBox

local speedValueInput = Instance.new("TextBox")
speedValueInput.Size = UDim2.fromOffset(32,20)
speedValueInput.Position = UDim2.fromOffset(22,0)
speedValueInput.BackgroundTransparency = 1
speedValueInput.ClearTextOnFocus = false
speedValueInput.Font = FONT_MAIN
speedValueInput.Text = tostring(speedBoosterValue)
speedValueInput.TextColor3 = UI_WHITE
speedValueInput.TextSize = 8
speedValueInput.TextXAlignment = Enum.TextXAlignment.Center
speedValueInput.ZIndex = 114
speedValueInput.Parent = targetSpeedBox

local speedPlus2 = Instance.new("TextButton")
speedPlus2.Size = UDim2.fromOffset(22,20)
speedPlus2.Position = UDim2.new(1,-22,0,0)
speedPlus2.BackgroundTransparency = 1
speedPlus2.AutoButtonColor = false
speedPlus2.Font = FONT_MAIN
speedPlus2.Text = "+"
speedPlus2.TextColor3 = UI_WHITE
speedPlus2.TextSize = 10
speedPlus2.ZIndex = 114
speedPlus2.Parent = targetSpeedBox

local function refreshTargetSpeed()
    speedBoosterValue = math.clamp(math.floor(tonumber(speedBoosterValue) or 27), 1, 200)
    speedValueInput.Text = tostring(speedBoosterValue)
end

speedValueInput.FocusLost:Connect(function()
    local n = tonumber(speedValueInput.Text)
    if not n then
        refreshTargetSpeed()
        return
    end
    speedBoosterValue = math.clamp(math.floor(n), 1, 200)
    refreshTargetSpeed()
    persistFeature("speedBoosterValue", speedBoosterValue)
    if speedBoosterEnabled then setSpeedBooster(true) end
end)

speedMinus2.MouseButton1Click:Connect(function()
    speedBoosterValue = math.max(1, speedBoosterValue - 1)
    refreshTargetSpeed()
    persistFeature("speedBoosterValue", speedBoosterValue)
    if speedBoosterEnabled then setSpeedBooster(true) end
end)

speedPlus2.MouseButton1Click:Connect(function()
    speedBoosterValue = math.min(200, speedBoosterValue + 1)
    refreshTargetSpeed()
    persistFeature("speedBoosterValue", speedBoosterValue)
    if speedBoosterEnabled then setSpeedBooster(true) end
end)

local targetActions = Instance.new("Frame")
targetActions.Size = UDim2.new(1,0,0,28)
targetActions.LayoutOrder = 5
targetActions.BackgroundTransparency = 1
targetActions.ZIndex = 111
targetActions.Parent = U.targetBody
local function targetActionBtn(text,x)
    local b = Instance.new("TextButton")
    b.Size = UDim2.new(0.5,-3,1,0)
    b.Position = UDim2.new(x,0,0,0)
    b.BackgroundColor3 = Color3.fromRGB(255,193,7)
    b.AutoButtonColor = false
    b.Font = FONT_MAIN
    b.Text = text
    b.TextColor3 = UI_WHITE
    b.TextSize = 8
    b.ZIndex = 112
    b.Parent = targetActions
    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0,8)
    c.Parent = b
    local st = Instance.new("UIStroke")
    st.Color = Color3.fromRGB(255,236,100)
    st.Thickness = 1
    st.Parent = b
    return b
end
U.dropBtn = targetActionBtn("DROP",0)
U.resetBtn = targetActionBtn("RESET",0.5)
U.dropBtn.MouseButton1Click:Connect(function()
    pcall(dropBrainrot)
end)
U.resetBtn.MouseButton1Click:Connect(function()
    persistAllSettings()
    doInstantReset()
end)

-- KEYBINDS WINDOW — click the key button to rebind
-- ------------------------------------------------------------
U.keybind = Instance.new("Frame")
U.keybind.Name = "KeybindsWindow"
U.keybind.Size = UDim2.fromOffset(220,245)
U.keybind.ZIndex = 500
U.keybind.Visible = false
U.keybind.Parent = screenGui
stylePanel2(U.keybind,15)

U.keyTitle = Instance.new("TextLabel")
U.keyTitle.Size = UDim2.new(1,-40,0,34)
U.keyTitle.Position = UDim2.fromOffset(12,0)
U.keyTitle.BackgroundTransparency = 1
U.keyTitle.Font = FONT_MAIN
U.keyTitle.Text = "KEYBINDS"
U.keyTitle.TextColor3 = UI_WHITE
U.keyTitle.TextSize = 12
U.keyTitle.ZIndex = 501
U.keyTitle.Parent = U.keybind

U.keyClose = Instance.new("TextButton")
U.keyClose.Size = UDim2.fromOffset(22,22)
U.keyClose.Position = UDim2.new(1,-30,0,7)
U.keyClose.BackgroundColor3 = UI_PURPLE_DARK
U.keyClose.AutoButtonColor = false
U.keyClose.Font = FONT_MAIN
U.keyClose.Text = "X"
U.keyClose.TextColor3 = UI_WHITE
U.keyClose.TextSize = 9
U.keyClose.ZIndex = 502
U.keyClose.Parent = U.keybind
local keyCloseCorner = Instance.new("UICorner")
keyCloseCorner.CornerRadius = UDim.new(0,7)
keyCloseCorner.Parent = U.keyClose

local keyLine = Instance.new("Frame")
keyLine.Size = UDim2.new(1,-34,0,1)
keyLine.Position = UDim2.fromOffset(17,40)
keyLine.BackgroundColor3 = Color3.fromRGB(95,95,102)
keyLine.BackgroundTransparency = 0.5
keyLine.BorderSizePixel = 0
keyLine.ZIndex = 501
keyLine.Parent = U.keybind

local kbBody = Instance.new("Frame")
kbBody.Size = UDim2.new(1,-24,1,-55)
kbBody.Position = UDim2.fromOffset(12,49)
kbBody.BackgroundTransparency = 1
kbBody.ZIndex = 501
kbBody.Parent = U.keybind

local kbSection = Instance.new("TextLabel")
kbSection.Size = UDim2.new(1,0,0,14)
kbSection.BackgroundTransparency = 1
kbSection.Font = FONT_MAIN
kbSection.Text = "MOVEMENT"
kbSection.TextColor3 = UI_PURPLE
kbSection.TextSize = 7
kbSection.TextXAlignment = Enum.TextXAlignment.Left
kbSection.ZIndex = 502
kbSection.Parent = kbBody

local ckRow = Instance.new("Frame")
ckRow.Size = UDim2.new(1,0,0,32)
ckRow.Position = UDim2.fromOffset(0,18)
ckRow.BackgroundColor3 = UI_PURPLE_DARK
ckRow.ZIndex = 502
ckRow.Parent = kbBody
local ckCorner = Instance.new("UICorner")
ckCorner.CornerRadius = UDim.new(0,9)
ckCorner.Parent = ckRow

local ckLabel = Instance.new("TextLabel")
ckLabel.Size = UDim2.new(1,-85,1,0)
ckLabel.Position = UDim2.fromOffset(10,0)
ckLabel.BackgroundTransparency = 1
ckLabel.Font = FONT_REGULAR
ckLabel.Text = "Carpet Speed"
ckLabel.TextColor3 = UI_WHITE
ckLabel.TextSize = 9
ckLabel.TextXAlignment = Enum.TextXAlignment.Left
ckLabel.ZIndex = 503
ckLabel.Parent = ckRow

U.keyButton = Instance.new("TextButton")
U.keyButton.Size = UDim2.fromOffset(58,22)
U.keyButton.Position = UDim2.new(1,-66,0.5,-11)
U.keyButton.BackgroundColor3 = UI_PURPLE_DARK
U.keyButton.AutoButtonColor = false
U.keyButton.Font = FONT_MAIN
U.keyButton.Text = KEYBIND_SPEED.Name
U.keyButton.TextColor3 = UI_WHITE
U.keyButton.TextSize = 8
U.keyButton.ZIndex = 503
U.keyButton.Parent = ckRow
local keyCorner = Instance.new("UICorner")
keyCorner.CornerRadius = UDim.new(0,7)
keyCorner.Parent = U.keyButton

-- SPEED BOOSTER KEYBIND
local boosterRow = Instance.new("Frame")
boosterRow.Size = UDim2.new(1,0,0,32)
boosterRow.Position = UDim2.fromOffset(0,54)
boosterRow.BackgroundColor3 = UI_PURPLE_DARK
boosterRow.ZIndex = 502
boosterRow.Parent = kbBody
local boosterCorner = Instance.new("UICorner")
boosterCorner.CornerRadius = UDim.new(0,9)
boosterCorner.Parent = boosterRow

local boosterLabel = Instance.new("TextLabel")
boosterLabel.Size = UDim2.new(1,-85,1,0)
boosterLabel.Position = UDim2.fromOffset(10,0)
boosterLabel.BackgroundTransparency = 1
boosterLabel.Font = FONT_REGULAR
boosterLabel.Text = "Speed Booster"
boosterLabel.TextColor3 = UI_WHITE
boosterLabel.TextSize = 9
boosterLabel.TextXAlignment = Enum.TextXAlignment.Left
boosterLabel.ZIndex = 503
boosterLabel.Parent = boosterRow

U.boosterKeyButton = Instance.new("TextButton")
U.boosterKeyButton.Size = UDim2.fromOffset(58,22)
U.boosterKeyButton.Position = UDim2.new(1,-66,0.5,-11)
U.boosterKeyButton.BackgroundColor3 = UI_PURPLE_DARK
U.boosterKeyButton.AutoButtonColor = false
U.boosterKeyButton.Font = FONT_MAIN
U.boosterKeyButton.Text = KEYBIND_BOOSTER.Name
U.boosterKeyButton.TextColor3 = UI_WHITE
U.boosterKeyButton.TextSize = 8
U.boosterKeyButton.ZIndex = 503
U.boosterKeyButton.Parent = boosterRow
local boosterKeyCorner = Instance.new("UICorner")
boosterKeyCorner.CornerRadius = UDim.new(0,7)
boosterKeyCorner.Parent = U.boosterKeyButton

local gearLabel = Instance.new("TextLabel")
gearLabel.Size = UDim2.new(1,0,0,15)
gearLabel.Position = UDim2.fromOffset(0,96)
gearLabel.BackgroundTransparency = 1
gearLabel.Font = FONT_MAIN
gearLabel.Text = "GEAR SPEED"
gearLabel.TextColor3 = UI_PURPLE
gearLabel.TextSize = 7
gearLabel.TextXAlignment = Enum.TextXAlignment.Left
gearLabel.ZIndex = 502
gearLabel.Parent = kbBody

U.gearSelectButton = Instance.new("TextButton")
U.gearSelectButton.Size = UDim2.new(1,0,0,34)
U.gearSelectButton.Position = UDim2.fromOffset(0,114)
U.gearSelectButton.BackgroundColor3 = UI_PURPLE_DARK
U.gearSelectButton.AutoButtonColor = false
U.gearSelectButton.Font = FONT_MAIN
U.gearSelectButton.TextColor3 = UI_WHITE
U.gearSelectButton.TextSize = 8
U.gearSelectButton.ZIndex = 503
U.gearSelectButton.Parent = kbBody
local gearSelectCorner = Instance.new("UICorner")
gearSelectCorner.CornerRadius = UDim.new(0,9)
gearSelectCorner.Parent = U.gearSelectButton
local gearSelectStroke = Instance.new("UIStroke")
gearSelectStroke.Color = UI_PURPLE_BORDER
gearSelectStroke.Thickness = 1
gearSelectStroke.Parent = U.gearSelectButton

local function updateKeyUI()
    U.keyButton.Text = KEYBIND_SPEED.Name
    U.boosterKeyButton.Text = KEYBIND_BOOSTER.Name
    U.gearSelectButton.Text = "SELECT GEAR SPEED  •  " .. tostring(selectedGearItem)
    U.flyBtn.Text = tostring(selectedGearItem):upper()
end
updateKeyUI()

local kbFooter = Instance.new("TextLabel")
kbFooter.Size = UDim2.new(1,0,0,16)
kbFooter.Position = UDim2.new(0,0,1,-20)
kbFooter.BackgroundTransparency = 1
kbFooter.Font = FONT_REGULAR
kbFooter.Text = "click the key • backspace clears"
kbFooter.TextColor3 = UI_MUTED
kbFooter.TextSize = 7
kbFooter.ZIndex = 502
kbFooter.Parent = U.keybind

makeDraggableAndPersistent(U.keybind,U.keyTitle,"KeybindsWindow",UDim2.new(0.5,-129,0.5,-118))

U.rebinding = false
U.boosterRebinding = false
U.keyButton.Activated:Connect(function()
    U.rebinding = true
    U.keyButton.Text = "PRESS KEY"
    U.keyButton.BackgroundColor3 = UI_PURPLE_DARK
end)

U.boosterKeyButton.Activated:Connect(function()
    U.boosterRebinding = true
    U.rebinding = false
    U.boosterKeyButton.Text = "PRESS KEY"
    U.boosterKeyButton.BackgroundColor3 = UI_PURPLE_DARK
end)

U.keyClose.MouseButton1Click:Connect(function()
    U.rebinding = false
    U.boosterRebinding = false
    U.keybind.Visible = false
end)

UserInputService.InputBegan:Connect(function(input,gp)
    if input.UserInputType ~= Enum.UserInputType.Keyboard then return end

    if U.rebinding then
        if input.KeyCode == Enum.KeyCode.Backspace then
            U.rebinding = false
            _G.XKPubSetMainSpeedKeybind(Enum.KeyCode.Unknown)
            U.keyButton.Text = "NONE"
            U.keyButton.BackgroundColor3 = UI_OFF
            return
        end
        if input.KeyCode ~= Enum.KeyCode.Unknown then
            _G.XKPubSetMainSpeedKeybind(input.KeyCode)
            U.rebinding = false
            U.keyButton.Text = input.KeyCode.Name
            U.keyButton.BackgroundColor3 = UI_PURPLE_DARK
            return
        end
    end

    if U.boosterRebinding then
        if input.KeyCode == Enum.KeyCode.Backspace then
            U.boosterRebinding = false
            _G.XKPubSetBoosterKeybind(Enum.KeyCode.Unknown)
            U.boosterKeyButton.Text = "NONE"
            U.boosterKeyButton.BackgroundColor3 = UI_OFF
            return
        end
        if input.KeyCode ~= Enum.KeyCode.Unknown then
            _G.XKPubSetBoosterKeybind(input.KeyCode)
            U.boosterRebinding = false
            U.boosterKeyButton.Text = input.KeyCode.Name
            U.boosterKeyButton.BackgroundColor3 = UI_PURPLE_DARK
            return
        end
    end
end)

-- SPEED BOOSTER KEY ACTIVATION (separate from Carpet Speed)
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.UserInputType ~= Enum.UserInputType.Keyboard then return end
    if U.rebinding or U.boosterRebinding then return end
    local bound = _G.KEYBIND_BOOSTER or KEYBIND_BOOSTER
    if bound == Enum.KeyCode.Unknown then return end
    if input.KeyCode == bound and _G.XKPubToggleSpeedBoosterKey then
        local state = _G.XKPubToggleSpeedBoosterKey()
        if U.speedBtn then
            U.speedBtn.Text = state and "ATIVADO" or "DESLIGADO"
        end
    end
end)

-- ------------------------------------------------------------
-- SELECT GEAR SPEED
-- ------------------------------------------------------------
U.gearSelect = Instance.new("Frame")
U.gearSelect.Name = "GearSelectWindow"
U.gearSelect.Size = UDim2.fromOffset(200,250)
U.gearSelect.ZIndex = 520
U.gearSelect.Visible = false
U.gearSelect.Parent = screenGui
U.gearSelect.AnchorPoint = Vector2.new(0.5, 0.5)
U.gearSelect.Position = UDim2.new(0.5, 0, 0.5, 0)
stylePanel2(U.gearSelect,15)

local gst = Instance.new("TextLabel")
gst.Size = UDim2.new(1,-40,0,30)
gst.Position = UDim2.fromOffset(12,0)
gst.BackgroundTransparency = 1
gst.Font = FONT_MAIN
gst.Text = "SELECT GEAR SPEED"
gst.TextColor3 = UI_WHITE
gst.TextSize = 11
gst.ZIndex = 521
gst.Parent = U.gearSelect

local gsc = Instance.new("TextButton")
gsc.Size = UDim2.fromOffset(22,22)
gsc.Position = UDim2.new(1,-30,0,7)
gsc.BackgroundColor3 = UI_PURPLE_DARK
gsc.AutoButtonColor = false
gsc.Font = FONT_MAIN
gsc.Text = "X"
gsc.TextColor3 = UI_WHITE
gsc.TextSize = 9
gsc.ZIndex = 522
gsc.Parent = U.gearSelect
local gscc = Instance.new("UICorner")
gscc.CornerRadius = UDim.new(0,7)
gscc.Parent = gsc

local gssub = Instance.new("TextLabel")
gssub.Size = UDim2.new(1,-24,0,20)
gssub.Position = UDim2.fromOffset(12,33)
gssub.BackgroundTransparency = 1
gssub.Font = FONT_REGULAR
gssub.Text = "Choose the gear used by Carpet Speed"
gssub.TextColor3 = UI_MUTED
gssub.TextSize = 7
gssub.ZIndex = 521
gssub.Parent = U.gearSelect

U.gearList = Instance.new("ScrollingFrame")
U.gearList.Size = UDim2.new(1,-18,1,-67)
U.gearList.Position = UDim2.fromOffset(9,58)
U.gearList.BackgroundTransparency = 1
U.gearList.BorderSizePixel = 0
U.gearList.ScrollBarThickness = 2
U.gearList.ScrollBarImageColor3 = UI_PURPLE
U.gearList.ZIndex = 521
U.gearList.Parent = U.gearSelect

U.gearLayout = Instance.new("UIListLayout")
U.gearLayout.Padding = UDim.new(0,5)
U.gearLayout.SortOrder = Enum.SortOrder.LayoutOrder
U.gearLayout.Parent = U.gearList

local function chooseGear2(name)
    selectedGearItem = name
    persistFeature("selectedGearItem", name)
    updateKeyUI()
    if speedEnabled then
        task.defer(function()
            equipSelectedTool()
        end)
    end
    if speedEnabled then
        speedEnabled = false
        stopEquipCheck()
        stopVelocity()
        task.wait(0.05)
        speedEnabled = true
        startEquipCheck()
        applyVelocity()
        if carpetSpeedToggleRef then
            carpetSpeedToggleRef.BackgroundColor3 = UI_PURPLE_DARK
            carpetSpeedToggleRef.TextColor3 = UI_WHITE
            carpetSpeedToggleRef.Text = "ON"
            local s = carpetSpeedToggleRef:FindFirstChildOfClass("UIStroke")
            if s then s.Color = UI_GREEN_BORDER end
        end
    end
end

function rebuildGearChoices()
    for _,child in ipairs(U.gearList:GetChildren()) do
        if child:IsA("TextButton") then child:Destroy() end
    end

    for i,name in ipairs(GEAR_ITEMS) do
        local b = Instance.new("TextButton")
        b.Size = UDim2.new(1,-4,0,37)
        b.BackgroundColor3 = (name == selectedGearItem) and UI_PURPLE_DARK or UI_OFF
        b.AutoButtonColor = false
        b.Font = FONT_MAIN
        b.Text = (name == selectedGearItem) and ("✓  "..name) or name
        b.TextColor3 = UI_WHITE
        b.TextSize = 8
        b.LayoutOrder = i
        b.ZIndex = 522
        b.Parent = U.gearList

        local c = Instance.new("UICorner")
        c.CornerRadius = UDim.new(0,9)
        c.Parent = b

        local s = Instance.new("UIStroke")
        s.Color = (name == selectedGearItem) and UI_PURPLE_BORDER or Color3.fromRGB(53,54,65)
        s.Thickness = 1
        s.Parent = b

        b.MouseButton1Click:Connect(function()
            chooseGear2(name)
            U.gearSelect.Visible = false
        end)
    end

    U.gearList.CanvasSize = UDim2.fromOffset(0,U.gearLayout.AbsoluteContentSize.Y + 4)
end

gsc.MouseButton1Click:Connect(function()
    U.gearSelect.Visible = false
end)

U.gearSelectButton.MouseButton1Click:Connect(function()
    rebuildGearChoices()
    U.gearSelect.Visible = true
    U.keybind.Visible = false
end)

-- ------------------------------------------------------------
-- ANTI DIE RESET WARNING
-- Anti Die is controlled by Speed Booster and has no standalone UI.
-- ------------------------------------------------------------
function showAntiDieResetToast()
    -- Intentionally hidden: Anti Die is now linked to Speed Booster.
end

-- SERVER JOB ID
-- ------------------------------------------------------------
U.job = Instance.new("Frame")
U.job.Name = "ServerJobIdWindow"
U.job.Size = UDim2.fromOffset(190,90)
U.job.ZIndex = 300
U.job.Parent = screenGui
stylePanel2(U.job,12)

U.jobTitle = Instance.new("TextLabel")
U.jobTitle.Size = UDim2.new(1,-18,0,20)
U.jobTitle.Position = UDim2.fromOffset(9,5)
U.jobTitle.BackgroundTransparency = 1
U.jobTitle.Font = Enum.Font.GothamBlack
U.jobTitle.Text = "JOB ID"
U.jobTitle.TextColor3 = UI_WHITE
U.jobTitle.TextSize = 10
U.jobTitle.TextXAlignment = Enum.TextXAlignment.Left
U.jobTitle.ZIndex = 301
U.jobTitle.Parent = U.job

U.jobValue = Instance.new("TextLabel")
U.jobValue.Size = UDim2.new(1,-18,0,27)
U.jobValue.Position = UDim2.fromOffset(9,22)
U.jobValue.BackgroundTransparency = 1
U.jobValue.Font = FONT_REGULAR
U.jobValue.Text = tostring(game.JobId)
U.jobValue.TextColor3 = Color3.fromRGB(178, 190, 205)
U.jobValue.TextSize = 7
U.jobValue.TextXAlignment = Enum.TextXAlignment.Left
U.jobValue.TextTruncate = Enum.TextTruncate.AtEnd
U.jobValue.ZIndex = 302
U.jobValue.Parent = U.job

local jobCopy = Instance.new("TextButton")
jobCopy.Size = UDim2.fromOffset(84,23)
jobCopy.Position = UDim2.fromOffset(9,61)
jobCopy.BackgroundColor3 = Color3.fromRGB(255, 193, 7)
jobCopy.AutoButtonColor = false
jobCopy.Font = FONT_MAIN
jobCopy.Text = "Copy ID"
jobCopy.TextColor3 = Color3.fromRGB(255,255,255)
jobCopy.TextSize = 8
jobCopy.ZIndex = 303
jobCopy.Parent = U.job

local jcc = Instance.new("UICorner")
jcc.CornerRadius = UDim.new(0,8)
jcc.Parent = jobCopy

local jcs = Instance.new("UIStroke")
jcs.Color = Color3.fromRGB(255, 236, 100)
jcs.Thickness = 1
jcs.Transparency = 0.1
jcs.Parent = jobCopy

local jobRejoin = Instance.new("TextButton")
jobRejoin.Size = UDim2.fromOffset(84,23)
jobRejoin.Position = UDim2.fromOffset(97,61)
jobRejoin.BackgroundColor3 = Color3.fromRGB(255, 193, 7)
jobRejoin.AutoButtonColor = false
jobRejoin.Font = FONT_MAIN
jobRejoin.Text = "Rejoin"
jobRejoin.TextColor3 = Color3.fromRGB(255,255,255)
jobRejoin.TextSize = 8
jobRejoin.ZIndex = 303
jobRejoin.Parent = U.job

local jrc = Instance.new("UICorner")
jrc.CornerRadius = UDim.new(0,8)
jrc.Parent = jobRejoin

local jrs = Instance.new("UIStroke")
jrs.Color = Color3.fromRGB(255, 236, 100)
jrs.Thickness = 1
jrs.Transparency = 0.1
jrs.Parent = jobRejoin

jobCopy.MouseButton1Click:Connect(function()
    if setclipboard then
        pcall(function() setclipboard(game.JobId) end)
        jobCopy.Text = "OK"
        task.delay(1, function()
            if jobCopy and jobCopy.Parent then
                jobCopy.Text = "Copy ID"
            end
        end)
    end
end)

jobRejoin.MouseButton1Click:Connect(function()
    pcall(function()
        TeleportService:TeleportToPlaceInstance(game.PlaceId, game.JobId, LocalPlayer)
    end)
end)

makeDraggableAndPersistent(U.job,U.jobTitle,"ServerJobIdWindow",UDim2.new(0,12,0,12))

-- ------------------------------------------------------------
-- NEW FEATURE TOAST
-- ------------------------------------------------------------
U.toast = Instance.new("Frame")
U.toast.Name = "NewFeatureToast"
U.toast.Size = UDim2.fromOffset(250,68)
U.toast.AnchorPoint = Vector2.new(0.5,0)
U.toast.Position = UDim2.fromScale(0.5,0.06)
U.toast.BackgroundColor3 = Color3.fromRGB(5,5,8)
U.toast.Visible = false
U.toast.ZIndex = 900
U.toast.Parent = screenGui

local tc = Instance.new("UICorner")
tc.CornerRadius = UDim.new(0,12)
tc.Parent = U.toast
local ts = Instance.new("UIStroke")
ts.Color = UI_PURPLE_BORDER
ts.Thickness = 1.2
ts.Parent = U.toast

local tt = Instance.new("TextLabel")
tt.Size = UDim2.new(1,0,0,27)
tt.BackgroundTransparency = 1
tt.Font = FONT_MAIN
tt.Text = "NEW FEATURE  •  SOON"
tt.TextColor3 = UI_WHITE
tt.TextSize = 11
tt.ZIndex = 901
tt.Parent = U.toast

local tsub = Instance.new("TextLabel")
tsub.Size = UDim2.new(1,0,0,18)
tsub.Position = UDim2.fromOffset(0,28)
tsub.BackgroundTransparency = 1
tsub.Font = FONT_REGULAR
tsub.Text = "NOT READY YET  •  CHECK BACK SOON"
tsub.TextColor3 = Color3.fromRGB(185, 195, 208)
tsub.TextSize = 8
tsub.ZIndex = 901
tsub.Parent = U.toast

local toastToken = 0
local SOON_SOUND_ID = "rbxassetid://8904888220"

local function playSoonSound()
    local sound = Instance.new("Sound")
    sound.Name = "SammySoonSound"
    sound.SoundId = SOON_SOUND_ID
    sound.Volume = 1.5
    sound.Parent = SoundService
    task.spawn(function()
        pcall(function()
            sound:Play()
        end)
        task.delay(4, function()
            pcall(function()
                sound:Destroy()
            end)
        end)
    end)
end

function showNotReadyToast()
    pcall(playSoonSound)
    toastToken += 1
    local token = toastToken
    U.toast.Visible = true
    task.delay(2,function()
        if token == toastToken and U.toast.Parent then
            U.toast.Visible = false
        end
    end)
end

U.newFeatureBtn.MouseButton1Click:Connect(showNotReadyToast)

-- ------------------------------------------------------------
-- PLAYER JOIN / LEAVE NOTIFICATION
-- ------------------------------------------------------------
U.playerNotify = Instance.new("Frame")
U.playerNotify.Name = "PlayerJoinNotification"
U.playerNotify.Size = UDim2.fromOffset(330,58)
U.playerNotify.AnchorPoint = Vector2.new(0.5,0)
U.playerNotify.Position = UDim2.fromScale(0.5,0.035)
U.playerNotify.BackgroundColor3 = Color3.fromRGB(12,16,22)
U.playerNotify.BackgroundTransparency = 0.04
U.playerNotify.Visible = false
U.playerNotify.ZIndex = 950
U.playerNotify.Parent = screenGui

local pnc = Instance.new("UICorner")
pnc.CornerRadius = UDim.new(0,14)
pnc.Parent = U.playerNotify
local pns = Instance.new("UIStroke")
pns.Color = UI_PURPLE_BORDER
pns.Thickness = 1.2
pns.Transparency = 0.08
pns.Parent = U.playerNotify

local pAvatar = Instance.new("ImageLabel")
pAvatar.Size = UDim2.fromOffset(42,42)
pAvatar.Position = UDim2.fromOffset(9,8)
pAvatar.BackgroundTransparency = 1
pAvatar.ZIndex = 951
pAvatar.Parent = U.playerNotify
local pavc = Instance.new("UICorner")
pavc.CornerRadius = UDim.new(1,0)
pavc.Parent = pAvatar

local pTitle = Instance.new("TextLabel")
pTitle.Size = UDim2.new(1,-65,0,20)
pTitle.Position = UDim2.fromOffset(59,8)
pTitle.BackgroundTransparency = 1
pTitle.Font = FONT_MAIN
pTitle.Text = "PLAYER JOINED"
pTitle.TextColor3 = UI_WHITE
pTitle.TextSize = 9
pTitle.TextXAlignment = Enum.TextXAlignment.Left
pTitle.ZIndex = 951
pTitle.Parent = U.playerNotify

local pName = Instance.new("TextLabel")
pName.Size = UDim2.new(1,-70,0,19)
pName.Position = UDim2.fromOffset(59,28)
pName.BackgroundTransparency = 1
pName.Font = FONT_REGULAR
pName.TextColor3 = Color3.fromRGB(178,190,205)
pName.TextSize = 8
pName.TextXAlignment = Enum.TextXAlignment.Left
pName.TextTruncate = Enum.TextTruncate.AtEnd
pName.ZIndex = 951
pName.Parent = U.playerNotify

local playerNotifyToken = 0
local function showPlayerNotify(player, joined)
    if not player or player == LocalPlayer then return end
    playerNotifyToken += 1
    local token = playerNotifyToken
    pTitle.Text = joined and "PLAYER JOINED" or "PLAYER LEFT"
    pName.Text = "@" .. tostring(player.Name)
    pcall(function()
        pAvatar.Image = Players:GetUserThumbnailAsync(
            player.UserId,
            Enum.ThumbnailType.HeadShot,
            Enum.ThumbnailSize.Size100x100
        )
    end)
    U.playerNotify.Visible = true
    pcall(playNotifySound)
    task.delay(3.2,function()
        if token == playerNotifyToken and U.playerNotify.Parent then
            U.playerNotify.Visible = false
        end
    end)
end

Players.PlayerAdded:Connect(function(player)
    showPlayerNotify(player,true)
end)
Players.PlayerRemoving:Connect(function(player)
    showPlayerNotify(player,false)
end)

-- ------------------------------------------------------------
-- TOP STATUS BAR — SON HUB / FPS / PING
-- ------------------------------------------------------------
U.statusBar = Instance.new("Frame")
U.statusBar.Name = "SonHubStatusBar"
U.statusBar.Size = UDim2.fromOffset(248, 34)
U.statusBar.Position = UDim2.new(0.5, -124, 0, 8)
U.statusBar.BackgroundColor3 = UI_PANEL
U.statusBar.BackgroundTransparency = 0.04
U.statusBar.BorderSizePixel = 0
U.statusBar.Active = true
U.statusBar.ZIndex = 950
U.statusBar.Parent = screenGui

local sonCorner = Instance.new("UICorner")
sonCorner.CornerRadius = UDim.new(0, 10)
sonCorner.Parent = U.statusBar

local sonStroke = Instance.new("UIStroke")
sonStroke.Color = UI_PURPLE_BORDER
sonStroke.Thickness = 1.1
sonStroke.Transparency = 0.05
sonStroke.Parent = U.statusBar

U.sonHubTitle = Instance.new("TextLabel")
U.sonHubTitle.Size = UDim2.fromOffset(80, 34)
U.sonHubTitle.Position = UDim2.fromOffset(10, 0)
U.sonHubTitle.BackgroundTransparency = 1
U.sonHubTitle.Font = FONT_MAIN
U.sonHubTitle.Text = "SPONGEBOB HUB"
U.sonHubTitle.TextColor3 = UI_WHITE
U.sonHubTitle.TextSize = 10
U.sonHubTitle.TextXAlignment = Enum.TextXAlignment.Left
U.sonHubTitle.ZIndex = 951
U.sonHubTitle.Parent = U.statusBar

U.sonHubStats = Instance.new("TextLabel")
U.sonHubStats.Size = UDim2.new(1, -122, 1, 0)
U.sonHubStats.Position = UDim2.fromOffset(88, 0)
U.sonHubStats.BackgroundTransparency = 1
U.sonHubStats.Font = FONT_REGULAR
U.sonHubStats.Text = "FPS --   •   PING --"
U.sonHubStats.TextColor3 = UI_WHITE
U.sonHubStats.TextSize = 8
U.sonHubStats.TextXAlignment = Enum.TextXAlignment.Right
U.sonHubStats.ZIndex = 951
U.sonHubStats.Parent = U.statusBar

-- Compact hub minimize button: keeps the status bar visible and collapses
-- the main control windows without touching their existing logic.
U.minimizeButton = Instance.new("TextButton")
U.minimizeButton.Name = "MinimizeButton"
U.minimizeButton.Size = UDim2.fromOffset(20,20)
U.minimizeButton.Position = UDim2.new(1,-25,0,7)
U.minimizeButton.BackgroundColor3 = UI_ROW
U.minimizeButton.BackgroundTransparency = 0.08
U.minimizeButton.BorderSizePixel = 0
U.minimizeButton.AutoButtonColor = false
U.minimizeButton.Font = Enum.Font.GothamBlack
U.minimizeButton.Text = "-"
U.minimizeButton.TextColor3 = UI_WHITE
U.minimizeButton.TextSize = 12
U.minimizeButton.ZIndex = 955
U.minimizeButton.Parent = U.statusBar
local minCorner = Instance.new("UICorner")
minCorner.CornerRadius = UDim.new(0,6)
minCorner.Parent = U.minimizeButton
local minStroke = Instance.new("UIStroke")
minStroke.Color = UI_PURPLE_BORDER
minStroke.Thickness = 1
minStroke.Transparency = 0.2
minStroke.Parent = U.minimizeButton

local hubMinimized = false
local previousKeybindVisible = false
local previousGearSelectVisible = false
local function setHubMinimized(state)
    hubMinimized = state == true
    local visible = not hubMinimized
    if U.gear then U.gear.Visible = visible end
    if U.target then U.target.Visible = visible end
    if U.job then U.job.Visible = visible end
    if hubMinimized then
        previousKeybindVisible = U.keybind and U.keybind.Visible or false
        previousGearSelectVisible = U.gearSelect and U.gearSelect.Visible or false
        if U.keybind then U.keybind.Visible = false end
        if U.gearSelect then U.gearSelect.Visible = false end
    else
        if U.keybind then U.keybind.Visible = previousKeybindVisible end
        if U.gearSelect then U.gearSelect.Visible = previousGearSelectVisible end
    end
    U.minimizeButton.Text = hubMinimized and "+" or "-"
end

U.minimizeButton.MouseButton1Click:Connect(function()
    setHubMinimized(not hubMinimized)
end)

-- SpongeBob visual treatment for every hub window.
addSpongeTexture(U.gear, 0.78, U.gear.ZIndex)
addSpongeTexture(U.target, 0.78, U.target.ZIndex)
addSpongeTexture(U.job, 0.78, U.job.ZIndex)
addSpongeTexture(U.keybind, 0.80, U.keybind.ZIndex)
addSpongeTexture(U.gearSelect, 0.80, U.gearSelect.ZIndex)

-- ------------------------------------------------------------
-- MINI BUTTONS FOR EVERY HUB WINDOW
-- ------------------------------------------------------------
local panelMinimizeStates = {}
local panelMinimizeMeta = {}

local function addHubPanelMinimize(panel, titleObject, expandedSize)
    if not panel or panelMinimizeMeta[panel] then return end

    local button = Instance.new("TextButton")
    button.Name = "PanelMinimizeButton"
    button.Size = UDim2.fromOffset(20,20)
    button.Position = UDim2.new(1,-27,0,7)
    button.BackgroundColor3 = UI_ROW
    button.BackgroundTransparency = 0.05
    button.BorderSizePixel = 0
    button.AutoButtonColor = false
    button.Font = Enum.Font.GothamBlack
    button.Text = "-"
    button.TextColor3 = UI_WHITE
    button.TextSize = 11
    button.ZIndex = (panel.ZIndex or 100) + 20
    button.Parent = panel

    local c = Instance.new("UICorner")
    c.CornerRadius = UDim.new(0,6)
    c.Parent = button
    local st = Instance.new("UIStroke")
    st.Color = UI_PURPLE_BORDER
    st.Thickness = 1
    st.Transparency = 0.15
    st.Parent = button

    local states = {}
    for _, child in ipairs(panel:GetChildren()) do
        if child ~= button and child ~= titleObject and not child:IsA("UIStroke") and not child:IsA("UICorner") then
            states[child] = child.Visible
        end
    end

    panelMinimizeMeta[panel] = {button = button, title = titleObject, expandedSize = expandedSize, states = states}

    button.MouseButton1Click:Connect(function()
        local meta = panelMinimizeMeta[panel]
        if not meta then return end
        local minimized = panelMinimizeStates[panel] == true
        minimized = not minimized
        panelMinimizeStates[panel] = minimized

        if minimized then
            for child in pairs(meta.states) do
                if child and child.Parent == panel then
                    child.Visible = false
                end
            end
            panel.Size = UDim2.fromOffset(expandedSize.X.Offset, 34)
            button.Text = "+"
        else
            for child, wasVisible in pairs(meta.states) do
                if child and child.Parent == panel then
                    child.Visible = wasVisible
                end
            end
            panel.Size = expandedSize
            button.Text = "-"
        end
    end)

    return button
end

-- Each independent hub/window gets its own small +/- control.
addHubPanelMinimize(U.gear, U.gearTitle, UDim2.fromOffset(190,300))
addHubPanelMinimize(U.target, U.targetTitle, UDim2.fromOffset(190,190))
addHubPanelMinimize(U.job, U.jobTitle, UDim2.fromOffset(190,90))
addHubPanelMinimize(U.keybind, U.keyTitle, UDim2.fromOffset(220,245))
addHubPanelMinimize(U.gearSelect, gst, UDim2.fromOffset(200,250))

local sonHubStatConnection = RunService.RenderStepped:Connect(function()
    if not U.statusBar or not U.statusBar.Parent then return end
    local fpsNow = tonumber(fpsValue) or 0
    local pingNow = "--"
    pcall(function()
        pingNow = getPingText()
    end)
    U.sonHubStats.Text = "FPS " .. tostring(fpsNow) .. "   •   PING " .. tostring(pingNow)
end)
connections[#connections + 1] = sonHubStatConnection
U.sonHubStatConnection = sonHubStatConnection

makeDraggableAndPersistent(
    U.statusBar,
    U.statusBar,
    "SonHubStatusBar",
    UDim2.new(0.5, -124, 0, 8)
)

-- ------------------------------------------------------------
-- STATUS BAR UI
-- ------------------------------------------------------------
-- Final placement / defaults
U.keybind.Visible = false
U.gearSelect.Visible = false
pcall(rebuildGearChoices)
pcall(function()
    newSlotESP()
    buildPlatforms()
end)
end

buildUI()
end

-- ============================================================
-- XKPUB INITIAL FEATURE STATE
-- ============================================================
if autoGrabEnabled then task.defer(function() pcall(openGrabInsta, true) end) end
if speedBoosterEnabled then
    task.defer(function() pcall(setSpeedBooster, true) end)
else
    antiDieEnabled = false
    _G.AntiDieDisabled = true
end
if baseESPEnabled then task.defer(rebuildBaseESP) end

-- ============================================================
-- CLEANUP
-- ============================================================

_G.__SammyApexCleanup = function()
    if unwalkConnection then pcall(function() unwalkConnection:Disconnect() end) unwalkConnection = nil end
    if unwalkEnabled then pcall(function() removeUnwalk(LocalPlayer.Character) end) end
    for _, c in ipairs(connections) do pcall(function() c:Disconnect() end) end
    table.clear(connections)
    if nextBaseProjectConnection then pcall(function() nextBaseProjectConnection:Disconnect() end) end
    if _podiumCleanup then pcall(_podiumCleanup) end
    stopEquipCheck()
    stopVelocity()
    antiRagdollCleanup()
    clearPlayerESP()
    if espHolder then pcall(function() espHolder:Destroy() end) end
    if anchor then pcall(function() anchor:Destroy() end) end
    if targetGui then pcall(function() targetGui:Destroy() end) end
    if fullGui then pcall(function() fullGui:Destroy() end) end
    if warningGui then pcall(function() warningGui:Destroy() end) end
    if joinNotifyGui then pcall(function() joinNotifyGui:Destroy() end) end
    if _G.U and U.sonHubStatConnection then pcall(function() U.sonHubStatConnection:Disconnect() end) end
    if _G.U and _G.U.statusBar then pcall(function() U.statusBar:Destroy() end) end
    if _G.U and _G.U.keybind then pcall(function() U.keybind:Destroy() end) end
    if _G.U and _G.U.gearSelect then pcall(function() U.gearSelect:Destroy() end) end
    if _G.U and _G.U.gear then pcall(function() U.gear:Destroy() end) end
    if _G.U and _G.U.target then pcall(function() U.target:Destroy() end) end
    if _G.__SaveNeptuneAutoGrabPosition then pcall(_G.__SaveNeptuneAutoGrabPosition) end
    if _G.U and _G.U.job then pcall(function() U.job:Destroy() end) end
    if screenGui then pcall(function() screenGui:Destroy() end) end
    if fpsBoostConnection then pcall(function() fpsBoostConnection:Disconnect() end) fpsBoostConnection = nil end
    if statusConnection then pcall(function() statusConnection:Disconnect() end) end
    if fpsConnection then pcall(function() fpsConnection:Disconnect() end) end
    _G.__SammyApexCleanup = nil
end

-- ============================================================
-- INICIALIZAÇÃO
-- ============================================================

pcall(function() newSlotESP() end)
pcall(function() buildPlatforms() end)

if unwalkEnabled and LocalPlayer.Character then
    task.spawn(function()
        task.wait(0.3)
        if LocalPlayer.Character then applyUnwalk(LocalPlayer.Character) end
    end)
end

print("[Sammy Apex] Integração concluída com sucesso!")
