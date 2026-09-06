--// VISION HUD ESP & AIMBOT - AUTOSAVE & MULTI-PLACE EDITION
--// LocalScript -> StarterPlayerScripts / StarterGui

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local HttpService = game:GetService("HttpService")
local TeleportService = game:GetService("TeleportService")

local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera
local PlayerGui = LocalPlayer:WaitForChild("PlayerGui")

local mousemoverel = mousemoverel or (Input and Input.MoveMouseRelative) or function(x, y)
    Camera.CFrame = Camera.CFrame * CFrame.Angles(0, math.rad(-x * 0.15), 0) * CFrame.Angles(math.rad(-y * 0.15), 0, 0)
end

--------------------------------------------------
-- CONFIGURATION & DEFAULT VALUES
--------------------------------------------------

local FILE_NAME = "VisionHUD_Config.json"

local CONFIG = {
    -- ESP Toggles
    ESPEnabled = true,
    ShowBoxes = true,
    ShowHealth = true,
    ShowNames = true,
    ShowHighlight = true,
    MaxDistance = 1000,
    AnimationSpeed = 30,

    -- Box & Fill Customization
    BoxFillEnabled = true,
    BoxColor = Color3.fromRGB(255, 255, 255),
    BoxThickness = 1.5,
    FillColor = Color3.fromRGB(255, 70, 70),
    FillTransparency = 0.65,
    OutlineColor = Color3.fromRGB(0, 0, 0),
    OutlineTransparency = 0.2,

    -- Gradient Customization
    UseGradient = false,
    GradientColor1 = Color3.fromRGB(255, 50, 50),
    GradientColor2 = Color3.fromRGB(50, 100, 255),
    GradientRotation = 90,

    -- Text & Bar Customization
    HealthBarColor = Color3.fromRGB(80, 255, 120),
    HealthBarBg = Color3.fromRGB(20, 20, 20),
    NameColor = Color3.fromRGB(255, 255, 255),
    DistanceColor = Color3.fromRGB(190, 190, 190),

    -- Highlight Customization
    HighlightDepthMode = Enum.HighlightDepthMode.AlwaysOnTop,

    -- Aimbot Settings
    AimEnabled = false,
    AimTargetPart = "Head",
    AimFOV = 150,
    ShowFOV = true,
    FOVColor = Color3.fromRGB(255, 255, 255),
    FOVThickness = 1.5,
    AimSmoothness = 5,
    AimKey = Enum.UserInputType.MouseButton2,
    AimMode = "Hold", -- "Hold", "Toggle", "Always"
    WallCheck = true,
}

local COLOR_PRESETS = {
    Color3.fromRGB(255, 255, 255),
    Color3.fromRGB(255, 60, 60),
    Color3.fromRGB(60, 255, 120),
    Color3.fromRGB(60, 140, 255),
    Color3.fromRGB(255, 220, 60),
    Color3.fromRGB(200, 60, 255),
    Color3.fromRGB(255, 130, 0),
    Color3.fromRGB(30, 30, 30)
}

local aimToggleActive = false

--------------------------------------------------
-- SERIALIZATION & SAVE/LOAD SYSTEM
--------------------------------------------------

local function ColorToTable(color)
    return {R = color.R, G = color.G, B = color.B}
end

local function TableToColor(tbl)
    if not tbl then return Color3.fromRGB(255, 255, 255) end
    return Color3.new(tbl.R, tbl.G, tbl.B)
end

local function EnumToString(enumVal)
    if typeof(enumVal) == "EnumItem" then
        return {Type = tostring(enumVal.EnumType), Name = enumVal.Name}
    end
    return nil
end

local function StringToEnum(tbl)
    if not tbl or not tbl.Type or not tbl.Name then return Enum.UserInputType.MouseButton2 end
    if tbl.Type == "Enum.UserInputType" then
        return Enum.UserInputType[tbl.Name]
    elseif tbl.Type == "Enum.KeyCode" then
        return Enum.KeyCode[tbl.Name]
    end
    return Enum.UserInputType.MouseButton2
end

local function SaveConfig()
    if not writefile then return end
    
    local exportData = {}
    for k, v in pairs(CONFIG) do
        if typeof(v) == "Color3" then
            exportData[k] = {__type = "Color3", Value = ColorToTable(v)}
        elseif typeof(v) == "EnumItem" then
            exportData[k] = {__type = "EnumItem", Value = EnumToString(v)}
        else
            exportData[k] = v
        end
    end

    local success, json = pcall(function()
        return HttpService:JSONEncode(exportData)
    end)

    if success then
        pcall(writefile, FILE_NAME, json)
    end
end

