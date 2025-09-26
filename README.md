-- Infinite Yield FE - Complete Rayfield UI 日本語版
-- 作者: くろすけ & AI
-- 全機能実装版 with Rayfield UI

local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- サービス
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Lighting = game:GetService("Lighting")
local PathfindingService = game:GetService("PathfindingService")
local Teams = game:GetService("Teams")
local GuiService = game:GetService("GuiService")
local SoundService = game:GetService("SoundService")
local StarterGui = game:GetService("StarterGui")
local MarketplaceService = game:GetService("MarketplaceService")
local TextService = game:GetService("TextService")
local CoreGui = game:GetService("CoreGui")
local ContextActionService = game:GetService("ContextActionService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local Chat = game:GetService("Chat")

-- プレイヤー参照
local LocalPlayer = Players.LocalPlayer
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")
local Mouse = LocalPlayer:GetMouse()

-- 状態変数とデータ保存
local gameData = {
    -- 移動関連
    flyEnabled = false,
    flyType = "Normal", -- Normal, VFly, CFrame, BodyVelocity
    noclipEnabled = false,
    speedEnabled = false,
    jumpPowerEnabled = false,
    infiniteJumpEnabled = false,
    swimEnabled = false,
    
    -- プレイヤー関連
    godModeEnabled = false,
    invisibilityEnabled = false,
    ghostEnabled = false,
    
    -- 視覚効果
    espEnabled = false,
    espType = "Highlight", -- Highlight, Box, Tracer
    wallhackEnabled = false,
    fullbrightEnabled = false,
    xrayEnabled = false,
    
    -- ツール
    clickTeleportEnabled = false,
    antiAFKEnabled = false,
    chatLogsEnabled = false,
    joinLogsEnabled = false,
    
    -- ワールド
    deleteToolsEnabled = false,
    
    -- 設定
    flySpeed = 50,
    walkSpeed = 16,
    jumpPower = 50,
    hipHeight = 0,
    
    -- ウェイポイント
    waypoints = {},
    
    -- エイリアス
    aliases = {},
    
    -- キーバインド
    keybinds = {},
    
    -- プラグイン
    plugins = {},
    
    -- ログ
    chatLogs = {},
    joinLogs = {},
    
    -- チーム
    teams = {},
    
    -- 管理者リスト
    admins = {},
    banned = {},
    
    -- 実行ログ
    commandHistory = {},
    
    -- フライ接続
    flyConnections = {}
}

-- ユーティリティ関数
local function getRootPart(player)
    local character = player.Character or player.CharacterAdded:Wait()
    return character:FindFirstChild("HumanoidRootPart") or character:FindFirstChild("Torso") or character:FindFirstChild("UpperTorso")
end

local function getHumanoid(player)
    local character = player.Character or player.CharacterAdded:Wait()
    return character:FindFirstChildOfClass("Humanoid")
end

local function findPlayer(name)
    if not name or name == "" then return nil end
    name = name:lower()
    
    -- 特別なキーワード
    if name == "me" or name == "自分" then
        return LocalPlayer
    end
    
    -- プレイヤー検索
    for _, player in ipairs(Players:GetPlayers()) do
        if player.Name:lower():sub(1, #name) == name or 
           player.DisplayName:lower():sub(1, #name) == name or
           player.Name:lower():find(name) or 
           player.DisplayName:lower():find(name) then
            return player
        end
    end
    return nil
end

local function findPlayers(name)
    if not name or name == "" then return {} end
    
    local foundPlayers = {}
    name = name:lower()
    
    -- 特別なキーワード
    if name == "all" or name == "everyone" or name == "全員" or name == "みんな" then
        return Players:GetPlayers()
    elseif name == "others" or name == "他の人" or name == "他" then
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                table.insert(foundPlayers, player)
            end
        end
        return foundPlayers
    elseif name == "me" or name == "自分" then
        return {LocalPlayer}
    elseif name == "random" or name == "ランダム" then
        local allPlayers = Players:GetPlayers()
        return {allPlayers[math.random(1, #allPlayers)]}
    elseif name == "youngest" or name == "最年少" then
        local youngest = Players:GetPlayers()[1]
        for _, player in ipairs(Players:GetPlayers()) do
            if player.AccountAge < youngest.AccountAge then
                youngest = player
            end
        end
        return {youngest}
    elseif name == "oldest" or name == "最年長" then
        local oldest = Players:GetPlayers()[1]
        for _, player in ipairs(Players:GetPlayers()) do
            if player.AccountAge > oldest.AccountAge then
                oldest = player
            end
        end
        return {oldest}
    else
        -- 通常の検索
        for _, player in ipairs(Players:GetPlayers()) do
            if player.Name:lower():find(name) or player.DisplayName:lower():find(name) then
                table.insert(foundPlayers, player)
            end
        end
    end
    
    return foundPlayers
end

local function notify(title, content, duration)
    Rayfield:Notify({
        Title = title,
        Content = content,
        Duration = duration or 3,
        Image = 4483362458,
    })
end

-- フライ機能の実装
local function enableFly(flyType)
    local rootPart = getRootPart(LocalPlayer)
    local humanoid = getHumanoid(LocalPlayer)
    if not rootPart or not humanoid then return end
    
    -- 既存のフライを停止
    for _, connection in pairs(gameData.flyConnections) do
        if connection then connection:Disconnect() end
    end
    gameData.flyConnections = {}
    
    if flyType == "Normal" then
        -- 通常のフライ（BodyVelocity使用）
        local bodyVelocity = Instance.new("BodyVelocity")
        local bodyAngularVelocity = Instance.new("BodyAngularVelocity")
        
        bodyVelocity.MaxForce = Vector3.new(4000, 4000, 4000)
        bodyVelocity.Velocity = Vector3.new(0, 0, 0)
        bodyVelocity.Parent = rootPart
        
        bodyAngularVelocity.MaxTorque = Vector3.new(0, 4000, 0)
        bodyAngularVelocity.AngularVelocity = Vector3.new(0, 0, 0)
        bodyAngularVelocity.Parent = rootPart
        
        local connection = RunService.Heartbeat:Connect(function()
            if not gameData.flyEnabled then
                bodyVelocity:Destroy()
                bodyAngularVelocity:Destroy()
                return
            end
            
            local camera = workspace.CurrentCamera
            local moveVector = humanoid.MoveDirection * gameData.flySpeed
            
            if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
                bodyVelocity.Velocity = Vector3.new(moveVector.X, gameData.flySpeed, moveVector.Z)
            elseif UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
                bodyVelocity.Velocity = Vector3.new(moveVector.X, -gameData.flySpeed, moveVector.Z)
            else
                bodyVelocity.Velocity = Vector3.new(moveVector.X, 0, moveVector.Z)
            end
        end)
        table.insert(gameData.flyConnections, connection)
        
    elseif flyType == "VFly" then
        -- VFly（Vector3ベース）
        local connection = RunService.Heartbeat:Connect(function()
            if not gameData.flyEnabled then return end
            
            local camera = workspace.CurrentCamera
            local moveVector = Vector3.new(0, 0, 0)
            
            if UserInputService:IsKeyDown(Enum.KeyCode.W) then
                moveVector = moveVector + camera.CFrame.LookVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.S) then
                moveVector = moveVector - camera.CFrame.LookVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.A) then
                moveVector = moveVector - camera.CFrame.RightVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.D) then
                moveVector = moveVector + camera.CFrame.RightVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
                moveVector = moveVector + Vector3.new(0, 1, 0)
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
                moveVector = moveVector - Vector3.new(0, 1, 0)
            end
            
            rootPart.Velocity = moveVector * gameData.flySpeed
        end)
        table.insert(gameData.flyConnections, connection)
        
    elseif flyType == "CFrame" then
        -- CFrameフライ
        local connection = RunService.Heartbeat:Connect(function()
            if not gameData.flyEnabled then return end
            
            local camera = workspace.CurrentCamera
            local moveVector = Vector3.new(0, 0, 0)
            
            if UserInputService:IsKeyDown(Enum.KeyCode.W) then
                moveVector = moveVector + camera.CFrame.LookVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.S) then
                moveVector = moveVector - camera.CFrame.LookVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.A) then
                moveVector = moveVector - camera.CFrame.RightVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.D) then
                moveVector = moveVector + camera.CFrame.RightVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
                moveVector = moveVector + Vector3.new(0, 1, 0)
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
                moveVector = moveVector - Vector3.new(0, 1, 0)
            end
            
            if moveVector.Magnitude > 0 then
                rootPart.CFrame = rootPart.CFrame + moveVector.Unit * (gameData.flySpeed / 10)
            end
        end)
        table.insert(gameData.flyConnections, connection)
        
    elseif flyType == "BodyPosition" then
        -- BodyPositionフライ
        local bodyPosition = Instance.new("BodyPosition")
        bodyPosition.MaxForce = Vector3.new(4000, 4000, 4000)
        bodyPosition.Position = rootPart.Position
        bodyPosition.Parent = rootPart
        
        local connection = RunService.Heartbeat:Connect(function()
            if not gameData.flyEnabled then
                bodyPosition:Destroy()
                return
            end
            
            local camera = workspace.CurrentCamera
            local moveVector = Vector3.new(0, 0, 0)
            
            if UserInputService:IsKeyDown(Enum.KeyCode.W) then
                moveVector = moveVector + camera.CFrame.LookVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.S) then
                moveVector = moveVector - camera.CFrame.LookVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.A) then
                moveVector = moveVector - camera.CFrame.RightVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.D) then
                moveVector = moveVector + camera.CFrame.RightVector
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.Space) then
                moveVector = moveVector + Vector3.new(0, 1, 0)
            end
            if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) then
                moveVector = moveVector - Vector3.new(0, 1, 0)
            end
            
            if moveVector.Magnitude > 0 then
                bodyPosition.Position = bodyPosition.Position + moveVector.Unit * (gameData.flySpeed / 5)
            end
        end)
        table.insert(gameData.flyConnections, connection)
    end
