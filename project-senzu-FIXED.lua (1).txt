-- First statement. This callback is public, readable code.
flow_preinit(function()
    -- Finish early setup here before returning.
    print("Preparing script")
end)

-- Optional: a separately protected key UI can follow.
flow_init(function(currentKey)
    return currentKey
end)

-- Your authenticated script begins here.
print("Access granted")

-- peler
local HttpService = game:GetService("HttpService")

pcall(function()
    if not isfolder("ProjectSenzu") then
        makefolder("ProjectSenzu")
    end
    if not isfolder("ProjectSenzu/Config") then
        makefolder("ProjectSenzu/Config")
    end
end)

local univId = tostring(game.GameId)
if univId == "0" or univId == "" then
    univId = tostring(game.PlaceId)
end

local ConfigFile = "ProjectSenzu/Config/ProjectSenzu_" .. univId .. ".json"

-- State library: sekarang LOCAL (dulu global -> bisa bentrok dengan script lain / window lain).
-- Satu-satunya global yang sengaja dipertahankan: notif() (dipakai script pemakai).
local ConfigData       = {}
local Elements         = {}
local CURRENT_VERSION  = nil

-- Simpan config dengan DEBOUNCE: banyak perubahan beruntun (mis. geser slider) cukup 1x writefile.
local SaveQueued = false
local function WriteConfigNow()
    SaveQueued = false
    if writefile then
        ConfigData._version = CURRENT_VERSION
        pcall(function()
            writefile(ConfigFile, HttpService:JSONEncode(ConfigData))
        end)
    end
end

local function SaveConfig()
    if SaveQueued then return end
    SaveQueued = true
    task.delay(0.5, WriteConfigNow)
end

-- Kunci config per elemen: "<Tipe>_<Flag atau Judul>". Judul yang sama tidak lagi saling menimpa:
-- yang pertama tetap "<Tipe>_<Judul>" (config lama tetap terbaca), berikutnya diberi akhiran "#2", "#3", ...
-- Mau kunci yang tetap walau judul diganti? Pakai parameter opsional  Flag = "NamaUnik"  di elemen.
local KeyUsed = {}
local function MakeKey(prefix, cfg)
    local base = prefix .. "_" .. tostring(cfg.Flag or cfg.Title)
    local n = (KeyUsed[base] or 0) + 1
    KeyUsed[base] = n
    if n == 1 then return base end
    -- judul ganda: kunci config dipisah otomatis (tanpa log). Pakai Flag = "..." supaya kunci tetap walau judul diganti.
    return base .. "#" .. n
end

-- Helper pembuat Instance: New("UICorner", { CornerRadius = UDim.new(0, 4) }, parent)
-- Properti diset dulu, Parent terakhir.
local function New(class, props, parent)
    local inst = Instance.new(class)
    for k, v in pairs(props or {}) do
        if k ~= "Parent" then inst[k] = v end
    end
    if parent then inst.Parent = parent end
    return inst
end

-- Auto-tinggi label deskripsi (wrap per kata). Tinggi diambil dari TextBounds.Y hasil wrap sebenarnya,
-- jadi tidak kepotong di HP / window sempit. apply(h) = atur tinggi frame induk,
-- onResized = dipanggil (di thread terpisah) tiap ukuran berubah setelah build awal.
local PendingResize = setmetatable({}, { __mode = "k" })
local function NotifyResized(fn)
    -- gabungkan banyak pemanggilan beruntun jadi satu (tiap section cukup hitung ulang sekali)
    if PendingResize[fn] then return end
    PendingResize[fn] = true
    task.delay(0.05, function()
        PendingResize[fn] = nil
        task.spawn(fn)
    end)
end

local function FitWrapLabel(label, minH, apply, onResized)
    label.TextWrapped = true
    label.TextTruncate = Enum.TextTruncate.None
    local last = -1
    local function refit(notify)
        if label.AbsoluteSize.X < 1 then return end -- belum ke-layout, tunggu signal berikutnya
        local h = math.max(minH, math.ceil(label.TextBounds.Y))
        if h == last then return end
        last = h
        label.Size = UDim2.new(label.Size.X.Scale, label.Size.X.Offset, 0, h)
        apply(h)
        if notify and onResized then NotifyResized(onResized) end
    end
    label:GetPropertyChangedSignal("TextBounds"):Connect(function() refit(true) end)
    label:GetPropertyChangedSignal("AbsoluteSize"):Connect(function() refit(true) end)
    refit(false)
    -- Tinggi item berubah SETELAH section sempat menghitung tingginya (item baru masuk dengan tinggi default).
    -- Tanpa ini section tetap pendek dan deskripsi panjang kepotong sampai window diperbesar.
    -- Jadi: ukur ulang setelah layout selesai, lalu suruh section menghitung ulang (sekali, walau tinggi tidak berubah).
    if onResized then
        task.defer(function()
            refit(false)
            NotifyResized(onResized)
        end)
    end
end

local function LoadConfigFromFile()
    if not CURRENT_VERSION then return end
    
    local isFileSuccess, isFileResult = false, false
    if isfile then
        isFileSuccess, isFileResult = pcall(function() return isfile(ConfigFile) end)
    end

    if isFileSuccess and isFileResult then
        local success, result = pcall(function()
            return HttpService:JSONDecode(readfile(ConfigFile))
        end)
        if success and type(result) == "table" then
            if result._version == CURRENT_VERSION then
                ConfigData = result
            else
                ConfigData = { _version = CURRENT_VERSION }
            end
        else
            ConfigData = { _version = CURRENT_VERSION }
        end
    else
        ConfigData = { _version = CURRENT_VERSION }
    end
end

local function LoadConfigElements()
    for key, element in pairs(Elements) do
        if ConfigData[key] ~= nil and element.Set then
            element:Set(ConfigData[key], true)
        end
    end
end

-- ============================================================
-- CONFIG PROFILE SYSTEM
-- ============================================================
local PROFILE_FOLDER = "ProjectSenzu/Profiles"

local function EnsureProfileFolder()
    pcall(function()
        if not isfolder("ProjectSenzu") then makefolder("ProjectSenzu") end
        if not isfolder("ProjectSenzu/Config") then makefolder("ProjectSenzu/Config") end
        if not isfolder(PROFILE_FOLDER) then makefolder(PROFILE_FOLDER) end
    end)
end

local function GetProfileList()
    local profiles = {}
    pcall(function()
        EnsureProfileFolder()
        local files = listfiles(PROFILE_FOLDER)
        for _, filePath in ipairs(files) do
            local normalized = tostring(filePath):gsub("\\", "/")
            local fileName = normalized:match("([^/]+)$") or ""
            if fileName:match("%.json$") then
                table.insert(profiles, (fileName:gsub("%.json$", "")))
            end
        end
    end)
    table.sort(profiles)
    return profiles
end

local function SaveProfile(name)
    if not name or name == "" then return false end
    EnsureProfileFolder()
    local ok, err = pcall(function()
        writefile(PROFILE_FOLDER .. "/" .. name .. ".json", HttpService:JSONEncode(ConfigData))
    end)
    return ok, err
end

local function LoadProfile(name)
    if not name or name == "" then return false end
    local path = PROFILE_FOLDER .. "/" .. name .. ".json"
    if not isfile then return false end
    local isFileSuccess, isFileResult = pcall(function() return isfile(path) end)
    if not isFileSuccess or not isFileResult then return false end
    local ok, data = pcall(function()
        return HttpService:JSONDecode(readfile(path))
    end)
    if ok and type(data) == "table" then
        for key, val in pairs(data) do
            ConfigData[key] = val
        end
        for key, element in pairs(Elements) do
            if ConfigData[key] ~= nil and element.Set then
                pcall(function()
                    element:Set(ConfigData[key])
                end)
                task.wait(0.03)
            end
        end
        SaveConfig()
        return true
    end
    return false
end

local function DeleteProfile(name)
    if not name or name == "" then return false end
    local path = PROFILE_FOLDER .. "/" .. name .. ".json"
    local ok = pcall(function()
        if isfile(path) then delfile(path) end
    end)
    return ok
end

local Icons = {
    player    = "rbxassetid://12120698352",
    web       = "rbxassetid://137601480983962",
    bag       = "rbxassetid://8601111810",
    shop      = "rbxassetid://4985385964",
    cart      = "rbxassetid://128874923961846",
    plug      = "rbxassetid://137601480983962",
    settings  = "rbxassetid://70386228443175",
    loop      = "rbxassetid://122032243989747",
    gps       = "rbxassetid://17824309485",
    compas    = "rbxassetid://125300760963399",
    gamepad   = "rbxassetid://84173963561612",
    boss      = "rbxassetid://13132186360",
    scroll    = "rbxassetid://114127804740858",
    menu      = "rbxassetid://6340513838",
    crosshair = "rbxassetid://12614416478",
    user      = "rbxassetid://108483430622128",
    stat      = "rbxassetid://12094445329",
    eyes      = "rbxassetid://14321059114",
    sword     = "rbxassetid://82472368671405",
    discord   = "rbxassetid://94434236999817",
    star      = "rbxassetid://107005941750079",
    skeleton  = "rbxassetid://17313330026",
    payment   = "rbxassetid://18747025078",
    scan      = "rbxassetid://109869955247116",
    alert     = "rbxassetid://73186275216515",
    question  = "rbxassetid://17510196486",
    idea      = "rbxassetid://16833255748",
    strom     = "rbxassetid://13321880293",
    water     = "rbxassetid://100076212630732",
    dcs       = "rbxassetid://15310731934",
    start     = "rbxassetid://108886429866687",
    next      = "rbxassetid://12662718374",
    rod       = "rbxassetid://103247953194129",
    fish      = "rbxassetid://97167558235554",
    enviicon  = "rbxassetid://101669656973003",
    nplnicon  = "rbxassetid://111895858615511",
    nplnv4    = "rbxassetid://87167468756710",
}

local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local LocalPlayer = game:GetService("Players").LocalPlayer
local Mouse = LocalPlayer:GetMouse()
-- Parent GUI: CoreGui -> gethui() -> PlayerGui (fallback otomatis kalau executor gak bisa nulis ke CoreGui)
local function getGuiParent()
    local ok, cg = pcall(function() return game:GetService("CoreGui") end)
    if ok and cg then
        local probe = Instance.new("ScreenGui")
        local canParent = pcall(function() probe.Parent = cg end)
        probe:Destroy()
        if canParent then return cg end
    end

    local okHui, hui = pcall(function() return gethui and gethui() end)
    if okHui and hui then return hui end

    return LocalPlayer:WaitForChild("PlayerGui")
end
local CoreGui = getGuiParent()
local viewport = workspace.CurrentCamera.ViewportSize

local function isMobileDevice()
    return UserInputService.TouchEnabled
        and not UserInputService.KeyboardEnabled
        and not UserInputService.MouseEnabled
end

local isMobile = isMobileDevice()

local function safeSize(pxWidth, pxHeight)
    local scaleX = pxWidth / viewport.X
    local scaleY = pxHeight / viewport.Y

    if isMobile then
        if scaleX > 0.5 then scaleX = 0.5 end
        if scaleY > 0.3 then scaleY = 0.3 end
    end

    return UDim2.new(scaleX, 0, scaleY, 0)
end

local function MakeDraggable(topbarobject, object, customSize, conns)
    -- conns: daftar koneksi window; semua listener UserInputService ditampung supaya bisa diputus saat window ditutup
    local function track(c)
        if conns then table.insert(conns, c) end
        return c
    end
    local function CustomPos(topbarobject, object)
        local Dragging, DragInput, DragStart, StartPosition

        local function UpdatePos(input)
            local Delta = input.Position - DragStart
            local pos = UDim2.new(
                StartPosition.X.Scale,
                StartPosition.X.Offset + Delta.X,
                StartPosition.Y.Scale,
                StartPosition.Y.Offset + Delta.Y
            )
            local Tween = TweenService:Create(object, TweenInfo.new(0.2), { Position = pos })
            Tween:Play()
        end

        topbarobject.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                Dragging = true
                DragStart = input.Position
                StartPosition = object.Position
                input.Changed:Connect(function()
                    if input.UserInputState == Enum.UserInputState.End then
                        Dragging = false
                    end
                end)
            end
        end)

        topbarobject.InputChanged:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
                DragInput = input
            end
        end)

        track(UserInputService.InputChanged:Connect(function(input)
            if input == DragInput and Dragging then
                UpdatePos(input)
            end
        end))
    end

    local function CustomSize(object, customSize)
        local Dragging, DragInput, DragStart, StartSize

        local minSizeX, minSizeY = 320, 200   -- sebelumnya 100x100 (lebih kecil dari lebar tab, UI jadi hancur)
        local defSizeX, defSizeY
        if isMobile then
            defSizeX, defSizeY = 470, 270
        else
            defSizeX = (customSize and customSize.X) or 620
            defSizeY = (customSize and customSize.Y) or 430
        end

        -- Batas ukuran = area layar yang tersedia (ukuran ScreenGui, sudah dikurangi top bar), sisakan margin 10px.
        -- Dibaca langsung tiap dipakai, jadi ikut berubah saat HP diputar / jendela game diubah ukurannya.
        local MARGIN = 10
        local function limits()
            local area = object.Parent and object.Parent.AbsoluteSize
            if not area or area.X < 1 or area.Y < 1 then
                local cam = workspace.CurrentCamera
                area = cam and cam.ViewportSize or Vector2.new(1280, 720)
            end
            local maxX = math.max(200, area.X - MARGIN * 2)
            local maxY = math.max(150, area.Y - MARGIN * 2)
            return math.min(minSizeX, maxX), math.min(minSizeY, maxY), maxX, maxY
        end

        do
            local _, _, maxX, maxY = limits()
            object.Size = UDim2.new(0, math.min(defSizeX, maxX), 0, math.min(defSizeY, maxY))
        end

        -- gagang resize di pojok kanan bawah: lebih besar di HP + penanda titik-titik supaya kelihatan
        local handleSize = isMobile and 56 or 40
        local half = handleSize / 2

        local changesizeobject = Instance.new("Frame")
        changesizeobject.AnchorPoint = Vector2.new(1, 1)
        changesizeobject.BackgroundTransparency = 1
        changesizeobject.Size = UDim2.new(0, handleSize, 0, handleSize)
        changesizeobject.Position = UDim2.new(1, half, 1, half)
        changesizeobject.Name = "changesizeobject"
        changesizeobject.Parent = object

        local grip = Instance.new("Frame")
        grip.Name = "Grip"
        grip.BackgroundTransparency = 1
        grip.BorderSizePixel = 0
        grip.Size = UDim2.new(0, 14, 0, 14)
        grip.Position = UDim2.new(0, half - 16, 0, half - 16) -- tepat di dalam sudut window
        grip.Parent = changesizeobject
        for col = 0, 2 do
            for row = 0, 2 do
                if col + row >= 2 then
                    local dot = Instance.new("Frame")
                    dot.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                    dot.BackgroundTransparency = 0.55
                    dot.BorderSizePixel = 0
                    dot.Size = UDim2.new(0, 2, 0, 2)
                    dot.Position = UDim2.new(0, 2 + col * 4, 0, 2 + row * 4)
                    dot.Parent = grip
                end
            end
        end

        local function UpdateSize(input)
            local Delta = input.Position - DragStart
            local minX, minY, maxX, maxY = limits()
            local newWidth = math.clamp(StartSize.X + Delta.X, minX, maxX)
            local newHeight = math.clamp(StartSize.Y + Delta.Y, minY, maxY)

            local Tween = TweenService:Create(object, TweenInfo.new(0.2), { Size = UDim2.new(0, newWidth, 0, newHeight) })
            Tween:Play()
        end

        changesizeobject.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                Dragging = true
                DragStart = input.Position
                StartSize = object.AbsoluteSize -- ukuran nyata (aman juga kalau sebelumnya fullscreen)
                input.Changed:Connect(function()
                    if input.UserInputState == Enum.UserInputState.End then
                        Dragging = false
                    end
                end)
            end
        end)

        changesizeobject.InputChanged:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
                DragInput = input
            end
        end)

        track(UserInputService.InputChanged:Connect(function(input)
            if input == DragInput and Dragging then
                UpdateSize(input)
            end
        end))

        -- layar berubah (HP diputar, jendela diperkecil): window yang kebesaran dikecilkan otomatis
        local function clampToScreen()
            if object.Size.X.Scale ~= 0 or object.Size.Y.Scale ~= 0 then return end -- sedang fullscreen
            local minX, minY, maxX, maxY = limits()
            local w = math.clamp(object.Size.X.Offset, minX, maxX)
            local h = math.clamp(object.Size.Y.Offset, minY, maxY)
            if w ~= object.Size.X.Offset or h ~= object.Size.Y.Offset then
                object.Size = UDim2.new(0, w, 0, h)
            end
        end
        if object.Parent then
            track(object.Parent:GetPropertyChangedSignal("AbsoluteSize"):Connect(clampToScreen))
        end
    end

    CustomSize(object, customSize)
    CustomPos(topbarobject, object)
end

-- ===== Sound effect (UI / Notification / Element) =====
-- Matikan: Window({ Sounds = false })  atau  ProjectSenzu.Sounds.Enabled = false
local SenzuSounds = {
    Enabled = true,
    Ready = false, -- baru true setelah window selesai dibuat (biar load config gak bunyi)
    Volume = 1,
    Ids = {
        UI = "rbxassetid://5852470908",
        Element = "rbxassetid://117649901456711",
        Notification = "rbxassetid://639589431",
    },
}
local function PlaySound(kind)
    if not (SenzuSounds.Enabled and SenzuSounds.Ready) then return end
    local id = SenzuSounds.Ids[kind]
    if not id then return end
    pcall(function()
        local snd = Instance.new("Sound")
        snd.SoundId = id
        snd.Volume = SenzuSounds.Volume
        snd.Name = "ProjectSenzuSound"
        snd.Parent = CoreGui
        snd:Play()
        task.delay(3, function() pcall(function() snd:Destroy() end) end)
    end)
end

