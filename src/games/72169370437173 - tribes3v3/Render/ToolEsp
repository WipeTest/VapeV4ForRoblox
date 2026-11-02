local ToolEsp
local ToolEspSettings = {
    ShowEquipped = true,
    ShowBackpack = true,
    MaxBackpack = 6,
    MaxDistance = 500,
    TeamCheck = true,
    OffsetY = 2.6,
    IconSize = 72,
    SmallSize = 26,
}

local ToolEspObjects = {}
local ToolEspConnections = {}

local function getTeamChannel(player)
    local ch = player:GetAttribute('TeamChannel')
    if ch and ch ~= '' and ch ~= 'team:0' then return ch end
    local team = player:GetAttribute('Team')
    if team ~= nil and team ~= 0 then return 'team:' .. tostring(team) end
    return nil
end

local function isTeammate(player)
    local lp = game:GetService('Players').LocalPlayer
    if player == lp then return true end
    local a, b = getTeamChannel(lp), getTeamChannel(player)
    if a and b then return a == b end
    if player.Team and lp.Team then return player.Team == lp.Team end
    return false
end

local function buildWeaponModel(tool, assets)
    if not tool then return nil end
    local weaponsFolder = assets and assets:FindFirstChild('Weapons')
    local wid = tool:GetAttribute('WeaponId')
    
    if wid and weaponsFolder then
        local tpl = weaponsFolder:FindFirstChild(wid)
        if tpl and tpl:IsA('Model') then
            local clone = tpl:Clone()
            for _, p in ipairs(clone:GetDescendants()) do
                if p:IsA('BasePart') then
                    p.Anchored = true
                    p.CanCollide = false
                end
            end
            return clone
        end
    end
    
    -- Fallback: clone parts from tool
    local model = Instance.new('Model')
    for _, d in ipairs(tool:GetDescendants()) do
        if d:IsA('BasePart') then
            local c = d:Clone()
            c.Anchored = true
            c.CanCollide = false
            c.Parent = model
        end
    end
    if #model:GetChildren() == 0 then model:Destroy(); return nil end
    return model
end

local function clearViewport(vp)
    for _, c in ipairs(vp:GetChildren()) do
        if c:IsA('Model') or c:IsA('Camera') then c:Destroy() end
    end
end

local function setupViewport(vp, tool, assets)
    clearViewport(vp)
    local model = buildWeaponModel(tool, assets)
    if not model then vp.Visible = false; return false end
    
    local ok = pcall(function()
        local cf = model:GetBoundingBox()
        local center = cf.Position
        for _, p in ipairs(model:GetDescendants()) do
            if p:IsA('BasePart') then p.CFrame = p.CFrame - center end
        end
        
        local _, size = model:GetBoundingBox()
        local maxDim = math.max(size.X, size.Y, size.Z)
        if maxDim < 0.01 then maxDim = 1 end
        
        local fov = 30
        local dist = (maxDim * 0.5) / math.tan(math.rad(fov * 0.5)) * 1.15
        
        local cam = Instance.new('Camera')
        cam.FieldOfView = fov
        cam.CFrame = CFrame.new(0, 0, -dist,
            -0.707106769, 0.707106769, 0,
            0.707106769, 0.707106769, 0,
            0, 0, -1
        )
        cam.Parent = vp
        vp.CurrentCamera = cam
        model.Parent = vp
    end)
    
    if not ok then model:Destroy(); vp.Visible = false; return false end
    vp.Visible = true
    return true
end

local function makeIconHolder(size)
    local holder = Instance.new('Frame')
    holder.Size = UDim2.fromOffset(size, size)
    holder.BackgroundColor3 = Color3.fromRGB(14, 14, 20)
    holder.BackgroundTransparency = 0.2
    holder.BorderSizePixel = 0
    local corner = Instance.new('UICorner')
    corner.CornerRadius = UDim.new(0, math.max(4, math.floor(size / 8)))
    corner.Parent = holder
    local stroke = Instance.new('UIStroke')
    stroke.Color = Color3.fromRGB(120, 170, 255)
    stroke.Thickness = size > 40 and 1.5 or 1
    stroke.Transparency = 0.15
    stroke.Parent = holder
    local vp = Instance.new('ViewportFrame')
    vp.Name = 'VP'
    vp.Size = UDim2.new(1, -6, 1, -6)
    vp.Position = UDim2.fromOffset(3, 3)
    vp.BackgroundTransparency = 1
    vp.Ambient = Color3.fromRGB(215, 215, 225)
    vp.LightColor = Color3.fromRGB(255, 255, 255)
    vp.LightDirection = Vector3.new(-1, -1, -1)
    vp.Parent = holder
    return holder, vp, stroke