end

-- ESP機能の実装
local function createESP(player, espType)
    if player == LocalPlayer then return end
    local character = player.Character
    if not character then return end
    
    -- 既存のESPを削除
    local existingESP = character:FindFirstChild("ESP_" .. espType)
    if existingESP then existingESP:Destroy() end
    
    if espType == "Highlight" then
        local highlight = Instance.new("Highlight")
        highlight.Name = "ESP_Highlight"
        highlight.FillColor = Color3.fromRGB(255, 0, 0)
        highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
        highlight.FillTransparency = 0.5
        highlight.OutlineTransparency = 0
        highlight.Parent = character
        
    elseif espType == "Box" then
        local rootPart = character:FindFirstChild("HumanoidRootPart")
        if not rootPart then return end
        
        local billboardGui = Instance.new("BillboardGui")
        billboardGui.Name = "ESP_Box"
        billboardGui.Size = UDim2.new(4, 0, 6, 0)
        billboardGui.StudsOffset = Vector3.new(0, 0, 0)
        billboardGui.AlwaysOnTop = true
        billboardGui.Parent = rootPart
        
        local frame = Instance.new("Frame")
        frame.Size = UDim2.new(1, 0, 1, 0)
        frame.BackgroundTransparency = 1
        frame.BorderSizePixel = 2
        frame.BorderColor3 = Color3.fromRGB(255, 0, 0)
        frame.Parent = billboardGui
        
    elseif espType == "Tracer" then
        local rootPart = character:FindFirstChild("HumanoidRootPart")
        if not rootPart then return end
        
        local beam = Instance.new("Beam")
        beam.Name = "ESP_Tracer"
        beam.Color = ColorSequence.new(Color3.fromRGB(255, 0, 0))
        beam.Transparency = NumberSequence.new(0.3)
        beam.Width0 = 0.5
        beam.Width1 = 0.5
        
        local attachment0 = Instance.new("Attachment")
        attachment0.Parent = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        
        local attachment1 = Instance.new("Attachment")
        attachment1.Parent = rootPart
        
        beam.Attachment0 = attachment0
        beam.Attachment1 = attachment1
        beam.Parent = workspace
    end
    
    -- 名前表示
    local head = character:FindFirstChild("Head")
    if head and not head:FindFirstChild("ESP_BillboardGui") then
        local billboard = Instance.new("BillboardGui")
        billboard.Name = "ESP_BillboardGui"
        billboard.Size = UDim2.new(0, 200, 0, 50)
        billboard.StudsOffset = Vector3.new(0, 2, 0)
        billboard.Parent = head
        
        local nameLabel = Instance.new("TextLabel")
        nameLabel.Size = UDim2.new(1, 0, 1, 0)
        nameLabel.BackgroundTransparency = 1
        nameLabel.Text = player.Name
        nameLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
        nameLabel.TextScaled = true
        nameLabel.Font = Enum.Font.SourceSansBold
        nameLabel.TextStrokeTransparency = 0
        nameLabel.Parent = billboard
    end