-- ===== Download asset (URL -> file lokal -> rbxasset) =====
-- Disimpan di ProjectSenzu/Assets. Return nil kalau executor gak support / gagal.
local ASSET_FOLDER = "ProjectSenzu/Assets"
local function DownloadAsset(url, fileName)
    local ok, res = pcall(function()
        local getAsset = getcustomasset or getsynasset
        if not (getAsset and writefile and isfile) then return nil end
        if makefolder and isfolder then
            if not isfolder("ProjectSenzu") then makefolder("ProjectSenzu") end
            if not isfolder(ASSET_FOLDER) then makefolder(ASSET_FOLDER) end
        end
        fileName = fileName or ((tostring(#url) .. "_" .. (url:match("([^/]+)$") or "asset")):gsub("[^%w%._%-]", "_"))
        local path = ASSET_FOLDER .. "/" .. fileName
        if not isfile(path) then
            writefile(path, game:HttpGet(url))
        end
        return getAsset(path)
    end)
    if ok then return res end
    return nil
end

local function CircleClick(Button, X, Y)
    PlaySound("UI")
    spawn(function()
        Button.ClipsDescendants = true
        local Circle = Instance.new("ImageLabel")
        Circle.Image = "rbxassetid://266543268"
        Circle.ImageColor3 = Color3.fromRGB(80, 80, 80)
        Circle.ImageTransparency = 0.8999999761581421
        Circle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        Circle.BackgroundTransparency = 1
        Circle.ZIndex = 10
        Circle.Name = "Circle"
        Circle.Parent = Button

        local NewX = X - Circle.AbsolutePosition.X
        local NewY = Y - Circle.AbsolutePosition.Y
        Circle.Position = UDim2.new(0, NewX, 0, NewY)
        local Size = 0
        if Button.AbsoluteSize.X > Button.AbsoluteSize.Y then
            Size = Button.AbsoluteSize.X * 1.5
        elseif Button.AbsoluteSize.X < Button.AbsoluteSize.Y then
            Size = Button.AbsoluteSize.Y * 1.5
        elseif Button.AbsoluteSize.X == Button.AbsoluteSize.Y then
            Size = Button.AbsoluteSize.X * 1.5
        end

        local Time = 0.5
        Circle:TweenSizeAndPosition(UDim2.new(0, Size, 0, Size), UDim2.new(0.5, -Size / 2, 0.5, -Size / 2), "Out", "Quad",
            Time, false, nil)
        for i = 1, 10 do
            Circle.ImageTransparency = Circle.ImageTransparency + 0.01
            wait(Time / 10)
        end
        Circle:Destroy()
    end)
end

local CurrentAccent = Color3.fromRGB(192, 192, 192)
local CurrentNotifyLogo = nil -- diisi dari Window (GuiConfig.WindowIMG) = logo di tengah UI

-- Pilih teks gelap atau putih otomatis sesuai terang-gelapnya warna aksen
local function GetContrastColor(color)
    local function lin(c)
        if c <= 0.03928 then return c / 12.92 end
        return ((c + 0.055) / 1.055) ^ 2.4
    end
    local lum = 0.2126 * lin(color.R) + 0.7152 * lin(color.G) + 0.0722 * lin(color.B)
    if lum > 0.25 then
        return Color3.fromRGB(10, 10, 10)
    end
    return Color3.fromRGB(255, 255, 255)
end

-- Ubah nama ikon ("eyes") / angka asset ID / URL gambar jadi content yang valid. Kosong -> nil.
local function ResolveIcon(icon)
    if icon == nil or icon == "" then return nil end
    if Icons[icon] then return Icons[icon] end
    if tostring(icon):match("^%d+$") then return "rbxassetid://" .. tostring(icon) end
    if tostring(icon):match("^https?://") then return DownloadAsset(tostring(icon)) end -- URL -> download -> getcustomasset
    return tostring(icon)
end

-- Gaya badge topbar: isi ungu gelap agak transparan + border tipis ungu + ikon kecil + teks putih
local function ApplyBadgeStyle(frame, label, iconId, accent)
    frame.BackgroundColor3 = accent:Lerp(Color3.new(0, 0, 0), 0.65)
    frame.BackgroundTransparency = 0.3

    local stroke = Instance.new("UIStroke")
    stroke.Color = accent
    stroke.Thickness = 1
    stroke.Transparency = 0.45
    stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    stroke.Parent = frame

    local list = Instance.new("UIListLayout")
    list.FillDirection = Enum.FillDirection.Horizontal
    list.VerticalAlignment = Enum.VerticalAlignment.Center
    list.SortOrder = Enum.SortOrder.LayoutOrder
    list.Padding = UDim.new(0, 5)
    list.Parent = frame

    if iconId then
        local icon = Instance.new("ImageLabel")
        icon.Name = "BadgeIcon"
        icon.LayoutOrder = 1
        icon.Size = UDim2.new(0, 13, 0, 13)
        icon.BackgroundTransparency = 1
        icon.Image = iconId
        icon.Parent = frame
    end

    label.LayoutOrder = 2
    label.TextColor3 = Color3.fromRGB(255, 255, 255)
end

local ProjectSenzu = {}
ProjectSenzu.Icons = Icons -- dipakai loader: Icon = "eyes" atau ProjectSenzu.Icons.eyes
-- Latar tiap item (toggle, slider, dropdown, input, dll).
-- 1 = polos, menyatu di dalam kartu section (gaya Zeroin). 0.935 = kotak abu per item (gaya lama).
-- Bisa diubah dari loader sebelum membuat window: ProjectSenzu.ItemBgTransparency = 0.935
ProjectSenzu.ItemBgTransparency = 1
-- URL backend chat (Vercel). Bisa di-set sekali di sini atau per chat lewat AddChat({ ApiBaseUrl = ... })
ProjectSenzu.ChatApiBaseUrl = "https://senzumithub.vercel.app"
-- Popup dropdown (panel yang muncul dari kanan pas dropdown diklik).
--   ProjectSenzu.DropdownColor = Color3.fromRGB(20, 12, 40)
--   ProjectSenzu.DropdownTransparency = 0.25  -- 0 = solid, 1 = hilang
-- Bisa juga lewat Window({ DropdownColor = ..., DropdownTransparency = ... })
-- atau kapan aja setelah window jadi: Tabs:SetDropdownStyle(color, transparency)
ProjectSenzu.DropdownColor = Color3.fromRGB(30, 30, 30)
ProjectSenzu.DropdownTransparency = 0.25
function ProjectSenzu:MakeNotify(NotifyConfig)
    PlaySound("Notification")
    local NotifyConfig = NotifyConfig or {}
    NotifyConfig.Title = NotifyConfig.Title or "Project Senzu"
    NotifyConfig.Description = NotifyConfig.Description or "Notification"
    NotifyConfig.Content = NotifyConfig.Content or "Content"
    -- Icon: kalau gak diisi, otomatis pakai logo tengah UI (WindowIMG)
    NotifyConfig.Color = NotifyConfig.Color or CurrentAccent
    NotifyConfig.Time = NotifyConfig.Time or 0.5
    NotifyConfig.Delay = NotifyConfig.Delay or 5
    local NotifyFunction = {}
    spawn(function()
        if not CoreGui:FindFirstChild("PZNotifyGui") then
            local PZNotifyGui = Instance.new("ScreenGui");
            PZNotifyGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
            PZNotifyGui.Name = "PZNotifyGui"
            PZNotifyGui.ResetOnSpawn = false
            PZNotifyGui.Parent = CoreGui
        end
        if not CoreGui.PZNotifyGui:FindFirstChild("PZNotifyLayout") then
            local PZNotifyLayout = Instance.new("Frame");
            PZNotifyLayout.AnchorPoint = Vector2.new(1, 1) -- pojok kanan bawah
            PZNotifyLayout.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            PZNotifyLayout.BackgroundTransparency = 0.9990000128746033
            PZNotifyLayout.BorderColor3 = Color3.fromRGB(0, 0, 0)
            PZNotifyLayout.BorderSizePixel = 0
            PZNotifyLayout.Position = UDim2.new(1, -30, 1, -30) -- jarak dari tepi kanan & bawah
            PZNotifyLayout.Size = UDim2.new(0, math.min(320, math.max(200, ((workspace.CurrentCamera and workspace.CurrentCamera.ViewportSize.X) or 1280) - 40)), 1, 0)
            PZNotifyLayout.Name = "PZNotifyLayout"
            PZNotifyLayout.Parent = CoreGui.PZNotifyGui
            CoreGui.PZNotifyGui.PZNotifyLayout.ChildRemoved:Connect(function()
                -- notif paling lama di paling bawah, yang baru numpuk ke atas
                local stackY = 0
                for i, v in CoreGui.PZNotifyGui.PZNotifyLayout:GetChildren() do
                    if v.Name ~= "NotifyFrame" then continue end
                    TweenService:Create(
                        v,
                        TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut),
                        { Position = UDim2.new(0, 0, 1, -stackY) }
                    ):Play()
                    stackY = stackY + v.Size.Y.Offset + 12
                end
            end)
        end
        local NotifyPosHeigh = 0 -- total tinggi notif yang masih tampil (jarak dari bawah)
        for i, v in CoreGui.PZNotifyGui.PZNotifyLayout:GetChildren() do
            if v.Name ~= "NotifyFrame" then continue end
            NotifyPosHeigh = NotifyPosHeigh + v.Size.Y.Offset + 12
        end
        local NotifyFrame = Instance.new("Frame");
        local NotifyFrameReal = Instance.new("Frame");
        local UICorner = Instance.new("UICorner");
        local DropShadowHolder = Instance.new("Frame");
        local DropShadow = Instance.new("ImageLabel");
        local Top = Instance.new("Frame");
        local TextLabel = Instance.new("TextLabel");
        local UICorner1 = Instance.new("UICorner");
        local TextLabel1 = Instance.new("TextLabel");
        local Close = Instance.new("TextButton");
        local ImageLabel = Instance.new("ImageLabel");
        local TextLabel2 = Instance.new("TextLabel");

        NotifyFrame.BackgroundColor3 = Color3.fromRGB(29, 30, 35)
        NotifyFrame.BorderColor3 = Color3.fromRGB(29, 30, 35)
        NotifyFrame.BorderSizePixel = 0
        NotifyFrame.Size = UDim2.new(1, 0, 0, 150)
        NotifyFrame.Name = "NotifyFrame"
        NotifyFrame.BackgroundTransparency = 1
        NotifyFrame.Parent = CoreGui.PZNotifyGui.PZNotifyLayout
        NotifyFrame.AnchorPoint = Vector2.new(0, 1) -- nempel dari bawah, tinggi berubah numbuhnya ke atas
        NotifyFrame.Position = UDim2.new(0, 0, 1, -NotifyPosHeigh)

        NotifyFrameReal.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        NotifyFrameReal.BorderColor3 = Color3.fromRGB(0, 0, 0)
        NotifyFrameReal.BorderSizePixel = 0
        NotifyFrameReal.Position = UDim2.new(0, 400, 0, 0)
        NotifyFrameReal.Size = UDim2.new(1, 0, 1, 0)
        NotifyFrameReal.Name = "NotifyFrameReal"
        NotifyFrameReal.Parent = NotifyFrame

        UICorner.Parent = NotifyFrameReal
        UICorner.CornerRadius = UDim.new(0, 8)

        DropShadowHolder.BackgroundTransparency = 1
        DropShadowHolder.BorderSizePixel = 0
        DropShadowHolder.Size = UDim2.new(1, 0, 1, 0)
        DropShadowHolder.ZIndex = 0
        DropShadowHolder.Name = "DropShadowHolder"
        DropShadowHolder.Parent = NotifyFrameReal

        local NotifIcon = Instance.new("ImageLabel")
        NotifIcon.Image = ResolveIcon(NotifyConfig.Icon or CurrentNotifyLogo) or "rbxassetid://108203634075572"
        NotifIcon.BackgroundTransparency = 1
        NotifIcon.ImageTransparency = 0
        NotifIcon.BorderSizePixel = 0
        NotifIcon.Size = UDim2.new(0, 60, 0, 60)
        NotifIcon.Position = UDim2.new(0., 0, 0.05, 0)
        NotifIcon.ScaleType = Enum.ScaleType.Fit
        NotifIcon.Name = "NotifIcon"
        NotifIcon.ZIndex = 0
        NotifIcon.Parent = NotifyFrameReal

        Top.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        Top.BackgroundTransparency = 0.9990000128746033
        Top.BorderColor3 = Color3.fromRGB(0, 0, 0)
        Top.BorderSizePixel = 0
        Top.Position = UDim2.new(0, 55, 0, 0)
        Top.Size = UDim2.new(1, -55, 0, 36)
        Top.Name = "Top"
        Top.Parent = NotifyFrameReal

        TextLabel.Font = Enum.Font.GothamBold
        TextLabel.Text = NotifyConfig.Title
        TextLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
        TextLabel.TextSize = 14
        TextLabel.TextXAlignment = Enum.TextXAlignment.Left
        TextLabel.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        TextLabel.BackgroundTransparency = 0.9990000128746033
        TextLabel.BorderColor3 = Color3.fromRGB(0, 0, 0)
        TextLabel.BorderSizePixel = 0
        TextLabel.Size = UDim2.new(1, 0, 1, 0)
        TextLabel.Parent = Top
        TextLabel.Position = UDim2.new(0, 10, 0, 0)

        UICorner1.Parent = Top
        UICorner1.CornerRadius = UDim.new(0, 5)

        TextLabel1.Font = Enum.Font.GothamBold
        TextLabel1.Text = NotifyConfig.Description
        TextLabel1.TextColor3 = NotifyConfig.Color
        TextLabel1.TextSize = 14
        TextLabel1.TextXAlignment = Enum.TextXAlignment.Left
        TextLabel1.BackgroundColor3 = Color3.fromRGB(255, 130, 130)
        TextLabel1.BackgroundTransparency = 0.9990000128746033
        TextLabel1.TextTransparency = 0.3
        TextLabel1.BorderColor3 = Color3.fromRGB(0, 0, 0)
        TextLabel1.BorderSizePixel = 0
        TextLabel1.TextTruncate = Enum.TextTruncate.AtEnd
        TextLabel1.Size = UDim2.new(1, -(TextLabel.TextBounds.X + 15 + 35), 1, 0)
        TextLabel1.Position = UDim2.new(0, TextLabel.TextBounds.X + 15, 0, 0)
        TextLabel1.Parent = Top

        Close.Font = Enum.Font.SourceSans
        Close.Text = ""
        Close.TextColor3 = Color3.fromRGB(0, 0, 0)
        Close.TextSize = 14
        Close.AnchorPoint = Vector2.new(1, 0.5)
        Close.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        Close.BackgroundTransparency = 0.9990000128746033
        Close.BorderColor3 = Color3.fromRGB(0, 0, 0)
        Close.BorderSizePixel = 0
        Close.Position = UDim2.new(1, -5, 0.5, 0)
        Close.Size = UDim2.new(0, 25, 0, 25)
        Close.Name = "Close"
        Close.Parent = Top

        ImageLabel.Image = "rbxassetid://9886659671"
        ImageLabel.AnchorPoint = Vector2.new(0.5, 0.5)
        ImageLabel.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        ImageLabel.BackgroundTransparency = 0.9990000128746033
        ImageLabel.BorderColor3 = Color3.fromRGB(0, 0, 0)
        ImageLabel.BorderSizePixel = 0
        ImageLabel.Position = UDim2.new(0.49000001, 0, 0.5, 0)
        ImageLabel.Size = UDim2.new(1, -8, 1, -8)
        ImageLabel.Parent = Close

        TextLabel2.Font = Enum.Font.GothamBold
        TextLabel2.TextColor3 = Color3.fromRGB(255, 255, 255)
        TextLabel2.TextSize = 13
        TextLabel2.Text = NotifyConfig.Content
        TextLabel2.TextXAlignment = Enum.TextXAlignment.Left
        TextLabel2.TextYAlignment = Enum.TextYAlignment.Top
        TextLabel2.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        TextLabel2.BackgroundTransparency = 0.9990000128746033
        TextLabel2.TextColor3 = Color3.fromRGB(150.0000062584877, 150.0000062584877, 150.0000062584877)
        TextLabel2.BorderColor3 = Color3.fromRGB(0, 0, 0)
        TextLabel2.BorderSizePixel = 0
        TextLabel2.Position = UDim2.new(0, 65, 0, 27)
        TextLabel2.Parent = NotifyFrameReal
        TextLabel2.Size = UDim2.new(1, -90, 0, 13)

        TextLabel2.TextWrapped = true
        TextLabel2.Size = UDim2.new(1, -90, 0, math.max(13, math.ceil(TextLabel2.TextBounds.Y)))

        if TextLabel2.Size.Y.Offset < 27 then
            NotifyFrame.Size = UDim2.new(1, 0, 0, 65)
        else
            NotifyFrame.Size = UDim2.new(1, 0, 0, TextLabel2.Size.Y.Offset + 40)
        end
        local waitbruh = false
        function NotifyFunction:Close()
            if waitbruh then
                return false
            end
            waitbruh = true
            TweenService:Create(
                NotifyFrameReal,
                TweenInfo.new(tonumber(NotifyConfig.Time), Enum.EasingStyle.Back, Enum.EasingDirection.InOut),
                { Position = UDim2.new(0, 400, 0, 0) }
            ):Play()
            task.wait(tonumber(NotifyConfig.Time) / 1.2)
            NotifyFrame:Destroy()
        end

        Close.Activated:Connect(function()
            NotifyFunction:Close()
        end)
        TweenService:Create(
            NotifyFrameReal,
            TweenInfo.new(tonumber(NotifyConfig.Time), Enum.EasingStyle.Back, Enum.EasingDirection.InOut),
            { Position = UDim2.new(0, 0, 0, 0) }
        ):Play()
        task.wait(tonumber(NotifyConfig.Delay))
        NotifyFunction:Close()
    end)
    return NotifyFunction
end

function notif(msg, delay, color, title, desc)
    return ProjectSenzu:MakeNotify({
        Title = title or "Project Senzu",
        Description = desc or "Notification",
        Content = msg or "Content",
        Color = color or CurrentAccent,
        Delay = delay or 4
    })
end

-- Background default (otomatis): download dari URL lalu jadikan asset via getcustomasset
local DEFAULT_BG_URL = "https://raw.githubusercontent.com/naells/NaellUI/refs/heads/main/img/background3.jpg"
local function ResolveBackground(src)
    if src == nil or src == "" then return nil end
    src = tostring(src)
    if src:match("^rbxassetid://") or src:match("^rbxasset://") then return src end
    if src:match("^%d+$") then return "rbxassetid://" .. src end
    if src:match("^https?://") then
        return DownloadAsset(src)
    end
    return src
end

ProjectSenzu.Sounds = SenzuSounds
function ProjectSenzu:DownloadAsset(url, fileName) return DownloadAsset(url, fileName) end

-- ===== Loading screen =====
-- local ls = ProjectSenzu:CreateLoadingScreen({ Title = "PROJECT SENZU", Target = "Nama Game", Detail = "...", Stages = {...}, Splash = "rbxassetid://..." })
-- ls:SetStep(2) / ls:set("download", "detail") / ls:SetTarget("Game") / ls:SetDetail("teks") / ls:Close()
function ProjectSenzu:CreateLoadingScreen(cfg)
    cfg = type(cfg) == "table" and cfg or {}
    local stages = cfg.Stages or {
        { key = "license",  label = "Selecting access mode" },
        { key = "session",  label = "Preparing access" },
        { key = "resolve",  label = "Resolving game script" },
        { key = "download", label = "Fetching build" },
        { key = "execute",  label = "Starting script" },
    }
    local WHITE, MUTED, TRACK = Color3.fromRGB(245, 245, 245), Color3.fromRGB(153, 153, 153), Color3.fromRGB(68, 68, 68)

    pcall(function()
        local old = CoreGui:FindFirstChild("ProjectSenzuLoader")
        if old then old:Destroy() end
    end)

    local function make(class, parent, props)
        local o = Instance.new(class)
        for k, v in pairs(props or {}) do o[k] = v end
        o.Parent = parent
        return o
    end
    local function tween(o, t, props)
        if o and o.Parent then
            pcall(function() TweenService:Create(o, TweenInfo.new(t, Enum.EasingStyle.Quint, Enum.EasingDirection.Out), props):Play() end)
        end
    end

    local Gui = make("ScreenGui", CoreGui, { Name = "ProjectSenzuLoader", DisplayOrder = 1000000, IgnoreGuiInset = true, ResetOnSpawn = false, ZIndexBehavior = Enum.ZIndexBehavior.Sibling })
    local Root = make("Frame", Gui, { Name = "Root", BackgroundTransparency = 1, Size = UDim2.fromScale(1, 1) })
    local Backdrop = make("Frame", Root, { Name = "Backdrop", BackgroundColor3 = Color3.new(0, 0, 0), BackgroundTransparency = 1, BorderSizePixel = 0, Size = UDim2.fromScale(1, 1), ZIndex = 2, Active = true })

    local Splash
    if cfg.Splash then
        Splash = make("ImageLabel", Root, { Name = "Splash", AnchorPoint = Vector2.new(0.5, 0.5), BackgroundTransparency = 1, Image = tostring(cfg.Splash), ImageTransparency = 1, Position = UDim2.fromScale(0.5, 0.5), ScaleType = Enum.ScaleType.Fit, Size = UDim2.new(0.82, 0, 0, 150), ZIndex = 3 })
        make("UISizeConstraint", Splash, { MaxSize = Vector2.new(560, 170), MinSize = Vector2.new(250, 90) })
    end

    local function label(name, pos, size, text, font, textSize, color, align)
        return make("TextLabel", Root, { Name = name, AnchorPoint = Vector2.new(0.5, 0.5), BackgroundTransparency = 1, Font = font, Position = pos, Size = size, Text = text, TextColor3 = color, TextSize = textSize, TextTransparency = 1, TextTruncate = Enum.TextTruncate.AtEnd, TextXAlignment = align or Enum.TextXAlignment.Center, ZIndex = 5 })
    end
    local Brand  = label("BrandTitle", UDim2.new(0.5, 0, 0.5, -52), UDim2.new(0.8, 0, 0, 22), tostring(cfg.Title or "PROJECT SENZU"), Enum.Font.GothamBold, 15, WHITE)
    local Target = label("Target", UDim2.new(0.5, 0, 0.5, -13), UDim2.new(0.84, 0, 0, 29), tostring(cfg.Target or "Detecting game"), Enum.Font.GothamBold, 21, WHITE)
    local Detail = label("Detail", UDim2.new(0.5, 0, 0.5, 15), UDim2.new(0.84, 0, 0, 30), tostring(cfg.Detail or "Preparing secure delivery"), Enum.Font.Gotham, 12, MUTED)
    Detail.TextWrapped = true
    local Phase  = label("Phase", UDim2.new(0.5, 0, 0.5, 57), UDim2.new(0.76, 0, 0, 20), stages[1].label, Enum.Font.GothamMedium, 11, WHITE)

    local Group = make("Frame", Root, { Name = "ProgressGroup", AnchorPoint = Vector2.new(0.5, 0), BackgroundTransparency = 1, Position = UDim2.new(0.5, 0, 0.5, 87), Size = UDim2.new(0.76, 0, 0, 30), ZIndex = 5 })
    make("UISizeConstraint", Group, { MaxSize = Vector2.new(360, 30), MinSize = Vector2.new(210, 30) })
    local Track = make("Frame", Group, { Name = "Track", BackgroundColor3 = TRACK, BackgroundTransparency = 1, BorderSizePixel = 0, ClipsDescendants = true, Size = UDim2.new(1, 0, 0, 3), ZIndex = 5 })
    Instance.new("UICorner", Track).CornerRadius = UDim.new(0, 2)
    local Fill = make("Frame", Track, { Name = "Fill", BackgroundColor3 = WHITE, BackgroundTransparency = 1, BorderSizePixel = 0, Size = UDim2.new(0, 0, 1, 0), ZIndex = 6 })
    Instance.new("UICorner", Fill).CornerRadius = UDim.new(0, 2)
    local Counter = make("TextLabel", Group, { Name = "StepCounter", BackgroundTransparency = 1, Font = Enum.Font.GothamMedium, Position = UDim2.fromOffset(0, 12), Size = UDim2.new(0.5, 0, 0, 15), Text = "", TextColor3 = MUTED, TextSize = 9, TextTransparency = 1, TextXAlignment = Enum.TextXAlignment.Left, ZIndex = 5 })
    local Percent = make("TextLabel", Group, { Name = "Percent", BackgroundTransparency = 1, Font = Enum.Font.GothamBold, Position = UDim2.new(0.5, 0, 0, 12), Size = UDim2.new(0.5, 0, 0, 15), Text = "", TextColor3 = WHITE, TextSize = 9, TextTransparency = 1, TextXAlignment = Enum.TextXAlignment.Right, ZIndex = 5 })

    local LS = { Closed = false, Visible = false, Screen = Gui, stages = stages, stageIndex = 1, stageKey = stages[1].key, progress = 1 / #stages }

    local function refresh()
        if LS.Closed then return end
        local p = math.clamp(LS.progress, 0, 1)
        Percent.Text = math.floor(p * 100 + 0.5) .. "%"
        Counter.Text = string.format("STEP %02d / %02d", LS.stageIndex, #LS.stages)
        if LS.Visible then tween(Fill, 0.3, { Size = UDim2.new(p, 0, 1, 0) }) else Fill.Size = UDim2.new(p, 0, 1, 0) end
    end

    function LS:SetDetail(text)
        if self.Closed or type(text) ~= "string" or text == "" then return end
        Detail.Text = text
    end
    function LS:SetTarget(name)
        if self.Closed or type(name) ~= "string" or name == "" then return end
        Target.Text = name
        self:SetDetail(name .. " - Secure delivery in progress")
    end
    function LS:SetStep(index, total, text)
        if self.Closed then return end
        local i = math.clamp(tonumber(index) or 1, 1, #self.stages)
        local st = self.stages[i]
        self.stageIndex, self.stageKey = i, st.key
        self.progress = math.clamp(i / math.max(1, tonumber(total) or #self.stages), 0, 1)
        Phase.Text = tostring(text or st.label)
        refresh()
    end
    function LS:set(key, detail)
        if self.Closed then return end
        for i, st in ipairs(self.stages) do
            if st.key == key then
                self:SetStep(i, #self.stages, st.label)
                if type(detail) == "string" and detail ~= "" then self:SetDetail(detail) end
                return
            end
        end
    end
    function LS:Close(instant)
        if self.Closed then return end
        self.Closed = true
        if instant or not Gui.Parent then pcall(function() Gui:Destroy() end) return end
        for _, d in ipairs(Gui:GetDescendants()) do
            if d:IsA("TextLabel") then tween(d, 0.18, { TextTransparency = 1, BackgroundTransparency = 1 })
            elseif d:IsA("ImageLabel") then tween(d, 0.18, { ImageTransparency = 1, BackgroundTransparency = 1 })
            elseif d:IsA("Frame") then tween(d, 0.18, { BackgroundTransparency = 1 }) end
        end
        task.delay(0.25, function() pcall(function() Gui:Destroy() end) end)
    end
    LS.setTarget = LS.SetTarget
    LS.close = LS.Close

    refresh()
    task.spawn(function()
        if Splash then
            tween(Splash, 0.65, { ImageTransparency = 0 })
            task.wait(0.9)
            if LS.Closed or not Gui.Parent then return end
            tween(Splash, 0.48, { ImageTransparency = 1 })
            task.wait(0.52)
            if LS.Closed or not Gui.Parent then return end
            Splash.Visible = false
        end
        LS.Visible = true
        tween(Backdrop, 0.38, { BackgroundTransparency = 0.4 })
        for _, o in ipairs({ Brand, Target, Detail, Phase, Counter, Percent }) do tween(o, 0.32, { TextTransparency = 0 }) end
        tween(Track, 0.32, { BackgroundTransparency = 0 })
        tween(Fill, 0.32, { BackgroundTransparency = 0 })
    end)
    return LS
end

function ProjectSenzu:Window(GuiConfig)
    GuiConfig              = GuiConfig or {}
    GuiConfig.Title        = GuiConfig.Title or "Project Senzu"
    GuiConfig.Footer       = GuiConfig.Footer or "Project Senzu >:D"
    -- Palet tetap: Hitam + Putih + Ungu (tanpa pilihan tema)
    GuiConfig.Color        = Color3.fromRGB(145, 70, 255)  -- ungu (aksen: border, tab, badge, toggle)
    GuiConfig.Color2       = Color3.fromRGB(60, 24, 115)   -- latar window: ungu gelap -> hitam (lewat gradient)
    CurrentAccent          = GuiConfig.Color
    GuiConfig["Tab Width"] = GuiConfig["Tab Width"] or 120
    GuiConfig.Version      = GuiConfig.Version or 1
    -- LogoHUB/WindowIMG/ToggleImage boleh ID angka atau URL gambar (PNG/JPG) -> otomatis didownload via DownloadAsset
    GuiConfig.LogoHUB      = ResolveBackground(GuiConfig.LogoHUB) or "rbxassetid://131719728791968" -- logo di kiri judul (topbar)
    GuiConfig.WindowIMG    = ResolveBackground(GuiConfig.WindowIMG) or GuiConfig.LogoHUB -- logo samar di tengah konten
    CurrentNotifyLogo      = GuiConfig.WindowIMG -- logo notif = logo tengah UI
    GuiConfig.DropdownColor        = GuiConfig.DropdownColor or ProjectSenzu.DropdownColor
    GuiConfig.DropdownTransparency = GuiConfig.DropdownTransparency or ProjectSenzu.DropdownTransparency

    CURRENT_VERSION        = GuiConfig.Version
    for k in pairs(KeyUsed) do KeyUsed[k] = nil end   -- window baru: hitungan kunci config mulai dari nol
    LoadConfigFromFile()

    local GuiFunc = {}

    if GuiConfig.Sounds ~= nil then SenzuSounds.Enabled = GuiConfig.Sounds and true or false end
    local ProjectSenzuOnTop = Instance.new("ScreenGui");
    local DropShadowHolder = Instance.new("Frame");
    local DropShadow = Instance.new("ImageLabel");
    local Main = Instance.new("ImageLabel");
    local UICorner = Instance.new("UICorner");
    local Top = Instance.new("Frame");
    local TextLabel = Instance.new("TextLabel");
    local UICorner1 = Instance.new("UICorner");
    local TextLabel1 = Instance.new("TextLabel");
    local Close = Instance.new("TextButton");
    local ImageLabel1 = Instance.new("ImageLabel");
    local Min = Instance.new("TextButton");
    local ImageLabel2 = Instance.new("ImageLabel");
    local FullScreen = Instance.new("TextButton")
    local ImageLabel3 = Instance.new("ImageLabel")
    local LayersTab = Instance.new("Frame");
    local AccountTab = Instance.new("Frame");
    local UICorner3 = Instance.new("UICorner")
    local DecideFrame = Instance.new("Frame");
    local Layers = Instance.new("Frame");
    local NameTab = Instance.new("TextLabel");
    local LayersReal = Instance.new("Frame");
    local LayersFolder = Instance.new("Folder");
    local LayersPageLayout = Instance.new("UIPageLayout");

    ProjectSenzuOnTop.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    ProjectSenzuOnTop.Name = "ProjectSenzuOnTop"
    ProjectSenzuOnTop.ResetOnSpawn = false
    ProjectSenzuOnTop.Parent = CoreGui

    DropShadowHolder.BackgroundTransparency = 1
	--DropShadowHolder.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    DropShadowHolder.BorderSizePixel = 0
    DropShadowHolder.AnchorPoint = Vector2.new(0.5, 0.5)
    DropShadowHolder.Position = UDim2.new(0.5, 0, 0.5, 0)
    if isMobile then
        DropShadowHolder.Size = UDim2.new(0, 470, 0, 270)
    else
        DropShadowHolder.Size = UDim2.new(0, 586, 0, 364)
    end
    DropShadowHolder.ZIndex = 0
    DropShadowHolder.Name = "DropShadowHolder"
    DropShadowHolder.Parent = ProjectSenzuOnTop

    DropShadowHolder.Position = UDim2.new(0.5, 0, 0.5, 0)
    DropShadow.Image = "rbxassetid://6015897843"
    DropShadow.ImageColor3 = Color3.fromRGB(0, 0, 0)
    DropShadow.ImageTransparency = 0.6
    DropShadow.ScaleType = Enum.ScaleType.Slice
    DropShadow.SliceCenter = Rect.new(49, 49, 450, 450)
    DropShadow.AnchorPoint = Vector2.new(0.5, 0.5)
    DropShadow.BackgroundTransparency = 1
    DropShadow.BorderSizePixel = 0
    DropShadow.Position = UDim2.new(0.5, 0, 0.5, 0)
    DropShadow.Size = UDim2.new(1, 47, 1, 47)
    DropShadow.ZIndex = 0
    DropShadow.Name = "DropShadow"
    DropShadow.Parent = DropShadowHolder

    Main.BackgroundColor3 = GuiConfig.Color2 -- Latar Warna Window
    -- Transparansi window. Ubah lewat Window({ BgTransparency = 0.25 }). 0 = solid, 1 = tembus total.
    Main.BackgroundTransparency = GuiConfig.BgTransparency or 0.2
    Main.ScaleType = Enum.ScaleType.Crop
    Main.ImageTransparency = GuiConfig.ThemeTransparency or 0.15
    do
        local BgImage = ResolveBackground(GuiConfig.Theme or DEFAULT_BG_URL)
        if BgImage then
            Main.Image = BgImage
            Main.BackgroundTransparency = GuiConfig.BgTransparency or 0.15
        end
    end

    Main.AnchorPoint = Vector2.new(0.5, 0.5)
    Main.BorderColor3 = Color3.fromRGB(0, 0, 0)
    Main.BorderSizePixel = 0
    Main.Position = UDim2.new(0.5, 0, 0.5, 0)
    Main.Size = UDim2.new(1, -47, 1, -47)
    Main.Name = "Main"
    Main.Parent = DropShadow

    local MainGradient = Instance.new("UIGradient")
    MainGradient.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 255, 255)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(80, 80, 80))
    }
    MainGradient.Rotation = 90
    MainGradient.Parent = Main

    UICorner3.Parent = Main
    UICorner3.CornerRadius = UDim.new(0.02, 0)

    do
        local DarkOverlay = Instance.new("Frame")
        DarkOverlay.Name = "DarkOverlay"
        DarkOverlay.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        DarkOverlay.BackgroundTransparency = GuiConfig.OverlayTransparency or 1
        DarkOverlay.BorderSizePixel = 0
        DarkOverlay.Size = UDim2.new(1, 0, 1, 0)
        DarkOverlay.ZIndex = 0
        DarkOverlay.Parent = Main
        Instance.new("UICorner", DarkOverlay).CornerRadius = UDim.new(0.02, 0)
    end

    local MainStroke = Instance.new("UIStroke")
    MainStroke.Color = GuiConfig.Color:Lerp(Color3.new(0, 0, 0), 0.45) -- Outline mengikuti warna tema
    MainStroke.Transparency = 0
    MainStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    MainStroke.Thickness = 1.2
    MainStroke.Parent = Main

    Top.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    Top.BackgroundTransparency = 0.9990000128746033
    Top.BorderColor3 = Color3.fromRGB(0, 0, 0)
    Top.BorderSizePixel = 0
    Top.Size = UDim2.new(1, 0, 0, 38)
    Top.Name = "Top"
    Top.Parent = Main

    TextLabel.Font = Enum.Font.GothamBold
    TextLabel.Text = GuiConfig.Title
    TextLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    TextLabel.TextSize = 14
    TextLabel.TextXAlignment = Enum.TextXAlignment.Left
    TextLabel.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    TextLabel.BackgroundTransparency = 0.9990000128746033
    TextLabel.BorderColor3 = Color3.fromRGB(0, 0, 0)
    TextLabel.BorderSizePixel = 0
    TextLabel.Size = UDim2.new(1, -100, 1, 0)
    TextLabel.Position = UDim2.new(0, 40, 0, 0)
    TextLabel.Parent = Top

    local LogoImg = Instance.new("ImageLabel")
    LogoImg.Image = GuiConfig.LogoHUB or "" -- sudah resolved (rbxassetid:// atau rbxasset:// dari DownloadAsset)
	LogoImg.BackgroundTransparency = 1
	LogoImg.BorderSizePixel = 0
	LogoImg.Size = UDim2.new(0, 28, 0, 28)
	LogoImg.Position = UDim2.new(0, 6, 0.5, -14)
	LogoImg.ScaleType = Enum.ScaleType.Fit
	LogoImg.Name = "LogoImg"
	LogoImg.Parent = Top

    UICorner1.Parent = Top

    local RightContainer = Instance.new("Frame")
    RightContainer.Name = "RightContainer"
    RightContainer.BackgroundTransparency = 1
    RightContainer.Size = UDim2.new(0.6, 0, 1, 0)
    RightContainer.Position = UDim2.new(1, -110, 0, 0)
    RightContainer.AnchorPoint = Vector2.new(1, 0)
    RightContainer.Parent = Top

    New("UIListLayout", {
        FillDirection = Enum.FillDirection.Horizontal,
        HorizontalAlignment = Enum.HorizontalAlignment.Right,
        VerticalAlignment = Enum.VerticalAlignment.Center,
        SortOrder = Enum.SortOrder.LayoutOrder,
        Padding = UDim.new(0, 8),
    }, RightContainer)

    local FooterFrame = Instance.new("Frame")
    FooterFrame.Name = "FooterFrame"
    FooterFrame.LayoutOrder = 1
    FooterFrame.Size = UDim2.new(0, 0, 0.6, 0)
    FooterFrame.BackgroundColor3 = GuiConfig.Color
    FooterFrame.BackgroundTransparency = 0.35
    FooterFrame.BorderSizePixel = 0
    FooterFrame.AutomaticSize = Enum.AutomaticSize.X
    FooterFrame.Parent = RightContainer

    New("UIPadding", { PaddingLeft = UDim.new(0, 10), PaddingRight = UDim.new(0, 10) }, FooterFrame)

    New("UICorner", { CornerRadius = UDim.new(0, 4) }, FooterFrame)

    TextLabel1.Name = "FooterText"
    TextLabel1.Size = UDim2.new(0, 0, 1, 0)
    TextLabel1.Position = UDim2.new(0, 0, 0, 0)
    TextLabel1.BackgroundTransparency = 1
    TextLabel1.Font = Enum.Font.GothamBold
    TextLabel1.Text = GuiConfig.Footer
    TextLabel1.TextColor3 = GetContrastColor(GuiConfig.Color)
    TextLabel1.TextSize = 12
    TextLabel1.TextXAlignment = Enum.TextXAlignment.Center
    TextLabel1.AutomaticSize = Enum.AutomaticSize.X
    TextLabel1.Parent = FooterFrame

    local execName = (identifyexecutor and identifyexecutor()) or "Unknown"
    local execText = "Executor: " .. tostring(execName)

    local Executor = Instance.new("Frame")
    Executor.Name = "Executor"
    Executor.LayoutOrder = 4
    Executor.Size = UDim2.new(0, 0, 0.6, 0)
    Executor.BackgroundColor3 = GuiConfig.Color
    Executor.BackgroundTransparency = 0.35
    Executor.BorderSizePixel = 0
    Executor.AutomaticSize = Enum.AutomaticSize.X
    Executor.Parent = RightContainer

    local UIPadding = Instance.new("UIPadding")
    UIPadding.PaddingLeft = UDim.new(0, 10)
    UIPadding.PaddingRight = UDim.new(0, 10)
    UIPadding.Parent = Executor

    New("UICorner", { CornerRadius = UDim.new(0, 4) }, Executor)

    local ExecutorTextLabel = Instance.new("TextLabel")
    ExecutorTextLabel.Name = "TextLabel"
    ExecutorTextLabel.Size = UDim2.new(0, 0, 1, 0)
    ExecutorTextLabel.Position = UDim2.new(0, 0, 0, 0)
    ExecutorTextLabel.BackgroundTransparency = 1
    ExecutorTextLabel.Font = Enum.Font.GothamBold
    ExecutorTextLabel.Text = execText
    ExecutorTextLabel.TextColor3 = GetContrastColor(GuiConfig.Color)
    ExecutorTextLabel.TextSize = 12
    ExecutorTextLabel.TextXAlignment = Enum.TextXAlignment.Center
    ExecutorTextLabel.AutomaticSize = Enum.AutomaticSize.X
    ExecutorTextLabel.Parent = Executor

    ApplyBadgeStyle(FooterFrame, TextLabel1, Icons.gamepad, GuiConfig.Color)
    ApplyBadgeStyle(Executor, ExecutorTextLabel, Icons.idea, GuiConfig.Color)

    -- ═══ FPS & PING (badge topbar) ═══
    -- Urutan topbar: [Game] [FPS] [Ping] [Executor]
    -- Mau dimatikan? Window({ ShowStats = false })
    local StatBadges = { list = {} }
    if GuiConfig.ShowStats ~= false then
        local RunService   = game:GetService("RunService")
        local StatsService = game:GetService("Stats")

        local function MakeStatBadge(name, order, text, iconId)
            local frame = Instance.new("Frame")
            frame.Name = name
            frame.LayoutOrder = order
            frame.Size = UDim2.new(0, 0, 0.6, 0)
            frame.BackgroundColor3 = GuiConfig.Color
            frame.BackgroundTransparency = 0.35
            frame.BorderSizePixel = 0
            frame.AutomaticSize = Enum.AutomaticSize.X
            frame.Parent = RightContainer

            local pad = Instance.new("UIPadding")
            pad.PaddingLeft = UDim.new(0, 10)
            pad.PaddingRight = UDim.new(0, 10)
            pad.Parent = frame

            New("UICorner", { CornerRadius = UDim.new(0, 4) }, frame)

            local label = Instance.new("TextLabel")
            label.Name = "TextLabel"
            label.Size = UDim2.new(0, 0, 1, 0)
            label.BackgroundTransparency = 1
            label.Font = Enum.Font.GothamBold
            label.Text = text
            label.TextColor3 = GetContrastColor(GuiConfig.Color)
            label.TextSize = 12
            label.TextXAlignment = Enum.TextXAlignment.Center
            label.AutomaticSize = Enum.AutomaticSize.X
            label.Parent = frame

            ApplyBadgeStyle(frame, label, iconId, GuiConfig.Color)
            table.insert(StatBadges.list, { frame = frame, label = label })
            return label
        end

        local fpsLabel  = MakeStatBadge("FpsBadge", 2, "-- FPS", Icons.stat)
        local pingLabel = MakeStatBadge("PingBadge", 3, "-- ms", Icons.web)

        local frames, lastT = 0, os.clock()
        local conn
        conn = RunService.Heartbeat:Connect(function()
            -- GUI sudah di-destroy -> hentikan loop
            if not ProjectSenzuOnTop.Parent then
                conn:Disconnect()
                return
            end
            frames = frames + 1
            local now = os.clock()
            if now - lastT >= 1 then
                local fps = math.floor(frames / (now - lastT) + 0.5)
                frames, lastT = 0, now
                fpsLabel.Text = fps .. " FPS"

                local ok, ping = pcall(function()
                    return StatsService.Network.ServerStatsItem["Data Ping"]:GetValue()
                end)
                if ok and type(ping) == "number" then
                    pingLabel.Text = math.floor(ping + 0.5) .. " ms"
                end
            end
        end)
    end

    -- ═══ ANTI-OVERFLOW TOPBAR ═══
    -- Badge tidak boleh menimpa judul. Dua tahap:
    --   1) kalau tidak muat -> mode KOMPAK (ikon badge disembunyikan, padding dirapatkan)
    --   2) masih tidak muat -> sembunyikan badge: Executor -> Game -> FPS (Ping dipertahankan paling akhir)
    do
        local HIDE_ORDER = { "Executor", "FooterFrame", "FpsBadge", "PingBadge" }
        local GAP, RIGHT_RESERVE = 8, 110
        local fitting = false

        local function Collect()
            local list = {}
            for _, name in ipairs(HIDE_ORDER) do
                local f = RightContainer:FindFirstChild(name)
                if f then
                    table.insert(list, {
                        frame = f,
                        label = f:FindFirstChildOfClass("TextLabel"),
                        icon  = f:FindFirstChild("BadgeIcon"),
                        pad   = f:FindFirstChildOfClass("UIPadding"),
                    })
                end
            end
            return list
        end

        local function WidthOf(it, compact)
            local textW = it.label and it.label.TextBounds.X or 0
            local iconW = (it.icon and not compact) and 18 or 0
            return (compact and 12 or 20) + iconW + textW
        end

        local function FitBadges()
            if fitting then return end
            fitting = true
            local items = Collect()
            local titleEnd = 40 + TextLabel.TextBounds.X + 14
            local avail = Top.AbsoluteSize.X - titleEnd - RIGHT_RESERVE

            local function Total(compact, skipHidden, hiddenCount)
                local t, n = 0, 0
                for i, it in ipairs(items) do
                    if not (skipHidden and i <= hiddenCount) then
                        t = t + WidthOf(it, compact); n = n + 1
                    end
                end
                return t + GAP * math.max(n - 1, 0)
            end

            local compact, hidden = false, 0
            if avail > 0 and Total(false, true, 0) > avail then
                compact = true
                while hidden < #items - 1 and Total(true, true, hidden) > avail do
                    hidden = hidden + 1
                end
            end

            for i, it in ipairs(items) do
                it.frame.Visible = i > hidden
                if it.icon then it.icon.Visible = not compact end
                if it.pad then
                    local p = compact and 6 or 10
                    it.pad.PaddingLeft, it.pad.PaddingRight = UDim.new(0, p), UDim.new(0, p)
                end
            end
            fitting = false
        end

        Top:GetPropertyChangedSignal("AbsoluteSize"):Connect(FitBadges)
        TextLabel:GetPropertyChangedSignal("TextBounds"):Connect(FitBadges)
        for _, it in ipairs(Collect()) do
            if it.label then it.label:GetPropertyChangedSignal("TextBounds"):Connect(FitBadges) end
        end
        task.defer(FitBadges)
    end

    -- ═══ UNLOCK FPS ═══
    -- Batas FPS dilepas (0 = tanpa batas), jadi FPS mengikuti kemampuan device.
    -- Mau dimatikan? Window({ UnlockFPS = false })
    if GuiConfig.UnlockFPS ~= false then
        pcall(function()
            if setfpscap then setfpscap(0) end
        end)
    end

    Close.Font = Enum.Font.SourceSans
    Close.Text = ""
    Close.TextColor3 = Color3.fromRGB(0, 0, 0)
    Close.TextSize = 14
    Close.AnchorPoint = Vector2.new(1, 0.5)
    Close.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    Close.BackgroundTransparency = 0.9990000128746033
    Close.BorderColor3 = Color3.fromRGB(0, 0, 0)
    Close.BorderSizePixel = 0
    Close.Position = UDim2.new(1, -8, 0.5, 0)
    Close.Size = UDim2.new(0, 25, 0, 25)
    Close.Name = "Close"
    Close.Parent = Top

    ImageLabel1.Image = "rbxassetid://9886659671"
    ImageLabel1.AnchorPoint = Vector2.new(0.5, 0.5)
    ImageLabel1.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    ImageLabel1.BackgroundTransparency = 0.9990000128746033
    ImageLabel1.BorderColor3 = Color3.fromRGB(0, 0, 0)
    ImageLabel1.BorderSizePixel = 0
    ImageLabel1.Position = UDim2.new(0.49, 0, 0.5, 0)
    ImageLabel1.Size = UDim2.new(1, -8, 1, -8)
    ImageLabel1.Parent = Close

    Min.Font = Enum.Font.SourceSans
    Min.Text = ""
    Min.TextColor3 = Color3.fromRGB(0, 0, 0)
    Min.TextSize = 14
    Min.AnchorPoint = Vector2.new(1, 0.5)
    Min.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    Min.BackgroundTransparency = 0.9990000128746033
    Min.BorderColor3 = Color3.fromRGB(0, 0, 0)
    Min.BorderSizePixel = 0
    Min.Position = UDim2.new(1, -68, 0.5, 0) --(1, -38, 0.5, 0)
    Min.Size = UDim2.new(0, 25, 0, 25)
    Min.Name = "Min"
    Min.Parent = Top

    ImageLabel2.Image = "rbxassetid://9886659276"
    ImageLabel2.AnchorPoint = Vector2.new(0.5, 0.5)
    ImageLabel2.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    ImageLabel2.BackgroundTransparency = 0.9990000128746033
    ImageLabel2.ImageTransparency = 0.2
    ImageLabel2.BorderColor3 = Color3.fromRGB(0, 0, 0)
    ImageLabel2.BorderSizePixel = 0
    ImageLabel2.Position = UDim2.new(0.5, 0, 0.5, 0)
    ImageLabel2.Size = UDim2.new(1, -9, 1, -9)
    ImageLabel2.Parent = Min

    FullScreen.Font = Enum.Font.SourceSans
    FullScreen.Text = ""
    FullScreen.TextColor3 = Color3.fromRGB(0, 0, 0)
    FullScreen.TextSize = 14
    FullScreen.AnchorPoint = Vector2.new(1, 0.5)
    FullScreen.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    FullScreen.BackgroundTransparency = 0.9990000128746033
    FullScreen.BorderColor3 = Color3.fromRGB(0, 0, 0)
    FullScreen.BorderSizePixel = 0
    FullScreen.Position = UDim2.new(1, -38, 0.5, 0) --(1, -68, 0.5, 0)
    FullScreen.Size = UDim2.new(0, 25, 0, 25)
    FullScreen.Name = "FullScreen"
    FullScreen.Parent = Top

    ImageLabel3.Image = "rbxassetid://113480421738477" -- ganti dengan icon fullscreen sesuai assetid kamu
    ImageLabel3.AnchorPoint = Vector2.new(0.5, 0.5)
    ImageLabel3.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    ImageLabel3.BackgroundTransparency = 0.9990000128746033
    ImageLabel3.ImageTransparency = 0.2
    ImageLabel3.BorderColor3 = Color3.fromRGB(0, 0, 0)
    ImageLabel3.BorderSizePixel = 0
    ImageLabel3.Position = UDim2.new(0.5, 0, 0.5, 0)
    ImageLabel3.Size = UDim2.new(1, -9, 1, -9)
    ImageLabel3.Parent = FullScreen

    LayersTab.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    LayersTab.BackgroundTransparency = 1
    LayersTab.BorderColor3 = Color3.fromRGB(0, 0, 0)
    LayersTab.BorderSizePixel = 0
    LayersTab.Position = UDim2.new(0, 9, 0, 50)
    LayersTab.Size = UDim2.new(0, GuiConfig["Tab Width"], 1, -95)
    LayersTab.Name = "LayersTab"
    LayersTab.Parent = Main

    local SearchBarFrame = Instance.new("Frame")
    SearchBarFrame.Name = "SearchBarFrame"
    SearchBarFrame.Size = UDim2.new(1, 0, 0, 26)
    SearchBarFrame.Position = UDim2.new(0, 0, 0, 0)
    SearchBarFrame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    SearchBarFrame.BackgroundTransparency = 0.93
    SearchBarFrame.Parent = LayersTab

    New("UICorner", { CornerRadius = UDim.new(0, 6) }, SearchBarFrame)

    local SearchBarStroke = Instance.new("UIStroke")
    SearchBarStroke.Thickness = 1
    SearchBarStroke.Transparency = 0.7
    SearchBarStroke.Color = GuiConfig.Color
    SearchBarStroke.Parent = SearchBarFrame

    local SearchBox = Instance.new("TextBox")
    SearchBox.Name = "SearchBox"
    SearchBox.BackgroundTransparency = 1
    SearchBox.Size = UDim2.new(1, -30, 1, 0)
    SearchBox.Position = UDim2.new(0, 24, 0, 0)
    SearchBox.TextSize = 11
    SearchBox.Font = Enum.Font.Gotham
    SearchBox.Text = ""
    SearchBox.PlaceholderText = "Search..."
    SearchBox.TextColor3 = Color3.fromRGB(255, 255, 255)
    SearchBox.PlaceholderColor3 = Color3.fromRGB(150, 150, 150)
    SearchBox.TextXAlignment = Enum.TextXAlignment.Left
    SearchBox.Parent = SearchBarFrame

    local SearchIcon = Instance.new("ImageLabel")
    SearchIcon.Name = "SearchIcon"
    SearchIcon.BackgroundTransparency = 1
    SearchIcon.Image = "rbxassetid://109869955247116"
    SearchIcon.AnchorPoint = Vector2.new(0, 0.5)
    SearchIcon.Position = UDim2.new(0, 7, 0.5, 0)
    SearchIcon.Size = UDim2.new(0, 13, 0, 13)
    SearchIcon.Parent = SearchBarFrame



	AccountTab.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    AccountTab.BackgroundTransparency = 0.9350000023841858
    AccountTab.BorderColor3 = Color3.fromRGB(0, 0, 0)
    AccountTab.BorderSizePixel = 0
    AccountTab.Position = UDim2.new(0, 9, 1, -45)
    AccountTab.Size = UDim2.new(0, GuiConfig["Tab Width"], 0, 35)
    AccountTab.Name = "AccountTab"
    AccountTab.Parent = Main

    UICorner.CornerRadius = UDim.new(0, 6)
    UICorner.Parent = AccountTab

    local AvatarCircle = Instance.new("Frame")
    AvatarCircle.Size = UDim2.new(0, 26, 0, 26)
    AvatarCircle.Position = UDim2.new(0, 5, 0.5, -13)
    AvatarCircle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    AvatarCircle.BackgroundTransparency = 1
    AvatarCircle.BorderSizePixel = 0
    AvatarCircle.ClipsDescendants = true  -- bikin bulat
    AvatarCircle.Name = "AvatarCircle"
    AvatarCircle.Parent = AccountTab

    local AvatarCorner = Instance.new("UICorner")
    AvatarCorner.CornerRadius = UDim.new(1, 0)  -- full bulat
    AvatarCorner.Parent = AvatarCircle

    local AvatarImg = Instance.new("ImageLabel")
    AvatarImg.Size = UDim2.new(1, 0, 1, 0)
    AvatarImg.BackgroundTransparency = 1
    AvatarImg.BorderSizePixel = 0
    AvatarImg.Name = "AvatarImg"
    AvatarImg.Parent = AvatarCircle

    -- Nama player di sebelah kanan avatar
    local AccountName = Instance.new("TextLabel")
    AccountName.Font = Enum.Font.GothamBold
    -- AccountName.Text = game:GetService("Players").LocalPlayer.Name
    AccountName.TextSize = 11
    AccountName.TextColor3 = Color3.fromRGB(231, 231, 231)
    AccountName.TextXAlignment = Enum.TextXAlignment.Left
    AccountName.BackgroundTransparency = 1
    AccountName.Position = UDim2.new(0, 36, 0, 0)
    AccountName.Size = UDim2.new(1, -40, 1, 0)
    AccountName.TextTruncate = Enum.TextTruncate.AtEnd  -- potong kalau kepanjangan
    AccountName.Parent = AccountTab

    local playerName = game:GetService("Players").LocalPlayer.Name

    if #playerName > 3 then
        local hiddenLength = #playerName - 3
        local visible = string.sub(playerName, -3)
        AccountName.Text = string.rep("*", hiddenLength) .. visible
    else
        AccountName.Text = playerName
    end

    -- Ambil foto avatar
    local Players = game:GetService("Players")
    local LocalPlayer = Players.LocalPlayer
    spawn(function()
        local url, _ = Players:GetUserThumbnailAsync(
            LocalPlayer.UserId,
            Enum.ThumbnailType.HeadShot,
            Enum.ThumbnailSize.Size420x420
        )
        AvatarImg.Image = url
    end)

    -- Kartu Discord (opsional): Window({ Discord = "discord.gg/xxxx" })
    -- Kalau diisi, menggantikan kartu avatar; klik = salin link ke clipboard.
    if GuiConfig.Discord then
        AvatarCircle.Visible = false
        AccountName.Visible = false

        local DcIcon = Instance.new("ImageLabel")
        DcIcon.Image = Icons.discord
        DcIcon.BackgroundTransparency = 1
        DcIcon.Size = UDim2.new(0, 20, 0, 20)
        DcIcon.Position = UDim2.new(0, 8, 0.5, -10)
        DcIcon.Parent = AccountTab

        local DcTitle = Instance.new("TextLabel")
        DcTitle.BackgroundTransparency = 1
        DcTitle.Font = Enum.Font.GothamBold
        DcTitle.Text = "Discord"
        DcTitle.TextSize = 11
        DcTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
        DcTitle.TextXAlignment = Enum.TextXAlignment.Left
        DcTitle.Position = UDim2.new(0, 34, 0, 4)
        DcTitle.Size = UDim2.new(1, -38, 0, 14)
        DcTitle.Parent = AccountTab

        local DcLink = Instance.new("TextLabel")
        DcLink.BackgroundTransparency = 1
        DcLink.Font = Enum.Font.Gotham
        DcLink.Text = tostring(GuiConfig.Discord)
        DcLink.TextSize = 9
        DcLink.TextColor3 = Color3.fromRGB(175, 170, 195)
        DcLink.TextXAlignment = Enum.TextXAlignment.Left
        DcLink.TextTruncate = Enum.TextTruncate.AtEnd
        DcLink.Position = UDim2.new(0, 34, 0, 18)
        DcLink.Size = UDim2.new(1, -38, 0, 12)
        DcLink.Parent = AccountTab

        local DcButton = Instance.new("TextButton")
        DcButton.BackgroundTransparency = 1
        DcButton.Text = ""
        DcButton.Size = UDim2.new(1, 0, 1, 0)
        DcButton.Parent = AccountTab
        DcButton.Activated:Connect(function()
            local copy = setclipboard or toclipboard
            if copy then
                pcall(copy, tostring(GuiConfig.Discord))
                notif("Discord link copied!", 3)
            else
                notif("Executor does not support clipboard!", 3)
            end
        end)
    end

    New("UICorner", { CornerRadius = UDim.new(0, 2) }, LayersTab)

    DecideFrame.AnchorPoint = Vector2.new(0.5, 0)
    DecideFrame.BackgroundColor3 = GuiConfig.Color
    DecideFrame.BackgroundTransparency = 0
    DecideFrame.BorderColor3 = Color3.fromRGB(0, 0, 0)
    DecideFrame.BorderSizePixel = 0
    DecideFrame.Position = UDim2.new(0.5, 0, 0, 38)
    DecideFrame.Size = UDim2.new(1, 0, 0, 1)
    DecideFrame.Name = "DecideFrame"
    DecideFrame.Parent = Main

    Layers.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    Layers.BackgroundTransparency = 0.9990000128746033
    Layers.BorderColor3 = Color3.fromRGB(0, 0, 0)
    Layers.BorderSizePixel = 0
    Layers.Position = UDim2.new(0, GuiConfig["Tab Width"] + 18, 0, 50)
    Layers.Size = UDim2.new(1, -(GuiConfig["Tab Width"] + 9 + 18), 1, -59)
    Layers.Name = "Layers"
    Layers.Parent = Main

    New("UICorner", { CornerRadius = UDim.new(0, 2) }, Layers)

	local WindowImg1 = Instance.new("ImageLabel")
	WindowImg1.Image = GuiConfig.LogoHUB or "" -- sudah resolved
	WindowImg1.BackgroundTransparency = 1
	WindowImg1.ImageTransparency = 0.8
	WindowImg1.BorderSizePixel = 0
	WindowImg1.Size = UDim2.new(0.5, 0, 0.9, 0)
	WindowImg1.Position = UDim2.new(0, 0, 0.1, 0)
	WindowImg1.ScaleType = Enum.ScaleType.Fit
	WindowImg1.Name = "WindowImg1"
	WindowImg1.ZIndex = -1
	WindowImg1.Parent = Layers
    WindowImg1.Visible = false

	local WindowImg2 = Instance.new("ImageLabel")
	WindowImg2.Image = GuiConfig.WindowIMG or "" -- sudah resolved
	WindowImg2.BackgroundTransparency = 1
	WindowImg2.ImageTransparency = 0.86
	WindowImg2.BorderSizePixel = 0
	WindowImg2.AnchorPoint = Vector2.new(0.5, 0.5)
	WindowImg2.Size = UDim2.new(0.75, 0, 0.85, 0)
	WindowImg2.Position = UDim2.new(0.5, 0, 0.5, 0)
	WindowImg2.ScaleType = Enum.ScaleType.Fit
	WindowImg2.Name = "WindowImg2"
	WindowImg2.ZIndex = 0
	WindowImg2.Parent = Layers
    WindowImg2.Visible = true

    -- NameTab.Font = Enum.Font.GothamBold
    -- NameTab.Text = ""
    -- NameTab.TextColor3 = Color3.fromRGB(255, 255, 255)
    -- NameTab.TextSize = 24
    -- NameTab.TextWrapped = true
    -- NameTab.TextXAlignment = Enum.TextXAlignment.Left
    -- NameTab.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    -- NameTab.BackgroundTransparency = 0.9990000128746033
    -- NameTab.BorderColor3 = Color3.fromRGB(0, 0, 0)
    -- NameTab.BorderSizePixel = 0
    -- NameTab.Size = UDim2.new(1, 0, 0, 30)
    -- NameTab.Name = "NameTab"
    -- NameTab.Parent = Layers

    LayersReal.AnchorPoint = Vector2.new(0, 1)
    LayersReal.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    LayersReal.BackgroundTransparency = 0.9990000128746033
    LayersReal.BorderColor3 = Color3.fromRGB(0, 0, 0)
    LayersReal.BorderSizePixel = 0
    LayersReal.ClipsDescendants = true
    LayersReal.Position = UDim2.new(0, 0, 1, 0)
    LayersReal.Size = UDim2.new(1, 0, 1, 0)
    LayersReal.Name = "LayersReal"
    LayersReal.Parent = Layers

    LayersFolder.Name = "LayersFolder"
    LayersFolder.Parent = LayersReal

    LayersPageLayout.SortOrder = Enum.SortOrder.LayoutOrder
    LayersPageLayout.Name = "LayersPageLayout"
    LayersPageLayout.Parent = LayersFolder
    LayersPageLayout.TweenTime = 0.5
    LayersPageLayout.EasingDirection = Enum.EasingDirection.InOut
    LayersPageLayout.EasingStyle = Enum.EasingStyle.Quad

    local ScrollTab = Instance.new("ScrollingFrame");
    local UIListLayout = Instance.new("UIListLayout");

    ScrollTab.CanvasSize = UDim2.new(0, 0, 1.10000002, 0)
    ScrollTab.ScrollBarImageColor3 = Color3.fromRGB(0, 0, 0)
    ScrollTab.ScrollBarThickness = 0
    ScrollTab.Active = true
    ScrollTab.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    ScrollTab.BackgroundTransparency = 0.9990000128746033
    ScrollTab.BorderColor3 = Color3.fromRGB(0, 0, 0)
    ScrollTab.BorderSizePixel = 0
    ScrollTab.Size = UDim2.new(1, 0, 0.88, 0)
	ScrollTab.Position = UDim2.new(0, 0, 0.11, 0)
    ScrollTab.Name = "ScrollTab"
    ScrollTab.Parent = LayersTab

    UIListLayout.Padding = UDim.new(0, 3)
    UIListLayout.SortOrder = Enum.SortOrder.LayoutOrder
    UIListLayout.Parent = ScrollTab

    local function UpdateSize1()
        local OffsetY = 0
        for _, child in ScrollTab:GetChildren() do
            if child.Name ~= "UIListLayout" then
                OffsetY = OffsetY + 3 + child.Size.Y.Offset
            end
        end
        ScrollTab.CanvasSize = UDim2.new(0, 0, 0, OffsetY)
    end
    ScrollTab.ChildAdded:Connect(UpdateSize1)
    ScrollTab.ChildRemoved:Connect(UpdateSize1)

    SearchBox:GetPropertyChangedSignal("Text"):Connect(function()
        local query = string.lower(SearchBox.Text)
        local jumpedToTab = false
        for _, scrolLayers in pairs(LayersFolder:GetChildren()) do
            if scrolLayers:IsA("ScrollingFrame") and scrolLayers.Name == "ScrolLayers" then
                for _, section in pairs(scrolLayers:GetChildren()) do
                    if section.Name == "Section" then
                        local sectionAdd = section:FindFirstChild("SectionAdd")
                        local sectionReal = section:FindFirstChild("SectionReal")
                        if sectionAdd and sectionReal then
                            local sectionVisible = false
                            for _, item in pairs(sectionAdd:GetChildren()) do
                                if item:IsA("GuiObject") then
                                    local match = false
                                    for _, desc in pairs(item:GetDescendants()) do
                                        if desc:IsA("TextLabel") and (string.find(desc.Name, "Title") or string.find(desc.Name, "Text")) then
                                            local txt = string.lower(desc.Text)
                                            if string.find(txt, query) then
                                                match = true
                                                if query ~= "" and not jumpedToTab then
                                                    jumpedToTab = true
                                                    for _, sideTab in pairs(ScrollTab:GetChildren()) do
                                                        if sideTab.Name == "Tab" and sideTab.LayoutOrder == scrolLayers.LayoutOrder then
                                                            local selectEvt = sideTab:FindFirstChild("SelectEvent")
                                                            if selectEvt then selectEvt:Fire() end
                                                            break
                                                        end
                                                    end
                                                end
                                                break
                                            end
                                        end
                                    end
                                    if query == "" then match = true end
                                    item.Visible = match
                                    if match then sectionVisible = true end
                                end
                            end
                            if query ~= "" then
                                section.Visible = sectionVisible
                            else
                                section.Visible = true
                            end
                        end
                    end
                end
            end
        end
    end)

    -- Semua koneksi global (drag tombol toggle, hotkey F3) ditampung supaya bisa diputus saat window ditutup
    GuiFunc._conns = {}

    function GuiFunc:DestroyGui()
        for _, c in ipairs(GuiFunc._conns) do
            pcall(function() c:Disconnect() end)
        end
        GuiFunc._conns = {}
        if SaveQueued then WriteConfigNow() end   -- simpan sisa perubahan sebelum window hilang

        -- Tombol toggle dibuat di parent yang sama dengan window (CoreGui / fallback)
        local toggleGui = CoreGui:FindFirstChild("ToggleUIProjectSenzu")
        if toggleGui then toggleGui:Destroy() end
        if ProjectSenzuOnTop then ProjectSenzuOnTop:Destroy() end
    end

    Min.Activated:Connect(function()
        CircleClick(Min, Mouse.X, Mouse.Y)
        DropShadowHolder.Visible = false
    end)
    Close.Activated:Connect(function()
        CircleClick(Close, Mouse.X, Mouse.Y)

        -- Cegah dialog dobel kalau tombol X diklik berkali-kali
        if Main:FindFirstChild("CloseOverlay") then return end

        -- Layar redup (fade in), menutup window persis mengikuti sudutnya
        local Overlay = Instance.new("Frame")
        Overlay.Name = "CloseOverlay"
        Overlay.Size = UDim2.new(1, 0, 1, 0)
        Overlay.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        Overlay.BackgroundTransparency = 1
        Overlay.BorderSizePixel = 0
        Overlay.Active = true -- tahan klik ke window di belakang dialog
        Overlay.ZIndex = 50
        Overlay.Parent = Main
        Instance.new("UICorner", Overlay).CornerRadius = UDim.new(0.02, 0)
        TweenService:Create(Overlay, TweenInfo.new(0.2), { BackgroundTransparency = 0.45 }):Play()

        -- Kartu dialog: gradient ungu gelap -> hampir hitam + border ungu
        local Dialog = Instance.new("Frame")
        Dialog.AnchorPoint = Vector2.new(0.5, 0.5)
        Dialog.Size = UDim2.new(0, 320, 0, 176)
        Dialog.Position = UDim2.new(0.5, 0, 0.5, 0)
        Dialog.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        Dialog.BorderSizePixel = 0
        Dialog.ZIndex = 51
        Dialog.Parent = Overlay
        Instance.new("UICorner", Dialog).CornerRadius = UDim.new(0, 10)

        local DialogGradient = Instance.new("UIGradient")
        DialogGradient.Color = ColorSequence.new({
            ColorSequenceKeypoint.new(0, GuiConfig.Color:Lerp(Color3.new(0, 0, 0), 0.7)),
            ColorSequenceKeypoint.new(1, Color3.fromRGB(12, 8, 20))
        })
        DialogGradient.Rotation = 90
        DialogGradient.Parent = Dialog

        New("UIStroke", {
            Color = GuiConfig.Color,
            Thickness = 1.5,
            Transparency = 0.3,
            ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
        }, Dialog)

        -- Animasi muncul (membesar sedikit)
        local DialogScale = Instance.new("UIScale")
        DialogScale.Scale = 0.88
        DialogScale.Parent = Dialog
        TweenService:Create(DialogScale, TweenInfo.new(0.2, Enum.EasingStyle.Back, Enum.EasingDirection.Out), { Scale = 1 }):Play()

        -- Ikon tanda seru dalam lingkaran
        local IconBg = Instance.new("Frame")
        IconBg.AnchorPoint = Vector2.new(0.5, 0)
        IconBg.Size = UDim2.new(0, 40, 0, 40)
        IconBg.Position = UDim2.new(0.5, 0, 0, 16)
        IconBg.BackgroundColor3 = GuiConfig.Color
        IconBg.BackgroundTransparency = 0.7
        IconBg.BorderSizePixel = 0
        IconBg.ZIndex = 52
        IconBg.Parent = Dialog
        Instance.new("UICorner", IconBg).CornerRadius = UDim.new(1, 0)

        New("UIStroke", { Color = GuiConfig.Color, Thickness = 1.2 }, IconBg)

        local IconText = Instance.new("TextLabel")
        IconText.Size = UDim2.new(1, 0, 1, 0)
        IconText.BackgroundTransparency = 1
        IconText.Font = Enum.Font.GothamBold
        IconText.Text = "!"
        IconText.TextSize = 22
        IconText.TextColor3 = Color3.fromRGB(255, 255, 255)
        IconText.ZIndex = 53
        IconText.Parent = IconBg

        local Title = Instance.new("TextLabel")
        Title.Size = UDim2.new(1, 0, 0, 24)
        Title.Position = UDim2.new(0, 0, 0, 60)
        Title.BackgroundTransparency = 1
        Title.Font = Enum.Font.GothamBold
        Title.Text = "Close Project Senzu?"
        Title.TextSize = 18
        Title.TextColor3 = Color3.fromRGB(255, 255, 255)
        Title.ZIndex = 52
        Title.Parent = Dialog

        local Message = Instance.new("TextLabel")
        Message.Size = UDim2.new(1, -20, 0, 36)
        Message.Position = UDim2.new(0, 10, 0, 86)
        Message.BackgroundTransparency = 1
        Message.Font = Enum.Font.Gotham
        Message.Text = "The window will be closed completely.\nYou will not be able to open it again."
        Message.TextSize = 12
        Message.TextColor3 = Color3.fromRGB(190, 185, 205)
        Message.TextWrapped = true
        Message.ZIndex = 52
        Message.Parent = Dialog

        -- Tombol: Yes (ungu solid) dan Cancel (outline)
        local Yes = Instance.new("TextButton")
        Yes.Size = UDim2.new(0.5, -22, 0, 32)
        Yes.Position = UDim2.new(0, 16, 1, -46)
        Yes.AutoButtonColor = false
        Yes.BackgroundColor3 = GuiConfig.Color
        Yes.BackgroundTransparency = 0
        Yes.Text = "Yes, close"
        Yes.Font = Enum.Font.GothamBold
        Yes.TextSize = 13
        Yes.TextColor3 = Color3.fromRGB(255, 255, 255)
        Yes.ZIndex = 52
        Yes.Name = "Yes"
        Yes.Parent = Dialog
        Instance.new("UICorner", Yes).CornerRadius = UDim.new(0, 6)

        local Cancel = Instance.new("TextButton")
        Cancel.Size = UDim2.new(0.5, -22, 0, 32)
        Cancel.Position = UDim2.new(0.5, 6, 1, -46)
        Cancel.AutoButtonColor = false
        Cancel.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        Cancel.BackgroundTransparency = 0.92
        Cancel.Text = "Cancel"
        Cancel.Font = Enum.Font.GothamBold
        Cancel.TextSize = 13
        Cancel.TextColor3 = Color3.fromRGB(255, 255, 255)
        Cancel.ZIndex = 52
        Cancel.Name = "Cancel"
        Cancel.Parent = Dialog
        Instance.new("UICorner", Cancel).CornerRadius = UDim.new(0, 6)

        New("UIStroke", {
            Color = Color3.fromRGB(255, 255, 255),
            Transparency = 0.8,
            Thickness = 1,
            ApplyStrokeMode = Enum.ApplyStrokeMode.Border,
        }, Cancel)

        -- Efek hover tombol
        local function Hover(btn, base, over)
            btn.MouseEnter:Connect(function()
                TweenService:Create(btn, TweenInfo.new(0.15), { BackgroundTransparency = over }):Play()
            end)
            btn.MouseLeave:Connect(function()
                TweenService:Create(btn, TweenInfo.new(0.15), { BackgroundTransparency = base }):Play()
            end)
        end
        Hover(Yes, 0, 0.2)
        Hover(Cancel, 0.92, 0.84)

        Yes.MouseButton1Click:Connect(function()
            GuiFunc:DestroyGui()
        end)

        Cancel.MouseButton1Click:Connect(function()
            Overlay:Destroy()
        end)
    end)

    local ToggleKey = Enum.KeyCode.F3
    table.insert(GuiFunc._conns, UserInputService.InputBegan:Connect(function(input, gpe)
        if gpe then return end
        if input.KeyCode == ToggleKey then
            if DropShadowHolder then
                DropShadowHolder.Visible = not DropShadowHolder.Visible
            end
        end
    end))

    function GuiFunc:ToggleUI()
        local ScreenGui = Instance.new("ScreenGui")
        ScreenGui.ResetOnSpawn = false
        ScreenGui.Parent = CoreGui
        ScreenGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
        ScreenGui.Name = "ToggleUIProjectSenzu"

        local MainButton = Instance.new("ImageLabel")
        MainButton.Parent = ScreenGui
        MainButton.AnchorPoint = Vector2.new(0.5, 0)          -- titik tumpu di tengah, biar bener-bener center
        MainButton.Size = UDim2.new(0, 52, 0, 52)             -- sebelumnya 40x40
        MainButton.Position = UDim2.new(0.5, 0, 0, 10)        -- tengah atas
        MainButton.BackgroundTransparency = 1
        MainButton.Image = (GuiConfig.ToggleImage and ResolveBackground(GuiConfig.ToggleImage)) or "rbxassetid://133666454974130" -- logo tombol toggle; boleh ID atau URL
        MainButton.ScaleType = Enum.ScaleType.Fit

        New("UIStroke", { Color = Color3.fromRGB(255, 255, 255), Thickness = 1.2, Transparency = 0.5 }, MainButton)

        local UICorner = Instance.new("UICorner")
        UICorner.CornerRadius = UDim.new(0, 6)
        UICorner.Parent = MainButton

        local Button = Instance.new("TextButton")
        Button.Parent = MainButton
        Button.Size = UDim2.new(1, 0, 1, 0)
        Button.BackgroundTransparency = 1
        Button.Text = ""

        Button.MouseButton1Click:Connect(function()
            if DropShadowHolder then
                DropShadowHolder.Visible = not DropShadowHolder.Visible
            end
        end)

        local dragging = false
        local dragStart, startPos

        local function update(input)
            local delta = input.Position - dragStart
            MainButton.Position = UDim2.new(
                startPos.X.Scale,
                startPos.X.Offset + delta.X,
                startPos.Y.Scale,
                startPos.Y.Offset + delta.Y
            )
        end

        Button.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = true
                dragStart = input.Position
                startPos = MainButton.Position
                input.Changed:Connect(function()
                    if input.UserInputState == Enum.UserInputState.End then
                        dragging = false
                    end
                end)
            end
        end)

        table.insert(GuiFunc._conns, game:GetService("UserInputService").InputChanged:Connect(function(input)
            if dragging and (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) then
                update(input)
            end
        end))
    end

    GuiFunc:ToggleUI()

    DropShadowHolder.Size = UDim2.new(0, 115 + TextLabel.TextBounds.X + 1 + TextLabel1.TextBounds.X, 0, 350)
    MakeDraggable(Top, DropShadowHolder, GuiConfig.Size, GuiFunc._conns) -- GuiConfig.Size = Vector2.new(lebar, tinggi), khusus PC
    local isFullscreen = false
    local originalSize = DropShadowHolder.Size
    local originalPos  = DropShadowHolder.Position

    FullScreen.Activated:Connect(function()
        CircleClick(FullScreen, Mouse.X, Mouse.Y)
        isFullscreen = not isFullscreen

        if isFullscreen then
            -- simpan size & pos sebelum fullscreen
            originalSize = DropShadowHolder.Size
            originalPos  = DropShadowHolder.Position

            TweenService:Create(DropShadowHolder, TweenInfo.new(0.4, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
                Size     = UDim2.new(0.980000019, 0, 0.949999988, 0),
                Position = UDim2.new(0.5, 0, 0.5, 0),
            }):Play()
        else
            TweenService:Create(DropShadowHolder, TweenInfo.new(0.4, Enum.EasingStyle.Quart, Enum.EasingDirection.Out), {
                Size     = originalSize,
                Position = originalPos,
            }):Play()
        end
    end)

    local MoreBlur = Instance.new("Frame");
    local DropShadowHolder1 = Instance.new("Frame");
    local DropShadow1 = Instance.new("ImageLabel");
    local ConnectButton = Instance.new("TextButton");

    MoreBlur.AnchorPoint = Vector2.new(1, 1)
    MoreBlur.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    MoreBlur.BackgroundTransparency = 0.999
    MoreBlur.BorderColor3 = Color3.fromRGB(0, 0, 0)
    MoreBlur.BorderSizePixel = 0
    MoreBlur.ClipsDescendants = true
    MoreBlur.Position = UDim2.new(1, 8, 1, 8)
    MoreBlur.Size = UDim2.new(1, 154, 1, 54)
    MoreBlur.Visible = false
    MoreBlur.Name = "MoreBlur"
    MoreBlur.Parent = Layers

    DropShadowHolder1.BackgroundTransparency = 1
    DropShadowHolder1.BorderSizePixel = 0
    DropShadowHolder1.Size = UDim2.new(1, 0, 1, 0)
    DropShadowHolder1.ZIndex = 0
    DropShadowHolder1.Name = "DropShadowHolder"
    DropShadowHolder1.Parent = MoreBlur

    DropShadow1.Image = "rbxassetid://6015897843"
    DropShadow1.ImageColor3 = Color3.fromRGB(0, 0, 0)
    DropShadow1.ImageTransparency = 1
    DropShadow1.ScaleType = Enum.ScaleType.Slice
    DropShadow1.SliceCenter = Rect.new(49, 49, 450, 450)
    DropShadow1.AnchorPoint = Vector2.new(0.5, 0.5)
    DropShadow1.BackgroundTransparency = 1
    DropShadow1.BorderSizePixel = 0
    DropShadow1.Position = UDim2.new(0.5, 0, 0.5, 0)
    DropShadow1.Size = UDim2.new(1, 35, 1, 35)
    DropShadow1.ZIndex = 0
    DropShadow1.Name = "DropShadow"
    DropShadow1.Parent = DropShadowHolder1

    New("UICorner", {  }, MoreBlur)

    ConnectButton.Font = Enum.Font.SourceSans
    ConnectButton.Text = ""
    ConnectButton.TextColor3 = Color3.fromRGB(0, 0, 0)
    ConnectButton.TextSize = 14
    ConnectButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    ConnectButton.BackgroundTransparency = 0.999
    ConnectButton.BorderColor3 = Color3.fromRGB(0, 0, 0)
    ConnectButton.BorderSizePixel = 0
    ConnectButton.Size = UDim2.new(1, 0, 1, 0)
    ConnectButton.Name = "ConnectButton"
    ConnectButton.Parent = MoreBlur

    local DropdownSelect = Instance.new("Frame");
    local UIStroke14 = Instance.new("UIStroke");
    local DropdownSelectReal = Instance.new("Frame");
    local DropdownFolder = Instance.new("Folder");
    local DropPageLayout = Instance.new("UIPageLayout");

    DropdownSelect.AnchorPoint = Vector2.new(1, 0.5)
    DropdownSelect.BackgroundColor3 = GuiConfig.DropdownColor
    DropdownSelect.BackgroundTransparency = GuiConfig.DropdownTransparency
    DropdownSelect.BorderColor3 = Color3.fromRGB(0, 0, 0)
    DropdownSelect.BorderSizePixel = 0
    DropdownSelect.LayoutOrder = 1
    DropdownSelect.Position = UDim2.new(1, 172, 0.5, 0)
    DropdownSelect.Size = UDim2.new(0, 160, 1, -16)
    DropdownSelect.Name = "DropdownSelect"
    DropdownSelect.ClipsDescendants = true
    DropdownSelect.Parent = MoreBlur

    ConnectButton.Activated:Connect(function()
        if MoreBlur.Visible then
            TweenService:Create(MoreBlur, TweenInfo.new(0.3), { BackgroundTransparency = 0.999 }):Play()
            TweenService:Create(DropdownSelect, TweenInfo.new(0.3), { Position = UDim2.new(1, 172, 0.5, 0) }):Play()
            task.wait(0.3)
            MoreBlur.Visible = false
        end
    end)
    New("UICorner", { CornerRadius = UDim.new(0, 3) }, DropdownSelect)

    UIStroke14.Color = GuiConfig.Color
    UIStroke14.Thickness = 2.5
    UIStroke14.Transparency = 0.8
    UIStroke14.Parent = DropdownSelect

    DropdownSelectReal.AnchorPoint = Vector2.new(0.5, 0.5)
    DropdownSelectReal.BackgroundColor3 = GuiConfig.DropdownColor
    DropdownSelectReal.BackgroundTransparency = 1 -- warna & transparansi diatur di DropdownSelect
    DropdownSelectReal.BorderColor3 = Color3.fromRGB(0, 0, 0)
    DropdownSelectReal.BorderSizePixel = 0
    DropdownSelectReal.LayoutOrder = 1
    DropdownSelectReal.Position = UDim2.new(0.5, 0, 0.5, 0)
    DropdownSelectReal.Size = UDim2.new(1, 1, 1, 1)
    DropdownSelectReal.Name = "DropdownSelectReal"
    DropdownSelectReal.Parent = DropdownSelect

    DropdownFolder.Name = "DropdownFolder"
    DropdownFolder.Parent = DropdownSelectReal

    DropPageLayout.EasingDirection = Enum.EasingDirection.InOut
    DropPageLayout.EasingStyle = Enum.EasingStyle.Quad
    DropPageLayout.TweenTime = 0.009999999776482582
    DropPageLayout.SortOrder = Enum.SortOrder.LayoutOrder
    DropPageLayout.FillDirection = Enum.FillDirection.Vertical
    DropPageLayout.Archivable = false
    DropPageLayout.Name = "DropPageLayout"
    DropPageLayout.Parent = DropdownFolder
    --// Tabs
    local Tabs = {}

    -- ===== Background API =====
    -- Tabs:SetBackground("https://...jpg" | "rbxassetid://123" | 123 | nil)  -> ganti gambar (nil/"" = hapus gambar)
    function Tabs:SetBackground(src)
        local img = ResolveBackground(src)
        Main.Image = img or ""
        if img and Main.BackgroundTransparency >= 0.99 then Main.BackgroundTransparency = 0.15 end
        return img ~= nil
    end
    Tabs.SetBackgroundImage = Tabs.SetBackground

    -- Transparansi gambar background (0 = jelas, 1 = hilang)
    function Tabs:SetThemeTransparency(t)
        Main.ImageTransparency = math.clamp(tonumber(t) or 0.15, 0, 1)
    end

    -- Transparansi lapisan gelap/terang di atas gambar (1 = tanpa overlay)
    function Tabs:SetOverlayTransparency(t)
        local ov = Main:FindFirstChild("DarkOverlay")
        if ov then ov.BackgroundTransparency = math.clamp(tonumber(t) or 1, 0, 1) end
    end

    -- ===== Tabs:AddBadge(text, icon) -> badge di topbar (sebelah FPS/Ping/Executor) =====
    -- local b = Tabs:AddBadge("Premium", "eyes"); b:SetTitle("Free"); b:SetIcon("stat"); b:Destroy()
    do
        local badgeOrder = 20
        local function resolveIcon(v)
            if v == nil then return nil end
            v = tostring(v)
            if Icons[v] then return Icons[v] end
            if v:match("^%d+$") then return "rbxassetid://" .. v end
            return v
        end
        function Tabs:AddBadge(text, icon)
            badgeOrder = badgeOrder + 1
            local frame = Instance.new("Frame")
            frame.Name = "CustomBadge"
            frame.LayoutOrder = badgeOrder
            frame.Size = UDim2.new(0, 0, 0.6, 0)
            frame.BorderSizePixel = 0
            frame.AutomaticSize = Enum.AutomaticSize.X
            frame.Parent = RightContainer
            New("UIPadding", { PaddingLeft = UDim.new(0, 10), PaddingRight = UDim.new(0, 10) }, frame)
            New("UICorner", { CornerRadius = UDim.new(0, 4) }, frame)

            local label = New("TextLabel", {
                Name = "TextLabel", Size = UDim2.new(0, 0, 1, 0), BackgroundTransparency = 1,
                Font = Enum.Font.GothamBold, Text = tostring(text or "Badge"), TextSize = 12,
                TextXAlignment = Enum.TextXAlignment.Center, AutomaticSize = Enum.AutomaticSize.X,
            }, frame)
            ApplyBadgeStyle(frame, label, resolveIcon(icon), GuiConfig.Color)
            local entry = { frame = frame, label = label }
            table.insert(StatBadges.list, entry)

            local Badge = { Frame = frame, TextLabel = label }
            function Badge:SetTitle(t) label.Text = tostring(t) end
            Badge.SetText = Badge.SetTitle
            function Badge:SetIcon(i)
                local img = frame:FindFirstChild("BadgeIcon")
                if not img then
                    img = New("ImageLabel", { Name = "BadgeIcon", LayoutOrder = 1, Size = UDim2.new(0, 13, 0, 13), BackgroundTransparency = 1 }, frame)
                end
                img.Image = resolveIcon(i) or ""
            end
            function Badge:Destroy()
                for k, b in ipairs(StatBadges.list) do
                    if b == entry then table.remove(StatBadges.list, k) break end
                end
                pcall(function() frame:Destroy() end)
            end
            return Badge
        end
        Tabs.Badge = Tabs.AddBadge
    end

    -- ===== Tema Light / Dark =====
    -- Tabs:SetMode("Light") / Tabs:SetMode("Dark")  (atau Window({ ThemeMode = "light" }))
    -- Warna abu/putih/hitam dibalik otomatis; warna aksen (ungu dll) tetap.
    do
        local original = setmetatable({}, { __mode = "k" })
        local currentMode = "dark"
        local descConn

        local function isNeutral(c)
            local _, sat = Color3.toHSV(c)
            return sat < 0.2
        end
        local function invert(c)
            local h, sat, v = Color3.toHSV(c)
            return Color3.fromHSV(h, sat, 1 - v)
        end
        local TEXT = { TextLabel = true, TextButton = true, TextBox = true }
        local IMG = { ImageLabel = true, ImageButton = true }

        local function convert(inst, light)
            if inst == Main or inst.Name == "DarkOverlay" or inst.Name == "Circle" or inst.Name == "DropShadow" then return end
            local o = original[inst]
            if not o then o = {}; original[inst] = o end
            pcall(function()
                if inst:IsA("GuiObject") and inst.BackgroundTransparency < 0.95 then
                    if light then
                        if o.bg == nil then o.bg = inst.BackgroundColor3 end
                        if isNeutral(o.bg) then inst.BackgroundColor3 = invert(o.bg) end
                    elseif o.bg ~= nil then inst.BackgroundColor3 = o.bg end
                end
                if TEXT[inst.ClassName] then
                    if light then
                        if o.tx == nil then o.tx = inst.TextColor3 end
                        if isNeutral(o.tx) then inst.TextColor3 = invert(o.tx) end
                    elseif o.tx ~= nil then inst.TextColor3 = o.tx end
                end
                if IMG[inst.ClassName] then
                    if light then
                        if o.im == nil then o.im = inst.ImageColor3 end
                        if isNeutral(o.im) then inst.ImageColor3 = invert(o.im) end
                    elseif o.im ~= nil then inst.ImageColor3 = o.im end
                end
            end)
        end

        function Tabs:SetMode(mode)
            local light = tostring(mode):lower() == "light" or tostring(mode):lower() == "white"
            currentMode = light and "light" or "dark"
            GuiConfig.ThemeMode = currentMode

            -- latar window + gradient
            local grad = Main:FindFirstChildOfClass("UIGradient")
            if light then
                Main.BackgroundColor3 = Color3.fromRGB(240, 240, 246)
                if grad then grad.Color = ColorSequence.new(Color3.fromRGB(255, 255, 255), Color3.fromRGB(205, 205, 212)) end
            else
                Main.BackgroundColor3 = GuiConfig.Color2
                if grad then grad.Color = ColorSequence.new(Color3.fromRGB(255, 255, 255), Color3.fromRGB(80, 80, 80)) end
            end
            local ov = Main:FindFirstChild("DarkOverlay")
            if ov then ov.BackgroundColor3 = light and Color3.fromRGB(255, 255, 255) or Color3.fromRGB(0, 0, 0) end

            for _, d in ipairs(Main:GetDescendants()) do convert(d, light) end

            if descConn then descConn:Disconnect(); descConn = nil end
            if light then
                descConn = Main.DescendantAdded:Connect(function(d)
                    task.defer(convert, d, true)
                end)
                table.insert(GuiFunc._conns, descConn)
            end
        end
        function Tabs:GetMode() return currentMode end
    end

    -- Ganti warna & transparansi popup dropdown kapan aja.
    -- Contoh: Tabs:SetDropdownStyle(Color3.fromRGB(20, 12, 40), 0.3)
    function Tabs:SetDropdownStyle(color, transparency)
        color = color or GuiConfig.DropdownColor
        if transparency == nil then transparency = GuiConfig.DropdownTransparency end
        GuiConfig.DropdownColor = color
        GuiConfig.DropdownTransparency = transparency

        -- warna teks otomatis menyesuaikan terang/gelapnya popup
        local lightPopup = GetContrastColor(color).R < 0.5
        local optionText = lightPopup and Color3.fromRGB(20, 20, 20) or Color3.fromRGB(230, 230, 230)
        local searchText = lightPopup and Color3.fromRGB(10, 10, 10) or Color3.fromRGB(255, 255, 255)
        GuiConfig._DropdownOptionText = optionText
        GuiConfig._DropdownSearchText = searchText
        GuiConfig._DropdownPlaceholder = searchText:Lerp(color, 0.25)

        TweenService:Create(DropdownSelect, TweenInfo.new(0.3), {
            BackgroundColor3 = color,
            BackgroundTransparency = transparency
        }):Play()
        DropdownSelectReal.BackgroundColor3 = color

        -- update dropdown yang udah kebuat
        for _, d in ipairs(DropdownFolder:GetDescendants()) do
            if d.Name == "OptionText" and d:IsA("TextLabel") then
                d.TextColor3 = optionText
            elseif d.Name == "SearchBox" and d:IsA("TextBox") then
                d.TextColor3 = searchText
                d.PlaceholderColor3 = GuiConfig._DropdownPlaceholder
            end
        end
    end
    Tabs:SetDropdownStyle(GuiConfig.DropdownColor, GuiConfig.DropdownTransparency)

    local CountTab = 0
    local CountDropdown = 0
    -- ===== Tabs:AddTabSection("Nama") -> judul grup di sidebar (taruh sebelum tab-tab yang mau dikelompokkan) =====
    function Tabs:AddTabSection(name)
        name = (type(name) == "string" and name) or (type(name) == "table" and name.Name) or "Section"
        name = tostring(name):match("^%s*(.-)%s*$")
        if name == "" then name = "Section" end

        local Header = Instance.new("Frame")
        Header.Name = "TabSection"
        Header.BackgroundTransparency = 1
        Header.BorderSizePixel = 0
        Header.Size = UDim2.new(1, 0, 0, 26)
        -- LayoutOrder sama dengan tab berikutnya; urutan dibuat lebih dulu = tampil di atas tab itu
        Header.LayoutOrder = CountTab
        Header.Parent = ScrollTab

        New("TextLabel", {
            Name = "SectionTitle", BackgroundTransparency = 1, Font = Enum.Font.GothamMedium,
            Text = name, TextSize = 11, TextColor3 = Color3.fromRGB(145, 145, 150), TextTransparency = 0.2,
            TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd,
            Size = UDim2.new(1, 0, 1, 0),
        }, Header)
        New("UIPadding", { PaddingLeft = UDim.new(0, 10), PaddingRight = UDim.new(0, 8) }, Header)
        return Header
    end

    function Tabs:AddTab(TabConfig)
        local TabConfig = TabConfig or {}
        TabConfig.Name = TabConfig.Name or "Tab"
        TabConfig.Icon = TabConfig.Icon or ""

        local ScrolLayers = Instance.new("ScrollingFrame");

        ScrolLayers.ScrollBarImageColor3 = Color3.fromRGB(80.00000283122063, 80.00000283122063, 80.00000283122063)
        ScrolLayers.ScrollBarThickness = 0
        ScrolLayers.Active = true
        ScrolLayers.LayoutOrder = CountTab
        ScrolLayers.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        ScrolLayers.BackgroundTransparency = 0.9990000128746033
        ScrolLayers.BorderColor3 = Color3.fromRGB(0, 0, 0)
        ScrolLayers.BorderSizePixel = 0
        ScrolLayers.Size = UDim2.new(1, 0, 1, 0)
        ScrolLayers.Name = "ScrolLayers"
        ScrolLayers.Parent = LayersFolder

        New("UIListLayout", { Padding = UDim.new(0, 3), SortOrder = Enum.SortOrder.LayoutOrder }, ScrolLayers)

        local Tab = Instance.new("Frame");
        local UICorner3 = Instance.new("UICorner");
        local TabButton = Instance.new("TextButton");
        local TabName = Instance.new("TextLabel")

        Tab.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        if CountTab == 0 then
            Tab.BackgroundTransparency = 0.9200000166893005
        else
            Tab.BackgroundTransparency = 0.9990000128746033
        end
        Tab.BorderColor3 = Color3.fromRGB(0, 0, 0)
        Tab.BorderSizePixel = 0
        Tab.LayoutOrder = CountTab
        Tab.Size = UDim2.new(1, 0, 0, 30)
        Tab.Name = "Tab"
        Tab.Parent = ScrollTab

        UICorner3.CornerRadius = UDim.new(0, 4)
        UICorner3.Parent = Tab

        TabButton.Font = Enum.Font.GothamBold
        TabButton.Text = ""
        TabButton.TextColor3 = Color3.fromRGB(255, 255, 255)
        TabButton.TextSize = 13
        TabButton.TextXAlignment = Enum.TextXAlignment.Left
        TabButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        TabButton.BackgroundTransparency = 0.9990000128746033
        TabButton.BorderColor3 = Color3.fromRGB(0, 0, 0)
        TabButton.BorderSizePixel = 0
        TabButton.Size = UDim2.new(1, 0, 1, 0)
        TabButton.Name = "TabButton"
        TabButton.Parent = Tab

        TabName.Font = Enum.Font.GothamBold
        TabName.Text = tostring(TabConfig.Name)
        TabName.TextColor3 = Color3.fromRGB(255, 255, 255)
        TabName.TextSize = 13
        TabName.TextXAlignment = Enum.TextXAlignment.Left
        TabName.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        TabName.BackgroundTransparency = 0.9990000128746033
        TabName.BorderColor3 = Color3.fromRGB(0, 0, 0)
        TabName.BorderSizePixel = 0
        TabName.Size = UDim2.new(1, 0, 1, 0)
        TabName.Position = UDim2.new(0, 8, 0, 0)
        TabName.Name = "TabName"
        TabName.Parent = Tab

        -- Ikon tab (nama dari tabel Icons, mis. "eyes", atau angka asset ID)
        local tabIconImage = ResolveIcon(TabConfig.Icon)
        if tabIconImage then
            local TabIcon = Instance.new("ImageLabel")
            TabIcon.Name = "TabIcon"
            TabIcon.Image = tabIconImage
            TabIcon.BackgroundTransparency = 1
            TabIcon.Size = UDim2.new(0, 16, 0, 16)
            TabIcon.Position = UDim2.new(0, 10, 0.5, -8)
            TabIcon.Parent = Tab
            TabName.Position = UDim2.new(0, 33, 0, 0)
            TabName.Size = UDim2.new(1, -33, 1, 0)
        end

        if CountTab == 0 then
            LayersPageLayout:JumpToIndex(0)
            NameTab.Text = TabConfig.Name
            local ChooseFrame = Instance.new("Frame");
            ChooseFrame.BackgroundColor3 = GuiConfig.Color
            ChooseFrame.BorderColor3 = Color3.fromRGB(0, 0, 0)
            ChooseFrame.BorderSizePixel = 0
            ChooseFrame.Position = UDim2.new(0, 2, 0, 9)
            ChooseFrame.Size = UDim2.new(0, 1, 0, 12)
            ChooseFrame.Name = "ChooseFrame"
            ChooseFrame.Parent = Tab

        New("UIStroke", { Color = GuiConfig.Color, Thickness = 1.600000023841858 }, ChooseFrame)

        New("UICorner", {  }, ChooseFrame)
        end

        local SelectEvent = Instance.new("BindableEvent")
        SelectEvent.Name = "SelectEvent"
        SelectEvent.Parent = Tab

        TabButton.Activated:Connect(function()
            CircleClick(TabButton, Mouse.X, Mouse.Y)
            SelectEvent:Fire()
        end)

        SelectEvent.Event:Connect(function()
            local FrameChoose
            for a, s in ScrollTab:GetChildren() do
                for i, v in s:GetChildren() do
                    if v.Name == "ChooseFrame" then
                        FrameChoose = v
                        break
                    end
                end
            end
            if FrameChoose ~= nil and Tab.LayoutOrder ~= LayersPageLayout.CurrentPage.LayoutOrder then
                for _, TabFrame in ScrollTab:GetChildren() do
                    if TabFrame.Name == "Tab" then
                        TweenService:Create(
                            TabFrame,
                            TweenInfo.new(0.3, Enum.EasingStyle.Back, Enum.EasingDirection.InOut),
                            { BackgroundTransparency = 0.9990000128746033 }
                        ):Play()
                    end
                end
                TweenService:Create(
                    Tab,
                    TweenInfo.new(0.6, Enum.EasingStyle.Back, Enum.EasingDirection.InOut),
                    { BackgroundTransparency = 0.9200000166893005 }
                ):Play()
                TweenService:Create(
                    FrameChoose,
                    TweenInfo.new(0.5, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut),
                    { Position = UDim2.new(0, 2, 0, 9 + math.floor(Tab.AbsolutePosition.Y - FrameChoose.Parent.AbsolutePosition.Y + 0.5)) }
                ):Play()
                LayersPageLayout:JumpToIndex(Tab.LayoutOrder)
                task.wait(0.05)
                NameTab.Text = TabConfig.Name
                TweenService:Create(
                    FrameChoose,
                    TweenInfo.new(0.35, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut),
                    { Size = UDim2.new(0, 1, 0, 20) }
                ):Play()
                task.wait(0.2)
                TweenService:Create(
                    FrameChoose,
                    TweenInfo.new(0.25, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut),
                    { Size = UDim2.new(0, 1, 0, 12) }
                ):Play()
            end
        end)
        --// Section
        local Sections = {}
        local CountSection = 0
        function Sections:AddSection(Title, AlwaysOpen, DefaultOpen)
            -- Default: section bisa buka-tutup (ada panah) & awalnya TERTUTUP.
            --   AddSection("Judul")              -> buka-tutup, awal tertutup
            --   AddSection("Judul", false, true) -> buka-tutup, awal terbuka
            --   AddSection("Judul", true)        -> selalu terbuka, tanpa panah
            if AlwaysOpen == nil then AlwaysOpen = false end
            if DefaultOpen == nil then DefaultOpen = false end
            local Title = Title or "Title"
            local Section = Instance.new("Frame");
            local UICorner1 = Instance.new("UICorner");
            local UIGradient = Instance.new("UIGradient");

            Section.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            Section.BackgroundTransparency = 0.9990000128746033
            Section.BorderColor3 = Color3.fromRGB(0, 0, 0)
            Section.BorderSizePixel = 0
            Section.LayoutOrder = CountSection
            Section.ClipsDescendants = true
            Section.LayoutOrder = 1
            Section.Size = UDim2.new(1, 0, 0, 30)
            Section.Name = "Section"
            Section.Parent = ScrolLayers

            local SectionReal = Instance.new("Frame");
            local UICorner = Instance.new("UICorner");
            local UIStroke = Instance.new("UIStroke");
            local SectionButton = Instance.new("TextButton");
            local FeatureFrame = Instance.new("Frame");
            local FeatureImg = Instance.new("ImageLabel");
            local SectionTitle = Instance.new("TextLabel");

            SectionReal.AnchorPoint = Vector2.new(0.5, 0)
            SectionReal.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            SectionReal.BackgroundTransparency = 1
            SectionReal.BorderColor3 = Color3.fromRGB(0, 0, 0)
            SectionReal.BorderSizePixel = 0
            SectionReal.LayoutOrder = 1
            SectionReal.Position = UDim2.new(0.5, 0, 0, 0)
            SectionReal.Size = UDim2.new(1, 1, 0, 30)
            SectionReal.Name = "SectionReal"
            SectionReal.Parent = Section

            UICorner.CornerRadius = UDim.new(0, 4)
            UICorner.Parent = SectionReal

            SectionButton.Font = Enum.Font.SourceSans
            SectionButton.Text = ""
            SectionButton.TextColor3 = Color3.fromRGB(0, 0, 0)
            SectionButton.TextSize = 14
            SectionButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            SectionButton.BackgroundTransparency = 0.9990000128746033
            SectionButton.BorderColor3 = Color3.fromRGB(0, 0, 0)
            SectionButton.BorderSizePixel = 0
            SectionButton.Size = UDim2.new(1, 0, 1, 0)
            SectionButton.Name = "SectionButton"
            SectionButton.Parent = SectionReal

            FeatureFrame.AnchorPoint = Vector2.new(1, 0.5)
            FeatureFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
            FeatureFrame.BackgroundTransparency = 0.9990000128746033
            FeatureFrame.BorderColor3 = Color3.fromRGB(0, 0, 0)
            FeatureFrame.BorderSizePixel = 0
            FeatureFrame.Position = UDim2.new(1, -5, 0.5, 0)
            FeatureFrame.Size = UDim2.new(0, 20, 0, 20)
            FeatureFrame.Name = "FeatureFrame"
            FeatureFrame.Parent = SectionReal

            FeatureImg.Image = "rbxassetid://16851841101"
            FeatureImg.AnchorPoint = Vector2.new(0.5, 0.5)
            FeatureImg.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            FeatureImg.BackgroundTransparency = 0.9990000128746033
            FeatureImg.BorderColor3 = Color3.fromRGB(0, 0, 0)
            FeatureImg.BorderSizePixel = 0
            FeatureImg.Position = UDim2.new(0.5, 0, 0.5, 0)
            FeatureImg.Rotation = -90
            FeatureImg.Size = UDim2.new(1, 6, 1, 6)
            FeatureImg.Name = "FeatureImg"
            FeatureImg.Parent = FeatureFrame

            SectionTitle.Font = Enum.Font.GothamBold
            SectionTitle.Text = Title
            SectionTitle.TextColor3 = Color3.fromRGB(230.77499270439148, 230.77499270439148, 230.77499270439148)
            SectionTitle.TextSize = 13
            SectionTitle.TextXAlignment = Enum.TextXAlignment.Left
            SectionTitle.TextYAlignment = Enum.TextYAlignment.Top
            SectionTitle.AnchorPoint = Vector2.new(0, 0.5)
            SectionTitle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            SectionTitle.BackgroundTransparency = 0.9990000128746033
            SectionTitle.BorderColor3 = Color3.fromRGB(0, 0, 0)
            SectionTitle.BorderSizePixel = 0
            SectionTitle.Position = UDim2.new(0, 10, 0.5, 0)
            SectionTitle.Size = UDim2.new(1, -50, 0, 13)
            SectionTitle.Name = "SectionTitle"
            SectionTitle.Parent = SectionReal

            --// Section Add
            local SectionAdd = Instance.new("Frame");

            SectionAdd.AnchorPoint = Vector2.new(0.5, 0)
            SectionAdd.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
            SectionAdd.BackgroundTransparency = 0.94 -- kartu: sedikit lebih terang dari window
            SectionAdd.BorderColor3 = Color3.fromRGB(0, 0, 0)
            SectionAdd.BorderSizePixel = 0
            SectionAdd.ClipsDescendants = true
            SectionAdd.LayoutOrder = 1
            SectionAdd.Position = UDim2.new(0.5, 0, 0, 32)
            SectionAdd.Size = UDim2.new(1, -6, 0, 100) -- -6: ruang untuk bingkai kartu (tidak terpotong)
            SectionAdd.Name = "SectionAdd"
            SectionAdd.Parent = Section

            New("UICorner", { CornerRadius = UDim.new(0, 6) }, SectionAdd)

            -- ═══ KARTU SECTION (bingkai tipis berwarna aksen + padding dalam) ═══
            local CARD_PAD = 5
            local CardStroke = Instance.new("UIStroke")
            CardStroke.Name = "CardStroke"
            CardStroke.Color = GuiConfig.Color
            CardStroke.Thickness = 1
            CardStroke.Transparency = 0.55
            CardStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
            CardStroke.Parent = SectionAdd

            local CardPadding = Instance.new("UIPadding")
            CardPadding.Name = "CardPadding"
            CardPadding.PaddingTop = UDim.new(0, CARD_PAD)
            CardPadding.PaddingBottom = UDim.new(0, CARD_PAD)
            CardPadding.PaddingLeft = UDim.new(0, CARD_PAD)
            CardPadding.PaddingRight = UDim.new(0, CARD_PAD)
            CardPadding.Parent = SectionAdd

            New("UIListLayout", { Padding = UDim.new(0, 3), SortOrder = Enum.SortOrder.LayoutOrder }, SectionAdd)

            local OpenSection = false

            local function UpdateSizeScroll()
                local OffsetY = 0
                for _, child in ScrolLayers:GetChildren() do
                    if child.Name ~= "UIListLayout" then
                        OffsetY = OffsetY + 3 + child.Size.Y.Offset
                    end
                end
                ScrolLayers.CanvasSize = UDim2.new(0, 0, 0, OffsetY)
            end

            local function UpdateSizeSection()
                if OpenSection then
                    local SectionSizeYWitdh = 38 + CARD_PAD * 2
                    for _, v in SectionAdd:GetChildren() do
                        if v:IsA("GuiObject") and v.Visible then
                            SectionSizeYWitdh = SectionSizeYWitdh + v.Size.Y.Offset + 3
                        end
                    end
                    TweenService:Create(FeatureFrame, TweenInfo.new(0.5), { Rotation = 90 }):Play()
                    TweenService:Create(SectionTitle, TweenInfo.new(0.5), { TextColor3 = Color3.fromRGB(255, 255, 255) }):Play()
                    TweenService:Create(Section, TweenInfo.new(0.5), { Size = UDim2.new(1, 1, 0, SectionSizeYWitdh) })
                        :Play()
                    TweenService:Create(SectionAdd, TweenInfo.new(0.5),
                        { Size = UDim2.new(1, -6, 0, SectionSizeYWitdh - 38) }):Play()
                    task.wait(0.5)
                    UpdateSizeScroll()
                end
            end

            if AlwaysOpen == true then
                SectionButton:Destroy()
                FeatureFrame:Destroy()
                OpenSection = true
            elseif DefaultOpen == true then
                OpenSection = true
            else
                OpenSection = false
            end

            if AlwaysOpen ~= true then
                SectionButton.Activated:Connect(function()
                    CircleClick(SectionButton, Mouse.X, Mouse.Y)
                    if OpenSection then
                        TweenService:Create(FeatureFrame, TweenInfo.new(0.5), { Rotation = 0 }):Play()
                        TweenService:Create(SectionTitle, TweenInfo.new(0.5), { TextColor3 = Color3.fromRGB(231, 231, 231) }):Play()
                        TweenService:Create(Section, TweenInfo.new(0.5), { Size = UDim2.new(1, 1, 0, 30) }):Play()
                        OpenSection = false
                        task.wait(0.5)
                        UpdateSizeScroll()
                    else
                        OpenSection = true
                        UpdateSizeSection()
                    end
                end)
            end

            if AlwaysOpen == true or DefaultOpen == true then
                OpenSection = true
                local SectionSizeYWitdh = 38 + CARD_PAD * 2
                for _, v in SectionAdd:GetChildren() do
                    if v:IsA("GuiObject") and v.Visible then
                        SectionSizeYWitdh = SectionSizeYWitdh + v.Size.Y.Offset + 3
                    end
                end
                FeatureFrame.Rotation = 90
                SectionTitle.TextColor3 = Color3.fromRGB(255, 255, 255)
                Section.Size = UDim2.new(1, 1, 0, SectionSizeYWitdh)
                SectionAdd.Size = UDim2.new(1, -6, 0, SectionSizeYWitdh - 38)
                UpdateSizeScroll()
            end

            SectionAdd.ChildAdded:Connect(function(child)
                if child:IsA("GuiObject") then
                    child:GetPropertyChangedSignal("Visible"):Connect(UpdateSizeSection)
                end
                UpdateSizeSection()
            end)
            SectionAdd.ChildRemoved:Connect(UpdateSizeSection)

            local layout = ScrolLayers:FindFirstChildOfClass("UIListLayout")
            if layout then
                layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
                    ScrolLayers.CanvasSize = UDim2.new(0, 0, 0, layout.AbsoluteContentSize.Y + 10)
                end)
            end

            local Items = {}
            local CountItem = 0

            -- ═══ ROW (item sejajar / multi kolom) ═══
            -- Saat AddRow berjalan, setiap item baru ditaruh ke kolom (slot) berikutnya.
            local RowSlots, RowSlotIndex = nil, 0
            local function ItemParent()
                if RowSlots then
                    RowSlotIndex = math.min(RowSlotIndex + 1, #RowSlots)
                    return RowSlots[RowSlotIndex]
                end
                return SectionAdd
            end

            function Items:AddParagraph(ParagraphConfig)
                local ParagraphConfig = ParagraphConfig or {}
                ParagraphConfig.Title = ParagraphConfig.Title or "Title"
                ParagraphConfig.Content = ParagraphConfig.Content or "Content"
                local ParagraphFunc = {}

                local Paragraph = Instance.new("Frame")
                local ParagraphTitle = Instance.new("TextLabel")
                local ParagraphContent = Instance.new("TextLabel")

                Paragraph.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                Paragraph.BackgroundTransparency = ProjectSenzu.ItemBgTransparency
                Paragraph.BorderSizePixel = 0
                Paragraph.LayoutOrder = CountItem
                Paragraph.Size = UDim2.new(1, 0, 0, 46)
                Paragraph.Name = "Paragraph"
                Paragraph.Parent = ItemParent()

                New("UICorner", { CornerRadius = UDim.new(0, 4) }, Paragraph)

                local iconOffset = 10
                if ParagraphConfig.Icon then
                    local IconImg = Instance.new("ImageLabel")
                    IconImg.Size = UDim2.new(0, 20, 0, 20)
                    IconImg.Position = UDim2.new(0, 8, 0, 12)
                    IconImg.BackgroundTransparency = 1
                    IconImg.Name = "ParagraphIcon"
                    IconImg.Parent = Paragraph

                    if Icons and Icons[ParagraphConfig.Icon] then
                        IconImg.Image = Icons[ParagraphConfig.Icon]
                    else
                        IconImg.Image = ParagraphConfig.Icon
                    end

                    iconOffset = 30
                end

                ParagraphTitle.Font = Enum.Font.GothamBold
                ParagraphTitle.Text = ParagraphConfig.Title
                ParagraphTitle.TextColor3 = Color3.fromRGB(231, 231, 231)
                ParagraphTitle.TextSize = 13
                ParagraphTitle.TextXAlignment = Enum.TextXAlignment.Left
                ParagraphTitle.TextYAlignment = Enum.TextYAlignment.Top
                ParagraphTitle.BackgroundTransparency = 1
                ParagraphTitle.Position = UDim2.new(0, iconOffset, 0, 10)
                ParagraphTitle.Size = UDim2.new(1, -16, 0, 13)
                ParagraphTitle.Name = "ParagraphTitle"
                ParagraphTitle.Parent = Paragraph

                ParagraphContent.Font = Enum.Font.Gotham
                ParagraphContent.Text = ParagraphConfig.Content
                ParagraphContent.TextColor3 = Color3.fromRGB(255, 255, 255)
                ParagraphContent.TextSize = 12
                ParagraphContent.TextXAlignment = Enum.TextXAlignment.Left
                ParagraphContent.TextYAlignment = Enum.TextYAlignment.Top
                ParagraphContent.BackgroundTransparency = 1
                ParagraphContent.Position = UDim2.new(0, iconOffset, 0, 25)
                ParagraphContent.Name = "ParagraphContent"
                ParagraphContent.TextWrapped = true
                ParagraphContent.RichText = true
                ParagraphContent.Parent = Paragraph

                ParagraphContent.Size = UDim2.new(1, -16, 0, 12)

                local ParagraphButton
                if ParagraphConfig.ButtonText then
                    ParagraphButton = Instance.new("TextButton")
                    ParagraphButton.Position = UDim2.new(0, 10, 0, 42)
                    ParagraphButton.Size = UDim2.new(1, -22, 0, 28)
                    ParagraphButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                    ParagraphButton.BackgroundTransparency = 0.935
                    ParagraphButton.Font = Enum.Font.GothamBold
                    ParagraphButton.TextSize = 12
                    ParagraphButton.TextTransparency = 0.3
                    ParagraphButton.TextColor3 = Color3.fromRGB(255, 255, 255)
                    ParagraphButton.Text = ParagraphConfig.ButtonText
                    ParagraphButton.Parent = Paragraph

                    New("UICorner", { CornerRadius = UDim.new(0, 6) }, ParagraphButton)

                    if ParagraphConfig.ButtonCallback then
                        ParagraphButton.MouseButton1Click:Connect(ParagraphConfig.ButtonCallback)
                    end
                end

                FitWrapLabel(ParagraphContent, 12, function(h)
                    local totalHeight = h + 33
                    if ParagraphButton then
                        ParagraphButton.Position = UDim2.new(0, 10, 0, 25 + h + 5)
                        totalHeight = totalHeight + ParagraphButton.Size.Y.Offset + 5
                    end
                    Paragraph.Size = UDim2.new(1, 0, 0, totalHeight)
                end, UpdateSizeSection)

                function ParagraphFunc:SetContent(content)
                    content = content or "Content"
                    ParagraphContent.Text = content
                end

                function ParagraphFunc:SetTitle(title)
                    ParagraphTitle.Text = title or "Title"
                end

                ParagraphFunc.Frame = Paragraph
                CountItem = CountItem + 1
                return ParagraphFunc
            end

            function Items:AddPanel(PanelConfig)
                PanelConfig = PanelConfig or {}
                PanelConfig.Title = PanelConfig.Title or "Title"
                PanelConfig.Content = PanelConfig.Content or ""
                PanelConfig.Placeholder = PanelConfig.Placeholder or nil
                PanelConfig.Default = PanelConfig.Default or ""
                PanelConfig.ButtonText = PanelConfig.Button or PanelConfig.ButtonText or "Confirm"
                PanelConfig.ButtonCallback = PanelConfig.Callback or PanelConfig.ButtonCallback or function() end
                PanelConfig.SubButtonText = PanelConfig.SubButton or PanelConfig.SubButtonText or nil
                PanelConfig.SubButtonCallback = PanelConfig.SubCallback or PanelConfig.SubButtonCallback or
                    function() end

                local configKey = MakeKey("Panel", PanelConfig)
                if ConfigData[configKey] ~= nil then
                    PanelConfig.Default = ConfigData[configKey]
                end

                local PanelFunc = { Value = PanelConfig.Default }

                local baseHeight = 50

                if PanelConfig.Placeholder then
                    baseHeight = baseHeight + 40
                end

                if PanelConfig.SubButtonText then
                    baseHeight = baseHeight + 40
                else
                    baseHeight = baseHeight + 36
                end

                local Panel = Instance.new("Frame")
                Panel.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                Panel.BackgroundTransparency = ProjectSenzu.ItemBgTransparency
                Panel.Size = UDim2.new(1, 0, 0, baseHeight)
                Panel.LayoutOrder = CountItem
                Panel.Parent = ItemParent()

                local UICorner = Instance.new("UICorner")
                UICorner.CornerRadius = UDim.new(0, 4)
                UICorner.Parent = Panel

                local Title = Instance.new("TextLabel")
                Title.Font = Enum.Font.GothamBold
                Title.Text = PanelConfig.Title
                Title.TextSize = 13
                Title.TextColor3 = Color3.fromRGB(255, 255, 255)
                Title.TextXAlignment = Enum.TextXAlignment.Left
                Title.BackgroundTransparency = 1
                Title.Position = UDim2.new(0, 10, 0, 10)
                Title.Size = UDim2.new(1, -20, 0, 13)
                Title.Parent = Panel

                local Content = Instance.new("TextLabel")
                Content.Font = Enum.Font.Gotham
                Content.Text = PanelConfig.Content
                Content.TextSize = 12
                Content.TextColor3 = Color3.fromRGB(255, 255, 255)
                Content.TextTransparency = 0
                Content.TextXAlignment = Enum.TextXAlignment.Left
                Content.BackgroundTransparency = 1
                Content.RichText = true
                Content.Position = UDim2.new(0, 10, 0, 28)
                Content.Size = UDim2.new(1, -20, 0, 14)
                Content.Parent = Panel

                local InputBox
                if PanelConfig.Placeholder then
                    local InputFrame = Instance.new("Frame")
                    InputFrame.AnchorPoint = Vector2.new(0.5, 0)
                    InputFrame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                    InputFrame.BackgroundTransparency = 0.95
                    InputFrame.Position = UDim2.new(0.5, 0, 0, 48)
                    InputFrame.Size = UDim2.new(1, -20, 0, 30)
                    InputFrame.Parent = Panel

                    New("UICorner", { CornerRadius = UDim.new(0, 4) }, InputFrame)

                    InputBox = Instance.new("TextBox")
                    InputBox.Font = Enum.Font.GothamBold
                    InputBox.PlaceholderText = PanelConfig.Placeholder
                    InputBox.PlaceholderColor3 = Color3.fromRGB(120, 120, 120)
                    InputBox.Text = PanelConfig.Default
                    InputBox.TextSize = 11
                    InputBox.TextColor3 = Color3.fromRGB(255, 255, 255)
                    InputBox.BackgroundTransparency = 1
                    InputBox.TextXAlignment = Enum.TextXAlignment.Left
                    InputBox.Size = UDim2.new(1, -10, 1, -6)
                    InputBox.Position = UDim2.new(0, 5, 0, 3)
                    InputBox.Parent = InputFrame
                end

                local yBtn = 0
                if PanelConfig.Placeholder then
                    yBtn = 88
                else
                    yBtn = 48
                end

                local ButtonMain = Instance.new("TextButton")
                ButtonMain.Font = Enum.Font.GothamBold
                ButtonMain.Text = PanelConfig.ButtonText
                ButtonMain.TextColor3 = Color3.fromRGB(255, 255, 255)
                ButtonMain.TextSize = 12
                ButtonMain.TextTransparency = 0.3
                ButtonMain.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                ButtonMain.BackgroundTransparency = 0.935
                ButtonMain.Size = PanelConfig.SubButtonText and UDim2.new(0.5, -12, 0, 30) or UDim2.new(1, -20, 0, 30)
                ButtonMain.Position = UDim2.new(0, 10, 0, yBtn)
                ButtonMain.Parent = Panel

                New("UICorner", { CornerRadius = UDim.new(0, 6) }, ButtonMain)

                ButtonMain.MouseButton1Click:Connect(function()
                    PanelConfig.ButtonCallback(InputBox and InputBox.Text or "")
                end)

                if PanelConfig.SubButtonText then
                    local SubButton = Instance.new("TextButton")
                    SubButton.Font = Enum.Font.GothamBold
                    SubButton.Text = PanelConfig.SubButtonText
                    SubButton.TextColor3 = Color3.fromRGB(255, 255, 255)
                    SubButton.TextSize = 12
                    SubButton.TextTransparency = 0.3
                    SubButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                    SubButton.BackgroundTransparency = 0.935
                    SubButton.Size = UDim2.new(0.5, -12, 0, 30)
                    SubButton.Position = UDim2.new(0.5, 2, 0, yBtn)
                    SubButton.Parent = Panel

                    New("UICorner", { CornerRadius = UDim.new(0, 6) }, SubButton)

                    SubButton.MouseButton1Click:Connect(function()
                        PanelConfig.SubButtonCallback(InputBox and InputBox.Text or "")
                    end)
                end

                if InputBox then
                    InputBox.FocusLost:Connect(function()
                        PanelFunc.Value = InputBox.Text
                        ConfigData[configKey] = InputBox.Text
                        SaveConfig()
                    end)
                end

                function PanelFunc:GetInput()
                    return InputBox and InputBox.Text or ""
                end

                CountItem = CountItem + 1
                return PanelFunc
            end

            function Items:AddButton(ButtonConfig)
                ButtonConfig = ButtonConfig or {}
                ButtonConfig.Title = ButtonConfig.Title or "Confirm"
                ButtonConfig.Callback = ButtonConfig.Callback or function() end
                ButtonConfig.SubTitle = ButtonConfig.SubTitle or nil
                ButtonConfig.SubCallback = ButtonConfig.SubCallback or function() end

                local Button = Instance.new("Frame")
                Button.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                Button.BackgroundTransparency = 1
                Button.Size = UDim2.new(1, 0, 0, 40)
                Button.LayoutOrder = CountItem
                Button.Parent = ItemParent()

                local UICorner = Instance.new("UICorner")
                UICorner.CornerRadius = UDim.new(0, 4)
                UICorner.Parent = Button

                -- Konfigurasi Animasi
                local tweenInfo = TweenInfo.new(0.15, Enum.EasingStyle.Quint, Enum.EasingDirection.Out)

                -- MAIN BUTTON
                local MainButton = Instance.new("TextButton")
                MainButton.Font = Enum.Font.GothamBold
                MainButton.Text = "    " .. ButtonConfig.Title
                MainButton.TextSize = 12
                MainButton.TextColor3 = Color3.fromRGB(255, 255, 255)
                MainButton.TextTransparency = 0.3
                MainButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                MainButton.BackgroundTransparency = 0.935
                MainButton.AutoButtonColor = false
                MainButton.TextXAlignment = Enum.TextXAlignment.Left

                local ButtonIcon = Instance.new("ImageLabel")
                ButtonIcon.Name = "Icon"
                ButtonIcon.Parent = MainButton -- Masuk ke dalam MainButton

                -- Ukuran & Posisi (Sesuaikan angka di bawah dengan kebutuhanmu)
                ButtonIcon.Size = UDim2.new(0, 20, 0, 20) -- Ganti 20, 20 dengan ukuranmu
                ButtonIcon.Position = UDim2.new(0.92, 0, 0.5, 0) -- Ganti 5 dengan posisi X keinginanmu
                ButtonIcon.AnchorPoint = Vector2.new(0, 0.5) -- Agar posisi Y (0.5) benar-benar di tengah vertikal

                -- Properti Tampilan
                ButtonIcon.BackgroundTransparency = 1 -- Agar background icon tidak kelihatan
                ButtonIcon.Image = "rbxassetid://90520342625816" -- Ganti 0 dengan ID gambarmu
                ButtonIcon.ImageColor3 = Color3.fromRGB(255, 255, 255) -- Warna icon
                ButtonIcon.ImageTransparency = 0.3 -- Biar estetik (cocok dengan teksmu yang transparan 0.3)
                ButtonIcon.ScaleType = Enum.ScaleType.Fit -- Agar gambar tidak gepeng
                
                local mainNormalSize = ButtonConfig.SubTitle
                    and UDim2.new(0.5, -4, 1, 0)
                    or  UDim2.new(1, 0, 1, 0)
                local mainShrinkSize = UDim2.new(
                    mainNormalSize.X.Scale,
                    mainNormalSize.X.Offset - 2,
                    mainNormalSize.Y.Scale,
                    mainNormalSize.Y.Offset - 2
                )

                MainButton.Size     = mainNormalSize
                MainButton.Position = UDim2.new(0, 0, 0, 0)
                MainButton.AnchorPoint = Vector2.new(0, 0)
                MainButton.Parent = Button


                New("UICorner", { CornerRadius = UDim.new(0, 4) }, MainButton)

                -- Efek Animasi Main Button
                MainButton.MouseButton1Down:Connect(function()
                    TweenService:Create(MainButton, tweenInfo, {Size = mainShrinkSize}):Play()
                end)
                MainButton.MouseButton1Up:Connect(function()
                    TweenService:Create(MainButton, tweenInfo, {Size = mainNormalSize}):Play()
                end)
                MainButton.MouseLeave:Connect(function()
                    TweenService:Create(MainButton, tweenInfo, {Size = mainNormalSize}):Play()
                end)

                MainButton.MouseButton1Click:Connect(ButtonConfig.Callback)

                -- SUB BUTTON
                if ButtonConfig.SubTitle then
                    local SubButton = Instance.new("TextButton")
                    SubButton.Font = Enum.Font.GothamBold
                    SubButton.Text = "    " .. ButtonConfig.SubTitle
                    SubButton.TextSize = 12
                    SubButton.TextTransparency = 0.3
                    SubButton.TextColor3 = Color3.fromRGB(255, 255, 255)
                    SubButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                    SubButton.BackgroundTransparency = 0.935
                    SubButton.AutoButtonColor = false
                    SubButton.TextXAlignment = Enum.TextXAlignment.Left
                    
                    local SubButtonIcon = Instance.new("ImageLabel")
                    SubButtonIcon.Name = "Icon"
                    SubButtonIcon.Parent = SubButton
                    SubButtonIcon.Size = UDim2.new(0, 20, 0, 20)
                    SubButtonIcon.Position = UDim2.new(0.92, 0, 0.5, 0)
                    SubButtonIcon.AnchorPoint = Vector2.new(0, 0.5)
                    SubButtonIcon.BackgroundTransparency = 1
                    SubButtonIcon.Image = "rbxassetid://90520342625816"
                    SubButtonIcon.ImageColor3 = Color3.fromRGB(255, 255, 255)
                    SubButtonIcon.ImageTransparency = 0.3
                    SubButtonIcon.ScaleType = Enum.ScaleType.Fit

                    local subNormalSize = UDim2.new(0.5, -4, 1, 0)
                    local subShrinkSize = UDim2.new(
                        subNormalSize.X.Scale,
                        subNormalSize.X.Offset - 2,
                        subNormalSize.Y.Scale,
                        subNormalSize.Y.Offset - 2
                    )

                    SubButton.Size     = subNormalSize
                    SubButton.Position = UDim2.new(0.5, 4, 0, 0)
                    SubButton.AnchorPoint = Vector2.new(0, 0)
                    SubButton.Parent = Button


                    New("UICorner", { CornerRadius = UDim.new(0, 4) }, SubButton)

                    -- Efek Animasi Sub Button
                    SubButton.MouseButton1Down:Connect(function()
                        TweenService:Create(SubButton, tweenInfo, {Size = subShrinkSize}):Play()
                    end)
                    SubButton.MouseButton1Up:Connect(function()
                        TweenService:Create(SubButton, tweenInfo, {Size = subNormalSize}):Play()
                    end)
                    SubButton.MouseLeave:Connect(function()
                        TweenService:Create(SubButton, tweenInfo, {Size = subNormalSize}):Play()
                    end)

                    SubButton.MouseButton1Click:Connect(ButtonConfig.SubCallback)
                end

                CountItem = CountItem + 1
            end

            -- ===== Items:SetHeaderToggle({Default=true, OnChanged=function(v) end, ActiveColor=Color3, Flag="x"}) =====
            -- Switch kecil di header section (sebelah panah). Nilai disimpan ke config.
            function Items:SetHeaderToggle(HeaderConfig)
                HeaderConfig = HeaderConfig or {}
                local onChanged = typeof(HeaderConfig.OnChanged) == "function" and HeaderConfig.OnChanged or function() end
                local activeColor = HeaderConfig.ActiveColor or GuiConfig.Color
                local offColor = Color3.fromRGB(75, 75, 80)

                local configKey = MakeKey("HeaderToggle", { Flag = HeaderConfig.Flag, Title = HeaderConfig.Title or SectionTitle.Text })
                local value = HeaderConfig.Default == true
                if ConfigData[configKey] ~= nil then value = ConfigData[configKey] == true end

                local arrowSpace = (FeatureFrame and FeatureFrame.Parent and FeatureFrame.Visible) and 28 or 8
                SectionTitle.Size = UDim2.new(1, -(arrowSpace + 56), 0, 13)

                local Switch = Instance.new("TextButton")
                Switch.Name = "SectionToggle"
                Switch.AutoButtonColor = false
                Switch.Text = ""
                Switch.AnchorPoint = Vector2.new(1, 0.5)
                Switch.Position = UDim2.new(1, -arrowSpace, 0.5, 0)
                Switch.Size = UDim2.fromOffset(36, 18)
                Switch.BorderSizePixel = 0
                Switch.ZIndex = 5
                Switch.BackgroundColor3 = value and activeColor or offColor
                Switch.Parent = SectionReal
                New("UICorner", { CornerRadius = UDim.new(1, 0) }, Switch)

                local Knob = New("Frame", {
                    Name = "Knob", BackgroundColor3 = Color3.fromRGB(255, 255, 255), BorderSizePixel = 0,
                    Size = UDim2.fromOffset(14, 14), ZIndex = 6,
                    Position = UDim2.new(0, value and 20 or 2, 0.5, -7),
                }, Switch)
                New("UICorner", { CornerRadius = UDim.new(1, 0) }, Knob)

                local HeaderFunc = {}
                local function apply(v, silent)
                    value = v and true or false
                    TweenService:Create(Switch, TweenInfo.new(0.16), { BackgroundColor3 = value and activeColor or offColor }):Play()
                    TweenService:Create(Knob, TweenInfo.new(0.16), { Position = UDim2.new(0, value and 20 or 2, 0.5, -7) }):Play()
                    ConfigData[configKey] = value
                    SaveConfig()
                    if not silent then pcall(onChanged, value) end
                end
                function HeaderFunc:Set(v) apply(v) end
                function HeaderFunc:GetValue() return value end

                Switch.Activated:Connect(function()
                    PlaySound("Element")
                    apply(not value)
                end)
                -- nilai awal (dari Default / config) diberitahukan ke OnChanged sekali
                task.defer(function() pcall(onChanged, value) end)
                return HeaderFunc
            end

            -- ===== Items:AddChat: chat global (backend: Vercel, lihat senzu-chat-api) =====
            -- Items:AddChat({ ApiBaseUrl="https://NAMA.vercel.app", Room="global", Height=280, RefreshSeconds=8 })
            -- Klik pesan = balas. Tier: Owner/Reseller dari UserId (server), Premium dari LRM_KeyTier, keyless/tanpa key = Free.
            function Items:AddChat(ChatConfig)
                ChatConfig = ChatConfig or {}
                local baseUrl = (tostring(ChatConfig.ApiBaseUrl or ProjectSenzu.ChatApiBaseUrl or "")):gsub("/+$", "")
                local room = tostring(ChatConfig.Room or "global")
                local height = tonumber(ChatConfig.Height) or 280
                local refreshSec = math.max(3, tonumber(ChatConfig.RefreshSeconds) or 8)
                local maxBytes = tonumber(ChatConfig.MaxMessageBytes) or 280
                local maxMessages = tonumber(ChatConfig.MaxMessages) or 50
                local keyTier = ChatConfig.KeyTier -- default: LRM_KeyTier (Luarmor), kosong kalau tanpa key

                local TIER_COLORS = {
                    Owner = Color3.fromRGB(255, 196, 64), Reseller = Color3.fromRGB(80, 170, 255),
                    Premium = Color3.fromRGB(190, 110, 255), Free = Color3.fromRGB(170, 170, 178),
                }
                local TIER_LABELS = { Owner = "| Owner", Reseller = "| Reseller", Premium = "| Premium", Free = "| Free" }

                local requestFn = (syn and syn.request) or http_request or request or (http and http.request)
                local state = { token = nil, tokenExpiry = 0, busy = false, loading = false, messages = {}, lastKey = "", reply = nil, alive = true, lastFetch = -1e9 }

                local function esc(t) return (tostring(t):gsub("&", "&amp;"):gsub("<", "&lt;"):gsub(">", "&gt;")) end
                local function hex(c)
                    return string.format("#%02x%02x%02x", math.floor(c.R * 255 + 0.5), math.floor(c.G * 255 + 0.5), math.floor(c.B * 255 + 0.5))
                end

                -- return: data, err, statusCode
                local function api(path, method, body, token)
                    if baseUrl == "" then return nil, "Set ApiBaseUrl" end
                    if not requestFn then return nil, "Executor tidak support http request" end
                    local headers = { ["Content-Type"] = "application/json" }
                    if token then headers["Authorization"] = "Bearer " .. token end
                    local ok, res = pcall(requestFn, {
                        Url = baseUrl .. path, Method = method, Headers = headers,
                        Body = body and HttpService:JSONEncode(body) or nil,
                    })
                    if not ok or type(res) ~= "table" then return nil, "Request gagal" end
                    local code = tonumber(res.StatusCode or res.Status)
                    local decoded
                    pcall(function() decoded = HttpService:JSONDecode(res.Body) end)
                    if code and code >= 200 and code < 300 and type(decoded) == "table" then return decoded, nil, code end
                    return nil, (type(decoded) == "table" and decoded.error) or ("HTTP " .. tostring(code)), code
                end

                local function ensureSession()
                    if state.token and state.tokenExpiry > os.time() + 60 then return true end
                    local data, err = api("/api/chat/session", "POST", {})
                    if not data or type(data.token) ~= "string" then return false, err end
                    state.token = data.token
                    state.tokenExpiry = tonumber(data.expiresAt) or (os.time() + 3000)
                    return true
                end

                -- ===== UI =====
                local Card = Instance.new("Frame")
                Card.Name = "Chat"
                Card.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                Card.BackgroundTransparency = 0.94
                Card.Size = UDim2.new(1, 0, 0, height)
                Card.LayoutOrder = CountItem
                Card.Parent = ItemParent()
                New("UICorner", { CornerRadius = UDim.new(0, 4) }, Card)
                New("UIStroke", { Color = GuiConfig.Color, Transparency = 0.7, Thickness = 1, ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, Card)

                local Status = New("TextLabel", {
                    Name = "Status", BackgroundTransparency = 1, Font = Enum.Font.GothamMedium, Text = "Chat",
                    TextSize = 10, TextColor3 = Color3.fromRGB(255, 255, 255), TextTransparency = 0.5,
                    TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd,
                    Position = UDim2.new(0, 8, 0, 3), Size = UDim2.new(1, -16, 0, 14),
                }, Card)

                local List = New("ScrollingFrame", {
                    Name = "Messages", BackgroundTransparency = 1, BorderSizePixel = 0, ScrollBarThickness = 3,
                    ScrollBarImageColor3 = Color3.fromRGB(120, 120, 120), CanvasSize = UDim2.new(0, 0, 0, 0),
                    AutomaticCanvasSize = Enum.AutomaticSize.Y, Position = UDim2.new(0, 6, 0, 20),
                }, Card)
                New("UIListLayout", { Padding = UDim.new(0, 5), SortOrder = Enum.SortOrder.LayoutOrder }, List)
                New("UIPadding", { PaddingRight = UDim.new(0, 4), PaddingBottom = UDim.new(0, 2) }, List)

                local ReplyBar = New("TextButton", {
                    Name = "ReplyBar", Visible = false, AutoButtonColor = false, BackgroundColor3 = Color3.fromRGB(255, 255, 255),
                    BackgroundTransparency = 0.9, Font = Enum.Font.Gotham, TextSize = 10, Text = "",
                    TextColor3 = Color3.fromRGB(255, 255, 255), TextXAlignment = Enum.TextXAlignment.Left,
                    TextTruncate = Enum.TextTruncate.AtEnd, Position = UDim2.new(0, 6, 1, -54), Size = UDim2.new(1, -12, 0, 16),
                }, Card)
                New("UICorner", { CornerRadius = UDim.new(0, 3) }, ReplyBar)
                New("UIPadding", { PaddingLeft = UDim.new(0, 6), PaddingRight = UDim.new(0, 6) }, ReplyBar)

                local Input = New("TextBox", {
                    Name = "Input", BackgroundColor3 = Color3.fromRGB(255, 255, 255), BackgroundTransparency = 0.9,
                    Font = Enum.Font.Gotham, TextSize = 12, Text = "", PlaceholderText = "Type a message...",
                    PlaceholderColor3 = Color3.fromRGB(150, 150, 155), TextColor3 = Color3.fromRGB(255, 255, 255),
                    TextXAlignment = Enum.TextXAlignment.Left, ClearTextOnFocus = false,
                    Position = UDim2.new(0, 6, 1, -34), Size = UDim2.new(1, -72, 0, 28),
                }, Card)
                New("UICorner", { CornerRadius = UDim.new(0, 4) }, Input)
                New("UIPadding", { PaddingLeft = UDim.new(0, 8), PaddingRight = UDim.new(0, 8) }, Input)

                local SendBtn = New("TextButton", {
                    Name = "Send", AutoButtonColor = false, Font = Enum.Font.GothamBold, Text = "Send", TextSize = 12,
                    TextColor3 = GetContrastColor(GuiConfig.Color), BackgroundColor3 = GuiConfig.Color,
                    AnchorPoint = Vector2.new(1, 0), Position = UDim2.new(1, -6, 1, -34), Size = UDim2.new(0, 58, 0, 28),
                }, Card)
                New("UICorner", { CornerRadius = UDim.new(0, 4) }, SendBtn)

                local function layout()
                    local bottom = state.reply and 60 or 42
                    List.Size = UDim2.new(1, -12, 1, -(20 + bottom))
                end
                layout()

                local function setStatus(t) if Status.Parent then Status.Text = tostring(t or "") end end

                local function setReply(msg)
                    state.reply = msg
                    ReplyBar.Visible = msg ~= nil
                    if msg then ReplyBar.Text = "Reply @" .. tostring(msg.username) .. ": " .. tostring(msg.message):sub(1, 40) .. "  (klik untuk batal)" end
                    layout()
                end
                ReplyBar.Activated:Connect(function() setReply(nil) end)

                local function render(scrollToEnd)
                    for _, c in ipairs(List:GetChildren()) do
                        if c:IsA("Frame") then c:Destroy() end
                    end
                    local first = math.max(1, #state.messages - maxMessages + 1)
                    for i = first, #state.messages do
                        local m = state.messages[i]
                        local tier = TIER_COLORS[m.tier] and m.tier or "Free"
                        local Row = New("Frame", { Name = "Msg", BackgroundTransparency = 1, LayoutOrder = i, Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y }, List)
                        New("UIListLayout", { Padding = UDim.new(0, 1), SortOrder = Enum.SortOrder.LayoutOrder }, Row)

                        New("TextLabel", {
                            LayoutOrder = 1, BackgroundTransparency = 1, Font = Enum.Font.GothamBold, TextSize = 11,
                            RichText = true, TextXAlignment = Enum.TextXAlignment.Left, TextWrapped = true,
                            Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, TextColor3 = Color3.fromRGB(255, 255, 255),
                            Text = string.format('<font color="%s">%s</font> <font color="%s">%s</font>',
                                hex(TIER_COLORS[tier]), esc(m.username or "?"), hex(TIER_COLORS[tier]:Lerp(Color3.new(0, 0, 0), 0.25)), TIER_LABELS[tier]),
                        }, Row)
                        if type(m.replyTo) == "table" then
                            New("TextLabel", {
                                LayoutOrder = 2, BackgroundTransparency = 1, Font = Enum.Font.Gotham, TextSize = 10,
                                TextColor3 = Color3.fromRGB(255, 255, 255), TextTransparency = 0.55, TextXAlignment = Enum.TextXAlignment.Left,
                                TextTruncate = Enum.TextTruncate.AtEnd, Size = UDim2.new(1, 0, 0, 12),
                                Text = "Reply @" .. tostring(m.replyTo.username) .. ": " .. tostring(m.replyTo.message),
                            }, Row)
                        end
                        New("TextLabel", {
                            LayoutOrder = 3, BackgroundTransparency = 1, Font = Enum.Font.Gotham, TextSize = 12,
                            RichText = false, TextWrapped = true, TextXAlignment = Enum.TextXAlignment.Left,
                            TextYAlignment = Enum.TextYAlignment.Top, TextColor3 = Color3.fromRGB(255, 255, 255), TextTransparency = 0.1,
                            Size = UDim2.new(1, 0, 0, 0), AutomaticSize = Enum.AutomaticSize.Y, Text = tostring(m.message or ""),
                        }, Row)

                        local Hit = New("TextButton", { Name = "Hit", BackgroundTransparency = 1, Text = "", Size = UDim2.new(1, 0, 1, 0), ZIndex = 3 }, Row)
                        Hit.Activated:Connect(function() setReply(m) end)
                    end
                    if scrollToEnd then
                        task.defer(function()
                            if List.Parent then List.CanvasPosition = Vector2.new(0, math.max(0, List.AbsoluteCanvasSize.Y - List.AbsoluteWindowSize.Y)) end
                        end)
                    end
                end

                local function keyOf(msgs)
                    local last = msgs[#msgs]
                    return tostring(#msgs) .. ":" .. tostring(last and last.id or "")
                end

                local function refresh()
                    if state.loading or state.busy then return end
                    state.loading = true
                    state.lastFetch = os.clock()
                    task.spawn(function()
                        local ok, err = ensureSession()
                        local data, e2, code
                        if ok then
                            data, e2, code = api("/api/chat/messages?room=" .. HttpService:UrlEncode(room), "GET", nil, state.token)
                            if code == 401 then
                                state.tokenExpiry = 0
                                if ensureSession() then data, e2 = api("/api/chat/messages?room=" .. HttpService:UrlEncode(room), "GET", nil, state.token) end
                            end
                        end
                        state.loading = false
                        if not Card.Parent then return end
                        if data and type(data.messages) == "table" then
                            local newKey = keyOf(data.messages)
                            state.messages = data.messages
                            if newKey ~= state.lastKey then
                                state.lastKey = newKey
                                render(true)
                            end
                            setStatus("Chat - " .. room)
                        else
                            setStatus(e2 or err or "Chat tidak tersedia")
                        end
                    end)
                end

                local function send()
                    local text = (Input.Text or ""):match("^%s*(.-)%s*$")
                    if text == "" or state.busy then return end
                    if #text > maxBytes then setStatus("Pesan terlalu panjang (maks " .. maxBytes .. " byte)") return end
                    state.busy = true
                    SendBtn.Text = "..."
                    task.spawn(function()
                        local function post()
                            return api("/api/chat/send", "POST", {
                                username = LocalPlayer.Name, displayName = LocalPlayer.DisplayName, userId = LocalPlayer.UserId,
                                message = text, room = room, replyToId = state.reply and state.reply.id or nil,
                                keyTier = tostring(keyTier or LRM_KeyTier or ""),
                            }, state.token)
                        end
                        local ok, err = ensureSession()
                        local data, e2, code
                        if ok then
                            data, e2, code = post()
                            if code == 401 then
                                state.tokenExpiry = 0
                                if ensureSession() then data, e2 = post() end
                            end
                        end
                        state.busy = false
                        if not Card.Parent then return end
                        SendBtn.Text = "Send"
                        if data and type(data.message) == "table" then
                            Input.Text = ""
                            setReply(nil)
                            table.insert(state.messages, data.message)
                            state.lastKey = keyOf(state.messages)
                            render(true)
                            setStatus("Chat - " .. room)
                            state.lastFetch = os.clock()
                        else
                            setStatus(e2 or err or "Gagal kirim")
                        end
                    end)
                end

                SendBtn.Activated:Connect(function()
                    CircleClick(SendBtn, Mouse.X, Mouse.Y)
                    send()
                end)
                Input.FocusLost:Connect(function(enter) if enter then send() end end)

                -- refresh otomatis: hanya kalau window terbuka & chat kelihatan
                local function isVisible()
                    if not DropShadowHolder.Visible then return false end
                    local cp, cs = Card.AbsolutePosition, Card.AbsoluteSize
                    local mp, ms = Main.AbsolutePosition, Main.AbsoluteSize
                    return cp.X < mp.X + ms.X and cp.X + cs.X > mp.X and cp.Y < mp.Y + ms.Y and cp.Y + cs.Y > mp.Y
                end
                task.spawn(function()
                    task.wait(0.5)
                    if baseUrl == "" then setStatus("Set ApiBaseUrl") else refresh() end
                    while state.alive do
                        task.wait(1)
                        if not Card:IsDescendantOf(game) then state.alive = false break end
                        if baseUrl ~= "" and os.clock() - state.lastFetch >= refreshSec and isVisible() then refresh() end
                    end
                end)

                local ChatFunc = { Frame = Card }
                function ChatFunc:Refresh() refresh() end
                function ChatFunc:GetMessages() return state.messages end
                function ChatFunc:SetApiBaseUrl(u) baseUrl = (tostring(u or "")):gsub("/+$", ""); state.token = nil; state.lastKey = ""; refresh() end
                function ChatFunc:Destroy() state.alive = false; pcall(function() Card:Destroy() end) end

                CountItem = CountItem + 1
                return ChatFunc
            end

            -- ===== AddButtonGroup: beberapa tombol sejajar dalam satu baris =====
            -- Items:AddButtonGroup({ Buttons = { {Title="A", Icon="eyes", Callback=function() end}, {Title="B"} } })
            function Items:AddButtonGroup(GroupConfig)
                GroupConfig = GroupConfig or {}
                local list = GroupConfig.Buttons or GroupConfig
                if type(list) ~= "table" or #list == 0 then return nil end
                local n, gap = #list, 4

                local Group = Instance.new("Frame")
                Group.Name = "ButtonGroup"
                Group.BackgroundTransparency = 1
                Group.Size = UDim2.new(1, 0, 0, 40)
                Group.LayoutOrder = CountItem
                Group.Parent = ItemParent()

                local GroupList = Instance.new("UIListLayout")
                GroupList.FillDirection = Enum.FillDirection.Horizontal
                GroupList.SortOrder = Enum.SortOrder.LayoutOrder
                GroupList.Padding = UDim.new(0, gap)
                GroupList.Parent = Group

                local result = { Frame = Group, Buttons = {} }
                for i, b in ipairs(list) do
                    b = type(b) == "table" and b or {}
                    local btn = Instance.new("TextButton")
                    btn.Name = "GroupButton" .. i
                    btn.LayoutOrder = i
                    btn.Font = Enum.Font.GothamBold
                    btn.Text = tostring(b.Title or "Confirm")
                    btn.TextSize = 12
                    btn.TextColor3 = Color3.fromRGB(255, 255, 255)
                    btn.TextTransparency = 0.3
                    btn.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                    btn.BackgroundTransparency = 0.935
                    btn.AutoButtonColor = false
                    btn.Size = UDim2.new(1 / n, -(gap * (n - 1)) / n, 1, 0)
                    btn.Parent = Group
                    New("UICorner", { CornerRadius = UDim.new(0, 4) }, btn)

                    if b.Icon then
                        local id = tostring(b.Icon)
                        id = Icons[id] or (id:match("^%d+$") and ("rbxassetid://" .. id)) or id
                        btn.Text = "      " .. btn.Text
                        New("ImageLabel", {
                            BackgroundTransparency = 1, Image = id, ImageTransparency = 0.3,
                            AnchorPoint = Vector2.new(0, 0.5), Position = UDim2.new(0, 10, 0.5, 0),
                            Size = UDim2.new(0, 14, 0, 14), ScaleType = Enum.ScaleType.Fit,
                        }, btn)
                    end

                    btn.MouseButton1Down:Connect(function()
                        TweenService:Create(btn, TweenInfo.new(0.12), { BackgroundTransparency = 0.85 }):Play()
                    end)
                    btn.MouseButton1Up:Connect(function()
                        TweenService:Create(btn, TweenInfo.new(0.12), { BackgroundTransparency = 0.935 }):Play()
                    end)
                    btn.MouseLeave:Connect(function()
                        TweenService:Create(btn, TweenInfo.new(0.12), { BackgroundTransparency = 0.935 }):Play()
                    end)
                    btn.Activated:Connect(function()
                        CircleClick(btn, Mouse.X, Mouse.Y)
                        if typeof(b.Callback) == "function" then pcall(b.Callback) end
                    end)
                    result.Buttons[i] = btn
                end

                CountItem = CountItem + 1
                return result
            end

            -- ===== AddKeybind: elemen pilih tombol (simpan ke config) =====
            -- Items:AddKeybind({ Title="Fly", Content="", Default=Enum.KeyCode.F, Mode="Toggle"|"Hold"|"Press",
            --                    Callback=function(state) end, ChangedCallback=function(keyName) end, Flag="fly_key" })
            -- Klik kotak tombol -> tekan tombol baru. Esc / Backspace = hapus. Mouse: MB2 / MB3 juga bisa.
            function Items:AddKeybind(KeybindConfig)
                KeybindConfig = KeybindConfig or {}
                KeybindConfig.Title = KeybindConfig.Title or "Keybind"
                KeybindConfig.Content = KeybindConfig.Content or ""
                local mode = tostring(KeybindConfig.Mode or "Toggle"):lower()
                local callback = typeof(KeybindConfig.Callback) == "function" and KeybindConfig.Callback or function() end
                local changed = typeof(KeybindConfig.ChangedCallback) == "function" and KeybindConfig.ChangedCallback or function() end

                local configKey = MakeKey("Keybind", KeybindConfig)

                local function nameOf(v)
                    if v == nil then return nil end
                    if typeof(v) == "EnumItem" then
                        if v == Enum.UserInputType.MouseButton1 then return "MB1" end
                        if v == Enum.UserInputType.MouseButton2 then return "MB2" end
                        if v == Enum.UserInputType.MouseButton3 then return "MB3" end
                        return v.Name
                    end
                    v = tostring(v)
                    if v == "" or v == "None" then return nil end
                    return v
                end
                local function inputName(input)
                    local t = input.UserInputType
                    if t == Enum.UserInputType.Keyboard then
                        if input.KeyCode == Enum.KeyCode.Unknown then return nil end
                        return input.KeyCode.Name
                    elseif t == Enum.UserInputType.MouseButton2 then return "MB2"
                    elseif t == Enum.UserInputType.MouseButton3 then return "MB3" end
                    return nil
                end

                local KeybindFunc = { Value = nil, State = false }
                local stored = ConfigData[configKey]
                if stored ~= nil then
                    KeybindFunc.Value = nameOf(stored)
                else
                    KeybindFunc.Value = nameOf(KeybindConfig.Default)
                end

                local hasContent = KeybindConfig.Content ~= ""
                local Row = Instance.new("Frame")
                Row.Name = "Keybind"
                Row.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                Row.BackgroundTransparency = ProjectSenzu.ItemBgTransparency
                Row.Size = UDim2.new(1, 0, 0, hasContent and 50 or 40)
                Row.LayoutOrder = CountItem
                Row.Parent = ItemParent()
                New("UICorner", { CornerRadius = UDim.new(0, 4) }, Row)

                New("TextLabel", {
                    Name = "Title", BackgroundTransparency = 1, Font = Enum.Font.GothamBold,
                    Text = KeybindConfig.Title, TextSize = 13, TextColor3 = Color3.fromRGB(255, 255, 255),
                    TextXAlignment = Enum.TextXAlignment.Left, TextTruncate = Enum.TextTruncate.AtEnd,
                    Position = UDim2.new(0, 10, 0, hasContent and 8 or 0),
                    Size = UDim2.new(1, -110, hasContent and 0 or 1, hasContent and 18 or 0),
                }, Row)
                if hasContent then
                    New("TextLabel", {
                        Name = "Content", BackgroundTransparency = 1, Font = Enum.Font.Gotham,
                        Text = KeybindConfig.Content, TextSize = 11, TextColor3 = Color3.fromRGB(255, 255, 255),
                        TextTransparency = 0.5, TextXAlignment = Enum.TextXAlignment.Left,
                        TextTruncate = Enum.TextTruncate.AtEnd,
                        Position = UDim2.new(0, 10, 0, 27), Size = UDim2.new(1, -110, 0, 16),
                    }, Row)
                end

                local BindButton = New("TextButton", {
                    Name = "BindButton", AutoButtonColor = false, Font = Enum.Font.GothamBold,
                    TextSize = 11, TextColor3 = Color3.fromRGB(255, 255, 255),
                    BackgroundColor3 = Color3.fromRGB(255, 255, 255), BackgroundTransparency = 0.92,
                    AnchorPoint = Vector2.new(1, 0.5), Position = UDim2.new(1, -10, 0.5, 0),
                    Size = UDim2.new(0, 70, 0, 24), Text = "None",
                }, Row)
                New("UICorner", { CornerRadius = UDim.new(0, 4) }, BindButton)
                local BindStroke = New("UIStroke", { Color = GuiConfig.Color, Transparency = 0.6, Thickness = 1, ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, BindButton)

                local recording = false
                local function render()
                    BindButton.Text = recording and "[ ... ]" or (KeybindFunc.Value and ("[ " .. KeybindFunc.Value .. " ]") or "None")
                    BindButton.Size = UDim2.new(0, math.max(60, BindButton.TextBounds.X + 20), 0, 24)
                    BindStroke.Transparency = recording and 0 or 0.6
                end
                render()

                function KeybindFunc:Set(key, silent)
                    self.Value = nameOf(key)
                    self.State = false
                    ConfigData[configKey] = self.Value or "None"
                    SaveConfig()
                    render()
                    if not silent then pcall(changed, self.Value) end
                end
                function KeybindFunc:GetValue() return self.Value end
                function KeybindFunc:GetState() return self.State end

                BindButton.Activated:Connect(function()
                    CircleClick(BindButton, Mouse.X, Mouse.Y)
                    recording = not recording
                    render()
                end)

                table.insert(GuiFunc._conns, UserInputService.InputBegan:Connect(function(input, gpe)
                    if recording then
                        if input.UserInputType == Enum.UserInputType.MouseButton1 then
                            recording = false; render(); return
                        end
                        local nm = inputName(input)
                        if not nm then return end
                        recording = false
                        if nm == "Escape" or nm == "Backspace" then
                            KeybindFunc:Set(nil)
                        else
                            KeybindFunc:Set(nm)
                        end
                        return
                    end
                    if gpe or not KeybindFunc.Value then return end
                    if inputName(input) ~= KeybindFunc.Value then return end
                    if mode == "hold" then
                        KeybindFunc.State = true
                        pcall(callback, true)
                    elseif mode == "press" then
                        pcall(callback, true)
                    else
                        KeybindFunc.State = not KeybindFunc.State
                        pcall(callback, KeybindFunc.State)
                    end
                end))
                table.insert(GuiFunc._conns, UserInputService.InputEnded:Connect(function(input)
                    if mode == "hold" and KeybindFunc.State and inputName(input) == KeybindFunc.Value then
                        KeybindFunc.State = false
                        pcall(callback, false)
                    end
                end))

                CountItem = CountItem + 1
                return KeybindFunc
            end

            -- ===== AddPricing: kartu paket harga =====
            -- Items:AddPricing({ Plans = { {Name="Weekly", Price="$1.99", LocalPrice="IDR 15.000", Description="...",
            --                               CheckoutUrl="https://...", Badge="BEST VALUE", Highlight=true} },
            --                    ButtonText="Get Plan", Callback=function(plan) end })
            -- Klik tombol: link checkout disalin ke clipboard (kalau executor support) + Callback(plan).
            function Items:AddPricing(PricingConfig)
                PricingConfig = PricingConfig or {}
                local plans = PricingConfig.Plans
                if type(plans) ~= "table" or #plans == 0 then return nil end
                local cols = math.min(#plans, 3)

                local Cards = Instance.new("Frame")
                Cards.Name = "PricingCards"
                Cards.BackgroundTransparency = 1
                Cards.Size = UDim2.new(1, 0, 0, 0)
                Cards.AutomaticSize = Enum.AutomaticSize.Y
                Cards.LayoutOrder = CountItem
                Cards.Parent = ItemParent()

                New("UIGridLayout", {
                    CellSize = UDim2.new(1 / cols, -(6 * (cols - 1)) / cols, 0, 184),
                    CellPadding = UDim2.new(0, 6, 0, 6),
                    SortOrder = Enum.SortOrder.LayoutOrder,
                }, Cards)

                local result = { Frame = Cards, Cards = {} }
                for i, plan in ipairs(plans) do
                    plan = type(plan) == "table" and plan or {}
                    local hi = plan.Highlight and true or false

                    local Card = Instance.new("Frame")
                    Card.Name = "Plan_" .. tostring(plan.Name or i)
                    Card.LayoutOrder = i
                    Card.BackgroundColor3 = hi and GuiConfig.Color:Lerp(Color3.new(0, 0, 0), 0.7) or Color3.fromRGB(255, 255, 255)
                    Card.BackgroundTransparency = hi and 0.35 or 0.92
                    Card.Parent = Cards
                    New("UICorner", { CornerRadius = UDim.new(0, 6) }, Card)
                    New("UIStroke", { Color = hi and GuiConfig.Color or Color3.fromRGB(255, 255, 255), Transparency = hi and 0.1 or 0.85, Thickness = 1, ApplyStrokeMode = Enum.ApplyStrokeMode.Border }, Card)
                    New("UIPadding", { PaddingTop = UDim.new(0, 8), PaddingBottom = UDim.new(0, 8), PaddingLeft = UDim.new(0, 8), PaddingRight = UDim.new(0, 8) }, Card)
                    New("UIListLayout", { Padding = UDim.new(0, 3), SortOrder = Enum.SortOrder.LayoutOrder }, Card)

                    local BadgeRow = New("Frame", { Name = "BadgeRow", LayoutOrder = 1, BackgroundTransparency = 1, Size = UDim2.new(1, 0, 0, 14) }, Card)
                    if plan.Badge then
                        local pill = New("TextLabel", {
                            BackgroundColor3 = GuiConfig.Color, BorderSizePixel = 0, Font = Enum.Font.GothamBold,
                            Text = tostring(plan.Badge), TextSize = 9, TextColor3 = GetContrastColor(GuiConfig.Color),
                            AutomaticSize = Enum.AutomaticSize.X, Size = UDim2.new(0, 0, 1, 0),
                        }, BadgeRow)
                        New("UIPadding", { PaddingLeft = UDim.new(0, 6), PaddingRight = UDim.new(0, 6) }, pill)
                        New("UICorner", { CornerRadius = UDim.new(0, 4) }, pill)
                    end

                    local function text(order, str, size, font, color, h, transp)
                        return New("TextLabel", {
                            LayoutOrder = order, BackgroundTransparency = 1, Font = font, Text = tostring(str or ""),
                            TextSize = size, TextColor3 = color, TextTransparency = transp or 0,
                            TextXAlignment = Enum.TextXAlignment.Left, TextYAlignment = Enum.TextYAlignment.Top,
                            TextWrapped = true, TextTruncate = Enum.TextTruncate.AtEnd, Size = UDim2.new(1, 0, 0, h),
                        }, Card)
                    end
                    text(2, plan.Name or "Plan", 13, Enum.Font.GothamBold, Color3.fromRGB(255, 255, 255), 17)
                    text(3, plan.Price or "", 19, Enum.Font.GothamBold, hi and GuiConfig.Color or Color3.fromRGB(255, 255, 255), 24)
                    text(4, plan.LocalPrice or "", 10, Enum.Font.GothamMedium, Color3.fromRGB(255, 255, 255), 13, 0.45)
                    text(5, plan.Description or "", 10, Enum.Font.Gotham, Color3.fromRGB(255, 255, 255), 38, 0.35)

                    local Buy = New("TextButton", {
                        Name = "Buy", LayoutOrder = 6, AutoButtonColor = false, Font = Enum.Font.GothamBold,
                        Text = tostring(plan.ButtonText or PricingConfig.ButtonText or "Get Plan"), TextSize = 11,
                        TextColor3 = hi and GetContrastColor(GuiConfig.Color) or Color3.fromRGB(255, 255, 255),
                        BackgroundColor3 = hi and GuiConfig.Color or Color3.fromRGB(255, 255, 255),
                        BackgroundTransparency = hi and 0 or 0.88, Size = UDim2.new(1, 0, 0, 26),
                    }, Card)
                    New("UICorner", { CornerRadius = UDim.new(0, 4) }, Buy)
                    Buy.Activated:Connect(function()
                        CircleClick(Buy, Mouse.X, Mouse.Y)
                        if plan.CheckoutUrl and setclipboard then
                            pcall(setclipboard, tostring(plan.CheckoutUrl))
                            pcall(function()
                                ProjectSenzu:MakeNotify({
                                    Title = tostring(plan.Name or "Plan"),
                                    Description = "Checkout",
                                    Content = "Link checkout disalin ke clipboard.",
                                })
                            end)
                        end
                        if typeof(PricingConfig.Callback) == "function" then pcall(PricingConfig.Callback, plan) end
                        if typeof(plan.Callback) == "function" then pcall(plan.Callback, plan) end
                    end)
                    result.Cards[i] = Card
                end

                CountItem = CountItem + 1
                return result
            end

            function Items:AddToggle(ToggleConfig)
                local ToggleConfig = ToggleConfig or {}
                ToggleConfig.Title = ToggleConfig.Title or "Title"
                ToggleConfig.Title2 = ToggleConfig.Title2 or ""
                ToggleConfig.Content = ToggleConfig.Content or ""
                ToggleConfig.Default = ToggleConfig.Default or false
                ToggleConfig.Callback = ToggleConfig.Callback or function() end
                ToggleConfig.Keybind = ToggleConfig.Keybind or false

                local configKey = MakeKey("Toggle", ToggleConfig)
                local keybindConfigKey = configKey .. "_Keybind"

                if ConfigData[configKey] ~= nil then
                    ToggleConfig.Default = ConfigData[configKey]
                end

                local currentKeybind = nil
                if ConfigData[keybindConfigKey] ~= nil then
                    currentKeybind = ConfigData[keybindConfigKey]
                end

                local ToggleFunc = { Value = ToggleConfig.Default }

                local Toggle = Instance.new("Frame")
                local ToggleTitle = Instance.new("TextLabel")
                local ToggleContent = Instance.new("TextLabel")
                local ToggleButton = Instance.new("TextButton")
                local FeatureFrame2 = Instance.new("Frame")
                local KeybindFrame = Instance.new("Frame")
                local UIStroke8 = Instance.new("UIStroke")
                local ToggleCircle = Instance.new("Frame")
                local KeybindButton = Instance.new("TextButton")

                Toggle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                Toggle.BackgroundTransparency = ProjectSenzu.ItemBgTransparency
                Toggle.BorderSizePixel = 0
                Toggle.LayoutOrder = CountItem
                Toggle.Name = "Toggle"
                Toggle.Parent = ItemParent()

                New("UICorner", { CornerRadius = UDim.new(0, 4) }, Toggle)

                ToggleTitle.Font = Enum.Font.GothamBold
                ToggleTitle.Text = ToggleConfig.Title
                ToggleTitle.TextSize = 13
                ToggleTitle.TextColor3 = Color3.fromRGB(231, 231, 231)
                ToggleTitle.TextXAlignment = Enum.TextXAlignment.Left
                ToggleTitle.TextTruncate = Enum.TextTruncate.AtEnd
                ToggleTitle.TextYAlignment = Enum.TextYAlignment.Top
                ToggleTitle.BackgroundTransparency = 1
                ToggleTitle.Position = UDim2.new(0, 10, 0, 10)
                ToggleTitle.Size = UDim2.new(1, -100, 0, 13)
                ToggleTitle.Name = "ToggleTitle"
                ToggleTitle.Parent = Toggle

                local ToggleTitle2 = Instance.new("TextLabel")
                ToggleTitle2.Font = Enum.Font.GothamBold
                ToggleTitle2.Text = ToggleConfig.Title2
                ToggleTitle2.TextSize = 12
                ToggleTitle2.TextColor3 = Color3.fromRGB(231, 231, 231)
                ToggleTitle2.TextXAlignment = Enum.TextXAlignment.Left
                ToggleTitle2.TextYAlignment = Enum.TextYAlignment.Top
                ToggleTitle2.BackgroundTransparency = 1
                ToggleTitle2.Position = UDim2.new(0, 10, 0, 23)
                ToggleTitle2.Size = UDim2.new(1, -100, 0, 12)
                ToggleTitle2.Name = "ToggleTitle2"
                ToggleTitle2.Parent = Toggle

                ToggleContent.Font = Enum.Font.GothamBold
                ToggleContent.Text = ToggleConfig.Content
                ToggleContent.TextColor3 = Color3.fromRGB(255, 255, 255)
                ToggleContent.TextSize = 12
                ToggleContent.TextTransparency = 0.6
                ToggleContent.TextXAlignment = Enum.TextXAlignment.Left
                ToggleContent.TextYAlignment = Enum.TextYAlignment.Bottom
                ToggleContent.BackgroundTransparency = 1
                ToggleContent.Size = UDim2.new(1, -100, 0, 12)
                ToggleContent.Name = "ToggleContent"
                ToggleContent.Parent = Toggle

                if ToggleConfig.Title2 ~= "" then
                    Toggle.Size = UDim2.new(1, 0, 0, 57)
                    ToggleContent.Position = UDim2.new(0, 10, 0, 36)
                    ToggleTitle2.Visible = true
                else
                    Toggle.Size = UDim2.new(1, 0, 0, 46)
                    ToggleContent.Position = UDim2.new(0, 10, 0, 23)
                    ToggleTitle2.Visible = false
                end

                FitWrapLabel(ToggleContent, 12, function(h)
                    Toggle.Size = UDim2.new(1, 0, 0, h + (ToggleConfig.Title2 ~= "" and 47 or 33))
                end, UpdateSizeSection)

                ToggleButton.Font = Enum.Font.SourceSans
                ToggleButton.Text = ""
                ToggleButton.BackgroundTransparency = 1
                ToggleButton.Size = UDim2.new(1, 0, 1, 0)
                ToggleButton.Name = "ToggleButton"
                ToggleButton.Parent = Toggle

                FeatureFrame2.AnchorPoint = Vector2.new(1, 0.5)
                FeatureFrame2.BackgroundTransparency = 0.92
                FeatureFrame2.BorderSizePixel = 0
                FeatureFrame2.Position = UDim2.new(1, -15, 0.5, 0)
                FeatureFrame2.Size = UDim2.new(0, 30, 0, 15)
                FeatureFrame2.Name = "FeatureFrame"
                FeatureFrame2.Parent = Toggle

                New("UICorner", {  }, FeatureFrame2)

                UIStroke8.Color = Color3.fromRGB(255, 255, 255)
                UIStroke8.Thickness = 2
                UIStroke8.Transparency = 0.9
                UIStroke8.Parent = FeatureFrame2

                ToggleCircle.BackgroundColor3 = Color3.fromRGB(230, 230, 230)
                ToggleCircle.BorderSizePixel = 0
                ToggleCircle.Size = UDim2.new(0, 14, 0, 14)
                ToggleCircle.Name = "ToggleCircle"
                ToggleCircle.Parent = FeatureFrame2

                New("UICorner", { CornerRadius = UDim.new(0, 15) }, ToggleCircle)

                KeybindFrame.AnchorPoint = Vector2.new(1, 0.5)
                KeybindFrame.BackgroundTransparency = 0.92
                KeybindFrame.BorderSizePixel = 0
                KeybindFrame.Position = UDim2.new(0.9, -20, 0.5, 0)
                KeybindFrame.Size = UDim2.new(0, 70, 0, 20)
                KeybindFrame.Name = "KeybindFrame"
                KeybindFrame.Parent = Toggle

                New("UICorner", {  }, KeybindFrame)

                KeybindButton.Font = Enum.Font.GothamBold
                KeybindButton.Text = "Keybind"
                KeybindButton.BackgroundTransparency = 1
                KeybindButton.Size = UDim2.new(1, 0, 1, 0)
                KeybindButton.Position = UDim2.new(0, 0, 0, 0)
                KeybindButton.TextXAlignment = Enum.TextXAlignment.Center
                KeybindButton.TextYAlignment = Enum.TextYAlignment.Center
                KeybindButton.Name = "KeybindButton"
                KeybindButton.TextColor3 = Color3.fromRGB(225, 225, 225)
                KeybindButton.TextSize = 12
                KeybindButton.Parent = KeybindFrame

                local isRecording = false
                local lastRecordCancel = 0

                local ignoredKeys = {
                    [Enum.KeyCode.LeftControl] = true, [Enum.KeyCode.RightControl] = true,
                    [Enum.KeyCode.LeftAlt] = true, [Enum.KeyCode.RightAlt] = true,
                    [Enum.KeyCode.LeftShift] = true, [Enum.KeyCode.RightShift] = true,
                    [Enum.KeyCode.Unknown] = true
                }

                local function GetModifiersString(excludeKey)
                    local keys = ""
                    if UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) and excludeKey ~= Enum.KeyCode.LeftControl then keys = keys .. "LeftControl + " end
                    if UserInputService:IsKeyDown(Enum.KeyCode.RightControl) and excludeKey ~= Enum.KeyCode.RightControl then keys = keys .. "RightControl + " end
                    if UserInputService:IsKeyDown(Enum.KeyCode.LeftAlt) and excludeKey ~= Enum.KeyCode.LeftAlt then keys = keys .. "LeftAlt + " end
                    if UserInputService:IsKeyDown(Enum.KeyCode.RightAlt) and excludeKey ~= Enum.KeyCode.RightAlt then keys = keys .. "RightAlt + " end
                    if UserInputService:IsKeyDown(Enum.KeyCode.LeftShift) and excludeKey ~= Enum.KeyCode.LeftShift then keys = keys .. "LeftShift + " end
                    if UserInputService:IsKeyDown(Enum.KeyCode.RightShift) and excludeKey ~= Enum.KeyCode.RightShift then keys = keys .. "RightShift + " end
                    return keys
                end

                KeybindButton:GetPropertyChangedSignal("TextBounds"):Connect(function()
                    KeybindFrame.Size = UDim2.new(0, KeybindButton.TextBounds.X + 20, 0, 20)
                end)

                if not ToggleConfig.Keybind or isMobile then
                    KeybindFrame.Visible = false
                else
                    if currentKeybind then
                        KeybindButton.Text = "[ " .. currentKeybind .. " ]"
                    else
                        KeybindButton.Text = "Keybind"
                    end

                    KeybindButton.MouseButton1Click:Connect(function()
                        if not isRecording and tick() - lastRecordCancel > 0.1 then
                            isRecording = true
                            KeybindButton.Text = "[ ... ]"
                        end
                    end)

                    table.insert(GuiFunc._conns, UserInputService.InputBegan:Connect(function(input, gameProcessed)
                        if isRecording then
                            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                                isRecording = false
                                lastRecordCancel = tick()
                                currentKeybind = nil
                                KeybindButton.Text = "Keybind"
                                ConfigData[keybindConfigKey] = nil
                                SaveConfig()
                            elseif input.UserInputType == Enum.UserInputType.Keyboard then
                                local key = input.KeyCode
                                if key == Enum.KeyCode.Escape or key == Enum.KeyCode.Backspace then
                                    isRecording = false
                                    currentKeybind = nil
                                    KeybindButton.Text = "Keybind"
                                    ConfigData[keybindConfigKey] = nil
                                    SaveConfig()
                                elseif ignoredKeys[key] then
                                    if key ~= Enum.KeyCode.Unknown then
                                        KeybindButton.Text = "[ " .. GetModifiersString() .. "... ]"
                                    end
                                else
                                    local finalBind = GetModifiersString() .. key.Name
                                    currentKeybind = finalBind
                                    KeybindButton.Text = "[ " .. finalBind .. " ]"
                                    isRecording = false
                                    ConfigData[keybindConfigKey] = currentKeybind
                                    SaveConfig()
                                end
                            end
                        else
                            if not gameProcessed and input.UserInputType == Enum.UserInputType.Keyboard then
                                if currentKeybind then
                                    local checkStr = GetModifiersString(input.KeyCode) .. input.KeyCode.Name
                                    if checkStr == currentKeybind then
                                        ToggleFunc.Value = not ToggleFunc.Value
                                        ToggleFunc:Set(ToggleFunc.Value)
                                    end
                                end
                            end
                        end
                    end))

                    table.insert(GuiFunc._conns, UserInputService.InputEnded:Connect(function(input, gameProcessed)
                        if isRecording and ignoredKeys[input.KeyCode] and input.KeyCode ~= Enum.KeyCode.Unknown then
                            currentKeybind = input.KeyCode.Name
                            KeybindButton.Text = "[ " .. currentKeybind .. " ]"
                            isRecording = false
                            ConfigData[keybindConfigKey] = currentKeybind
                            SaveConfig()
                        end
                    end))
                end

                ToggleButton.Activated:Connect(function()
                    ToggleFunc.Value = not ToggleFunc.Value
                    ToggleFunc:Set(ToggleFunc.Value)
                end)

                function ToggleFunc:Set(Value)
                    PlaySound("Element")
                    if typeof(ToggleConfig.Callback) == "function" then
                        local ok, err = pcall(function()
                            ToggleConfig.Callback(Value)
                        end)
                        -- error callback diabaikan (tidak dicetak ke F9)
                    end
                    ConfigData[configKey] = Value
                    SaveConfig()
                    if Value then
                        TweenService:Create(ToggleTitle, TweenInfo.new(0.2), { TextColor3 = Color3.fromRGB(255, 255, 255) }):Play()
                        TweenService:Create(ToggleCircle, TweenInfo.new(0.2), { Position = UDim2.new(0, 15, 0, 0), BackgroundColor3 = GetContrastColor(GuiConfig.Color) })
                            :Play()
                        TweenService:Create(UIStroke8, TweenInfo.new(0.2), { Color = GuiConfig.Color, Transparency = 0 })
                            :Play()
                        TweenService:Create(FeatureFrame2, TweenInfo.new(0.2),
                            { BackgroundColor3 = GuiConfig.Color, BackgroundTransparency = 0 }):Play()
                    else
                        TweenService:Create(ToggleTitle, TweenInfo.new(0.2),
                            { TextColor3 = Color3.fromRGB(230, 230, 230) }):Play()
                        TweenService:Create(ToggleCircle, TweenInfo.new(0.2), { Position = UDim2.new(0, 0, 0, 0), BackgroundColor3 = Color3.fromRGB(230, 230, 230) }):Play()
                        TweenService:Create(UIStroke8, TweenInfo.new(0.2),
                            { Color = Color3.fromRGB(255, 255, 255), Transparency = 0.9 }):Play()
                        TweenService:Create(FeatureFrame2, TweenInfo.new(0.2),
                            { BackgroundColor3 = Color3.fromRGB(255, 255, 255), BackgroundTransparency = 0.92 }):Play()
                    end
                end

                ToggleFunc:Set(ToggleFunc.Value)
                CountItem = CountItem + 1
                Elements[configKey] = ToggleFunc
                return ToggleFunc
            end

            function Items:AddSlider(SliderConfig)
                local SliderConfig = SliderConfig or {}
                SliderConfig.Title = SliderConfig.Title or "Slider"
                SliderConfig.Content = SliderConfig.Content or ""
                SliderConfig.Increment = SliderConfig.Increment or 1
                SliderConfig.Min = SliderConfig.Min or 0
                SliderConfig.Max = SliderConfig.Max or 100
                SliderConfig.Default = SliderConfig.Default or 50
                SliderConfig.Callback = SliderConfig.Callback or function() end

                local configKey = MakeKey("Slider", SliderConfig)
                if ConfigData[configKey] ~= nil then
                    SliderConfig.Default = ConfigData[configKey]
                end

                local SliderFunc = { Value = SliderConfig.Default }

                local Slider = Instance.new("Frame");
                local SliderTitle = Instance.new("TextLabel");
                local SliderContent = Instance.new("TextLabel");
                local SliderInput = Instance.new("Frame");
                local TextBox = Instance.new("TextBox");
                local SliderFrame = Instance.new("Frame");
                local SliderDraggable = Instance.new("Frame");
                local UIStroke5 = Instance.new("UIStroke");
                local SliderCircle = Instance.new("Frame");
                local UIStroke7 = Instance.new("UIStroke");

                Slider.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                Slider.BackgroundTransparency = ProjectSenzu.ItemBgTransparency
                Slider.BorderColor3 = Color3.fromRGB(0, 0, 0)
                Slider.BorderSizePixel = 0
                Slider.LayoutOrder = CountItem
                Slider.Size = UDim2.new(1, 0, 0, 46)
                Slider.Name = "Slider"
                Slider.Parent = ItemParent()

                New("UICorner", { CornerRadius = UDim.new(0, 4) }, Slider)

                SliderTitle.Font = Enum.Font.GothamBold
                SliderTitle.Text = SliderConfig.Title
                SliderTitle.TextColor3 = Color3.fromRGB(230.77499270439148, 230.77499270439148, 230.77499270439148)
                SliderTitle.TextSize = 13
                SliderTitle.TextXAlignment = Enum.TextXAlignment.Left
                SliderTitle.TextTruncate = Enum.TextTruncate.AtEnd
                SliderTitle.TextYAlignment = Enum.TextYAlignment.Top
                SliderTitle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                SliderTitle.BackgroundTransparency = 0.9990000128746033
                SliderTitle.BorderColor3 = Color3.fromRGB(0, 0, 0)
                SliderTitle.BorderSizePixel = 0
                SliderTitle.Position = UDim2.new(0, 10, 0, 10)
                SliderTitle.Size = UDim2.new(1, -180, 0, 13)
                SliderTitle.Name = "SliderTitle"
                SliderTitle.Parent = Slider

                SliderContent.Font = Enum.Font.GothamBold
                SliderContent.Text = SliderConfig.Content
                SliderContent.TextColor3 = Color3.fromRGB(255, 255, 255)
                SliderContent.TextSize = 12
                SliderContent.TextTransparency = 0.6000000238418579
                SliderContent.TextXAlignment = Enum.TextXAlignment.Left
                SliderContent.TextYAlignment = Enum.TextYAlignment.Bottom
                SliderContent.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                SliderContent.BackgroundTransparency = 0.9990000128746033
                SliderContent.BorderColor3 = Color3.fromRGB(0, 0, 0)
                SliderContent.BorderSizePixel = 0
                SliderContent.Position = UDim2.new(0, 10, 0, 25)
                SliderContent.Size = UDim2.new(1, -180, 0, 12)
                SliderContent.Name = "SliderContent"
                SliderContent.Parent = Slider

                FitWrapLabel(SliderContent, 12, function(h)
                    Slider.Size = UDim2.new(1, 0, 0, math.max(46, h + 33))
                end, UpdateSizeSection)

                SliderInput.AnchorPoint = Vector2.new(0, 0.5)
                SliderInput.BackgroundColor3 = GuiConfig.Color
                SliderInput.BorderColor3 = Color3.fromRGB(0, 0, 0)
                SliderInput.BackgroundTransparency = 1
                SliderInput.BorderSizePixel = 0
                SliderInput.Position = UDim2.new(1, -155, 0.5, 0)
                SliderInput.Size = UDim2.new(0, 28, 0, 20)
                SliderInput.Name = "SliderInput"
                SliderInput.Parent = Slider

                New("UICorner", { CornerRadius = UDim.new(0, 2) }, SliderInput)

                TextBox.Font = Enum.Font.GothamBold
                TextBox.Text = "90"
                TextBox.TextColor3 = Color3.fromRGB(255, 255, 255)
                TextBox.TextSize = 13
                TextBox.TextWrapped = true
                TextBox.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
                TextBox.BackgroundTransparency = 0.9990000128746033
                TextBox.BorderColor3 = Color3.fromRGB(0, 0, 0)
                TextBox.BorderSizePixel = 0
                TextBox.Position = UDim2.new(0, -1, 0, 0)
                TextBox.Size = UDim2.new(1, 0, 1, 0)
                TextBox.Parent = SliderInput

                SliderFrame.AnchorPoint = Vector2.new(1, 0.5)
                SliderFrame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                SliderFrame.BackgroundTransparency = 0.800000011920929
                SliderFrame.BorderColor3 = Color3.fromRGB(0, 0, 0)
                SliderFrame.BorderSizePixel = 0
                SliderFrame.Position = UDim2.new(1, -20, 0.5, 0)
                SliderFrame.Size = UDim2.new(0, 100, 0, 3)
                SliderFrame.Name = "SliderFrame"
                SliderFrame.Parent = Slider

                New("UICorner", {  }, SliderFrame)

                SliderDraggable.AnchorPoint = Vector2.new(0, 0.5)
                SliderDraggable.BackgroundColor3 = GuiConfig.Color
                SliderDraggable.BorderColor3 = Color3.fromRGB(0, 0, 0)
                SliderDraggable.BorderSizePixel = 0
                SliderDraggable.Position = UDim2.new(0, 0, 0.5, 0)
                SliderDraggable.Size = UDim2.new(0.899999976, 0, 0, 1)
                SliderDraggable.Name = "SliderDraggable"
                SliderDraggable.Parent = SliderFrame

                New("UICorner", {  }, SliderDraggable)

                SliderCircle.AnchorPoint = Vector2.new(1, 0.5)
                SliderCircle.BackgroundColor3 = GuiConfig.Color
                SliderCircle.BorderColor3 = Color3.fromRGB(0, 0, 0)
                SliderCircle.BorderSizePixel = 0
                SliderCircle.Position = UDim2.new(1, 4, 0.5, 0)
                SliderCircle.Size = UDim2.new(0, 8, 0, 8)
                SliderCircle.Name = "SliderCircle"
                SliderCircle.Parent = SliderDraggable

                New("UICorner", {  }, SliderCircle)

                New("UIStroke", { Color = GuiConfig.Color }, SliderCircle)

                local Dragging = false
                -- Pembulatan ke kelipatan Factor (benar untuk angka negatif) + buang noise floating point (0.30000000000000004)
                local function Round(Number, Factor)
                    if not Factor or Factor <= 0 then Factor = 1 end
                    local Result = math.floor(Number / Factor + 0.5) * Factor
                    local dec, f = 0, Factor
                    while dec < 6 and math.abs(f - math.floor(f + 1e-9)) > 1e-9 do
                        f = f * 10
                        dec = dec + 1
                    end
                    return tonumber(string.format("%." .. dec .. "f", Result)) + 0   -- "+ 0" membuang -0
                end
                function SliderFunc:Set(Value)
                    Value = math.clamp(Round(Value, SliderConfig.Increment), SliderConfig.Min, SliderConfig.Max)
                    SliderFunc.Value = Value
                    TextBox.Text = tostring(Value)
                    TweenService:Create(
                        SliderDraggable,
                        TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
                        { Size = UDim2.fromScale((Value - SliderConfig.Min) / (SliderConfig.Max - SliderConfig.Min), 1) }
                    ):Play()

                    SliderConfig.Callback(Value)
                    ConfigData[configKey] = Value
                    SaveConfig()
                end

                SliderFrame.InputBegan:Connect(function(Input)
                    if Input.UserInputType == Enum.UserInputType.MouseButton1 or Input.UserInputType == Enum.UserInputType.Touch then
                        Dragging = true
                        TweenService:Create(
                            SliderCircle,
                            TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
                            { Size = UDim2.new(0, 14, 0, 14) }
                        ):Play()
                        local SizeScale = math.clamp(
                            (Input.Position.X - SliderFrame.AbsolutePosition.X) / SliderFrame.AbsoluteSize.X,
                            0,
                            1
                        )
                        SliderFunc:Set(SliderConfig.Min + ((SliderConfig.Max - SliderConfig.Min) * SizeScale))
                    end
                end)

                SliderFrame.InputEnded:Connect(function(Input)
                    if Input.UserInputType == Enum.UserInputType.MouseButton1 or Input.UserInputType == Enum.UserInputType.Touch then
                        Dragging = false   -- Callback sudah dipanggil oleh Set(), tidak perlu dipanggil lagi di sini
                        TweenService:Create(
                            SliderCircle,
                            TweenInfo.new(0.2, Enum.EasingStyle.Quad, Enum.EasingDirection.Out),
                            { Size = UDim2.new(0, 8, 0, 8) }
                        ):Play()
                    end
                end)

                table.insert(GuiFunc._conns, UserInputService.InputChanged:Connect(function(Input)
                    if Dragging and (Input.UserInputType == Enum.UserInputType.MouseMovement or Input.UserInputType == Enum.UserInputType.Touch) then
                        local SizeScale = math.clamp(
                            (Input.Position.X - SliderFrame.AbsolutePosition.X) / SliderFrame.AbsoluteSize.X,
                            0,
                            1
                        )
                        SliderFunc:Set(SliderConfig.Min + ((SliderConfig.Max - SliderConfig.Min) * SizeScale))
                    end
                end))

                -- Saat mengetik: hanya buang karakter non-angka (nilai TIDAK dipaksa/di-clamp sampai selesai mengetik)
                TextBox:GetPropertyChangedSignal("Text"):Connect(function()
                    local allowed = (SliderConfig.Min < 0) and "[^%d%.%-]" or "[^%d%.]"
                    local clean = (TextBox.Text:gsub(allowed, ""))
                    if clean ~= TextBox.Text then TextBox.Text = clean end
                end)
                -- Selesai mengetik (Enter / klik di luar): baru dibulatkan, di-clamp dan diterapkan
                TextBox.FocusLost:Connect(function()
                    local typed = tonumber(TextBox.Text)
                    if typed then
                        SliderFunc:Set(math.clamp(typed, SliderConfig.Min, SliderConfig.Max))
                    else
                        TextBox.Text = tostring(SliderFunc.Value)
                    end
                end)
                SliderFunc:Set(SliderConfig.Default)
                CountItem = CountItem + 1
                Elements[configKey] = SliderFunc
                return SliderFunc
            end

            function Items:AddInput(InputConfig)
                local InputConfig = InputConfig or {}
                InputConfig.Title = InputConfig.Title or "Title"
                InputConfig.Content = InputConfig.Content or ""
                InputConfig.Callback = InputConfig.Callback or function() end
                InputConfig.Default = InputConfig.Default or ""

                local configKey = MakeKey("Input", InputConfig)
                if ConfigData[configKey] ~= nil then
                    InputConfig.Default = ConfigData[configKey]
                end

                local InputFunc = { Value = InputConfig.Default }

                local Input = Instance.new("Frame");
                local InputTitle = Instance.new("TextLabel");
                local InputContent = Instance.new("TextLabel");
                local InputFrame = Instance.new("Frame");
                local InputTextBox = Instance.new("TextBox");

                Input.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                Input.BackgroundTransparency = ProjectSenzu.ItemBgTransparency
                Input.BorderColor3 = Color3.fromRGB(0, 0, 0)
                Input.BorderSizePixel = 0
                Input.LayoutOrder = CountItem
                Input.Size = UDim2.new(1, 0, 0, 46)
                Input.Name = "Input"
                Input.Parent = ItemParent()

                New("UICorner", { CornerRadius = UDim.new(0, 4) }, Input)

                InputTitle.Font = Enum.Font.GothamBold
                InputTitle.Text = InputConfig.Title or "TextBox"
                InputTitle.TextColor3 = Color3.fromRGB(230.77499270439148, 230.77499270439148, 230.77499270439148)
                InputTitle.TextSize = 13
                InputTitle.TextXAlignment = Enum.TextXAlignment.Left
                InputTitle.TextYAlignment = Enum.TextYAlignment.Top
                InputTitle.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                InputTitle.BackgroundTransparency = 0.9990000128746033
                InputTitle.BorderColor3 = Color3.fromRGB(0, 0, 0)
                InputTitle.BorderSizePixel = 0
                InputTitle.Position = UDim2.new(0, 10, 0, 10)
                InputTitle.Size = UDim2.new(1, -20, 0, 13)
                InputTitle.Name = "InputTitle"
                InputTitle.Parent = Input

                InputContent.Font = Enum.Font.GothamBold
                InputContent.Text = InputConfig.Content or "This is a TextBox"
                InputContent.TextColor3 = Color3.fromRGB(255, 255, 255)
                InputContent.TextSize = 12
                InputContent.TextTransparency = 0.6000000238418579
                InputContent.TextWrapped = true
                InputContent.TextXAlignment = Enum.TextXAlignment.Left
                InputContent.TextYAlignment = Enum.TextYAlignment.Bottom
                InputContent.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                InputContent.BackgroundTransparency = 0.9990000128746033
                InputContent.BorderColor3 = Color3.fromRGB(0, 0, 0)
                InputContent.BorderSizePixel = 0
                InputContent.Position = UDim2.new(0, 10, 0, 25)
                InputContent.Size = UDim2.new(1, -20, 0, 12)
                InputContent.Name = "InputContent"
                InputContent.Parent = Input

                if InputConfig.Content == "" then
                    InputContent.Visible = false
                    InputContent.Size = UDim2.new(1, -20, 0, 0)
                    Input.Size = UDim2.new(1, 0, 0, 73)
                else
                    FitWrapLabel(InputContent, 12, function(h)
                        Input.Size = UDim2.new(1, 0, 0, h + 75)
                        InputFrame.Position = UDim2.new(0.5, 0, 0, InputContent.Position.Y.Offset + h + 10)
                    end, UpdateSizeSection)
                end

                InputFrame.AnchorPoint = Vector2.new(0.5, 0)
                InputFrame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                InputFrame.BackgroundTransparency = 0.949999988079071
                InputFrame.BorderColor3 = Color3.fromRGB(0, 0, 0)
                InputFrame.BorderSizePixel = 0
                InputFrame.ClipsDescendants = true
                if InputConfig.Content == "" then
                    InputFrame.Position = UDim2.new(0.5, 0, 0, 33)
                else
                    InputFrame.Position = UDim2.new(0.5, 0, 0, InputContent.Position.Y.Offset + InputContent.Size.Y.Offset + 10)
                end
                InputFrame.Size = UDim2.new(1, -20, 0, 30)
                InputFrame.Name = "InputFrame"
                InputFrame.Parent = Input

                New("UICorner", { CornerRadius = UDim.new(0, 4) }, InputFrame)

                InputTextBox.CursorPosition = -1
                InputTextBox.Font = Enum.Font.GothamBold
                InputTextBox.PlaceholderColor3 = Color3.fromRGB(120.00000044703484, 120.00000044703484,
                    120.00000044703484)
                InputTextBox.PlaceholderText = "Write ur input here!"
                InputTextBox.Text = InputConfig.Default
                InputTextBox.TextColor3 = Color3.fromRGB(255, 255, 255)
                InputTextBox.TextSize = 12
                InputTextBox.TextXAlignment = Enum.TextXAlignment.Left
                InputTextBox.AnchorPoint = Vector2.new(0, 0.5)
                InputTextBox.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                InputTextBox.BackgroundTransparency = 0.9990000128746033
                InputTextBox.BorderColor3 = Color3.fromRGB(0, 0, 0)
                InputTextBox.BorderSizePixel = 0
                InputTextBox.Position = UDim2.new(0, 5, 0.5, 0)
                InputTextBox.Size = UDim2.new(1, -10, 1, -8)
                InputTextBox.Name = "InputTextBox"
                InputTextBox.Parent = InputFrame
                function InputFunc:Set(Value)
                    InputTextBox.Text = Value
                    InputFunc.Value = Value
                    InputConfig.Callback(Value)
                    ConfigData[configKey] = Value
                    SaveConfig()
                end

                InputFunc:Set(InputFunc.Value)

                InputTextBox.FocusLost:Connect(function()
                    InputFunc:Set(InputTextBox.Text)
                end)
                CountItem = CountItem + 1
                Elements[configKey] = InputFunc
                return InputFunc
            end
            
            function Items:AddDropdown(DropdownConfig)
                local DropdownConfig = DropdownConfig or {}
                DropdownConfig.Title = DropdownConfig.Title or "Title"
                DropdownConfig.Content = DropdownConfig.Content or ""
                DropdownConfig.Multi = DropdownConfig.Multi or false
                DropdownConfig.Options = DropdownConfig.Options or {}
                DropdownConfig.Default = DropdownConfig.Default or (DropdownConfig.Multi and {} or nil)
                DropdownConfig.Callback = DropdownConfig.Callback or function() end

                local configKey = MakeKey("Dropdown", DropdownConfig)
                if ConfigData[configKey] ~= nil then
                    DropdownConfig.Default = ConfigData[configKey]
                end

                local DropdownFunc = { Value = DropdownConfig.Default, Options = DropdownConfig.Options }

                local Dropdown = Instance.new("Frame")
                local DropdownButton = Instance.new("TextButton")
                local DropdownTitle = Instance.new("TextLabel")
                local DropdownContent = Instance.new("TextLabel")
                local SelectOptionsFrame = Instance.new("Frame")
                local OptionSelecting = Instance.new("TextLabel")
                local OptionImg = Instance.new("ImageLabel")

                Dropdown.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                Dropdown.BackgroundTransparency = ProjectSenzu.ItemBgTransparency
                Dropdown.BorderSizePixel = 0
                Dropdown.LayoutOrder = CountItem
                Dropdown.Size = UDim2.new(1, 0, 0, 46)
                Dropdown.Name = "Dropdown"
                Dropdown.Parent = ItemParent()

                DropdownButton.Text = ""
                DropdownButton.BackgroundTransparency = 1
                DropdownButton.Size = UDim2.new(1, 0, 1, 0)
                DropdownButton.Name = "ToggleButton"
                DropdownButton.Parent = Dropdown

                New("UICorner", { CornerRadius = UDim.new(0, 4) }, Dropdown)

                DropdownTitle.Font = Enum.Font.GothamBold
                DropdownTitle.Text = DropdownConfig.Title
                DropdownTitle.TextColor3 = Color3.fromRGB(230, 230, 230)
                DropdownTitle.TextSize = 13
                DropdownTitle.TextXAlignment = Enum.TextXAlignment.Left
                DropdownTitle.TextTruncate = Enum.TextTruncate.AtEnd
                DropdownTitle.BackgroundTransparency = 1
                DropdownTitle.Position = UDim2.new(0, 10, 0, 10)
                DropdownTitle.Size = UDim2.new(1, -180, 0, 13)
                DropdownTitle.Name = "DropdownTitle"
                DropdownTitle.Parent = Dropdown

                DropdownContent.Font = Enum.Font.GothamBold
                DropdownContent.Text = DropdownConfig.Content
                DropdownContent.TextColor3 = Color3.fromRGB(255, 255, 255)
                DropdownContent.TextSize = 12
                DropdownContent.TextTransparency = 0.6
                DropdownContent.TextWrapped = true
                DropdownContent.TextXAlignment = Enum.TextXAlignment.Left
                DropdownContent.BackgroundTransparency = 1
                DropdownContent.Position = UDim2.new(0, 10, 0, 25)
                DropdownContent.Size = UDim2.new(1, -180, 0, 12)
                DropdownContent.TextYAlignment = Enum.TextYAlignment.Top
                DropdownContent.Name = "DropdownContent"
                DropdownContent.Parent = Dropdown

                FitWrapLabel(DropdownContent, 12, function(h)
                    Dropdown.Size = UDim2.new(1, 0, 0, math.max(46, h + 33))
                end, UpdateSizeSection)

                SelectOptionsFrame.AnchorPoint = Vector2.new(1, 0.5)
                SelectOptionsFrame.BackgroundTransparency = 0.95
                SelectOptionsFrame.Position = UDim2.new(1, -7, 0.5, 0)
                SelectOptionsFrame.Size = UDim2.new(0, 148, 0, 30)
                SelectOptionsFrame.Name = "SelectOptionsFrame"
                SelectOptionsFrame.LayoutOrder = CountDropdown
                SelectOptionsFrame.Parent = Dropdown

                New("UICorner", { CornerRadius = UDim.new(0, 4) }, SelectOptionsFrame)

                DropdownButton.Activated:Connect(function()
                    if not MoreBlur.Visible then
                        MoreBlur.Visible = true
                        DropPageLayout:JumpToIndex(SelectOptionsFrame.LayoutOrder)
                        TweenService:Create(MoreBlur, TweenInfo.new(0.3), { BackgroundTransparency = 1 }):Play()
                        TweenService:Create(DropdownSelect, TweenInfo.new(0.3), { Position = UDim2.new(1, -11, 0.5, 0) })
                            :Play()
                    end
                end)

                OptionSelecting.Font = Enum.Font.GothamBold
                OptionSelecting.Text = "None"
                OptionSelecting.TextColor3 = Color3.fromRGB(255, 255, 255)
                OptionSelecting.TextSize = 12
                OptionSelecting.TextTransparency = 0.6
                OptionSelecting.TextXAlignment = Enum.TextXAlignment.Left
                OptionSelecting.AnchorPoint = Vector2.new(0, 0.5)
                OptionSelecting.BackgroundTransparency = 1
                OptionSelecting.Position = UDim2.new(0, 5, 0.5, 0)
                OptionSelecting.Size = UDim2.new(1, -30, 1, -8)
                OptionSelecting.Name = "OptionSelecting"
                OptionSelecting.Parent = SelectOptionsFrame

                OptionImg.Image = "rbxassetid://16851841101"
                OptionImg.ImageColor3 = Color3.fromRGB(230, 230, 230)
                OptionImg.AnchorPoint = Vector2.new(1, 0.5)
                OptionImg.BackgroundTransparency = 1
                OptionImg.Position = UDim2.new(1, 0, 0.5, 0)
                OptionImg.Size = UDim2.new(0, 25, 0, 25)
                OptionImg.Name = "OptionImg"
                OptionImg.Parent = SelectOptionsFrame

                local DropdownContainer = Instance.new("Frame")
                DropdownContainer.Size = UDim2.new(1, 0, 1, 0)
                DropdownContainer.BackgroundTransparency = 1
                DropdownContainer.Parent = DropdownFolder

                local SearchBox = Instance.new("TextBox")
                SearchBox.PlaceholderText = "Search"
                SearchBox.Font = Enum.Font.Gotham
                SearchBox.Text = ""
                SearchBox.TextSize = 12
                SearchBox.TextColor3 = GuiConfig._DropdownSearchText
                SearchBox.PlaceholderColor3 = GuiConfig._DropdownPlaceholder
                SearchBox.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
                SearchBox.BackgroundTransparency = 0.9
                SearchBox.BorderSizePixel = 0
                SearchBox.Size = UDim2.new(1, 0, 0, 25)
                SearchBox.Position = UDim2.new(0, 0, 0, 0)
                SearchBox.ClearTextOnFocus = false
                SearchBox.Name = "SearchBox"
                SearchBox.Parent = DropdownContainer

                local ScrollSelect = Instance.new("ScrollingFrame")
                ScrollSelect.Size = UDim2.new(1, 0, 1, -30)
                ScrollSelect.Position = UDim2.new(0, 0, 0, 30)
                ScrollSelect.ScrollBarImageTransparency = 1
                ScrollSelect.BorderSizePixel = 0
                ScrollSelect.BackgroundTransparency = 1
                ScrollSelect.ScrollBarThickness = 0
                ScrollSelect.CanvasSize = UDim2.new(0, 0, 0, 0)
                ScrollSelect.Name = "ScrollSelect"
                ScrollSelect.Parent = DropdownContainer

                local UIListLayout4 = Instance.new("UIListLayout")
                UIListLayout4.Padding = UDim.new(0, 3)
                UIListLayout4.SortOrder = Enum.SortOrder.LayoutOrder
                UIListLayout4.Parent = ScrollSelect

                UIListLayout4:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
                    ScrollSelect.CanvasSize = UDim2.new(0, 0, 0, UIListLayout4.AbsoluteContentSize.Y)
                end)

                SearchBox:GetPropertyChangedSignal("Text"):Connect(function()
                    local query = string.lower(SearchBox.Text)
                    for _, option in pairs(ScrollSelect:GetChildren()) do
                        if option.Name == "Option" and option:FindFirstChild("OptionText") then
                            local text = string.lower(option.OptionText.Text)
                            option.Visible = query == "" or string.find(text, query, 1, true)
                        end
                    end
                    ScrollSelect.CanvasSize = UDim2.new(0, 0, 0, UIListLayout4.AbsoluteContentSize.Y)
                end)

                local DropCount = 0

                function DropdownFunc:Clear()
                    for _, DropFrame in ScrollSelect:GetChildren() do
                        if DropFrame.Name == "Option" then
                            DropFrame:Destroy()
                        end
                    end
                    DropdownFunc.Value = DropdownConfig.Multi and {} or nil
                    DropdownFunc.Options = {}
                    OptionSelecting.Text = "None"
                    DropCount = 0
                end

                function DropdownFunc:AddOption(option)
                    local label, value
                    if typeof(option) == "table" and option.Label and option.Value ~= nil then
                        label = tostring(option.Label)
                        value = option.Value
                    else
                        label = tostring(option)
                        value = option
                    end

                    local Option = Instance.new("Frame")
                    local OptionButton = Instance.new("TextButton")
                    local OptionText = Instance.new("TextLabel")
                    local ChooseFrame = Instance.new("Frame")

                    Option.BackgroundTransparency = 1
                    Option.Size = UDim2.new(1, 0, 0, 30)
                    Option.Name = "Option"
                    Option.Parent = ScrollSelect

                    New("UICorner", { CornerRadius = UDim.new(0, 3) }, Option)

                    OptionButton.BackgroundTransparency = 1
                    OptionButton.Size = UDim2.new(1, 0, 1, 0)
                    OptionButton.Text = ""
                    OptionButton.Name = "OptionButton"
                    OptionButton.Parent = Option

                    OptionText.Font = Enum.Font.GothamBold
                    OptionText.Text = label
                    OptionText.TextSize = 13
                    OptionText.TextColor3 = GuiConfig._DropdownOptionText
                    OptionText.Position = UDim2.new(0, 8, 0, 8)
                    OptionText.Size = UDim2.new(1, -100, 0, 13)
                    OptionText.BackgroundTransparency = 1
                    OptionText.TextXAlignment = Enum.TextXAlignment.Left
                    OptionText.Name = "OptionText"
                    OptionText.Parent = Option

                    Option:SetAttribute("RealValue", value)

                    ChooseFrame.AnchorPoint = Vector2.new(0, 0.5)
                    ChooseFrame.BackgroundColor3 = GuiConfig.Color
                    ChooseFrame.Position = UDim2.new(0, 2, 0.5, 0)
                    ChooseFrame.Size = UDim2.new(0, 0, 0, 0)
                    ChooseFrame.Name = "ChooseFrame"
                    ChooseFrame.Parent = Option

                    New("UIStroke", { Color = GuiConfig.Color, Thickness = 1.6, Transparency = 0.999 }, ChooseFrame)
                    New("UICorner", {  }, ChooseFrame)

                    OptionButton.Activated:Connect(function()
                        if DropdownConfig.Multi then
                            if not table.find(DropdownFunc.Value, value) then
                                table.insert(DropdownFunc.Value, value)
                            else
                                for i, v in pairs(DropdownFunc.Value) do
                                    if v == value then
                                        table.remove(DropdownFunc.Value, i)
                                        break
                                    end
                                end
                            end
                        else
                            DropdownFunc.Value = value
                        end
                        DropdownFunc:Set(DropdownFunc.Value)
                    end)
                end

                function DropdownFunc:Set(Value)
                    if DropdownConfig.Multi then
                        DropdownFunc.Value = type(Value) == "table" and Value or {}
                    else
                        DropdownFunc.Value = (type(Value) == "table" and Value[1]) or Value
                    end

                    ConfigData[configKey] = DropdownFunc.Value
                    SaveConfig()

                    local texts = {}
                    for _, Drop in ScrollSelect:GetChildren() do
                        if Drop.Name == "Option" and Drop:FindFirstChild("OptionText") then
                            local v = Drop:GetAttribute("RealValue")
                            local selected = DropdownConfig.Multi and table.find(DropdownFunc.Value, v) or
                                DropdownFunc.Value == v

                            if selected then
                                TweenService:Create(Drop.ChooseFrame, TweenInfo.new(0.2),
                                    { Size = UDim2.new(0, 1, 0, 12) }):Play()
                                TweenService:Create(Drop.ChooseFrame.UIStroke, TweenInfo.new(0.2), { Transparency = 0 })
                                    :Play()
                                TweenService:Create(Drop, TweenInfo.new(0.2), { BackgroundTransparency = 0.935 }):Play()
                                table.insert(texts, Drop.OptionText.Text)
                            else
                                TweenService:Create(Drop.ChooseFrame, TweenInfo.new(0.1),
                                    { Size = UDim2.new(0, 0, 0, 0) }):Play()
                                TweenService:Create(Drop.ChooseFrame.UIStroke, TweenInfo.new(0.1),
                                    { Transparency = 0.999 }):Play()
                                TweenService:Create(Drop, TweenInfo.new(0.1), { BackgroundTransparency = 0.999 }):Play()
                            end
                        end
                    end

                    OptionSelecting.Text = (#texts == 0)
                        and ("None")
                        or table.concat(texts, ", ")

                    if DropdownConfig.Callback then
                        if DropdownConfig.Multi then
                            DropdownConfig.Callback(DropdownFunc.Value)
                        else
                            local str = (DropdownFunc.Value ~= nil) and tostring(DropdownFunc.Value) or ""
                            DropdownConfig.Callback(str)
                        end
                    end
                end

                function DropdownFunc:SetValue(val)
                    self:Set(val)
                end

                function DropdownFunc:GetValue()
                    return self.Value
                end

                function DropdownFunc:SetValues(newList, selecting)
                    newList = newList or {}
                    selecting = selecting or (DropdownConfig.Multi and {} or nil)
                    DropdownFunc:Clear()
                    for _, v in ipairs(newList) do
                        DropdownFunc:AddOption(v)
                    end
                    DropdownFunc.Options = newList
                    DropdownFunc:Set(selecting)
                end

                DropdownFunc:SetValues(DropdownFunc.Options, DropdownFunc.Value)

                CountItem = CountItem + 1
                CountDropdown = CountDropdown + 1
                Elements[configKey] = DropdownFunc
                return DropdownFunc
            end

            function Items:AddDivider()
                local Divider = Instance.new("Frame")
                Divider.Name = "Divider"
                Divider.Parent = ItemParent()
                Divider.AnchorPoint = Vector2.new(0.5, 0)
                Divider.Position = UDim2.new(0.5, 0, 0, 0)
                Divider.Size = UDim2.new(1, 0, 0, 2)
                Divider.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                Divider.BackgroundTransparency = 0
                Divider.BorderSizePixel = 0
                Divider.LayoutOrder = CountItem

                local UIGradient = Instance.new("UIGradient")
                UIGradient.Color = ColorSequence.new {
                    ColorSequenceKeypoint.new(0, Color3.fromRGB(20, 20, 20)),
                    ColorSequenceKeypoint.new(0.5, GuiConfig.Color),
                    ColorSequenceKeypoint.new(1, Color3.fromRGB(20, 20, 20))
                }
                UIGradient.Parent = Divider

                local UICorner = Instance.new("UICorner")
                UICorner.CornerRadius = UDim.new(0, 2)
                UICorner.Parent = Divider

                CountItem = CountItem + 1
                return Divider
            end

            function Items:AddSubSection(title)
                title = title or "Sub Section"

                local SubSection = Instance.new("Frame")
                SubSection.Name = "SubSection"
                SubSection.Parent = ItemParent()
                SubSection.BackgroundTransparency = 1
                SubSection.Size = UDim2.new(1, 0, 0, 22)
                SubSection.LayoutOrder = CountItem

                local Background = Instance.new("Frame")
                Background.Parent = SubSection
                Background.Size = UDim2.new(1, 0, 1, 0)
                Background.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
                Background.BackgroundTransparency = 1
                Background.BorderSizePixel = 0
                Instance.new("UICorner", Background).CornerRadius = UDim.new(0, 6)

                local Label = Instance.new("TextLabel")
                Label.Parent = SubSection
                Label.AnchorPoint = Vector2.new(0, 0.5)
                Label.Position = UDim2.new(0, 10, 0.5, 0)
                Label.Size = UDim2.new(1, -20, 1, 0)
                Label.BackgroundTransparency = 1
                Label.Font = Enum.Font.GothamBold
                Label.Text = title
                Label.TextColor3 = GuiConfig.Color
                Label.TextSize = 12
                Label.TextXAlignment = Enum.TextXAlignment.Left

                CountItem = CountItem + 1
                return SubSection
            end

            -- Items:AddRow(jumlahKolom, function(Row) ... end)
            -- Contoh 2 kolom sejajar:
            --   Section:AddRow(2, function(Row)
            --       Row:AddToggle({ Title = "A" })
            --       Row:AddToggle({ Title = "B" })
            --   end)
            -- Item pertama masuk kolom 1, kedua kolom 2, dst. Tinggi baris mengikuti item tertinggi.
            function Items:AddRow(Count, Builder)
                if type(Count) == "function" then
                    Builder, Count = Count, 2
                end
                Count = math.max(1, math.floor(tonumber(Count) or 2))
                local GAP = 6

                local Row = Instance.new("Frame")
                Row.Name = "Row"
                Row.BackgroundTransparency = 1
                Row.BorderSizePixel = 0
                Row.LayoutOrder = CountItem
                Row.Size = UDim2.new(1, 0, 0, 46)
                Row.Parent = SectionAdd

                New("UIListLayout", {
                    FillDirection = Enum.FillDirection.Horizontal,
                    SortOrder = Enum.SortOrder.LayoutOrder,
                    Padding = UDim.new(0, GAP),
                }, Row)

                local Slots = {}
                for i = 1, Count do
                    local Slot = Instance.new("Frame")
                    Slot.Name = "RowSlot"
                    Slot.BackgroundTransparency = 1
                    Slot.BorderSizePixel = 0
                    Slot.LayoutOrder = i
                    Slot.Size = UDim2.new(1 / Count, -(GAP * (Count - 1)) / Count, 0, 46)
                    Slot.Parent = Row

                    New("UIListLayout", { SortOrder = Enum.SortOrder.LayoutOrder, Padding = UDim.new(0, 3) }, Slot)

                    Slots[i] = Slot
                end

                -- tinggi baris = kolom tertinggi
                local function RecalcRow()
                    local RowHeight = 0
                    for _, Slot in Slots do
                        local Sum, N = 0, 0
                        for _, c in Slot:GetChildren() do
                            if c:IsA("GuiObject") and c.Visible then
                                Sum = Sum + c.Size.Y.Offset
                                N = N + 1
                            end
                        end
                        if N > 1 then Sum = Sum + 3 * (N - 1) end
                        Slot.Size = UDim2.new(Slot.Size.X.Scale, Slot.Size.X.Offset, 0, Sum)
                        RowHeight = math.max(RowHeight, Sum)
                    end
                    Row.Size = UDim2.new(1, 0, 0, RowHeight)
                    UpdateSizeSection()
                end

                RowSlots, RowSlotIndex = Slots, 0
                local ok, err = pcall(Builder or function() end, Items)
                RowSlots, RowSlotIndex = nil, 0
                -- error di dalam Builder diabaikan (tidak dicetak ke F9)

                for _, Slot in Slots do
                    for _, c in Slot:GetChildren() do
                        if c:IsA("GuiObject") then
                            c:GetPropertyChangedSignal("Size"):Connect(RecalcRow)
                            c:GetPropertyChangedSignal("Visible"):Connect(RecalcRow)
                        end
                    end
                end
                RecalcRow()

                CountItem = CountItem + 1
                return Items
            end

            CountSection = CountSection + 1
            return Items
        end

        CountTab = CountTab + 1
        local safeName = TabConfig.Name:gsub("%s+", "_")
        _G[safeName] = Sections
        return Sections
    end

    -- ═══════════════════════════════════════════════════
    -- SETTINGS TAB (Config Profile System)
    -- ═══════════════════════════════════════════════════
    function Tabs:AddConfigTab()
        pcall(function()
            local SettingsTab = Tabs:AddTab({ Name = "Config", Icon = "settings" })

            -- helper notif dengan title "Config" (global notif: msg, delay, color, title)
            local function cnotif(msg, delay)
                notif(msg, delay, GuiConfig.Color, "Config")
            end

            -- ═══ INFO SECTION ═══
            -- local InfoSection = SettingsTab:AddSection("Informasi", true)
            -- InfoSection:AddParagraph({
            --     Title   = "Auto-Save Aktif",
            --     Content = "Semua perubahan (toggle, slider, dropdown) otomatis tersimpan ke file saat kamu ubah. Saat relog, setting kamu akan otomatis ter-load kembali.\n\nGunakan Profile untuk menyimpan beberapa preset yang berbeda."
            -- })

            -- ═══ AUTO-LOAD SAAT STARTUP ═══
            -- Panggil LoadConfigElements setelah semua element dibuat
            task.defer(function()
                pcall(function()
                    LoadConfigElements()
                end)
            end)

            -- ═══ PROFILE SECTION ═══
            local ConfigSection = SettingsTab:AddSection("Config Profile")

            local profileDropdown
            local selectedProfile = "None"

            local function RefreshProfileDropdown()
                local fresh = GetProfileList()
                if #fresh == 0 then table.insert(fresh, "None") end
                if profileDropdown and profileDropdown.SetValues then
                    profileDropdown:SetValues(fresh, fresh[1] or "None")
                end
                selectedProfile = fresh[1] or "None"
                return fresh
            end

            local profiles = GetProfileList()
            if #profiles == 0 then table.insert(profiles, "None") end

            profileDropdown = ConfigSection:AddDropdown({
                Title   = "Select Profile",
                Content = "Select the profile you want to load or delete",
                Options = profiles,
                Default = profiles[1] or "None",
                Multi   = false,
                Callback = function(val)
                    selectedProfile = val
                end
            })

            ConfigSection:AddButton({
                Title = "Refresh",
                Callback = function()
                    local fresh = RefreshProfileDropdown()
                    cnotif("Found " .. tostring(#fresh) .. " profiles", 3)
                end
            })

            local profileNameInput = ConfigSection:AddInput({
                Title   = "New Profile Name",
                Content = "Type a name to save new profile",
                Default = "",
                Callback = function(val) end
            })


            ConfigSection:AddButton({
                Title    = "Save Profile",
                SubTitle = "Load Profile",
                Callback = function()
                    local name = profileNameInput.Value
                    if not name or name == "" or name == "None" then
                        cnotif("Type a profile name first!", 3)
                        return
                    end
                    name = name:gsub("[^%w_%-]", "_")
                    local ok = SaveProfile(name)
                    if ok then
                        cnotif("Profile '" .. name .. "' saved successfully!", 4)
                        task.wait(0.2)
                        local newList = GetProfileList()
                        if not table.find(newList, name) then
                            table.insert(newList, name)
                            table.sort(newList)
                        end
                        if #newList > 1 then
                            for i = #newList, 1, -1 do
                                if newList[i] == "None" then table.remove(newList, i) end
                            end
                        end
                        if profileDropdown and profileDropdown.SetValues then
                            profileDropdown:SetValues(newList, name)
                        end
                        selectedProfile = name
                    else
                        cnotif("Failed to save profile!", 3)
                    end
                end,
                SubCallback = function()
                    if not selectedProfile or selectedProfile == "" or selectedProfile == "None" then
                        cnotif("Select a profile from the dropdown first!", 3)
                        return
                    end
                    cnotif("Loading '" .. selectedProfile .. "'...", 2)
                    task.spawn(function()
                        local ok = LoadProfile(selectedProfile)
                        if ok then
                            cnotif("Profile '" .. selectedProfile .. "' loaded successfully!", 4)
                        else
                            cnotif("Failed to load profile!", 3)
                        end
                    end)
                end
            })

            ConfigSection:AddButton({
                Title    = "Delete Profile",
                SubTitle = "Copy Config",
                Callback = function()
                    if not selectedProfile or selectedProfile == "" or selectedProfile == "None" then
                        cnotif("Select the profile you want to delete!", 3)
                        return
                    end
                    local ok = DeleteProfile(selectedProfile)
                    if ok then
                        cnotif("Profile '" .. selectedProfile .. "' dihapus!", 3)
                        selectedProfile = "None"
                        local newList = GetProfileList()
                        if #newList == 0 then table.insert(newList, "None") end
                        if profileDropdown and profileDropdown.SetValues then
                            profileDropdown:SetValues(newList, newList[1])
                        end
                    else
                        cnotif("Failed to delete profile!", 3)
                    end
                end,
                SubCallback = function()
                    -- Copy current ConfigData as JSON to clipboard
                    if setclipboard then
                        local json = HttpService:JSONEncode(ConfigData)
                        setclipboard(json)
                        cnotif("Config copied to clipboard successfully!", 3)
                    else
                        cnotif("Executor does not support clipboard!", 3)
                    end
                end
            })

            -- ═══ IMPORT CONFIG ═══
            local ImportSection = SettingsTab:AddSection("Import Config")

            local importInput = ImportSection:AddPanel({
                Title       = "Paste JSON Config",
                Placeholder = "Paste JSON Config here...",
                ButtonText  = "Import & Load",
                Callback    = function(val)
                    if not val or val == "" then
                        cnotif("JSON Input is empty!", 3)
                        return
                    end
                    local ok, data = pcall(function()
                        return HttpService:JSONDecode(val)
                    end)
                    if not ok or type(data) ~= "table" then
                        cnotif("Invalid JSON!", 3)
                        return
                    end
                    -- Apply ke ConfigData dan semua elemen
                    for key, v in pairs(data) do
                        ConfigData[key] = v
                    end
                    SaveConfig()
                    for key, element in pairs(Elements) do
                        if ConfigData[key] ~= nil and element.Set then
                            pcall(function()
                                element:Set(ConfigData[key])
                            end)
                            task.wait(0.02)
                        end
                    end
                    cnotif("Config imported successfully!", 4)
                end
            })
        end)
    end


    -- Kumpulkan referensi elemen warna yang bisa diubah secara dinamis
    local _themeColorElements = {
        titleLabel  = TextLabel,
        footerFrm   = FooterFrame,
        mainStroke  = MainStroke,
        decideFrm   = DecideFrame,
        mainBG      = Main,
        executorFrm = Executor,
    }
    local _themeTabChooseFrames = {} -- referensi semua ChooseFrame tab

    function Tabs:SetTheme(newColor, newColor2)
        GuiConfig.Color  = newColor
        GuiConfig.Color2 = newColor2 or newColor
        CurrentAccent    = newColor

        -- Update topbar (Title & Footer text)
        -- TweenService:Create(_themeColorElements.titleLabel, TweenInfo.new(0.4), { TextColor3 = newColor }):Play()
        if _themeColorElements.footerFrm then
            TweenService:Create(_themeColorElements.footerFrm, TweenInfo.new(0.4), { BackgroundColor3 = newColor }):Play()
        end
        if _themeColorElements.executorFrm then
            TweenService:Create(_themeColorElements.executorFrm, TweenInfo.new(0.4), { BackgroundColor3 = newColor }):Play()
        end

        -- Update border stroke & DecideFrame
        TweenService:Create(_themeColorElements.mainStroke, TweenInfo.new(0.4), { Color = newColor:Lerp(Color3.new(0, 0, 0), 0.45) }):Play()
        TweenService:Create(UIStroke14, TweenInfo.new(0.4), { Color = newColor }):Play()
        TweenService:Create(TextLabel1, TweenInfo.new(0.4), { TextColor3 = GetContrastColor(newColor) }):Play()
        TweenService:Create(ExecutorTextLabel, TweenInfo.new(0.4), { TextColor3 = GetContrastColor(newColor) }):Play()
        for _, badge in ipairs(StatBadges.list) do
            TweenService:Create(badge.frame, TweenInfo.new(0.4), { BackgroundColor3 = newColor }):Play()
            TweenService:Create(badge.label, TweenInfo.new(0.4), { TextColor3 = GetContrastColor(newColor) }):Play()
        end
        TweenService:Create(SearchBarStroke, TweenInfo.new(0.4), { Color = newColor }):Play()
        TweenService:Create(_themeColorElements.decideFrm, TweenInfo.new(0.4), { BackgroundColor3 = newColor }):Play()

        -- Update window background (Color2)
        if GuiConfig.ThemeMode ~= "light" then
            TweenService:Create(_themeColorElements.mainBG, TweenInfo.new(0.4), { BackgroundColor3 = GuiConfig.Color2 }):Play()
        end

        -- Update ChooseFrame & UIStroke di semua tab
        for _, tab in ScrollTab:GetChildren() do
            if tab.Name == "Tab" then
                local cf = tab:FindFirstChild("ChooseFrame")
                if cf then
                    TweenService:Create(cf, TweenInfo.new(0.3), { BackgroundColor3 = newColor }):Play()
                    local stk = cf:FindFirstChildOfClass("UIStroke")
                    if stk then
                        TweenService:Create(stk, TweenInfo.new(0.3), { Color = newColor }):Play()
                    end
                end
            end
        end

        -- Update semua elemen di dalam ProjectSenzuOnTop secara langsung
        for _, gui in ProjectSenzuOnTop:GetDescendants() do
            if gui.Name == "SectionTitle" and gui:IsA("TextLabel") then
                local sectionReal = gui.Parent
                if sectionReal and sectionReal.Name == "SectionReal" then
                    local featureFrame = sectionReal:FindFirstChild("FeatureFrame")
                    -- Jika FeatureFrame tidak ada (AlwaysOpen=true) atau rotasinya 90 (terbuka)
                    if not featureFrame or featureFrame.Rotation > 45 then
                        TweenService:Create(gui, TweenInfo.new(0.4), { TextColor3 = Color3.fromRGB(255, 255, 255) }):Play()
                    end
                end
            end

            -- Bingkai kartu section
            if gui.Name == "CardStroke" and gui:IsA("UIStroke") then
                TweenService:Create(gui, TweenInfo.new(0.4), { Color = newColor }):Play()
            end

            -- SubSection label
            if gui.Name == "SubSection" and gui:IsA("Frame") then
                local lbl = gui:FindFirstChildOfClass("TextLabel")
                if lbl then
                    TweenService:Create(lbl, TweenInfo.new(0.4), { TextColor3 = newColor }):Play()
                end
            end

            -- Divider gradient
            if gui.Name == "Divider" and gui:IsA("Frame") then
                local grad = gui:FindFirstChildOfClass("UIGradient")
                if grad then
                    grad.Color = ColorSequence.new({
                        ColorSequenceKeypoint.new(0, Color3.fromRGB(20, 20, 20)),
                        ColorSequenceKeypoint.new(0.5, newColor),
                        ColorSequenceKeypoint.new(1, Color3.fromRGB(20, 20, 20))
                    })
                end
            end

            -- Dropdown ChooseFrame option
            if gui.Name == "ChooseFrame" and gui:IsA("Frame") and gui.Parent and gui.Parent.Name == "Option" then
                TweenService:Create(gui, TweenInfo.new(0.3), { BackgroundColor3 = newColor }):Play()
                local stk = gui:FindFirstChildOfClass("UIStroke")
                if stk then
                    TweenService:Create(stk, TweenInfo.new(0.3), { Color = newColor }):Play()
                end
            end

            -- Slider (garis terisi, bulatan, dan outline bulatan)
            if (gui.Name == "SliderDraggable" or gui.Name == "SliderCircle") and gui:IsA("Frame") then
                TweenService:Create(gui, TweenInfo.new(0.3), { BackgroundColor3 = newColor }):Play()
                local stk = gui:FindFirstChildOfClass("UIStroke")
                if stk then
                    TweenService:Create(stk, TweenInfo.new(0.3), { Color = newColor }):Play()
                end
            end

            -- Toggle yang sedang ON (FeatureFrame.BackgroundTransparency ≈ 0)
            if gui.Name == "FeatureFrame" and gui:IsA("Frame") and gui.Parent and gui.Parent.Name == "Toggle" then
                if gui.BackgroundTransparency < 0.1 then
                    -- toggle ini ON, update warnanya langsung
                    TweenService:Create(gui, TweenInfo.new(0.3), { BackgroundColor3 = newColor }):Play()
                    local stk = gui:FindFirstChildOfClass("UIStroke")
                    if stk then
                        TweenService:Create(stk, TweenInfo.new(0.3), { Color = newColor }):Play()
                    end
                    -- knob toggle ikut kontras terhadap warna aksen baru
                    local knob = gui:FindFirstChild("ToggleCircle")
                    if knob then
                        TweenService:Create(knob, TweenInfo.new(0.3), { BackgroundColor3 = GetContrastColor(newColor) }):Play()
                    end
                    -- update juga teks judul toggle yang ON
                    local toggleParent = gui.Parent
                    if toggleParent then
                        local ttl = toggleParent:FindFirstChild("ToggleTitle")
                        if ttl then
                            TweenService:Create(ttl, TweenInfo.new(0.3), { TextColor3 = Color3.fromRGB(255, 255, 255) }):Play()
                        end
                    end
                end
            end

        end
    end

    -- API publik: tutup & bersihkan window (semua koneksi global diputus). Berguna sebelum menjalankan ulang script.
    function Tabs:Destroy()
        GuiFunc:DestroyGui()
    end

    task.defer(function()
        if tostring(GuiConfig.ThemeMode or ""):lower() == "light" then Tabs:SetMode("light") end
        task.wait(1)
        SenzuSounds.Ready = true
    end)
    return Tabs
end

-- Info key (Luarmor): tampilkan sisa waktu key user.
local function formatTime(seconds)
    if seconds == math.huge then return "Lifetime" end
    local days = math.floor(seconds / 86400)
    local hours = math.floor(seconds % 86400 / 3600)
    return string.format("%dd %dh left", days, hours)
end

-- Aman kalau script belum diproteksi Luarmor (variabel LRM_ nil).
function ProjectSenzu:GetKeyInfo()
    return {
        Tier = LRM_KeyTier or "unknown",                 -- "premium", "free" atau "keyless"
        TimeLeft = formatTime(tonumber(LRM_SecondsLeft) or 0),
        Runs = LRM_TotalExecutions or 0,
        DiscordID = LRM_LinkedDiscordID or "0",
    }
end

do
    local info = ProjectSenzu:GetKeyInfo()
    print("Plan:", info.Tier)
    print("Time:", info.TimeLeft)
    print("Runs:", info.Runs)
    if info.DiscordID ~= "0" then
        print("Discord user:", info.DiscordID)
    end
end

return ProjectSenzu