end

local function createDisplay(character, config)
    local holderH = config.ICON_SIZE + 10 + ((config.SHOW_BACKPACK and config.MAX_BACKPACK > 0) and (config.SMALL_SIZE + 4) or 0)
    local container = Instance.new('Frame')
    container.Name = 'ToolDisplay_' .. character.Name
    container.Size = UDim2.fromOffset(config.ICON_SIZE + 10, holderH)
    container.BackgroundTransparency = 1
    container.AnchorPoint = Vector2.new(0.5, 1)
    container.Visible = false
    container.Parent = ToolEspObjects.iconLayer
    
    local mainHolder, mainVP, mainStroke = makeIconHolder(config.ICON_SIZE + 10)
    mainHolder.Name = 'MainHolder'
    mainHolder.Position = UDim2.new(0.5, 0, 0, 0)
    mainHolder.AnchorPoint = Vector2.new(0.5, 0)
    mainHolder.Parent = container
    
    local row = Instance.new('Frame')
    row.Name = 'Row'
    row.Size = UDim2.new(0, config.ICON_SIZE + 10, 0, config.SMALL_SIZE)
    row.Position = UDim2.new(0.5, 0, 1, 0)
    row.AnchorPoint = Vector2.new(0.5, 1)
    row.BackgroundTransparency = 1
    row.Parent = container
    
    local layout = Instance.new('UIListLayout')
    layout.FillDirection = Enum.FillDirection.Horizontal
    layout.HorizontalAlignment = Enum.HorizontalAlignment.Center
    layout.VerticalAlignment = Enum.VerticalAlignment.Center
    layout.Padding = UDim.new(0, 4)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent = row
    
    return {
        container = container,
        mainHolder = mainHolder,
        mainVP = mainVP,
        mainStroke = mainStroke,
        row = row,
        signature = nil,
        rowHasIcons = false,
    }
end

local function destroyDisplay(character)
    local data = ToolEspObjects.displays[character]
    if data and data.container then data.container:Destroy() end
    ToolEspObjects.displays[character] = nil
end

local function getTools(player, character)
    local equipped = character:FindFirstChildOfClass('Tool')
    local list = {}
    if equipped then table.insert(list, equipped) end
    local backpack = player:FindFirstChildOfClass('Backpack')
    if backpack then
        for _, t in ipairs(backpack:GetChildren()) do
            if t:IsA('Tool') then table.insert(list, t) end
        end
    end
    return list, equipped
end

local function toolKey(tool)
    if not tool then return 'none' end
    return tool.Name .. ':' .. tostring(tool:GetAttribute('WeaponId'))
end

local function rebuildRow(data, tools, equipped, config, assets)
    for _, c in ipairs(data.row:GetChildren()) do
        if c:IsA('Frame') then c:Destroy() end
    end
    
    local order = 0
    for _, tool in ipairs(tools) do
        if tool ~= equipped and order < config.MAX_BACKPACK then
            order += 1
            local holder, vp = makeIconHolder(config.SMALL_SIZE)
            holder.LayoutOrder = order
            holder.Parent = data.row
            setupViewport(vp, tool, assets)
        end
    end
    
    data.rowHasIcons = order > 0
    data.row.Visible = config.SHOW_BACKPACK and order > 0
end