local function LoadConfig()
    if not readfile or not isfile or not isfile(FILE_NAME) then return end

    local success, json = pcall(readfile, FILE_NAME)
    if not success or not json then return end

    local decodeSuccess, importedData = pcall(function()
        return HttpService:JSONDecode(json)
    end)

    if decodeSuccess and typeof(importedData) == "table" then
        for k, v in pairs(importedData) do
            if typeof(v) == "table" and v.__type then
                if v.__type == "Color3" then
                    CONFIG[k] = TableToColor(v.Value)
                elseif v.__type == "EnumItem" then
                    CONFIG[k] = StringToEnum(v.Value)
                end
            else
                CONFIG[k] = v
            end
        end
    end
end

LoadConfig()

--------------------------------------------------
-- GUI BUILD & BASE STRUCTURE
--------------------------------------------------

local ScreenGui = Instance.new("ScreenGui")
ScreenGui.Name = "VisionHUD_" .. math.random(1000, 9999)
ScreenGui.ResetOnSpawn = false
ScreenGui.IgnoreGuiInset = true
ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
ScreenGui.Parent = PlayerGui

local ESPFolder = Instance.new("Folder")
ESPFolder.Name = "VisionESP"
ESPFolder.Parent = workspace

-- FOV Circle
local FOVCircle = Instance.new("Frame")
FOVCircle.Name = "FOVCircle"
FOVCircle.AnchorPoint = Vector2.new(0.5, 0.5)
FOVCircle.BackgroundTransparency = 1
FOVCircle.BorderSizePixel = 0
FOVCircle.Visible = false
FOVCircle.ZIndex = 5
FOVCircle.Parent = ScreenGui

local FOVCorner = Instance.new("UICorner")
FOVCorner.CornerRadius = UDim.new(1, 0)
FOVCorner.Parent = FOVCircle

local FOVStroke = Instance.new("UIStroke")
FOVStroke.Color = CONFIG.FOVColor
FOVStroke.Thickness = CONFIG.FOVThickness
FOVStroke.Parent = FOVCircle

-- Main Window
local MainFrame = Instance.new("Frame")
MainFrame.Name = "MainFrame"
MainFrame.Size = UDim2.new(0, 560, 0, 420)
MainFrame.Position = UDim2.new(0.5, -280, 0.5, -210)
MainFrame.BackgroundColor3 = Color3.fromRGB(16, 16, 20)
MainFrame.BorderSizePixel = 0
MainFrame.Parent = ScreenGui

local MainCorner = Instance.new("UICorner")
MainCorner.CornerRadius = UDim.new(0, 10)
MainCorner.Parent = MainFrame

local MainStroke = Instance.new("UIStroke")
MainStroke.Color = Color3.fromRGB(40, 40, 50)
MainStroke.Thickness = 1.5
MainStroke.Parent = MainFrame

-- Topbar
local Topbar = Instance.new("Frame")
Topbar.Name = "Topbar"
Topbar.Size = UDim2.new(1, 0, 0, 35)
Topbar.BackgroundColor3 = Color3.fromRGB(22, 22, 28)
Topbar.BorderSizePixel = 0
Topbar.Parent = MainFrame

local TopbarCorner = Instance.new("UICorner")
TopbarCorner.CornerRadius = UDim.new(0, 10)
TopbarCorner.Parent = Topbar

local TopbarFix = Instance.new("Frame")
TopbarFix.Size = UDim2.new(1, 0, 0, 10)
TopbarFix.Position = UDim2.new(0, 0, 1, -10)
TopbarFix.BackgroundColor3 = Color3.fromRGB(22, 22, 28)
TopbarFix.BorderSizePixel = 0
TopbarFix.Parent = Topbar

local MenuTitle = Instance.new("TextLabel")
MenuTitle.Size = UDim2.new(1, -20, 1, 0)
MenuTitle.Position = UDim2.new(0, 12, 0, 0)
MenuTitle.BackgroundTransparency = 1
MenuTitle.Font = Enum.Font.GothamBold
MenuTitle.Text = "VISION HUD // AUTOSAVE SYSTEM READY"
MenuTitle.TextColor3 = Color3.fromRGB(220, 220, 235)
MenuTitle.TextSize = 11
MenuTitle.TextXAlignment = Enum.TextXAlignment.Left
MenuTitle.Parent = Topbar

--------------------------------------------------
-- DRAG & TOGGLE MECHANIC
--------------------------------------------------

local dragging, dragInput, dragStart, startPos

Topbar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = MainFrame.Position

        input.Changed:Connect(function()
            if input.UserInputState == Enum.UserInputState.End then
                dragging = false
            end
        end)
    end
end)

Topbar.InputChanged:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseMovement then
        dragInput = input
    end
end)

UserInputService.InputChanged:Connect(function(input)
    if input == dragInput and dragging then
        local delta = input.Position - dragStart
        MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)

local menuVisible = true
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if input.KeyCode == Enum.KeyCode.Insert then
        menuVisible = not menuVisible
        MainFrame.Visible = menuVisible
    end
end)

--------------------------------------------------
-- UI CONTROLS & PAGES SETUP
--------------------------------------------------

