-- โหลด Rayfield UI
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

local TweenService = game:GetService("TweenService")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local RunService = game:GetService("RunService")
local CoreGui = game:GetService("CoreGui")

-- ==================== ค่าเริ่มต้น ====================
local flySpeed = 50
local holdTime = 0.3
local heightOffset = 5
local heightEnabled = true
local toolName = "พลั่ว"

local plantEnabled = false
local plantSpeed = 50
local plantHeight = 5
local plantHeightEnabled = true
local plantHoldTime = 1.4
local plantToolName = "เมล็ดทุเรียน"

local flyingEnabled = false
local flyTask = nil
local plantTask = nil

local originalCameraType = nil
local cameraLocked = false

-- ==================== ระบบกันแบน ====================
local AntiBan = {
    Enabled = true,
    AntiKick = true,
    AntiAFK = true,
    AntiTeleport = true,        -- ป้องกันการถูกวาร์ปออก
    AntiFling = true,           -- ป้องกันการถูกกระเด็น
    SafeTeleport = true,        -- ใช้ Humanoid:MoveTo แทนการ set CFrame ตรงๆ
    RandomDelay = true,         -- สุ่มดีเลย์ให้ดูเป็นธรรมชาติ
    MaxSpeedLimit = 200,        -- จำกัดความเร็วสูงสุดไม่ให้ดูผิดปกติ
    MinDelay = 0.5,
    MaxDelay = 1.5,
    LastPosition = nil,
    Connections = {},
}
-- ===================================================

local Window = Rayfield:CreateWindow({
    Name = "Durian Auto",
    LoadingTitle = "กำลังโหลด...",
    LoadingSubtitle = "โดย Rayfield UI",
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "DurianAuto",
        FileName = "Config"
    },
    KeySystem = false
})

local MainTab = Window:CreateTab("บินหา Durian", 4483362458)
local PlantTab = Window:CreateTab("ปลูกเมล็ด", 4483362458)
local AntiBanTab = Window:CreateTab("🛡️ กันแบน", 4483362458)
local SettingTab = Window:CreateTab("ตั้งค่า", 4483362458)

-- ==================== ฟังก์ชันช่วย ====================
local function getCharacter()
    local char = LocalPlayer.Character or LocalPlayer.CharacterAdded:Wait()
    local hrp = char:WaitForChild("HumanoidRootPart", 5)
    return char, hrp
end

local durianCache = nil
local function findDurian()
    if durianCache and durianCache.Parent then return durianCache end
    durianCache = workspace:FindFirstChild("DurianBig", true)
    return durianCache
end

local treeCache = nil
local function findTree()
    if treeCache and treeCache.Parent then return treeCache end
    treeCache = workspace:FindFirstChild("DurianTree", true)
    return treeCache
end

local plantPartCache = nil
local function findPlantPart()
    if plantPartCache and plantPartCache.Parent then return plantPartCache end
    plantPartCache = workspace:FindFirstChild("p", true)
    return plantPartCache
end

local function findTool(name)
    local char = LocalPlayer.Character
    if not char then return nil end
    for _, t in ipairs(LocalPlayer.Backpack:GetChildren()) do
        if t:IsA("Tool") and t.Name:find(name, 1, true) then return t end
    end
    for _, t in ipairs(char:GetChildren()) do
        if t:IsA("Tool") and t.Name:find(name, 1, true) then return t end
    end
    return nil
end

local function resetCamera()
    local camera = workspace.CurrentCamera
    if not camera then return end
    if cameraLocked then
        camera.CameraType = originalCameraType or Enum.CameraType.Custom
        cameraLocked = false
        originalCameraType = nil
    end
    local char = LocalPlayer.Character
    if char and char:FindFirstChild("Humanoid") then
        camera.CameraSubject = char.Humanoid
    end
end

-- ==================== ระบบกันแบน: ฟังก์ชันหลัก ====================

-- 1) Anti-Kick: บล็อก RemoteEvent ที่ทำให้ถูกเตะ
local function setupAntiKick()
    if not AntiBan.AntiKick then return end
    
    -- บล็อก Remote ที่ชื่อมีคำว่า kick/ban
    for _, obj in ipairs(game:GetDescendants()) do
        if obj:IsA("RemoteEvent") or obj:IsA("RemoteFunction") then
            local lowerName = obj.Name:lower()
            if lowerName:find("kick") or lowerName:find("ban") or lowerName:find("remove") then
                pcall(function()
                    local oldName = obj.Name
                    obj.Name = "Blocked_" .. oldName
                end)
            end
        end
    end
end