end

-- ウィンドウ作成
local Window = Rayfield:CreateWindow({
    Name = "Infinite Yield FE - Complete 日本語版 v6.3.3",
    LoadingTitle = "Infinite Yield 読み込み中...",
    LoadingSubtitle = "全機能実装版 by くろすけ & AI",
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "InfiniteYieldComplete",
        FileName = "Config"
    },
    Discord = {
        Enabled = false,
        Invite = "",
        RememberJoins = true
    },
    KeySystem = false,
})

-- 移動タブ
local MovementTab = Window:CreateTab("移動 (Movement)", 4483362458)

-- フライタイプ選択
MovementTab:CreateDropdown({
    Name = "フライタイプ選択 (Fly Type)",
    Options = {"Normal", "VFly", "CFrame", "BodyPosition"},
    CurrentOption = "Normal",
    Flag = "FlyTypeDropdown",
    Callback = function(Option)
        gameData.flyType = Option
        if gameData.flyEnabled then
            enableFly(Option)
        end
    end,
})

MovementTab:CreateToggle({
    Name = "フライ (Fly)",
    CurrentValue = false,
    Flag = "FlyToggle",
    Callback = function(Value)
        gameData.flyEnabled = Value
        
        if gameData.flyEnabled then
            enableFly(gameData.flyType)
            notify("フライ", gameData.flyType .. " フライモードを有効にしました")
        else
            -- フライを停止
            for _, connection in pairs(gameData.flyConnections) do
                if connection then connection:Disconnect() end
            end
            gameData.flyConnections = {}
            notify("フライ", "フライモードを無効にしました")
        end
    end,
})