local Sidebar = Instance.new("Frame")
Sidebar.Size = UDim2.new(0, 120, 1, -35)
Sidebar.Position = UDim2.new(0, 0, 0, 35)
Sidebar.BackgroundColor3 = Color3.fromRGB(19, 19, 24)
Sidebar.BorderSizePixel = 0
Sidebar.Parent = MainFrame

local TabContainer = Instance.new("Frame")
TabContainer.Size = UDim2.new(1, -130, 1, -45)
TabContainer.Position = UDim2.new(0, 125, 0, 40)
TabContainer.BackgroundTransparency = 1
TabContainer.Parent = MainFrame

local function CreateTabBtn(text, pos)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -10, 0, 30)
    btn.Position = UDim2.new(0, 5, 0, pos)
    btn.BackgroundColor3 = Color3.fromRGB(28, 28, 36)
    btn.BorderSizePixel = 0
    btn.Font = Enum.Font.GothamBold
    btn.Text = text
    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
    btn.TextSize = 10
    btn.Parent = Sidebar

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = btn

    return btn
end

local ESPMainBtn = CreateTabBtn("ESP MAIN", 8)
local ESPStyleBtn = CreateTabBtn("BOX & GRADIENT", 42)
local ColorsBtn = CreateTabBtn("COLORS & TEXT", 76)
local AimTabBtn = CreateTabBtn("AIMBOT", 110)

local function CreatePage()
    local page = Instance.new("ScrollingFrame")
    page.Size = UDim2.new(1, 0, 1, 0)
    page.BackgroundTransparency = 1
    page.BorderSizePixel = 0
    page.ScrollBarThickness = 3
    page.ScrollBarImageColor3 = Color3.fromRGB(70, 70, 90)
    page.Visible = false
    page.Parent = TabContainer

    local layout = Instance.new("UIListLayout")
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Padding = UDim.new(0, 6)
    layout.Parent = page

    return page
end

local ESPMainPage = CreatePage()
local ESPStylePage = CreatePage()
local ColorsPage = CreatePage()
local AimPage = CreatePage()

ESPMainPage.Visible = true

local function SwitchTab(targetPage)
    ESPMainPage.Visible = false
    ESPStylePage.Visible = false
    ColorsPage.Visible = false
    AimPage.Visible = false
    targetPage.Visible = true
end

ESPMainBtn.MouseButton1Click:Connect(function() SwitchTab(ESPMainPage) end)
ESPStyleBtn.MouseButton1Click:Connect(function() SwitchTab(ESPStylePage) end)
ColorsBtn.MouseButton1Click:Connect(function() SwitchTab(ColorsPage) end)
AimTabBtn.MouseButton1Click:Connect(function() SwitchTab(AimPage) end)

--------------------------------------------------
-- UI COMPONENT CREATORS (WITH AUTOSAVE)
--------------------------------------------------

local function CreateToggle(parent, text, defaultConfig, callback)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -10, 0, 32)
    frame.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
    frame.BorderSizePixel = 0
    frame.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.7, 0, 1, 0)
    label.Position = UDim2.new(0, 10, 0, 0)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.Gotham
    label.Text = text
    label.TextColor3 = Color3.fromRGB(200, 200, 210)
    label.TextSize = 11
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0, 38, 0, 18)
    button.Position = UDim2.new(1, -48, 0.5, -9)
    button.BackgroundColor3 = defaultConfig and Color3.fromRGB(60, 200, 100) or Color3.fromRGB(50, 50, 60)
    button.BorderSizePixel = 0
    button.Text = ""
    button.Parent = frame

    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(1, 0)
    btnCorner.Parent = button

    local state = defaultConfig
    button.MouseButton1Click:Connect(function()
        state = not state
        TweenService:Create(button, TweenInfo.new(0.2), {
            BackgroundColor3 = state and Color3.fromRGB(60, 200, 100) or Color3.fromRGB(50, 50, 60)
        }):Play()
        callback(state)
        SaveConfig()
    end)
end

local function CreateSlider(parent, text, min, max, default, callback)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -10, 0, 42)
    frame.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
    frame.BorderSizePixel = 0
    frame.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1, -20, 0, 18)
    label.Position = UDim2.new(0, 10, 0, 3)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.Gotham
    label.Text = text .. ": " .. tostring(default)
    label.TextColor3 = Color3.fromRGB(200, 200, 210)
    label.TextSize = 11
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local sliderBg = Instance.new("Frame")
    sliderBg.Size = UDim2.new(1, -20, 0, 5)
    sliderBg.Position = UDim2.new(0, 10, 0, 26)
    sliderBg.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
    sliderBg.BorderSizePixel = 0
    sliderBg.Parent = frame

    local sliderBgCorner = Instance.new("UICorner")
    sliderBgCorner.CornerRadius = UDim.new(1, 0)
    sliderBgCorner.Parent = sliderBg

    local sliderFill = Instance.new("Frame")
    sliderFill.Size = UDim2.new((default - min) / (max - min), 0, 1, 0)
    sliderFill.BackgroundColor3 = Color3.fromRGB(80, 140, 255)
    sliderFill.BorderSizePixel = 0
    sliderFill.Parent = sliderBg

    local sliderFillCorner = Instance.new("UICorner")
    sliderFillCorner.CornerRadius = UDim.new(1, 0)
    sliderFillCorner.Parent = sliderFill

    local isSliding = false
    local function updateSlider(input)
        local posX = math.clamp((input.Position.X - sliderBg.AbsolutePosition.X) / sliderBg.AbsoluteSize.X, 0, 1)
        local val = math.floor((min + (max - min) * posX) * 100) / 100
        sliderFill.Size = UDim2.new(posX, 0, 1, 0)
        label.Text = text .. ": " .. tostring(val)
        callback(val)
    end

    sliderBg.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            isSliding = true
            updateSlider(input)
        end
    end)

    UserInputService.InputChanged:Connect(function(input)
        if isSliding and input.UserInputType == Enum.UserInputType.MouseMovement then
            updateSlider(input)
        end
    end)

    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            isSliding = false
            SaveConfig()
        end
    end)