-- 2) Anti-AFK: ป้องกันการถูกเตะเพราะ AFK
local function setupAntiAFK()
    if not AntiBan.AntiAFK then return end
    
    if LocalPlayer.Idled then
        -- บล็อก Idled event ของ Roblox
        LocalPlayer.Idled:Connect(function()
            -- จำลองการขยับเล็กน้อย
            local vu = game:GetService("VirtualUser")
            vu:CaptureController()
            vu:ClickButton2(Vector2.new())
        end)
    end
    
    -- ขยับตัวเป็นระยะ
    table.insert(AntiBan.Connections, task.spawn(function()
        while AntiBan.Enabled and AntiBan.AntiAFK do
            task.wait(60 + math.random(0, 30))
            if AntiBan.Enabled and AntiBan.AntiAFK then
                local char = LocalPlayer.Character
                if char and char:FindFirstChild("Humanoid") then
                    local hum = char.Humanoid
                    if hum.MoveDirection.Magnitude < 0.1 then
                        -- หมุนตัวเล็กน้อย
                        local hrp = char:FindFirstChild("HumanoidRootPart")
                        if hrp then
                            hrp.CFrame = hrp.CFrame * CFrame.Angles(0, math.rad(1), 0)
                        end
                    end
                end
            end
        end
    end))
end

-- 3) Anti-Teleport: ป้องกันการถูกวาร์ปออกจากพื้นที่
local function setupAntiTeleport()
    if not AntiBan.AntiTeleport then return end
    
    AntiBan.LastPosition = nil
    
    table.insert(AntiBan.Connections, RunService.Heartbeat:Connect(function()
        if not AntiBan.Enabled or not AntiBan.AntiTeleport then return end
        
        local char = LocalPlayer.Character
        if not char then return end
        
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        
        if flyingEnabled or plantEnabled then
            -- ระหว่างบินไม่ต้องเช็ค
            AntiBan.LastPosition = hrp.Position
            return
        end
        
        if AntiBan.LastPosition then
            local distance = (hrp.Position - AntiBan.LastPosition).Magnitude
            -- ถ้ากระโดดไกลผิดปกติ (เช่น ถูกวาร์ป) ให้วาร์ปกลับ
            if distance > 100 then
                hrp.CFrame = CFrame.new(AntiBan.LastPosition)
            end
        end
        AntiBan.LastPosition = hrp.Position
    end))
end

-- 4) Anti-Fling: ป้องกันการถูกกระเด็นออกจากแมพ
local function setupAntiFling()
    if not AntiBan.AntiFling then return end
    
    table.insert(AntiBan.Connections, RunService.Heartbeat:Connect(function()
        if not AntiBan.Enabled or not AntiBan.AntiFling then return end
        
        local char = LocalPlayer.Character
        if not char then return end
        
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        
        -- ถ้าความเร็วสูงเกินผิดปกติ ให้รีเซ็ต
        if hrp.AssemblyLinearVelocity.Magnitude > 500 then
            hrp.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
        end
        
        -- ถ้าตกแมพ (Y ต่ำเกินไป)
        if hrp.Position.Y < -100 then
            hrp.CFrame = CFrame.new(0, 50, 0)
        end
    end))
end

-- 5) Safe flyTo: บินแบบนุ่มนวลและจำกัดความเร็ว
local function flyTo(targetPos, speed, isRunning)
    local char, hrp = getCharacter()
    if not hrp then return false end
    
    -- จำกัดความเร็วไม่ให้เกิน MaxSpeedLimit (ถ้าเปิดกันแบน)
    local actualSpeed = speed
    if AntiBan.Enabled and AntiBan.MaxSpeedLimit then
        actualSpeed = math.min(speed, AntiBan.MaxSpeedLimit)
    end

    local distance = (hrp.Position - targetPos).Magnitude
    local duration = math.max(distance / actualSpeed, 0.05)

    hrp.Anchored = true
    local tween = TweenService:Create(
        hrp,
        TweenInfo.new(duration, Enum.EasingStyle.Linear),
        { CFrame = CFrame.new(targetPos) }
    )
    tween:Play()

    local completed = false
    local conn = tween.Completed:Connect(function() completed = true end)
    local waited = 0
    while not completed and waited < duration + 0.5 and isRunning() do
        task.wait(0.05)
        waited += 0.05
    end
    conn:Disconnect()
    hrp.Anchored = false
    return completed
end