MovementTab:CreateSlider({
    Name = "フライ速度 (Fly Speed)",
    Range = {1, 500},
    Increment = 1,
    Suffix = "",
    CurrentValue = 50,
    Flag = "FlySpeedSlider",
    Callback = function(Value)
        gameData.flySpeed = Value
    end,
})

MovementTab:CreateToggle({
    Name = "ノークリップ (Noclip)",
    CurrentValue = false,
    Flag = "NoclipToggle", 
    Callback = function(Value)
        gameData.noclipEnabled = Value
        local character = LocalPlayer.Character
        
        if gameData.noclipEnabled and character then
            local connection
            connection = RunService.Stepped:Connect(function()
                if not gameData.noclipEnabled then
                    -- パーツの衝突を元に戻す
                    for _, part in pairs(character:GetChildren()) do
                        if part:IsA("BasePart") then
                            part.CanCollide = true
                        end
                    end
                    connection:Disconnect()
                    return
                end
                
                for _, part in pairs(character:GetChildren()) do
                    if part:IsA("BasePart") then
                        part.CanCollide = false
                    end
                end
            end)
            notify("ノークリップ", "ノークリップを有効にしました")
        else
            notify("ノークリップ", "ノークリップを無効にしました")
        end
    end,
})

MovementTab:CreateSlider({
    Name = "歩行速度 (Walk Speed)",
    Range = {0, 500},
    Increment = 1,
    Suffix = "",
    CurrentValue = 16,
    Flag = "WalkSpeedSlider",
    Callback = function(Value)
        gameData.walkSpeed = Value
        local humanoid = getHumanoid(LocalPlayer)
        if humanoid then
            humanoid.WalkSpeed = Value
        end
    end,
})

MovementTab:CreateSlider({
    Name = "ジャンプパワー (Jump Power)",
    Range = {0, 500},
    Increment = 1,
    Suffix = "",
    CurrentValue = 50,
    Flag = "JumpPowerSlider",
    Callback = function(Value)
        gameData.jumpPower = Value
        local humanoid = getHumanoid(LocalPlayer)
        if humanoid then
            humanoid.JumpPower = Value
        end
    end,
})

MovementTab:CreateSlider({
    Name = "ヒップ高さ (Hip Height)",
    Range = {-50, 50},
    Increment = 0.5,
    Suffix = "",
    CurrentValue = 0,
    Flag = "HipHeightSlider",
    Callback = function(Value)
        gameData.hipHeight = Value
        local humanoid = getHumanoid(LocalPlayer)
        if humanoid then
            humanoid.HipHeight = Value
        end
    end,
})

MovementTab:CreateToggle({
    Name = "無限ジャンプ (Infinite Jump)",
    CurrentValue = false,
    Flag = "InfiniteJumpToggle",
    Callback = function(Value)
        gameData.infiniteJumpEnabled = Value
        
        if gameData.infiniteJumpEnabled then
            local connection
            connection = UserInputService.JumpRequest:Connect(function()
                if not gameData.infiniteJumpEnabled then
                    connection:Disconnect()
                    return
                end
                
                local humanoid = getHumanoid(LocalPlayer)
                if humanoid then
                    humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
                end
            end)
            notify("無限ジャンプ", "無限ジャンプを有効にしました")
        else
            notify("無限ジャンプ", "無限ジャンプを無効にしました")
        end
    end,
})

MovementTab:CreateToggle({
    Name = "スイム (Swim)",
    CurrentValue = false,
    Flag = "SwimToggle",
    Callback = function(Value)
        gameData.swimEnabled = Value
        local humanoid = getHumanoid(LocalPlayer)
        
        if gameData.swimEnabled and humanoid then
            humanoid:ChangeState(Enum.HumanoidStateType.Swimming)
     
