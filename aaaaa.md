--[[
    ═══════════════════════════════════════════════════════════════
    UZU HUB - AUTO PERFECT BLOCK V4.1 LINORIA
    Grand Piece Online - Smart Combat Detection System
    ═══════════════════════════════════════════════════════════════
    
    V4.1 LINORIA FEATURES:
    - LinoriaLib UI (modern and clean interface)
    - Smart combat state detection
    - Player combo awareness (won't block during your combo)
    - Customizable action keys
    - Block delay for legitimacy
    - On-screen status display
    - Unknown animation recorder with AUTO NAME EXTRACTION
    - Auto-block logging system
    - Fixed cooldown system
    - Proper F key pressing method
    - Animation player with stop functionality
    - Theme Manager & Save Manager support
    
    Author: UzuDev
    Version: 4.1 Linoria
]]

-- ════════════════════════════════════════════════════════════════
-- COMPATIBILITY CHECK
-- ════════════════════════════════════════════════════════════════


pcall(checkGame)

-- ════════════════════════════════════════════════════════════════
-- SERVICES
-- ════════════════════════════════════════════════════════════════

local Players = game:GetService("Players")
local Workspace = game:GetService("Workspace")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer

-- ════════════════════════════════════════════════════════════════
-- LINORIA UI LIBRARY
-- ════════════════════════════════════════════════════════════════

local repo = 'https://raw.githubusercontent.com/violin-suzutsuki/LinoriaLib/main/'

local Library = loadstring(game:HttpGet(repo .. 'Library.lua'))()
local ThemeManager = loadstring(game:HttpGet(repo .. 'addons/ThemeManager.lua'))()
local SaveManager = loadstring(game:HttpGet(repo .. 'addons/SaveManager.lua'))()

local Window = Library:CreateWindow({
    Title = 'UZU HUB GOON ON UI',
    Center = true,
    AutoShow = true,
    TabPadding = 8,
    MenuFadeTime = 0.2
})

-- ════════════════════════════════════════════════════════════════
-- FILE SYSTEM SETUP
-- ════════════════════════════════════════════════════════════════

local folderName = "uzu auto pb"
local unknownAnimFile = folderName .. "/unknown_animations.txt"
local customAnimFile = folderName .. "/custom_animations.txt"

if not isfolder(folderName) then
    makefolder(folderName)
end

if not isfile(unknownAnimFile) then
    writefile(unknownAnimFile, "-- Unknown Animations Log --\n-- Format: AnimationID | Timestamp | Target Name | Animation Name (Auto-Detected)\n\n")
end

if not isfile(customAnimFile) then
    writefile(customAnimFile, "-- Custom Animations --\n-- Format: AnimationID | Timing | Name (optional)\n-- Example: 1234567890 | 0.2 | My Custom Attack\n\n")
end

-- ════════════════════════════════════════════════════════════════
-- ON-SCREEN STATUS DISPLAY
-- ════════════════════════════════════════════════════════════════

local StatusGui = Instance.new("ScreenGui")
StatusGui.Name = "UzuHubStatus"
StatusGui.ResetOnSpawn = false
StatusGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling

local StatusLabel = Instance.new("TextLabel")
StatusLabel.Name = "StatusLabel"
StatusLabel.Parent = StatusGui
StatusLabel.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
StatusLabel.BackgroundTransparency = 0.5
StatusLabel.BorderSizePixel = 0
StatusLabel.Position = UDim2.new(0.5, -150, 0, 60)
StatusLabel.Size = UDim2.new(0, 300, 0, 40)
StatusLabel.Font = Enum.Font.GothamBold
StatusLabel.Text = "Status: Idle"
StatusLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
StatusLabel.TextSize = 18
StatusLabel.TextStrokeTransparency = 0.5

local Corner = Instance.new("UICorner")
Corner.CornerRadius = UDim.new(0, 8)
Corner.Parent = StatusLabel

local function safeParent()
    local success, err = pcall(function()
        if game:GetService("CoreGui") then
            StatusGui.Parent = game:GetService("CoreGui")
        else
            StatusGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
        end
    end)
    
    if not success then
        StatusGui.Parent = LocalPlayer:WaitForChild("PlayerGui")
    end
end

safeParent()

local function updateStatusDisplay(text, color)
    StatusLabel.Text = "Status: " .. text
    StatusLabel.TextColor3 = color or Color3.fromRGB(255, 255, 255)
end

-- ════════════════════════════════════════════════════════════════
-- TABS
-- ════════════════════════════════════════════════════════════════

local Tabs = {
    Main = Window:AddTab('Main'),
    Anims = Window:AddTab('Animations'),
    ['UI Settings'] = Window:AddTab('UI Settings'),
}

-- ════════════════════════════════════════════════════════════════
-- VARIABLES
-- ════════════════════════════════════════════════════════════════

local PerfectBlockSettings = {
    Enabled = false,
    Keybind = Enum.KeyCode.P,
    BlockPlayers = true,
    MaxDistance = 150,
    BlockDelay = 0,
    
    -- Prediction
    PredictionEnabled = false,
    
    -- Combat Detection
    CombatKeys = {
        [Enum.UserInputType.MouseButton1] = false,
        [Enum.KeyCode.E] = false,
        [Enum.KeyCode.R] = false,
        [Enum.KeyCode.T] = false,
        [Enum.KeyCode.Q] = false,
        [Enum.KeyCode.Z] = false,
        [Enum.KeyCode.X] = false,
        [Enum.KeyCode.C] = false,
        [Enum.KeyCode.V] = false,
    },
    
    -- Recorder
    RecorderEnabled = true,
    RecorderRadius = 150,
    BlockUnknown = true,
    RecorderCustomName = "",
}

local CombatState = {
    PlayerInCombo = false,
    LastActionTime = 0,
    ComboCooldown = 0.5,
    CurrentStatus = "Idle",
    ActiveKeys = {},
    LastKeyAllowedBlocking = nil,
}

-- ════════════════════════════════════════════════════════════════
-- LOGGING SYSTEM WITH NOTIFICATION QUEUE
-- ════════════════════════════════════════════════════════════════

local BlockLogger = {
    TotalBlocks = 0,
    PlayerBlocks = 0,
    PredictedBlocks = 0,
}

-- Notification queue system
local NotificationQueue = {
    queue = {},
    maxNotifications = 7,
    isProcessing = false,
    notificationCooldown = 0.5, -- Minimum time between notifications
    lastNotificationTime = 0,
}

function NotificationQueue:Add(message, duration)
    -- Limit queue size
    if #self.queue >= self.maxNotifications then
        return -- Don't add if queue is full
    end
    
    -- Check if similar notification already in queue (prevent duplicates)
    for _, notif in ipairs(self.queue) do
        if notif.message == message then
            return -- Already queued
        end
    end
    
    table.insert(self.queue, {
        message = message,
        duration = duration or 1
    })
    
    if not self.isProcessing then
        self:Process()
    end
end

function NotificationQueue:Process()
    self.isProcessing = true
    
    task.spawn(function()
        while #self.queue > 0 do
            local currentTime = tick()
            
            -- Wait for cooldown
            if currentTime - self.lastNotificationTime < self.notificationCooldown then
                task.wait(self.notificationCooldown - (currentTime - self.lastNotificationTime))
            end
            
            local notif = table.remove(self.queue, 1)
            if notif then
                Library:Notify(notif.message, notif.duration)
                self.lastNotificationTime = tick()
            end
            
            if Library.Unloaded then break end
        end
        
        self.isProcessing = false
    end)
end

local function logBlock(targetName, animationId, wasPredicted)
    BlockLogger.TotalBlocks = BlockLogger.TotalBlocks + 1
    BlockLogger.PlayerBlocks = BlockLogger.PlayerBlocks + 1
    
    if wasPredicted then
        BlockLogger.PredictedBlocks = BlockLogger.PredictedBlocks + 1
    end
    
    local emoji = "🛡️"
    local message = string.format("%s %s | AnimID: %s", emoji, targetName or "Unknown", animationId or "Unknown")
    
    if wasPredicted then
        message = message .. " (Predicted)"
    end
    
    -- Use notification queue instead of direct notify
    NotificationQueue:Add(message, 0.5)
end

-- ════════════════════════════════════════════════════════════════
-- IGNORED ANIMATIONS
-- ════════════════════════════════════════════════════════════════

local IgnoredAnimations = {
    ['102847582739519'] = true,
    ['9710431811'] = true,
    ['9703995286'] = true,
    ['4910485611'] = true,
    ['9711831861'] = true,
    ['134877403213295'] = true,
    ['18841102170'] = true,
    ['18841081185'] = true,
    ['106763696159860'] = true,
    ['92811333533670'] = true,
    ['78584318919493'] = true,
    ['118247933649869'] = true,
    ['138848037207334'] = true,
    ['98963988224403'] = true,
    ['9712102429'] = true,
    ['98166532936064'] = true,
    ['7584947295'] = true,
    ['18841080472'] = true,
    ['2942644324'] = true,
    ['2942643830'] = true,
    ['2942641670'] = true,
    ['5392930263'] = true,
    ['5392869763'] = true,
    ['6032355961'] = true,
    ['6032356414'] = true,
    ['6028142920'] = true,
    ['6026084898'] = true,
    ['6026085284'] = true,
    ['6026082598'] = true,
    ['11838527249'] = true,
    ['11838531612'] = true,
    ['11838526480'] = true,
    ['4910406979'] = true,
    ['3044991033'] = true,
    ['13243427337'] = true,
    ['129305042300099'] = true,
    ['74744607717391'] = true,
    ['82497770045941'] = true,
    ['11838527981'] = true,
    ['119334509242358'] = true,
    ['11838529559'] = true,
    ['11838530329'] = true,
    ['11838528512'] = true,
    ['13243423773'] = true,
    ['4899959433'] = true,
    ['99564327193459'] = true,
    ['90004694910626'] = true,
    ['4907577925'] = true,
    ['14986407007'] = true,
    ['3027864591'] = true,
    ['3027717390'] = true,
    ['3027719314'] = true,
    ['15059163245'] = true,
    ['15374681990'] = true,
    ['15059161952'] = true,
    ['72729463849772'] = true,
    ['15382115992'] = true,
    ['15382065457'] = true,
    ['13630769186'] = true,
    ['17650838050'] = true,
    ['5517298834'] = true,
    ['11093531300'] = true,
    ['6043954920'] = true,
    ['7075728341'] = true,
    ['10001705684'] = true,
    ['507765644'] = true,
    ['4563261864'] = true,
    ['2095054253'] = true,
    ['9984793787'] = true,
    ['5796457289'] = true,
    ['4126956669'] = true,
    ['5796460384'] = true,
    ['10001707271'] = true,
    ['507766388'] = true,
    ['507766666'] = true,
    ['507766951'] = true,
    ['507767234'] = true,
    ['507767714'] = true,
    ['913376220'] = true,
    ['913402848'] = true,
    ['913403323'] = true,
    ['913403938'] = true,
    ['913384386'] = true,
    ['10921082554'] = true,
    ['10921083856'] = true,
    ['507784897'] = true,
    ['507785072'] = true,
    ['507765000'] = true,
    ['507767968'] = true,
    ['507768133'] = true,
    ['507768375'] = true,
    ['507768851'] = true,
    ['3333499508'] = true,
    ['3333497031'] = true,
    ['4841397952'] = true,
    ['3695333486'] = true,
    ['3695335779'] = true,
    ['3333136415'] = true,
    ['4049037604'] = true,
    ['3337966527'] = true,
    ['3360686498'] = true,
    ['3576686446'] = true,
    ['3576968026'] = true,
    ['10921127235'] = true,
    ['3541114300'] = true,
    ['3541111181'] = true,
}

-- ════════════════════════════════════════════════════════════════
-- COMBAT ANIMATIONS DATABASE WITH TIMINGS
-- ════════════════════════════════════════════════════════════════

local AnimationTimings = {
    -- PUNCHES
    ['4087684389'] = 0,
    ['4087685071'] = 0,
    ['3993561070'] = 0,
    ['3027112133'] = 0,
    ['3993564465'] = 0,
    ['3993593264'] = 0,
    ['3993592261'] = 0,
    ['4048678023'] = 0,
    ['3993590202'] = 0,
    ['4810797117'] = 0,
    ['4563258318'] = 0,
    ['10970023685'] = 0,
    ['10970024047'] = 0,
    ['10970024392'] = 0,
    ['10970024769'] = 0,
    ['10970025138'] = 0,
    ['10970025632'] = 0,
    ['10970026107'] = 0,
    ['4610052185'] = 0,
    
    -- HEAVY
    ['4158016136'] = 0.1,
    ['4158017967'] = 0.1,
    
    -- KATANA/SWORDS
    ['10620309597'] = 0,
    ['10620310103'] = 0,
    ['10620310578'] = 0,
    ['10620311262'] = 0,
    ['10620311612'] = 0,
    ['11315312194'] = 0,
    ['11315312585'] = 0,
    ['11315312964'] = 0,
    ['11315313251'] = 0,
    ['11315313647'] = 0,
    ['11382679819'] = 0,
    ['11382680257'] = 0,
    ['11382680700'] = 0,
    ['11382681299'] = 0,
    ['11382681645'] = 0,
    ['11382682075'] = 0,
    ['11382682398'] = 0,
    
    -- SLASHES
    ['11838525488'] = 0,
    ['11838524829'] = 0,
    ['128995825208007'] = 0.2,
    ['121388817190480'] = 0.1,
    ['111488507714748'] = 0,
    ['112308753483647'] = 0,
    ['97426652978104'] = 0,
    ['115904308263891'] = 0,
    ['109907155268070'] = 0.1,
    ['93998517237288'] = 0,
    ['112862351309418'] = 0,
    ['70683948961049'] = 0.1,
    ['72539583003867'] = 0.2,
    ['4862846183'] = 0.5,
    ['4867467527'] = 0.1,
    ['9711777437'] = 0.5,
    ['11839454205'] = 0.2,
    ['77208200591918'] = 0,
    ['11838708179'] = 0.1,
    ['108522752957198'] = 0.2,
    ['139348436441539'] = 0.1,
    ['11838519181'] = 0,
    ['11838522300'] = 0,
    ['11838521789'] = 0,
    ['11838521234'] = 0,
    ['11838520277'] = 0,
    ['11838532486'] = 0,
    ['6028142451'] = 0.2,
    ['6026081621'] = 0,
    ['6026081303'] = 0.1,
    ['6026081955'] = 0,
    ['6026082936'] = 0.2,
    ['74939812314067'] = 0.1,
    ['108685213656196'] = 0.1,
    ['140420232174716'] = 0,
    ['134100362364915'] = 0.3,
    ['10970027277'] = 0.2,
    ['87153817279767'] = 0,
    ['86832118981204'] = 0,
    ['92320977675828'] = 0,
    ['137602132498103'] = 0,
    ['136537815800828'] = 0.1,
    ['11509753562'] = 0,
    ['11509753722'] = 0,
    ['11509754018'] = 0,
    ['11509754281'] = 0,
    ['11509754564'] = 0,
    ['11509754904'] = 0,
    ['15490770972'] = 0,
    ['15490767452'] = 0,
    
    -- SPECIALS
    ['10984727647'] = 0.2,
    ['11094732558'] = 0.2,
    ['11545972545'] = 0.2,
    ['15635213147'] = 0.2,
    ['10827636518'] = 0.2,
    ['12378039588'] = 0.2,
    ['11643324933'] = 0.2,
    ['11643325301'] = 0.2,
    ['120163859318804'] = 0.2,
    ['87695031726881'] = 0.2,
    ['5048955925'] = 0.2,
    
    -- KICKS
    ['110432084683680'] = 0,
    ['4563260213'] = 0,
    ['4760375217'] = 0,
    ['10519097075'] = 0,
    
    -- DEVIL FRUITS
    ['98540572072150'] = 0.1,
    ['87003860237022'] = 0.1,
    ['120123207439754'] = 0.2,
    ['4563257350'] = 0.3,
    ['5798506723'] = 0.3,
    ['5800509406'] = 0.3,
    ['4563256454'] = 0.3,
    ['4563257080'] = 0.3,
    ['102869887710887'] = 0.3,
    ['74395261231638'] = 0.3,
    ['4760307723'] = 0.3,
    ['11548109927'] = 0,
    ['11548110975'] = 0,
    ['12371016840'] = 0.2,
    ['12446605660'] = 0,
    ['88497352212383'] = 0,
    ['111758607864066'] = 0,
    
    -- FIGHTING STYLES
    ['5746701412'] = 0,
    ['5930373942'] = 0.1,
    ['5930374302'] = 0.1,
    
    -- CHARGE
    ['111860306703353'] = 0.2,
    ['111411037243512'] = 0.2,
    ['110371782727867'] = 0.2,
    ['106700313119224'] = 0.2,
    ['10492139931'] = 0.2,
    ['4563261003'] = 0.2,
    
    -- BANDIT BOSS COMBOS
    ['13243418555'] = 0,
    ['13243419833'] = 0,
    ['13243420683'] = 0,
    ['13243421318'] = 0,
    ['13243421985'] = 0,
    ['13243424357'] = 0,
    ['13243425294'] = 0,
    ['13243426159'] = 0,
    ['13243428011'] = 0,
}

-- ════════════════════════════════════════════════════════════════
-- LOAD CUSTOM ANIMATIONS
-- ════════════════════════════════════════════════════════════════

local function loadCustomAnimations()
    if not isfile(customAnimFile) then return 0 end
    
    local content = readfile(customAnimFile)
    local count = 0
    
    for line in content:gmatch("[^\r\n]+") do
        if not line:match("^%-%-") and line:match("%S") then
            local animId, timing, name = line:match("^(%d+)%s*|%s*([%d%.]+)%s*|?%s*(.*)$")
            
            if animId and timing then
                AnimationTimings[animId] = tonumber(timing) or 0
                count = count + 1
            end
        end
    end
    
    return count
end

local customAnimCount = loadCustomAnimations()

-- ════════════════════════════════════════════════════════════════
-- UNKNOWN ANIMATION RECORDER (WITH AUTOMATIC GAME NAME EXTRACTION)
-- ════════════════════════════════════════════════════════════════

local RecordedAnimations = {}
local RecordingCache = {}

local function loadRecordedAnimations()
    if not isfile(unknownAnimFile) then
        writefile(unknownAnimFile, "-- Unknown Animations Log --\n-- Format: AnimationID | Timestamp | Target Name | Animation Name (Auto-Detected)\n\n")
    end
    
    local content = readfile(unknownAnimFile)
    for line in content:gmatch("[^\r\n]+") do
        local animId = line:match("^(%d+)")
        if animId then
            RecordedAnimations[animId] = true
            RecordingCache[animId] = true
        end
    end
end

loadRecordedAnimations()

local function getAnimationName(animationInstance)
    local animName = "Unknown Animation"
    
    if animationInstance and animationInstance:IsA("Animation") then
        if animationInstance.Name and animationInstance.Name ~= "" and animationInstance.Name ~= "Animation" then
            animName = animationInstance.Name
        end
        
        local parent = animationInstance.Parent
        if parent and parent.Name and parent.Name ~= "" and parent.Name ~= "Animator" and parent.Name ~= "Humanoid" then
            if animName == "Unknown Animation" or animName == "Animation" then
                animName = parent.Name
            end
        end
    end
    
    return animName
end

local function recordUnknownAnimation(animId, targetName, animationInstance)
    if not PerfectBlockSettings.RecorderEnabled then return end
    if RecordedAnimations[animId] or RecordingCache[animId] then return end
    if IgnoredAnimations[animId] then return end
    
    RecordedAnimations[animId] = true
    RecordingCache[animId] = true
    
    local timestamp = os.date("%Y-%m-%d %H:%M:%S")
    local animName = getAnimationName(animationInstance)
    
    if animName == "Unknown Animation" or animName == "Animation" then
        if PerfectBlockSettings.RecorderCustomName ~= "" then
            animName = PerfectBlockSettings.RecorderCustomName
        end
    end
    
    local logEntry = string.format("%s | %s | %s | [%s]\n", 
        animId, 
        timestamp, 
        targetName or "Unknown",
        animName
    )
    
    appendfile(unknownAnimFile, logEntry)
    
    -- Use notification queue
    NotificationQueue:Add(string.format("📝 New Animation: [%s] ID: %s", animName, animId), 3)
end

-- ════════════════════════════════════════════════════════════════
-- COMBAT STATE DETECTION
-- ════════════════════════════════════════════════════════════════

local function isPlayerInCombat()
    for keyName, keyAllowsBlocking in pairs(CombatState.ActiveKeys) do
        if not keyAllowsBlocking then
            return true
        end
    end
    
    local currentTime = tick()
    if currentTime - CombatState.LastActionTime <= CombatState.ComboCooldown then
        if CombatState.LastKeyAllowedBlocking == false then
            return true
        end
    end
    
    return false
end

local function updateCombatState()
    CombatState.PlayerInCombo = isPlayerInCombat()
    
    if CombatState.PlayerInCombo then
        CombatState.CurrentStatus = "Player Combo"
        updateStatusDisplay("Player Combo", Color3.fromRGB(255, 170, 0))
    else
        if PerfectBlockSettings.PredictionEnabled then
            CombatState.CurrentStatus = "Prediction"
            updateStatusDisplay("Prediction", Color3.fromRGB(0, 255, 127))
        else
            CombatState.CurrentStatus = "Ready"
            updateStatusDisplay("Ready", Color3.fromRGB(255, 255, 255))
        end
    end
end

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    
    local keyAllowsBlocking = nil
    local keyIdentifier = nil
    
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        keyAllowsBlocking = PerfectBlockSettings.CombatKeys[Enum.UserInputType.MouseButton1]
        if keyAllowsBlocking ~= nil then
            keyIdentifier = "MouseButton1"
        end
    else
        keyAllowsBlocking = PerfectBlockSettings.CombatKeys[input.KeyCode]
        if keyAllowsBlocking ~= nil then
            keyIdentifier = input.KeyCode.Name
        end
    end
    
    if keyIdentifier and keyAllowsBlocking ~= nil then
        CombatState.ActiveKeys[keyIdentifier] = keyAllowsBlocking
        CombatState.LastActionTime = tick()
        CombatState.LastKeyAllowedBlocking = keyAllowsBlocking
    end
end)