end

local function CreateColorPicker(parent, text, defaultColor, callback)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -10, 0, 32)
    frame.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
    frame.BorderSizePixel = 0
    frame.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.5, 0, 1, 0)
    label.Position = UDim2.new(0, 10, 0, 0)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.Gotham
    label.Text = text
    label.TextColor3 = Color3.fromRGB(200, 200, 210)
    label.TextSize = 11
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local previewBtn = Instance.new("TextButton")
    previewBtn.Size = UDim2.new(0, 24, 0, 18)
    previewBtn.Position = UDim2.new(1, -34, 0.5, -9)
    previewBtn.BackgroundColor3 = defaultColor
    previewBtn.BorderSizePixel = 0
    previewBtn.Text = ""
    previewBtn.Parent = frame

    local prevCorner = Instance.new("UICorner")
    prevCorner.CornerRadius = UDim.new(0, 4)
    prevCorner.Parent = previewBtn

    local colorIdx = 1
    previewBtn.MouseButton1Click:Connect(function()
        colorIdx = (colorIdx % #COLOR_PRESETS) + 1
        local selectedColor = COLOR_PRESETS[colorIdx]
        previewBtn.BackgroundColor3 = selectedColor
        callback(selectedColor)
        SaveConfig()
    end)
end

local function CreateKeybind(parent, text, defaultKey, callback)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -10, 0, 32)
    frame.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
    frame.BorderSizePixel = 0
    frame.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.5, 0, 1, 0)
    label.Position = UDim2.new(0, 10, 0, 0)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.Gotham
    label.Text = text
    label.TextColor3 = Color3.fromRGB(200, 200, 210)
    label.TextSize = 11
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0, 90, 0, 20)
    button.Position = UDim2.new(1, -100, 0.5, -10)
    button.BackgroundColor3 = Color3.fromRGB(40, 40, 52)
    button.BorderSizePixel = 0
    
    local function GetKeyName(key)
        if typeof(key) == "EnumItem" then
            if key.EnumType == Enum.UserInputType then
                if key == Enum.UserInputType.MouseButton1 then return "MB1" end
                if key == Enum.UserInputType.MouseButton2 then return "MB2" end
                if key == Enum.UserInputType.MouseButton3 then return "MB3" end
            end
            return key.Name
        end
        return "None"
    end

    button.Text = GetKeyName(defaultKey)
    button.Font = Enum.Font.GothamBold
    button.TextColor3 = Color3.fromRGB(255, 255, 255)
    button.TextSize = 10
    button.Parent = frame

    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 4)
    btnCorner.Parent = button

    local listening = false
    button.MouseButton1Click:Connect(function()
        if listening then return end
        listening = true
        button.Text = "..."
        button.BackgroundColor3 = Color3.fromRGB(80, 140, 255)

        local connection
        connection = UserInputService.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.Keyboard then
                if input.KeyCode ~= Enum.KeyCode.Unknown and input.KeyCode ~= Enum.KeyCode.Insert then
                    CONFIG.AimKey = input.KeyCode
                    button.Text = GetKeyName(input.KeyCode)
                    listening = false
                    button.BackgroundColor3 = Color3.fromRGB(40, 40, 52)
                    connection:Disconnect()
                    callback(input.KeyCode)
                    SaveConfig()
                end
            elseif input.UserInputType == Enum.UserInputType.MouseButton1 
                or input.UserInputType == Enum.UserInputType.MouseButton2 
                or input.UserInputType == Enum.UserInputType.MouseButton3 then
                
                CONFIG.AimKey = input.UserInputType
                button.Text = GetKeyName(input.UserInputType)
                listening = false
                button.BackgroundColor3 = Color3.fromRGB(40, 40, 52)
                connection:Disconnect()
                callback(input.UserInputType)
                SaveConfig()
            end
        end)
    end)
end