local function updatePlayer(player, config, assets, iconLayer, camera, lp, lplr)
    if player == lp and not config.SHOW_SELF then
        destroyDisplay(player.Character)
        return
    end
    
    if config.TEAM_CHECK and player ~= lp and isTeammate(player) then
        destroyDisplay(player.Character)
        return
    end
    
    local character = player.Character
    if not character then return end
    
    local data = ToolEspObjects.displays[character]
    if not data or not data.container or not data.container.Parent then
        data = createDisplay(character, ToolEspSettings)
        ToolEspObjects.displays[character] = data
    end
    
    local tools, equipped = getTools(player, character)
    if #tools == 0 then
        data.container.Visible = false
        return
    end
    
    local keys = {}
    for _, t in ipairs(tools) do table.insert(keys, toolKey(t)) end
    local signature = table.concat(keys, '|')
    
    if signature ~= data.signature then
        data.signature = signature
        if equipped and setupViewport(data.mainVP, equipped, assets) then
            data.mainHolder.Visible = true
        else
            clearViewport(data.mainVP)
            data.mainHolder.Visible = false
        end
        rebuildRow(data, tools, equipped, ToolEspSettings, assets)
    end
    
    local head = character:FindFirstChild('Head') or character.PrimaryPart
    if not camera or not head then
        data.container.Visible = false
        return
    end
    
    local worldPos = head.Position + Vector3.new(0, config.OFFSET_Y, 0)
    local screenPos, onScreen = camera:WorldToViewportPoint(worldPos)
    local distance = (camera.CFrame.Position - worldPos).Magnitude
    
    if not onScreen or screenPos.Z <= 0 or distance > config.MAX_DISTANCE then
        data.container.Visible = false
        return
    end
    
    data.container.Position = UDim2.fromOffset(math.floor(screenPos.X), math.floor(screenPos.Y))
    data.container.Visible = true
    data.visible = true
    
    local alpha = 1
    if distance > config.FADE_START then
        alpha = 1 - ((distance - config.FADE_START) / math.max(1, config.MAX_DISTANCE - config.FADE_START))
        alpha = math.clamp(alpha, 0, 1)
    end
    data.mainHolder.BackgroundTransparency = 1 - (alpha * 0.8)
    data.mainStroke.Transparency = 1 - (alpha * 0.85)
    data.row.Visible = config.SHOW_BACKPACK and data.rowHasIcons and alpha > 0.25
end

local function toolKey(tool)
    if not tool then return 'none' end
    return tool.Name .. ':' .. tostring(tool:GetAttribute('WeaponId'))
end

local function getTools(player, character)
    local equipped = character:FindFirstChildOfClass('Tool')
    local list = {}
    if equipped then table.insert(list, equipped) end
    local backpack = player:FindFirstChildOfClass('Backpack')
    if backpack then
        for _, t in ipairs(backpack:GetChildren()) do
            if t:IsA('Tool') then table.insert(list, t) end
        end
    end
    return list, equipped
end

local function getTeamChannel(player)
    local ch = player:GetAttribute('TeamChannel')
    if ch and ch ~= '' and ch ~= 'team:0' then return ch end
    local team = player:GetAttribute('Team')
    if team ~= nil and team ~= 0 then return 'team:' .. tostring(team) end
    return nil
end

local function isTeammate(player)
    local lp = game:GetService('Players').LocalPlayer
    if player == lp then return true end
    local a, b = getTeamChannel(lp), getTeamChannel(player)
    if a and b then return a == b end
    if player.Team and lp.Team then return player.Team == lp.Team end
    return false
end