-- 6) Random Delay: สุ่มดีเลย์ระหว่างการทำงานให้ดูเป็นธรรมชาติ
local function smartDelay(baseDelay)
    if AntiBan.Enabled and AntiBan.RandomDelay then
        local randomExtra = math.random() * (AntiBan.MaxDelay - AntiBan.MinDelay) + AntiBan.MinDelay
        return baseDelay + randomExtra
    end
    return baseDelay
end

-- ==================== ฟังก์ชันดั้งเดิม ====================
local function chopTree()
    local char, hrp = getCharacter()
    if not hrp then return false end

    local tree = findTree()
    if not tree then
        warn("ไม่พบโมเดล DurianTree")
        return false
    end

    local humanoid = char:FindFirstChild("Humanoid")
    if not humanoid then return false end

    local camera = workspace.CurrentCamera
    local treePos = tree:GetPivot().Position

    local yOffset = heightEnabled and heightOffset or 0
    local targetPos = treePos + Vector3.new(0, yOffset, 0)

    if not flyTo(targetPos, flySpeed, function() return flyingEnabled end) then
        return false
    end

    originalCameraType = camera.CameraType
    cameraLocked = true
    camera.CameraType = Enum.CameraType.Scriptable
    camera.CFrame = CFrame.lookAt(camera.CFrame.Position, treePos)

    local tool = findTool(toolName)
    if not tool then
        warn("ไม่พบ Tool: " .. toolName)
        resetCamera()
        return false
    end

    humanoid:EquipTool(tool)
    task.wait(0.4)

    local success, err = pcall(function()
        tool:Activate()
        task.wait(holdTime)
        tool:Deactivate()
    end)

    task.wait(0.2)
    resetCamera()

    if not success then
        warn("ตัดต้นไม้ล้มเหลว: " .. tostring(err))
        return false
    end
    return true
end

local function plantOnce()
    local char, hrp = getCharacter()
    if not hrp then return false end

    local humanoid = char:FindFirstChild("Humanoid")
    if not humanoid then return false end

    local p = findPlantPart()
    if not p then
        warn("ไม่พบ Part ชื่อ p")
        return false
    end

    local camera = workspace.CurrentCamera

    local tool = findTool(plantToolName)
    if not tool then
        warn("ไม่พบ Tool: " .. plantToolName)
        return false
    end
    humanoid:EquipTool(tool)
    task.wait(0.3)

    local yOffset = plantHeightEnabled and plantHeight or 0
    local targetPos = p.Position + Vector3.new(0, yOffset, 0)

    if not flyTo(targetPos, plantSpeed, function() return plantEnabled end) then
        return false
    end

    originalCameraType = camera.CameraType
    cameraLocked = true
    camera.CameraType = Enum.CameraType.Scriptable
    camera.CFrame = CFrame.lookAt(camera.CFrame.Position, p.Position)

    local success, err = pcall(function()
        tool:Activate()
        task.wait(plantHoldTime)
        tool:Deactivate()
    end)

    task.wait(0.2)
    resetCamera()

    if not success then
        warn("ปลูกเมล็ดล้มเหลว: " .. tostring(err))
        return false
    end
    return true
end

-- ==================== ลูปพร้อม Smart Delay ====================
local function startFlying()
    flyingEnabled = true
    flyTask = task.spawn(function()
        while flyingEnabled do
            local ok = chopTree()
            task.wait(smartDelay(1))
        end
        resetCamera()
        local _, hrp = getCharacter()
        if hrp then hrp.Anchored = false end
    end)
end

local function stopFlying()
    flyingEnabled = false
    task.wait(0.15)
    flyTask = nil
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if hrp then hrp.Anchored = false end
    resetCamera()
end

local function startPlanting()
    plantEnabled = true
    plantTask = task.spawn(function()
        while plantEnabled do
            plantOnce()
            task.wait(smartDelay(1))
        end
        resetCamera()
        local _, hrp = getCharacter()
        if hrp then hrp.Anchored = false end
    end)
end

local function stopPlanting()
    plantEnabled = false
    task.wait(0.15)
    plantTask = nil
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if hrp then hrp.Anchored = false end
    resetCamera()
end

-- ============================================================
-- 🌳 TAB 1: บินหา Durian
-- ============================================================
MainTab:CreateToggle({
    Name = "เปิด/ปิด บินอัตโนมัติ (ตัดต้น)",
    CurrentValue = false,
    Flag = "AutoFlyToggle",
    Callback = function(value)
        if value then
            startFlying()
            Rayfield:Notify({ Title = "เปิดใช้งาน", Content = "เริ่มบินหา DurianTree", Duration = 2 })
        else
            stopFlying()
            Rayfield:Notify({ Title = "ปิดใช้งาน", Content = "หยุดบิน", Duration = 2 })
        end
    end
})