UserInputService.InputEnded:Connect(function(input, gameProcessed)
    local keyIdentifier = nil
    
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        keyIdentifier = "MouseButton1"
    else
        keyIdentifier = input.KeyCode.Name
    end
    
    if keyIdentifier and CombatState.ActiveKeys[keyIdentifier] ~= nil then
        CombatState.ActiveKeys[keyIdentifier] = nil
        CombatState.LastActionTime = tick()
    end
end)

RunService.Heartbeat:Connect(function()
    if PerfectBlockSettings.Enabled then
        updateCombatState()
    else
        updateStatusDisplay("Disabled", Color3.fromRGB(200, 200, 200))
    end
end)

-- ════════════════════════════════════════════════════════════════
-- PERFECT BLOCK EXECUTION
-- ════════════════════════════════════════════════════════════════

local function executePerfectBlock()
    local success, err = pcall(function()
        if PerfectBlockSettings.BlockDelay > 0 then
            task.wait(PerfectBlockSettings.BlockDelay)
        end
        
        keypress(0x46)
        task.wait(0.05)
        keyrelease(0x46)
    end)
    
    if not success then
        warn("[CRITICAL ERROR] Perfect block execution failed: " .. tostring(err))
    end
    
    return success
end

-- ════════════════════════════════════════════════════════════════
-- ANIMATION MONITORING
-- ════════════════════════════════════════════════════════════════