ToolEsp = vape.Categories.Render:CreateModule({
    Name = 'ToolEsp',
    Function = function(callback)
        if callback then
            local lp = game:GetService('Players').LocalPlayer
            local camera = workspace.CurrentCamera
            local replicatedStorage = game:GetService('ReplicatedStorage')
            local playersService = game:GetService('Players')
            local runService = game:GetService('RunService')
            
            ToolEspObjects = {
                displays = {},
                iconLayer = nil,
                screenGui = nil,
            }
            
            -- Cleanup previous
            if _G.__ToolEspToken then
                _G.__ToolEspToken.active = false
            end
            local token = {active = true}
            _G.__ToolEspToken = token
            
            local playerGui = lp:FindFirstChild('PlayerGui')
            local oldGui = playerGui and playerGui:FindFirstChild('ToolEspGui')
            if oldGui then oldGui:Destroy() end
            
            -- Create GUI
            local screenGui = Instance.new('ScreenGui')
            screenGui.Name = 'ToolEspGui'
            screenGui.IgnoreGuiInset = true
            screenGui.ResetOnSpawn = false
            screenGui.DisplayOrder = 50
            screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
            screenGui.Parent = lp:WaitForChild('PlayerGui')
            
            local iconLayer = Instance.new('Frame')
            iconLayer.Name = 'Icons'
            iconLayer.Size = UDim2.new(1, 0, 1, 0)
            iconLayer.BackgroundTransparency = 1
            iconLayer.Active = false
            iconLayer.Parent = screenGui
            
            ToolEspObjects.iconLayer = iconLayer
            ToolEspObjects.screenGui = screenGui
            
            -- Assets
            local assets = game:GetService('ReplicatedStorage'):FindFirstChild('Assets')
            local config = {
                OFFSET_Y = 2.6,
                MAX_DISTANCE = 500,
                SHOW_SELF = false,
                SHOW_BACKPACK = true,
                MAX_BACKPACK = 6,
                ICON_SIZE = 72,
                SMALL_SIZE = 26,
                ROW_GAP = 4,
                FADE_START = 150,
                TEAM_CHECK = true,
            }
            
            local function updatePlayer(player)
                if player == game:GetService('Players').LocalPlayer and not ToolEspSettings.ShowSelf then
                    destroyDisplay(player.Character)
                    return
                end
                if ToolEspSettings.TeamCheck and player ~= game:GetService('Players').LocalPlayer and isTeammate(player) then
                    destroyDisplay(player.Character)
                    return
                end
                updatePlayer(player, ToolEspSettings, game:GetService('ReplicatedStorage'):FindFirstChild('Assets'), ToolEspObjects.iconLayer, workspace.CurrentCamera, game:GetService('Players').LocalPlayer, game:GetService('Players').LocalPlayer)
            end
            
            local function hookPlayer(player)
                table.insert(ToolEspConnections, player.CharacterAdded:Connect(function()
                    task.wait(0.2)
                    if token.active then updatePlayer(player) end
                end))
                table.insert(ToolEspConnections, player.CharacterRemoving:Connect(function(character)
                    destroyDisplay(character)
                end))
                if player.Character then updatePlayer(player) end
            end
            
            for _, player in ipairs(game:GetService('Players'):GetPlayers()) do
                hookPlayer(player)
            end
            table.insert(ToolEspConnections, game:GetService('Players').PlayerAdded:Connect(hookPlayer))
            table.insert(ToolEspConnections, game:GetService('Players').PlayerRemoving:Connect(function(player)
                if player.Character then destroyDisplay(player.Character) end
            end))
            
            local renderConn = game:GetService('RunService').RenderStepped:Connect(function()
                if not token.active then return end
                for _, player in ipairs(game:GetService('Players'):GetPlayers()) do
                    updatePlayer(player)
                end
            end)
            
            local conn = runService.RenderStepped:Connect(function()
                for _, player in ipairs(game:GetService('Players'):GetPlayers()) do
                    updatePlayer(player)
                end
            end)
            
            ToolEsp:Clean(function()
                token.active = false
                for _, conn in ipairs(ToolEspConnections) do
                    pcall(function() conn:Disconnect() end)
                end
                ToolEspConnections = {}
                for _, player in ipairs(game:GetService('Players'):GetPlayers()) do
                    destroyDisplay(player.Character)
                end
                if screenGui then screenGui:Destroy() end
            end)
        else
            -- Disable handled by Clean function
        end
    end,
    Tooltip = 'Shows what weapons/tools enemies are holding above their heads. Uses the game\'s own 3D weapon models.'
})

-- Settings
ToolEsp:CreateToggle({
    Name = 'Team Check',
    Function = function(callback) ToolEspSettings.TeamCheck = callback end,
    Default = true,
    Tooltip = 'Hides teammates\' tool displays'
})

ToolEsp:CreateToggle({
    Name = 'Show Self',
    Function = function(callback) ToolEspSettings.ShowSelf = callback end,
    Default = false,
    Tooltip = 'Shows your own tool display'
})

ToolEsp:CreateToggle({
    Name = 'Show Equipped',
    Function = function(callback) ToolEspSettings.ShowEquipped = callback end,
    Default = true,
    Tooltip = 'Shows the weapon the enemy has equipped'
})

ToolEsp:CreateToggle({
    Name = 'Show Backpack',
    Function = function(callback) ToolEspSettings.ShowBackpack = callback end,
    Default = true,
    Tooltip = 'Shows small icons of other tools in backpack'
})

ToolEsp:CreateSlider({
    Name = 'Max Backpack Icons',
    Function = function(val) ToolEspSettings.MaxBackpack = val end,
    Min = 1,
    Max = 10,
    Default = 6,
    Tooltip = 'Maximum number of backpack tool icons to show'
})

ToolEsp:CreateSlider({
    Name = 'Max Distance',
    Function = function(val) ToolEspSettings.MaxDistance = val end,
    Min = 50,
    Max = 1000,
    Default = 500,
    Tooltip = 'Maximum distance to show tool displays'
})

ToolEsp:CreateSlider({
    Name = 'Icon Size',
    Function = function(val) 
        ToolEspSettings.IconSize = val
        ToolEspSettings.SmallSize = math.floor(val * 0.35)
    end,
    Min = 40,
    Max = 120,
    Default = 72,
    Tooltip = 'Size of the main weapon icon'
})

ToolEsp:CreateSlider({
    Name = 'Offset Y',
    Function = function(val) ToolEspSettings.OffsetY = val end,
    Min = 1,
    Max = 10,
    Default = 2.6,
    Decimal = 10,
    Tooltip = 'Vertical offset above head in studs'
})