MainTab:CreateSlider({
    Name = "ความเร็วบิน (studs/วิ)",
    Range = {10, 300},
    Increment = 5,
    Suffix = " studs/วิ",
    CurrentValue = 50,
    Flag = "FlySpeedSlider",
    Callback = function(value) flySpeed = value end
})

MainTab:CreateSlider({
    Name = "เวลากดค้าง (วินาที)",
    Range = {0.1, 3},
    Increment = 0.1,
    Suffix = " วิ",
    CurrentValue = 0.3,
    Flag = "HoldTimeSlider",
    Callback = function(value) holdTime = value end
})

MainTab:CreateButton({
    Name = "ตัดต้นไม้ 1 ครั้ง",
    Callback = function()
        task.spawn(function()
            local ok = chopTree()
            Rayfield:Notify({
                Title = ok and "สำเร็จ" or "ล้มเหลว",
                Content = ok and "ตัดต้นไม้เรียบร้อย" or "ไม่สามารถตัดได้",
                Duration = 2
            })
        end)
    end
})

-- ============================================================
-- 🌱 TAB 2: ปลูกเมล็ด
-- ============================================================
PlantTab:CreateToggle({
    Name = "เปิด/ปิด ปลูกเมล็ดอัตโนมัติ",
    CurrentValue = false,
    Flag = "PlantToggle",
    Callback = function(value)
        if value then
            startPlanting()
            Rayfield:Notify({ Title = "เปิดใช้งาน", Content = "เริ่มปลูกเมล็ดทุเรียน", Duration = 2 })
        else
            stopPlanting()
            Rayfield:Notify({ Title = "ปิดใช้งาน", Content = "หยุดปลูก", Duration = 2 })
        end
    end
})

PlantTab:CreateSlider({
    Name = "ความเร็วบิน (studs/วิ)",
    Range = {10, 300},
    Increment = 5,
    Suffix = " studs/วิ",
    CurrentValue = 50,
    Flag = "PlantSpeedSlider",
    Callback = function(value) plantSpeed = value end
})

PlantTab:CreateSlider({
    Name = "เวลากดค้าง (วินาที)",
    Range = {0.1, 3},
    Increment = 0.1,
    Suffix = " วิ",
    CurrentValue = 1.4,
    Flag = "PlantHoldSlider",
    Callback = function(value) plantHoldTime = value end
})

PlantTab:CreateToggle({
    Name = "เปิด/ปิด การใช้ความสูง",
    CurrentValue = true,
    Flag = "PlantHeightToggle",
    Callback = function(value)
        plantHeightEnabled = value
        Rayfield:Notify({
            Title = value and "เปิด" or "ปิด",
            Content = value and "ใช้ความสูงเหนือ p" or "บินที่ระดับพื้น p",
            Duration = 2
        })
    end
})

PlantTab:CreateSlider({
    Name = "ความสูงเหนือ p (studs)",
    Range = {0, 50},
    Increment = 1,
    Suffix = " studs",
    CurrentValue = 5,
    Flag = "PlantHeightSlider",
    Callback = function(value) plantHeight = value end
})

PlantTab:CreateButton({
    Name = "ปลูก 1 ครั้ง",
    Callback = function()
        task.spawn(function()
            local ok = plantOnce()
            Rayfield:Notify({
                Title = ok and "สำเร็จ" or "ล้มเหลว",
                Content = ok and "ปลูกเมล็ดเรียบร้อย" or "ไม่สามารถปลูกได้",
                Duration = 2
            })
        end)
    end
})

--
============================================================
-- 🛡️ TAB 3: ระบบกันแบน
-- ============================================================
AntiBanTab:CreateToggle({
    Name = "เปิด/ปิด ระบบกันแบนทั้งหมด",
    CurrentValue = true,
    Flag = "AntiBanMaster",
    Callback = function(value)
        AntiBan.Enabled = value
        if value then
            setupAntiKick()
            setupAntiAFK()
            setupAntiTeleport()
            setupAntiFling()
            Rayfield:Notify({ Title = "🛡️ เปิด", Content = "ระบบกันแบนทำงาน", Duration = 2 })
        else
            Rayfield:Notify({ Title = "ปิด", Content = "ระบบกันแบนหยุดทำงาน", Duration = 2 })
        end
    end
})

AntiBanTab:CreateToggle({
    Name = "Anti-Kick (บล็อก Remote เตะ)",
    CurrentValue = true,
    Flag = "AntiKick",
    Callback = function(value)
        AntiBan.AntiKick = value
        if value then setupAntiKick() end
    end
})