local blockedAnimations = {}

local function setupCharacter(character)
    if not character then return end
    
    local humanoid = character:WaitForChild("Humanoid", 5)
    if not humanoid then return end
    
    local animator = humanoid:WaitForChild("Animator", 5)
    if not animator then return end
    
    local isPlayer = Players:GetPlayerFromCharacter(character) ~= nil
    
    if isPlayer and not PerfectBlockSettings.BlockPlayers then
        return
    end
    
    local myChar = LocalPlayer.Character
    if not myChar then return end
    
    local myHrp = myChar:FindFirstChild("HumanoidRootPart")
    if not myHrp then return end
    
    animator.AnimationPlayed:Connect(function(animTrack)
        local hrp = character:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        
        local distance = (hrp.Position - myHrp.Position).Magnitude
        if distance > PerfectBlockSettings.MaxDistance then return end
        
        local animId = animTrack.Animation.AnimationId:match("%d+")
        if not animId then return end
        
        if IgnoredAnimations[animId] then return end
        
        -- FIXED: Use character + animId without tick() to prevent duplicates
        local blockKey = tostring(character) .. "_" .. animId
        if blockedAnimations[blockKey] then return end
        
        local animTiming = AnimationTimings[animId]
        
        if not animTiming and PerfectBlockSettings.RecorderEnabled then
            local targetName = character.Name
            if distance <= PerfectBlockSettings.RecorderRadius then
                recordUnknownAnimation(animId, targetName, animTrack.Animation)
            end
        end
        
        if not PerfectBlockSettings.Enabled then return end
        
        if not animTiming and not PerfectBlockSettings.BlockUnknown then 
            return 
        end
        
        local inCombat = isPlayerInCombat()
        
        if inCombat then 
            return 
        end
        
        -- Mark as blocked to prevent duplicates
        blockedAnimations[blockKey] = true
        
        -- Clear the block marker after animation would reasonably be done (1 second)
        task.delay(1, function()
            blockedAnimations[blockKey] = nil
        end)
        
        -- Execute block with timing
        task.spawn(function()
            local waitTime = animTiming or 0
            
            if waitTime > 0 then
                task.wait(waitTime)
            end
            
            local blockSuccess = executePerfectBlock()
            if blockSuccess then
                logBlock(character.Name, animId, PerfectBlockSettings.PredictionEnabled)
            end
        end)
    end)