local function CreateDropdown(parent, text, options, defaultOption, callback)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, -10, 0, 32)
    frame.BackgroundColor3 = Color3.fromRGB(24, 24, 30)
    frame.BorderSizePixel = 0
    frame.Parent = parent

    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 6)
    corner.Parent = frame

    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(0.5, 0, 1, 0)
    label.Position = UDim2.new(0, 10, 0, 0)
    label.BackgroundTransparency = 1
    label.Font = Enum.Font.Gotham
    label.Text = text
    label.TextColor3 = Color3.fromRGB(200, 200, 210)
    label.TextSize = 11
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Parent = frame

    local button = Instance.new("TextButton")
    button.Size = UDim2.new(0, 90, 0, 20)
    button.Position = UDim2.new(1, -100, 0.5, -10)
    button.BackgroundColor3 = Color3.fromRGB(40, 40, 52)
    button.BorderSizePixel = 0
    button.Font = Enum.Font.GothamBold
    button.Text = tostring(defaultOption)
    button.TextColor3 = Color3.fromRGB(255, 255, 255)
    button.TextSize = 10
    button.Parent = frame

    local btnCorner = Instance.new("UICorner")
    btnCorner.CornerRadius = UDim.new(0, 4)
    btnCorner.Parent = button

    local currentIndex = table.find(options, defaultOption) or 1
    button.MouseButton1Click:Connect(function()
        currentIndex = (currentIndex % #options) + 1
        local selected = options[currentIndex]
        button.Text = tostring(selected)
        callback(selected)
        SaveConfig()
    end)
end

--------------------------------------------------
-- REGISTER UI CONTROLS
--------------------------------------------------

-- Tab 1: ESP Main
CreateToggle(ESPMainPage, "Master ESP Toggle", CONFIG.ESPEnabled, function(v) CONFIG.ESPEnabled = v end)
CreateToggle(ESPMainPage, "Show Boxes", CONFIG.ShowBoxes, function(v) CONFIG.ShowBoxes = v end)
CreateToggle(ESPMainPage, "Show Health Bars", CONFIG.ShowHealth, function(v) CONFIG.ShowHealth = v end)
CreateToggle(ESPMainPage, "Show Names & Distance", CONFIG.ShowNames, function(v) CONFIG.ShowNames = v end)
CreateToggle(ESPMainPage, "Show Highlight (Chams)", CONFIG.ShowHighlight, function(v) CONFIG.ShowHighlight = v end)
CreateSlider(ESPMainPage, "Max Distance", 100, 3000, CONFIG.MaxDistance, function(v) CONFIG.MaxDistance = v end)

-- Tab 2: Box & Gradient Customization
CreateToggle(ESPStylePage, "Enable Box Fill", CONFIG.BoxFillEnabled, function(v) CONFIG.BoxFillEnabled = v end)
CreateToggle(ESPStylePage, "Enable Custom Gradient", CONFIG.UseGradient, function(v) CONFIG.UseGradient = v end)
CreateSlider(ESPStylePage, "Fill Transparency", 0, 1, CONFIG.FillTransparency, function(v) CONFIG.FillTransparency = v end)
CreateSlider(ESPStylePage, "Box Thickness", 1, 5, CONFIG.BoxThickness, function(v) CONFIG.BoxThickness = v end)
CreateSlider(ESPStylePage, "Gradient Angle", 0, 360, CONFIG.GradientRotation, function(v) CONFIG.GradientRotation = v end)
CreateColorPicker(ESPStylePage, "Gradient Color 1", CONFIG.GradientColor1, function(c) CONFIG.GradientColor1 = c end)
CreateColorPicker(ESPStylePage, "Gradient Color 2", CONFIG.GradientColor2, function(c) CONFIG.GradientColor2 = c end)

-- Tab 3: Colors & Text
CreateColorPicker(ColorsPage, "Box Border Color", CONFIG.BoxColor, function(c) CONFIG.BoxColor = c end)
CreateColorPicker(ColorsPage, "Box Fill Color", CONFIG.FillColor, function(c) CONFIG.FillColor = c end)
CreateColorPicker(ColorsPage, "Name Text Color", CONFIG.NameColor, function(c) CONFIG.NameColor = c end)
CreateColorPicker(ColorsPage, "Distance Color", CONFIG.DistanceColor, function(c) CONFIG.DistanceColor = c end)

-- Tab 4: Aimbot
CreateToggle(AimPage, "Master Aimbot", CONFIG.AimEnabled, function(v) CONFIG.AimEnabled = v end)
CreateToggle(AimPage, "Show FOV Circle", CONFIG.ShowFOV, function(v) CONFIG.ShowFOV = v end)
CreateToggle(AimPage, "Wall Check (Visible Only)", CONFIG.WallCheck, function(v) CONFIG.WallCheck = v end)
CreateKeybind(AimPage, "Aim Keybind", CONFIG.AimKey, function(k) CONFIG.AimKey = k end)
CreateDropdown(AimPage, "Aim Mode", {"Hold", "Toggle", "Always"}, CONFIG.AimMode, function(mode) 
    CONFIG.AimMode = mode 
    aimToggleActive = false 
end)
CreateSlider(AimPage, "FOV Radius", 30, 500, CONFIG.AimFOV, function(v) CONFIG.AimFOV = v end)
CreateSlider(AimPage, "Smoothness", 1, 20, CONFIG.AimSmoothness, function(v) CONFIG.AimSmoothness = v end)
CreateColorPicker(AimPage, "FOV Circle Color", CONFIG.FOVColor, function(c) CONFIG.FOVColor = c end)

--------------------------------------------------
-- ESP & RENDERING LOGIC
--------------------------------------------------

local function IsCharacterValid(character)
    return character and character.Parent and character:IsDescendantOf(workspace)
end

local function GetHumanoid(character)
    return character and character:FindFirstChildOfClass("Humanoid")
end

local function GetRoot(character)
    return character and character:FindFirstChild("HumanoidRootPart")
end

local function GetBoundingBox2D(character)
    local hrp = GetRoot(character)
    if not hrp then return nil end

    local hrpPos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
    if not onScreen or hrpPos.Z <= 0 then return nil end

    local characterHeight3D = 5.6
    local characterWidth3D = 3.2

    local viewportSize = Camera.ViewportSize
    local fov = math.rad(Camera.FieldOfView)
    local distance = hrpPos.Z

    local height = (characterHeight3D / (math.tan(fov / 2) * distance * 2)) * viewportSize.Y
    local width = height * (characterWidth3D / characterHeight3D)

    return {
        Position = Vector2.new(hrpPos.X - width / 2, hrpPos.Y - height / 2),
        Size = Vector2.new(width, height)
    }
end

local ESPCache = {}

local function CreateESP(player)
    if ESPCache[player] then return ESPCache[player] end

    local boxFrame = Instance.new("Frame")
    boxFrame.Name = "Box"
    boxFrame.BackgroundColor3 = CONFIG.FillColor
    boxFrame.BackgroundTransparency = CONFIG.FillTransparency
    boxFrame.BorderSizePixel = 0
    boxFrame.Visible = false
    boxFrame.ZIndex = 10
    boxFrame.Parent = ScreenGui

    local uiGradient = Instance.new("UIGradient")
    uiGradient.Name = "CustomGradient"
    uiGradient.Rotation = CONFIG.GradientRotation
    uiGradient.Enabled = CONFIG.UseGradient
    uiGradient.Parent = boxFrame

    local boxStroke = Instance.new("UIStroke")
    boxStroke.Color = CONFIG.BoxColor
    boxStroke.Thickness = CONFIG.BoxThickness
    boxStroke.Transparency = 0
    boxStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    boxStroke.Parent = boxFrame

    local healthBg = Instance.new("Frame")
    healthBg.Name = "HealthBackground"
    healthBg.AnchorPoint = Vector2.new(1, 0)
    healthBg.Position = UDim2.new(0, -5, 0, 0)
    healthBg.Size = UDim2.new(0, 4, 1, 0)
    healthBg.BackgroundColor3 = CONFIG.HealthBarBg
    healthBg.BackgroundTransparency = 0.2
    healthBg.BorderSizePixel = 0
    healthBg.Visible = false
    healthBg.ZIndex = 12
    healthBg.Parent = boxFrame

    local healthBgCorner = Instance.new("UICorner")
    healthBgCorner.CornerRadius = UDim.new(0, 2)
    healthBgCorner.Parent = healthBg

    local healthFill = Instance.new("Frame")
    healthFill.Name = "Health"
    healthFill.AnchorPoint = Vector2.new(0, 1)
    healthFill.Position = UDim2.new(0, 0, 1, 0)
    healthFill.Size = UDim2.new(1, 0, 1, 0)
    healthFill.BackgroundColor3 = CONFIG.HealthBarColor
    healthFill.BorderSizePixel = 0
    healthFill.ZIndex = 13
    healthFill.Parent = healthBg

    local healthCorner = Instance.new("UICorner")
    healthCorner.CornerRadius = UDim.new(0, 2)
    healthCorner.Parent = healthFill

    local nameLabel = Instance.new("TextLabel")
    nameLabel.Name = "Name"
    nameLabel.AnchorPoint = Vector2.new(0.5, 1)
    nameLabel.Position = UDim2.new(0.5, 0, 0, -5)
    nameLabel.Size = UDim2.fromOffset(250, 22)
    nameLabel.BackgroundTransparency = 1
    nameLabel.Text = ""
    nameLabel.TextColor3 = CONFIG.NameColor
    nameLabel.TextSize = 14
    nameLabel.Font = Enum.Font.GothamBold
    nameLabel.TextStrokeTransparency = 0.25
    nameLabel.Visible = false
    nameLabel.ZIndex = 20
    nameLabel.Parent = boxFrame

    local distanceLabel = Instance.new("TextLabel")
    distanceLabel.Name = "Distance"
    distanceLabel.AnchorPoint = Vector2.new(0.5, 0)
    distanceLabel.Position = UDim2.new(0.5, 0, 1, 5)
    distanceLabel.Size = UDim2.fromOffset(250, 18)
    distanceLabel.BackgroundTransparency = 1
    distanceLabel.Text = ""
    distanceLabel.TextColor3 = CONFIG.DistanceColor
    distanceLabel.TextSize = 11
    distanceLabel.Font = Enum.Font.GothamMedium
    distanceLabel.TextStrokeTransparency = 0.4
    distanceLabel.Visible = false
    distanceLabel.ZIndex = 20
    distanceLabel.Parent = boxFrame

    local highlight = Instance.new("Highlight")
    highlight.Name = "VisionHighlight"
    highlight.Adornee = nil
    highlight.DepthMode = CONFIG.HighlightDepthMode
    highlight.FillColor = CONFIG.FillColor
    highlight.FillTransparency = CONFIG.FillTransparency
    highlight.OutlineColor = CONFIG.BoxColor
    highlight.OutlineTransparency = 0
    highlight.Enabled = false
    highlight.Parent = ESPFolder

    local data = {
        Player = player,
        Box = boxFrame,
        Gradient = uiGradient,
        BoxStroke = boxStroke,
        HealthBg = healthBg,
        HealthFill = healthFill,
        Name = nameLabel,
        Distance = distanceLabel,
        Highlight = highlight,
        CurrentPos = nil,
        CurrentSize = nil,
    }

    ESPCache[player] = data
    return data
end

local function HideESP(data)
    if not data then return end
    data.Box.Visible = false
    data.Name.Visible = false
    data.Distance.Visible = false
    data.HealthBg.Visible = false
    if data.Highlight then
        data.Highlight.Enabled = false
        data.Highlight.Adornee = nil
    end
end

local function RemoveESP(player)
    local data = ESPCache[player]
    if not data then return end
    if data.Box then data.Box:Destroy() end
    if data.Highlight then data.Highlight:Destroy() end
    ESPCache[player] = nil
end

--------------------------------------------------
-- AIMBOT & RAYCAST LOGIC
--------------------------------------------------

local function IsPartVisible(part, character)
    if not CONFIG.WallCheck then return true end
    local ignoreList = {Camera, LocalPlayer.Character, character}
    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = ignoreList

    local ray = workspace:Raycast(Camera.CFrame.Position, part.Position - Camera.CFrame.Position, params)
    return ray == nil
end

local function GetClosestPlayer()
    local closestPlayer = nil
    local shortestDistance = CONFIG.AimFOV
    local mousePos = UserInputService:GetMouseLocation()

    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local char = player.Character
            if IsCharacterValid(char) then
                local hum = GetHumanoid(char)
                local targetPart = char:FindFirstChild(CONFIG.AimTargetPart) or GetRoot(char)

                if hum and hum.Health > 0 and targetPart then
                    local screenPos, onScreen = Camera:WorldToViewportPoint(targetPart.Position)
                    if onScreen then
                        local distance = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                        if distance < shortestDistance then
                            if IsPartVisible(targetPart, char) then
                                shortestDistance = distance
                                closestPlayer = targetPart
                            end
                        end
                    end
                end
            end
        end
    end

    return closestPlayer
end

-- Обработка нажатий клавиш для переключения режима Toggle
UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    if CONFIG.AimMode == "Toggle" and typeof(CONFIG.AimKey) == "EnumItem" then
        if CONFIG.AimKey.EnumType == Enum.UserInputType then
            if input.UserInputType == CONFIG.AimKey then
                aimToggleActive = not aimToggleActive
            end
        elseif CONFIG.AimKey.EnumType == Enum.KeyCode then
            if input.KeyCode == CONFIG.AimKey then
                aimToggleActive = not aimToggleActive
            end
        end
    end
end)