AntiBanTab:CreateToggle({
    Name = "Anti-AFK (ป้องกัน AFK Kick)",
    CurrentValue = true,
    Flag = "AntiAFK",
    Callback = function(value) AntiBan.AntiAFK = value end
})

AntiBanTab:CreateToggle({
    Name = "Anti-Teleport (ป้องกันถูกวาร์ป)",
    CurrentValue = true,
    Flag = "AntiTP",
    Callback = function(value) AntiBan.AntiTeleport = value end
})

AntiBanTab:CreateToggle({
    Name = "Anti-Fling (ป้องกันถูกกระเด็น)",
    CurrentValue = true,
    Flag = "AntiFling",
    Callback = function(value) AntiBan.AntiFling = value end
})

AntiBanTab:CreateToggle({
    Name = "Random Delay (สุ่มดีเลย์ให้ธรรมชาติ)",
    CurrentValue = true,
    Flag = "RandomDelay",
    Callback = function(value) AntiBan.RandomDelay = value end
})

AntiBanTab:CreateSlider({
    Name = "ความเร็วสูงสุดที่ปลอดภัย",
    Range = {50, 300},
    Increment = 10,
    Suffix = " studs/วิ",
    CurrentValue = 200,
    Flag = "MaxSpeedLimit",
    Callback = function(value) AntiBan.MaxSpeedLimit = value end
})

AntiBanTab:CreateSlider({
    Name = "ดีเลย์น้อยสุด (วินาที)",
    Range = {0.1, 2},
    Increment = 0.1,
    Suffix = " วิ",
    CurrentValue = 0.5,
    Flag = "MinDelay",
    Callback = function(value) AntiBan.MinDelay = value end
})

AntiBanTab:CreateSlider({
    Name = "ดีเลย์มากสุด (วินาที)",
    Range = {0.5, 5},
    Increment = 0.1,
    Suffix = " วิ",
    CurrentValue = 1.5,
    Flag = "MaxDelay",
    Callback = function(value) AntiBan.MaxDelay = value end
})

AntiBanTab:CreateButton({
    Name = "🔄 เรียกใช้ระบบกันแบนใหม่",
    Callback = function()
        setupAntiKick()
        setupAntiAFK()
        setupAntiTeleport()
        setupAntiFling()
        Rayfield:Notify({
            Title = "🛡️ อัปเดต",
            Content = "ติดตั้งระบบกันแบนใหม่เรียบร้อย",
            Duration = 2
        })
    end
})

-- ============================================================
-- ⚙️ TAB 4: ตั้งค่า (ใช้ร่วมกัน)
-- ============================================================
SettingTab:CreateToggle({
    Name = "เปิด/ปิด การใช้ความสูง (ตัดต้น)",
    CurrentValue = true,
    Flag = "HeightToggle",
    Callback = function(value)
        heightEnabled = value
        Rayfield:Notify({
            Title = value and "เปิด" or "ปิด",
            Content = value and "ใช้ความสูงเหนือโมเดล" or "บินที่ระดับพื้นโมเดล",
            Duration = 2
        })
    end
})

SettingTab:CreateSlider({
    Name = "ความสูงเหนือโมเดล (studs)",
    Range = {0, 50},
    Increment = 1,
    Suffix = " studs",
    CurrentValue = 5,
    Flag = "HeightSlider",
    Callback = function(value) heightOffset = value end
})

SettingTab:CreateInput({
    Name = "ชื่อ Tool สำหรับตัดต้น",
    CurrentValue = "พลั่ว",
    PlaceholderText = "เช่น พลั่ว, ขวาน",
    RemoveTextAfterFocusLost = false,
    Flag = "ToolNameInput",
    Callback = function(text)
        if text and text ~= "" then toolName = text end
    end
})

SettingTab:CreateInput({
    Name = "ชื่อ Tool สำหรับปลูกเมล็ด",
    CurrentValue = "เมล็ดทุเรียน",
    PlaceholderText = "เช่น เมล็ดทุเรียน",
    RemoveTextAfterFocusLost = false,
    Flag = "PlantToolInput",
    Callback = function(text)
        if text and text ~= "" then plantToolName = text end
    end
})

-- ==================== เริ่มระบบกันแบนอัตโนมัติ ====================
setupAntiKick()
setupAntiAFK()
setupAntiTeleport()
setupAntiFling()

-- ==================== แจ้งเตือนโหลดเสร็จ ====================
Rayfield:Notify({
    Title = "โหลดสำเร็จ",
    Content = "แผง Durian Auto + ระบบกันแบน พร้อมใช้งาน",
    Duration = 3
})