end

-- ════════════════════════════════════════════════════════════════
-- MAIN TAB UI - LEFT SIDE
-- ════════════════════════════════════════════════════════════════

local MainLeftUpper = Tabs.Main:AddLeftGroupbox('Auto Perfect Block')

MainLeftUpper:AddToggle('MainToggle', {
    Text = 'Enable Auto Block',
    Default = false,
    Tooltip = 'Toggle auto perfect block (Press P to toggle quickly)',
    
    Callback = function(Value)
        PerfectBlockSettings.Enabled = Value
        
        if Value then
            Library:Notify('✅ Auto Block Enabled | Press P to disable', 3)
            updateStatusDisplay("Ready", Color3.fromRGB(0, 255, 127))
            
            for _, char in pairs(Workspace.PlayerCharacters:GetChildren()) do
                setupCharacter(char)
            end
        else
            Library:Notify('❌ Auto Block Disabled | Press P to enable', 3)
            updateStatusDisplay("Disabled", Color3.fromRGB(200, 200, 200))
        end
    end
})

MainLeftUpper:AddToggle('PredictionToggle', {
    Text = 'Enable Prediction',
    Default = false,
    Tooltip = 'Show prediction status (visual indicator only)',
    
    Callback = function(Value)
        PerfectBlockSettings.PredictionEnabled = Value
    end
})