-- Проверка условия активации аима
local function ShouldAimBeActive()
    if CONFIG.AimMode == "Always" then
        return true
    elseif CONFIG.AimMode == "Toggle" then
        return aimToggleActive
    elseif CONFIG.AimMode == "Hold" then
        if typeof(CONFIG.AimKey) == "EnumItem" then
            if CONFIG.AimKey.EnumType == Enum.UserInputType then
                return UserInputService:IsMouseButtonPressed(CONFIG.AimKey)
            elseif CONFIG.AimKey.EnumType == Enum.KeyCode then
                return UserInputService:IsKeyDown(CONFIG.AimKey)
            end
        end
    end
    return false
end

--------------------------------------------------
-- RENDER STEPS
--------------------------------------------------

local function UpdateESP(player, data, dt)
    if not CONFIG.ESPEnabled or player == LocalPlayer then
        HideESP(data)
        return
    end

    local character = player.Character
    if not IsCharacterValid(character) then
        HideESP(data)
        return
    end

    local humanoid = GetHumanoid(character)
    local root = GetRoot(character)

    if not humanoid or not root or humanoid.Health <= 0 then
        HideESP(data)
        return
    end

    local distance = (Camera.CFrame.Position - root.Position).Magnitude
    if distance > CONFIG.MaxDistance then
        HideESP(data)
        return
    end

    local box2D = GetBoundingBox2D(character)
    if not box2D then
        HideESP(data)
        return
    end

    local smooth = 1 - math.exp(-CONFIG.AnimationSpeed * dt)
    if not data.CurrentPos then data.CurrentPos = box2D.Position end
    if not data.CurrentSize then data.CurrentSize = box2D.Size end

    data.CurrentPos = data.CurrentPos:Lerp(box2D.Position, smooth)
    data.CurrentSize = data.CurrentSize:Lerp(box2D.Size, smooth)

    if CONFIG.ShowBoxes then
        data.Box.Visible = true
        data.Box.Position = UDim2.fromOffset(math.floor(data.CurrentPos.X + 0.5), math.floor(data.CurrentPos.Y + 0.5))
        data.Box.Size = UDim2.fromOffset(math.floor(data.CurrentSize.X + 0.5), math.floor(data.CurrentSize.Y + 0.5))
        data.Box.BackgroundColor3 = CONFIG.FillColor
        data.Box.BackgroundTransparency = CONFIG.BoxFillEnabled and CONFIG.FillTransparency or 1
        data.BoxStroke.Color = CONFIG.BoxColor
        data.BoxStroke.Thickness = CONFIG.BoxThickness

        data.Gradient.Enabled = CONFIG.UseGradient and CONFIG.BoxFillEnabled
        data.Gradient.Rotation = CONFIG.GradientRotation
        data.Gradient.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, CONFIG.GradientColor1),
            ColorSequenceKeypoint.new(1, CONFIG.GradientColor2)
        })
    else
        data.Box.Visible = false
    end

    if CONFIG.ShowHealth then
        data.HealthBg.Visible = true
        local maxHealth = math.max(humanoid.MaxHealth, 1)
        local health = math.clamp(humanoid.Health / maxHealth, 0, 1)
        data.HealthFill.Size = UDim2.new(1, 0, health, 0)
    else
        data.HealthBg.Visible = false
    end

    if CONFIG.ShowNames then
        data.Name.Visible = true
        data.Name.TextColor3 = CONFIG.NameColor
        data.Name.Text = player.DisplayName

        data.Distance.Visible = true
        data.Distance.TextColor3 = CONFIG.DistanceColor
        data.Distance.Text = string.format("%dm", math.floor(distance + 0.5))
    else
        data.Name.Visible = false
        data.Distance.Visible = false
    end

    if data.Highlight then
        data.Highlight.Enabled = CONFIG.ShowHighlight
        data.Highlight.Adornee = character
        data.Highlight.FillColor = CONFIG.FillColor
        data.Highlight.FillTransparency = CONFIG.FillTransparency
        data.Highlight.OutlineColor = CONFIG.BoxColor
    end