MainLeftUpper:AddToggle('BlockPlayers', {
    Text = 'Block Players',
    Default = true,
    Tooltip = 'Block player attacks',
    
    Callback = function(Value)
        PerfectBlockSettings.BlockPlayers = Value
    end
})

MainLeftUpper:AddDivider()

MainLeftUpper:AddSlider('MaxDistance', {
    Text = 'Max Distance',
    Default = 150,
    Min = 10,
    Max = 300,
    Rounding = 0,
    Suffix = ' studs',
    
    Callback = function(Value)
        PerfectBlockSettings.MaxDistance = Value
    end
})

MainLeftUpper:AddSlider('BlockDelay', {
    Text = 'Block Delay',
    Default = 0,
    Min = 0,
    Max = 0.4,
    Rounding = 2,
    Suffix = 's',
    
    Callback = function(Value)
        PerfectBlockSettings.BlockDelay = Value
    end
})

MainLeftUpper:AddDivider()
MainLeftUpper:AddLabel('Keybind: Press P to toggle')
MainLeftUpper:AddLabel('0s = Instant | 0.2s = Human-like')

-- Combat Keys on Left side, second groupbox
local MainLeftLower = Tabs.Main:AddLeftGroupbox('Combat Keys')

MainLeftLower:AddLabel('ON = Can block while pressing')
MainLeftLower:AddLabel('OFF = Cannot block (your combo)')
MainLeftLower:AddDivider()

-- ════════════════════════════════════════════════════════════════
-- MAIN TAB - COMBAT KEYS (CONTINUING LEFT SIDE)
-- ════════════════════════════════════════════════════════════════

local combatKeyList = {
    {key = Enum.UserInputType.MouseButton1, name = 'M1 (Left Click)', flag = 'M1Key'},
    {key = Enum.KeyCode.E, name = 'E Key', flag = 'EKey'},
    {key = Enum.KeyCode.R, name = 'R Key', flag = 'RKey'},
    {key = Enum.KeyCode.T, name = 'T Key', flag = 'TKey'},
    {key = Enum.KeyCode.Q, name = 'Q Key', flag = 'QKey'},
    {key = Enum.KeyCode.Z, name = 'Z Key', flag = 'ZKey'},
    {key = Enum.KeyCode.X, name = 'X Key', flag = 'XKey'},
    {key = Enum.KeyCode.C, name = 'C Key', flag = 'CKey'},
    {key = Enum.KeyCode.V, name = 'V Key', flag = 'VKey'},
}

for _, keyData in ipairs(combatKeyList) do
    MainLeftLower:AddToggle(keyData.flag, {
        Text = keyData.name,
        Default = false,
        
        Callback = function(Value)
            PerfectBlockSettings.CombatKeys[keyData.key] = Value
        end
    })
end

MainLeftLower:AddDivider()

MainLeftLower:AddSlider('ComboCooldown', {
    Text = 'Combo Cooldown',
    Default = 0.5,
    Min = 0.1,
    Max = 2,
    Rounding = 1,
    Suffix = 's',
    
    Callback = function(Value)
        CombatState.ComboCooldown = Value
    end
})

-- ════════════════════════════════════════════════════════════════
-- MAIN TAB - RIGHT SIDE (STATISTICS)
-- ════════════════════════════════════════════════════════════════

local MainRightBox = Tabs.Main:AddRightGroupbox('Statistics')

MainRightBox:AddLabel('Total Blocks: 0')
MainRightBox:AddLabel('Player Blocks: 0')
MainRightBox:AddLabel('Predicted Blocks: 0')
MainRightBox:AddDivider()
MainRightBox:AddLabel('Status: Waiting...')

-- Update stats every 2 seconds
task.spawn(function()
    while task.wait(2) do
        pcall(function()
            local statsBox = MainRightBox
            -- Stats will be updated via labels
        end)
        if Library.Unloaded then break end
    end
end)

-- ════════════════════════════════════════════════════════════════
-- CUSTOM ANIMATIONS TAB UI (LEFT SIDE OF ANIMS TAB)
-- ════════════════════════════════════════════════════════════════

local CustomAnimGroupBox = Tabs.Anims:AddLeftGroupbox('Add Custom Animation')

local customAnimId = ""
local customTiming = "0"
local customName = ""

CustomAnimGroupBox:AddInput('CustomAnimID', {
    Default = '',
    Numeric = true,
    Finished = true,
    Text = 'Animation ID',
    Tooltip = 'Enter animation ID (numbers only)',
    Placeholder = 'Enter animation ID',
    
    Callback = function(Value)
        customAnimId = Value
    end
})

CustomAnimGroupBox:AddInput('CustomAnimTiming', {
    Default = '0',
    Numeric = true,
    Finished = true,
    Text = 'Timing (seconds)',
    Tooltip = 'Delay before block (0, 0.1, 0.2, etc)',
    Placeholder = '0, 0.1, 0.2, etc',
    
    Callback = function(Value)
        customTiming = Value
    end
})

CustomAnimGroupBox:AddInput('CustomAnimName', {
    Default = '',
    Text = 'Name (Optional)',
    Tooltip = 'e.g., Dragon Claw',
    Placeholder = 'e.g., Dragon Claw',
    
    Callback = function(Value)
        customName = Value
    end
})