end

--------------------------------------------------
-- LIFECYCLE LISTENERS & TELEPORTATION HANDLING
--------------------------------------------------

TeleportService.TeleportInitFailed:Connect(SaveConfig)
game:GetService("Players").LocalPlayer.OnTeleport:Connect(function(State)
    if State == Enum.TeleportState.Started then
        SaveConfig()
    end
end)

Players.PlayerAdded:Connect(function(player)
    CreateESP(player)
end)

Players.PlayerRemoving:Connect(RemoveESP)

for _, player in ipairs(Players:GetPlayers()) do
    if player ~= LocalPlayer then
        CreateESP(player)
    end
end

RunService.RenderStepped:Connect(function(dt)
    Camera = workspace.CurrentCamera
    if not Camera then return end

    if CONFIG.ShowFOV and CONFIG.AimEnabled then
        local mousePos = UserInputService:GetMouseLocation()
        FOVCircle.Visible = true
        FOVCircle.Size = UDim2.fromOffset(CONFIG.AimFOV * 2, CONFIG.AimFOV * 2)
        FOVCircle.Position = UDim2.fromOffset(mousePos.X, mousePos.Y)
        FOVStroke.Color = CONFIG.FOVColor
    else
        FOVCircle.Visible = false
    end

    for player, data in pairs(ESPCache) do
        if player.Parent == Players then
            UpdateESP(player, data, dt)
        else
            RemoveESP(player)
        end
    end

    if CONFIG.AimEnabled and ShouldAimBeActive() then
        local targetPart = GetClosestPlayer()
        if targetPart then
            local targetScreenPos = Camera:WorldToViewportPoint(targetPart.Position)
            local mousePos = UserInputService:GetMouseLocation()

            local deltaX = (targetScreenPos.X - mousePos.X) / CONFIG.AimSmoothness
            local deltaY = (targetScreenPos.Y - mousePos.Y) / CONFIG.AimSmoothness

            mousemoverel(deltaX, deltaY)
        end
    end
end)