CustomAnimGroupBox:AddButton({
    Text = 'Add Animation',
    Func = function()
        if customAnimId == "" or customAnimId:match("%D") then
            Library:Notify('❌ Invalid Animation ID (numbers only)', 3)
            return
        end
        
        local timing = tonumber(customTiming) or 0
        if timing < 0 or timing > 5 then
            Library:Notify('❌ Timing must be between 0 and 5 seconds', 3)
            return
        end
        
        AnimationTimings[customAnimId] = timing
        
        local entry = string.format("%s | %s | %s\n", customAnimId, timing, customName)
        appendfile(customAnimFile, entry)
        
        Library:Notify(string.format('✅ Animation Added: ID %s | Timing %ss', customAnimId, timing), 3)
    end,
    Tooltip = 'Add the animation to database'
})

-- Animation Player
local AnimPlayerGroupBox = Tabs.Anims:AddRightGroupbox('Animation Player')

local playerAnimId = ""
local currentPlayingTrack = nil

AnimPlayerGroupBox:AddInput('PlayerAnimID', {
    Default = '',
    Numeric = true,
    Finished = true,
    Text = 'Animation ID to Play',
    Tooltip = 'Enter animation ID to test',
    Placeholder = 'Enter animation ID',
    
    Callback = function(Value)
        playerAnimId = Value
    end
})

AnimPlayerGroupBox:AddButton({
    Text = 'Play Animation',
    Func = function()
        if playerAnimId == "" or playerAnimId:match("%D") then
            Library:Notify('❌ Invalid Animation ID', 3)
            return
        end
        
        local success = pcall(function()
            local char = LocalPlayer.Character
            local humanoid = char:FindFirstChild("Humanoid")
            local animator = humanoid:FindFirstChild("Animator")
            
            if currentPlayingTrack then
                currentPlayingTrack:Stop()
            end
            
            local anim = Instance.new("Animation")
            anim.AnimationId = "rbxassetid://" .. playerAnimId
            
            local track = animator:LoadAnimation(anim)
            currentPlayingTrack = track
            track:Play()
            
            Library:Notify('🎭 Playing Animation: ' .. playerAnimId, 3)
        end)
        
        if not success then
            Library:Notify('❌ Failed to play animation', 3)
        end
    end,
    Tooltip = 'Test the animation on your character'
})

AnimPlayerGroupBox:AddButton({
    Text = 'Stop All Animations',
    Func = function()
        local success = pcall(function()
            local char = LocalPlayer.Character
            local humanoid = char:FindFirstChild("Humanoid")
            local animator = humanoid:FindFirstChild("Animator")
            
            local playingTracks = animator:GetPlayingAnimationTracks()
            for _, track in ipairs(playingTracks) do
                track:Stop()
            end
            
            currentPlayingTrack = nil
            
            Library:Notify('⏹️ All animations stopped', 2)
        end)
        
        if not success then
            Library:Notify('❌ Failed to stop animations', 3)
        end
    end,
    Tooltip = 'Stop all playing animations'
})

AnimPlayerGroupBox:AddDivider()

local customAnimCountLabel = AnimPlayerGroupBox:AddLabel('Custom Animations: ' .. customAnimCount)

AnimPlayerGroupBox:AddButton({
    Text = 'Reload Custom Animations',
    Func = function()
        local count = loadCustomAnimations()
        customAnimCountLabel:SetText('Custom Animations: ' .. (count or 0))
        Library:Notify(string.format('📝 Loaded %d custom animations', count or 0), 3)
    end,
    Tooltip = 'Reload custom animations from file'
})

-- ════════════════════════════════════════════════════════════════
-- RECORDER UI (LEFT SIDE OF ANIMS TAB, SECOND GROUPBOX)
-- ════════════════════════════════════════════════════════════════

local RecorderGroupBox = Tabs.Anims:AddLeftGroupbox('Animation Recorder')

RecorderGroupBox:AddToggle('RecorderEnabled', {
    Text = 'Enable Recorder',
    Default = true,
    Tooltip = 'Record unknown animations',
    
    Callback = function(Value)
        PerfectBlockSettings.RecorderEnabled = Value
    end
})

RecorderGroupBox:AddToggle('BlockUnknown', {
    Text = 'Block Unknown Animations',
    Default = true,
    Tooltip = 'Block animations not in database',
    
    Callback = function(Value)
        PerfectBlockSettings.BlockUnknown = Value
    end
})

RecorderGroupBox:AddSlider('RecorderRadius', {
    Text = 'Recorder Radius',
    Default = 150,
    Min = 10,
    Max = 300,
    Rounding = 0,
    Suffix = ' studs',
    
    Callback = function(Value)
        PerfectBlockSettings.RecorderRadius = Value
    end
})

-- Auto Name Detection
local AutoNameGroupBox = Tabs.Anims:AddRightGroupbox('Auto Name Detection')

AutoNameGroupBox:AddLabel('Script auto-detects animation names!')
AutoNameGroupBox:AddLabel('Fallback name used if detection fails')
AutoNameGroupBox:AddDivider()

AutoNameGroupBox:AddInput('FallbackName', {
    Default = '',
    Text = 'Fallback Custom Name',
    Tooltip = 'Used when game name not found',
    Placeholder = 'e.g., M1, E Skill, etc.',
    
    Callback = function(Value)
        PerfectBlockSettings.RecorderCustomName = Value
    end
})

AutoNameGroupBox:AddDivider()

-- Update recorded count label dynamically
local recordedCountLabel = AutoNameGroupBox:AddLabel('Recorded: 0')

task.spawn(function()
    while task.wait(2) do
        local count = 0
        for _ in pairs(RecordedAnimations) do 
            count = count + 1 
        end
        recordedCountLabel:SetText('Recorded Animations: ' .. count)
        
        if Library.Unloaded then break end
    end
end)

AutoNameGroupBox:AddButton({
    Text = 'Clear All Recordings',
    Func = function()
        RecordedAnimations = {}
        RecordingCache = {}
        writefile(unknownAnimFile, "-- Unknown Animations Log --\n-- Format: AnimationID | Timestamp | Target Name | Animation Name (Auto-Detected)\n\n")
        
        Library:Notify('📝 All recordings cleared', 3)
    end
})

-- ════════════════════════════════════════════════════════════════
-- UI SETTINGS TAB
-- ════════════════════════════════════════════════════════════════

-- Menu Controls
local MenuGroup = Tabs['UI Settings']:AddLeftGroupbox('Menu')

MenuGroup:AddButton({
    Text = 'Unload',
    Func = function() 
        Library:Unload() 
    end,
    DoubleClick = false,
    Tooltip = 'Unload the UI'
})

MenuGroup:AddLabel('Menu bind'):AddKeyPicker('MenuKeybind', { 
    Default = 'End', 
    NoUI = true, 
    Text = 'Menu keybind' 
})

Library.ToggleKeybind = Options.MenuKeybind

-- Watermark
MenuGroup:AddDivider()
MenuGroup:AddToggle('WatermarkToggle', {
    Text = 'Show Watermark',
    Default = false,
    Tooltip = 'Show/hide watermark',
    
    Callback = function(Value)
        -- Handled by OnChanged below
    end
})

-- Keybind Frame
MenuGroup:AddToggle('KeybindToggle', {
    Text = 'Show Keybind Frame',
    Default = false,
    Tooltip = 'Show/hide keybind list',
    
    Callback = function(Value)
        Library.KeybindFrame.Visible = Value
    end
})

-- Theme Manager & Save Manager
ThemeManager:SetLibrary(Library)
SaveManager:SetLibrary(Library)

SaveManager:IgnoreThemeSettings()
SaveManager:SetIgnoreIndexes({ 'MenuKeybind' })

ThemeManager:SetFolder('UzuHub')
SaveManager:SetFolder('UzuHub/AutoPerfectBlock')

SaveManager:BuildConfigSection(Tabs['UI Settings'])
ThemeManager:ApplyToTab(Tabs['UI Settings'])

-- Watermark system
Library:SetWatermarkVisibility(false)
Library.KeybindFrame.Visible = false

local FrameTimer = tick()
local FrameCounter = 0
local FPS = 60
local WatermarkConnection = nil

local function startWatermark()
    if WatermarkConnection then return end
    
    WatermarkConnection = game:GetService('RunService').RenderStepped:Connect(function()
        FrameCounter = FrameCounter + 1
        
        if (tick() - FrameTimer) >= 1 then
            FPS = FrameCounter
            FrameTimer = tick()
            FrameCounter = 0
        end
        
        Library:SetWatermark(('UZU HUB | %s fps | %s ms'):format(
            math.floor(FPS),
            math.floor(game:GetService('Stats').Network.ServerStatsItem['Data Ping']:GetValue())
        ))
    end)
end

local function stopWatermark()
    if WatermarkConnection then
        WatermarkConnection:Disconnect()
        WatermarkConnection = nil
    end
end

Toggles.WatermarkToggle:OnChanged(function()
    if Toggles.WatermarkToggle.Value then
        Library:SetWatermarkVisibility(true)
        startWatermark()
    else
        Library:SetWatermarkVisibility(false)
        stopWatermark()
    end
end)

Library:OnUnload(function()
    stopWatermark()
    print('UZU HUB Unloaded!')
    Library.Unloaded = true
end)

-- Load autoload config
SaveManager:LoadAutoloadConfig()

-- ════════════════════════════════════════════════════════════════
-- MONITORING
-- ════════════════════════════════════════════════════════════════

for _, char in pairs(Workspace.PlayerCharacters:GetChildren()) do
    task.spawn(function()
        setupCharacter(char)
    end)
end

Workspace.PlayerCharacters.ChildAdded:Connect(function(character)
    task.wait(0.5)
    setupCharacter(character)
end)

-- ════════════════════════════════════════════════════════════════
-- KEYBIND HANDLER
-- ════════════════════════════════════════════════════════════════

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    
    if input.KeyCode == Enum.KeyCode.P then
        PerfectBlockSettings.Enabled = not PerfectBlockSettings.Enabled
        Toggles.MainToggle:SetValue(PerfectBlockSettings.Enabled)
        
        if PerfectBlockSettings.Enabled then
            Library:Notify('✅ Auto Block Enabled | Press P to disable', 3)
            updateStatusDisplay("Ready", Color3.fromRGB(0, 255, 127))
            
            for _, char in pairs(Workspace.PlayerCharacters:GetChildren()) do
                setupCharacter(char)
            end
        else
            Library:Notify('❌ Auto Block Disabled | Press P to enable', 3)
            updateStatusDisplay("Disabled", Color3.fromRGB(200, 200, 200))
        end
    end
end)

-- ════════════════════════════════════════════════════════════════
-- INITIALIZATION
-- ════════════════════════════════════════════════════════════════

Library:Notify('UZU HUB SEXY UI', 5)
updateStatusDisplay("Disabled", Color3.fromRGB(200, 200, 200))
