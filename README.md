-- fulldump.lua — full client-side DataModel dump, single file, paste into the executor.
-- no readfile / dofile / getgenv dependency: run it, then open the runner's workspace\fulldump\
--
-- usage: paste this whole file, then call:
--   getgenv().Fulldump.run()      -- everything
--   getgenv().Fulldump.quick()    -- skip require() + live net sniff
--   getgenv().Fulldump.sniff(60)  -- just the live net window
--   getgenv().Fulldump.report()   -- phase timings + file list from 00_meta.json

local Http = game:GetService("HttpService")
local Workspace = game:GetService("Workspace")
local Players = game:GetService("Players")

----------------------------------------------------------------------
-- CONFIG
----------------------------------------------------------------------
local CONFIG = {
    NET_SECONDS        = 60,      -- live remote sniff window
    NETLOG_MAX         = 20000,   -- ring cap so a firefight can't eat the heap
    NODE_BUDGET        = 300000,  -- hard ceiling on instances walked
    MAX_QUEUE          = 250000,
    PART_DETAIL_MAX    = 20000,   -- parts that get position/size detail
    VALUE_MAX          = 20000,
    SCRIPT_MAX         = 6000,
    REMOTE_MAX         = 4000,
    WORLD_MAX          = 6000,
    NAME_MAX           = 40000,
    REQUIRE_MODULES    = true,
    MODULE_TIME_BUDGET = 20,      -- seconds; the walk stops cleanly here
    GC_LIMIT           = 4000,
    CONNECTION_LIMIT   = 2000,
    GUI_MAX_DEPTH      = 3,
    OUT_DIR            = "fulldump", -- "." writes into the runner workspace root
    HTML_INDEX         = true,
    PROGRESS_EVERY     = 25000,   -- progress lines during the walk
}

-- user overrides without editing the file: getgenv().FulldumpConfig = { NET_SECONDS = 30 }
do
    local ov
    if type(getgenv) == "function" then ov = getgenv().FulldumpConfig end
    if type(ov) == "table" then
        for k, v in pairs(ov) do CONFIG[k] = v end
    end
end

----------------------------------------------------------------------
-- output registry: every file this script can produce is declared up front so
-- index.html and 00_meta.json list the whole set regardless of write order
----------------------------------------------------------------------
local FILE_DESC = {
    ["00_meta.json"]       = { "json", "capabilities, counts, timings, file list" },
    ["01_tree.ndjson"]     = { "ndjson", "one JSON object per instance (streaming)" },
    ["02_values.json"]     = { "json", "every value object: game config lives here" },
    ["03_remotes.json"]    = { "json", "all RemoteEvent/RemoteFunction/UnreliableRemoteEvent" },
    ["04_scripts.json"]    = { "json", "all Script/LocalScript/ModuleScript instances" },
    ["05_modules.json"]    = { "json", "require() results + constants/upvalues per export" },
    ["06_netlog.ndjson"]   = { "ndjson", "one JSON object per remote call" },
    ["07_net_summary.json"] = { "json", "per-remote counts, intervals, rpm, arg shapes" },
    ["08_connections.json"] = { "json", "signal handlers with src:line" },
    ["09_gc.json"]         = { "json", "every function in getgc with constants/upvalues" },
    ["10_env.json"]        = { "json", "_G / getgenv / shared, base globals filtered" },
    ["11_players.json"]    = { "json", "per player: character, humanoid, tools, gui" },
    ["12_world.json"]      = { "json", "models, asset ids, npcs, spawns, sounds, terrain, lighting" },
    ["names.txt"]          = { "txt", "every unique instance name with a count" },
    ["index.html"]         = { "html", "offline browser for this dump" },
}

local FILE_ORDER = {
    "00_meta.json", "01_tree.ndjson", "02_values.json", "03_remotes.json",
    "04_scripts.json", "05_modules.json", "06_netlog.ndjson", "07_net_summary.json",
    "08_connections.json", "09_gc.json", "10_env.json", "11_players.json",
    "12_world.json", "names.txt", "index.html",
}

local FILES, FILE_SEEN = {}, {}

local function registerFile(name, kind, desc, extra)
    local d = FILE_DESC[name]
    kind = kind or (d and d[1]) or ""
    desc = desc or (d and d[2]) or ""
    local rec = FILE_SEEN[name]
    if rec then
        rec.kind, rec.desc = kind ~= "" and kind or rec.kind, desc ~= "" and desc or rec.desc
        rec.extra = extra or rec.extra
        return rec
    end
    rec = { file = name, kind = kind, desc = desc, extra = extra }
    FILE_SEEN[name] = rec
    FILES[#FILES + 1] = rec
    return rec
end

----------------------------------------------------------------------
-- utils
----------------------------------------------------------------------
local function safeCall(fn, ...)
    if type(fn) ~= "function" then return nil end
    local ok, a, b, c, d, e, f = pcall(fn, ...)
    if not ok then return nil end
    return a, b, c, d, e, f
end

local function now() return os.clock() end

local function mkdirp(p)
    if p == "." or p == "" then return true end
    if type(makefolder) ~= "function" then
        print("[!] makefolder unavailable — the subdir probe will decide the fallback")
        return false
    end
    local ok, err = pcall(makefolder, p)
    if not ok then print("[!] makefolder('" .. p .. "') failed: " .. tostring(err)) end
    return ok
end

local function outPath(name)
    if CONFIG.OUT_DIR == "." or CONFIG.OUT_DIR == "" then return name end
    return CONFIG.OUT_DIR .. "/" .. name
end

local function writeOut(name, data)
    if type(writefile) ~= "function" then
        print("[!] writefile missing — nothing can be saved")
        return false
    end
    local ok, err = pcall(writefile, outPath(name), data)
    if not ok then print("[!] write failed " .. name .. ": " .. tostring(err)) end
    return ok
end

-- write probe: prove the destination works before spending 30s walking the
-- DataModel. an executor that cannot create subdirs still writes to its root.
local function probeWrite(dir)
    if type(writefile) ~= "function" then return false end
    local path = (dir == "." or dir == "") and "_fd_probe.txt" or (dir .. "/_fd_probe.txt")
    local ok = pcall(writefile, path, "fulldump write probe")
    if not ok then return false end
    if type(delfile) == "function" then
        pcall(delfile, path)
    else
        -- no delfile: blank it so the probe leaves no readable junk behind
        pcall(writefile, path, "")
    end
    return true
end

local function resolveOutput()
    local want = CONFIG.OUT_DIR
    mkdirp(want)
    if probeWrite(want) then
        print("[+] output -> " .. outPath("<name>"))
        return want
    end
    if want ~= "." and probeWrite(".") then
        print("[!] cannot write into '" .. tostring(want) ..
              "' on this executor — falling back to the workspace root")
        CONFIG.OUT_DIR = "."
        print("[+] output -> " .. outPath("<name>"))
        return "."
    end
    error("fulldump: this executor cannot write files (writefile probe failed in '" ..
          tostring(want) .. "' and in the workspace root)")
end

-- nesting budget for the serializer. player records reach 8 levels
-- (players > tools > values > value), so this must stay well above that.
local SER_MAX_DEPTH = 12

local function nameset(list)
    local t = {}
    for _, v in ipairs(list) do t[v] = true end
    return t
end

local TN_VECTOR  = nameset({ "vector", "vector3", "Vector3" })
local TN_VECTOR3I = nameset({ "vector3int16", "Vector3int16" })
local TN_VECTOR2 = nameset({ "vector2", "Vector2" })
local TN_COLOR   = nameset({ "color3", "Color3" })
local TN_CFRAME  = nameset({ "CFrame", "cframe" })
local TN_UDIM    = nameset({ "UDim2", "udim2", "UDim", "udim" })

local function ser(v, depth, seen)
    depth = depth or 0
    seen = seen or {}
    local tv = type(v)
    if tv == "nil" or tv == "boolean" or tv == "number" then return v end
    if tv == "string" then
        if #v > 400 then return v:sub(1, 400) .. ("…(+%d)"):format(#v - 400) end
        return v
    end
    -- Roblox reports engine value types inconsistently across versions and
    -- executors: type() returns "vector"/"vector3"/"userdata" depending on the
    -- build, and typeof() returns "Vector3"/"Color3". normalize against both,
    -- otherwise every part position serializes as "<vector>" and the map is dead.
    local tn = tv
    if tv == "userdata" or tv == "table" then
        local ok, t = pcall(typeof, v)
        if ok and type(t) == "string" then tn = t end
    end
    if TN_VECTOR[tn] then return { t = "V3", x = v.X, y = v.Y, z = v.Z } end
    if TN_VECTOR3I[tn] then return { t = "V3i", x = v.X, y = v.Y, z = v.Z } end
    if TN_VECTOR2[tn] then return { t = "V2", x = v.X, y = v.Y } end
    if TN_COLOR[tn] then return { t = "C3", r = v.R, g = v.G, b = v.B } end
    if TN_CFRAME[tn] then
        local pos = safeCall(function() return v.Position end)
        local rot = safeCall(function()
            local rx, ry, rz = v:ToEulerAnglesYXZ()
            return { rx, ry, rz }
        end)
        return {
            t = "CF",
            p = pos and { pos.X, pos.Y, pos.Z } or nil,
            r = rot and { math.floor(rot[1] * 100) / 100, math.floor(rot[2] * 100) / 100,
                          math.floor(rot[3] * 100) / 100 } or nil,
        }
    end
    if TN_UDIM[tn] then
        return { t = "UDim", x = v.X and v.X.Scale or 0, xo = v.X and v.X.Offset or 0,
                 y = v.Y and v.Y.Scale or 0, yo = v.Y and v.Y.Offset or 0 }
    end
    if depth > SER_MAX_DEPTH then return "<maxdepth>" end
    if tv == "function" then return "<fn>" end
    if tn == "Instance" or tv == "userdata" then
        local ok, isInst = pcall(function() return v:IsA("Instance") end)
        if ok and isInst then return { t = "Inst", class = v.ClassName, path = safeCall(function() return v:GetFullName() end) } end
        return "<" .. (tn ~= tv and tn or tv) .. ">"
    end
    if tv == "table" then
        if seen[v] then return "<cycle>" end
        seen[v] = true
        local count, sequential = 0, true
        for k in pairs(v) do
            if type(k) ~= "number" or k % 1 ~= 0 then sequential = false break end
            count = count + 1
        end
        if sequential then
            for i = 1, count do
                if v[i] == nil then sequential = false break end
            end
        end
        if sequential and count > 0 then
            local arr = {}
            for i = 1, math.min(count, 150) do arr[i] = ser(v[i], depth + 1, seen) end
            seen[v] = nil
            return arr
        end
        local out, n = {}, 0
        for k, val in pairs(v) do
            n = n + 1
            if n > 150 then break end
            local key = type(k) == "string" and k or ("[" .. tostring(k) .. "]")
            out[key] = ser(val, depth + 1, seen)
        end
        seen[v] = nil
        return out
    end
    return "<" .. tv .. ">"
end

local function enc(v)
    local ok, s = pcall(function() return Http:JSONEncode(ser(v)) end)
    if ok and type(s) == "string" then return s end
    return "{\"__encodeError\":true}"
end

local function writeJson(name, v)
    registerFile(name)
    return writeOut(name, enc(v))
end

----------------------------------------------------------------------
-- streaming writer: appendfile when the runner has it, chunked parts when it doesn't
----------------------------------------------------------------------
local CHUNK_LINES = 5000

local function makeWriter(name, kind, desc)
    local w = {
        name = name, path = outPath(name),
        append = type(appendfile) == "function",
        buf = {}, chunks = 0, count = 0,
    }
    if w.append then
        local ok = pcall(appendfile, w.path, "")
        if not ok then
            -- never trust the api's presence: prove it writes, or buffer instead
            print("[!] appendfile unusable for " .. name .. " — buffering in memory")
            w.append = false
        end
    end
    function w:flushChunk()
        if #self.buf == 0 then return end
        self.chunks = self.chunks + 1
        -- first chunk keeps the plain name so index.html links never dangle
        local target = self.chunks == 1 and self.path
            or string.format("%s.part-%d", self.path, self.chunks)
        local blob = table.concat(self.buf, "\n") .. "\n"
        self.buf = {}
        if type(writefile) ~= "function" then
            print("[!] writefile missing — cannot flush " .. target)
            return
        end
        local ok, err = pcall(writefile, target, blob)
        if not ok then print("[!] chunk write failed " .. target .. ": " .. tostring(err)) end
    end
    function w:put(obj)
        self.count = self.count + 1
        local line = enc(obj)
        if self.append then
            pcall(appendfile, self.path, line .. "\n")
        else
            self.buf[#self.buf + 1] = line
            if #self.buf >= CHUNK_LINES then self:flushChunk() end
        end
    end
    function w:finish()
        if not self.append then
            -- an empty run still has to produce the file: index.html links it
            if self.chunks == 0 and type(writefile) == "function" then
                pcall(writefile, self.path, "")
            end
            self:flushChunk()
        end
        local note = nil
        if self.chunks > 1 then
            note = "buffered: " .. self.path .. " + " .. (self.chunks - 1) .. " .part-N file(s)"
        end
        registerFile(name, kind, desc, note)
        return self.count
    end
    return w
end

----------------------------------------------------------------------
-- function forensics
----------------------------------------------------------------------
local HAS_DEBUG_INFO = type(debug) == "table" and type(debug.info) == "function"
local HAS_CONSTANTS = type(debug) == "table" and type(debug.getconstants) == "function"
local HAS_UPVALUES = type(debug) == "table" and type(debug.getupvalues) == "function"
local HAS_CONNECTIONS = type(debug) == "table" and type(debug.getconnections) == "function"

local function funcInfo(fn)
    if type(fn) ~= "function" then return nil end
    local info = { addr = tostring(fn) }
    if HAS_DEBUG_INFO then
        pcall(function() info.src = debug.info(fn, "s") end)
        pcall(function() info.line = debug.info(fn, "l") end)
        pcall(function() info.name = debug.info(fn, "n") end)
    end
    if HAS_CONSTANTS then
        local consts = safeCall(function() return debug.getconstants(fn, true) end)
        if type(consts) == "table" and #consts > 0 then
            local keep = {}
            for i = 1, math.min(#consts, 200) do
                local c = consts[i]
                local ct = type(c)
                if ct == "string" then
                    if #c <= 160 then keep[#keep + 1] = c end
                elseif ct == "number" or ct == "boolean" then
                    keep[#keep + 1] = c
                end
            end
            if #keep > 0 then info.constants = keep end
        end
    end
    if HAS_UPVALUES then
        local ups = safeCall(function() return debug.getupvalues(fn) end)
        if type(ups) == "table" then
            local keep = {}
            for i = 1, math.min(#ups, 40) do
                local n, v = ups[i], ups[i + 1]
                if type(n) == "string" then
                    local vt = type(v)
                    if vt == "string" or vt == "number" or vt == "boolean" then
                        keep[#keep + 1] = { n = n, t = vt, v = v }
                    elseif vt == "table" then
                        keep[#keep + 1] = { n = n, t = "table", v = "<table>" }
                    end
                end
            end
            if #keep > 0 then info.upvalues = keep end
        end
    end
    return info
end

----------------------------------------------------------------------
-- shared state
----------------------------------------------------------------------
local S = {
    stats = {},
    names = {},
    nameCount = 0,
    values = {},
    scripts = {},
    remotes = {},
    modules = {},
    world = { models = {}, assets = {}, npcs = {}, spawns = {}, sounds = {}, anims = {}, parts = 0 },
    netlog = {},
    netCount = 0,
    shotCount = 0,
    seenNodes = {},        -- tree walk dedupe
    hookedRemotes = {},    -- remote hook bookkeeping (must not share with seenNodes)
    clientNetHooked = {},
    timings = {},
}

local function phase(name, fn)
    local t0 = now()
    print("[phase] " .. name .. " ...")
    local ok, err = pcall(fn)
    S.timings[name] = { seconds = now() - t0, ok = ok, err = ok and nil or tostring(err) }
    print(string.format("[phase] %s %s (%.2fs)", name, ok and "done" or ("FAILED: " .. tostring(err)), now() - t0))
    return ok
end

----------------------------------------------------------------------
-- classification
----------------------------------------------------------------------
local EXACT = {
    NumberValue = "value", IntValue = "value", StringValue = "value",
    BoolValue = "value", ObjectValue = "value",
    RemoteEvent = "remote", RemoteFunction = "remote", UnreliableRemoteEvent = "remote",
    Script = "script", LocalScript = "script", ModuleScript = "module",
    Humanoid = "humanoid", AnimationController = "anim",
    MeshPart = "mesh", UnionOperation = "mesh", SpawnLocation = "spawn",
    Decal = "decal", Texture = "texture", Sound = "sound",
    Workspace = "container", ReplicatedStorage = "container", ServerScriptService = "container",
    StarterPack = "container", StarterGui = "container", StarterPlayerScripts = "container",
}

local function classify(node)
    local cn = node.ClassName
    local hit = EXACT[cn]
    if hit then return hit end
    if node:IsA("BasePart") then return "part" end
    if node:IsA("Model") then return "model" end
    if node:IsA("Player") then return "player" end
    if node:IsA("Configuration") then return "config" end
    if node:IsA("Script") then return "script" end
    if node:IsA("GuiObject") then return "gui" end
    return "other"
end

local function noteName(name)
    if not name then return end
    local rec = S.names[name]
    if rec then
        rec.n = rec.n + 1
    elseif S.nameCount < CONFIG.NAME_MAX then
        S.names[name] = { n = 1 }
        S.nameCount = S.nameCount + 1
    end
end

----------------------------------------------------------------------
-- phase 1 — instance tree (streaming) + everything harvested in the same walk
----------------------------------------------------------------------
local function walkTree()
    local writer = makeWriter("01_tree.ndjson", "ndjson", "one JSON object per instance (streaming)")
    local queue = { { node = game, path = "game" } }
    local head = 1
    local nodes, parts, meshes, truncated = 0, 0, 0, false
    local CS = safeCall(function() return game:GetService("CollectionService") end)

    writer:put({
        class = "DataModel", name = "game",
        gameId = game.GameId, placeId = game.PlaceId,
        jobId = safeCall(function() return game.JobId end),
        root = true,
    })

    while head <= #queue do
        if nodes >= CONFIG.NODE_BUDGET or #queue - head > CONFIG.MAX_QUEUE then
            truncated = true
            break
        end
        local item = queue[head]
        head = head + 1
        local node, path = item.node, item.path
        if not S.seenNodes[node] then
            S.seenNodes[node] = true
            local kind = classify(node)
            local rec = { class = node.ClassName, name = node.Name, path = path, kind = kind }

            local attrs = safeCall(function() return node:GetAttributes() end)
            if type(attrs) == "table" and next(attrs) ~= nil then rec.attrs = ser(attrs, 3) end
            if CS then
                local tg = safeCall(function() return CS:GetTags(node) end)
                if type(tg) == "table" and #tg > 0 then rec.tags = tg end
            end

            if kind == "value" then
                local val = safeCall(function() return node.Value end)
                local vt = type(val)
                if vt == "Instance" then
                    rec.value = { inst = val.ClassName .. "/" .. val.Name }
                else
                    rec.value = ser(val, 3)
                end
                if #S.values < CONFIG.VALUE_MAX then
                    S.values[#S.values + 1] = {
                        path = path, class = node.ClassName, name = node.Name,
                        value = rec.value, parent = node.Parent and node.Parent.Name,
                    }
                end
            elseif kind == "part" then
                parts = parts + 1
                S.world.parts = S.world.parts + 1
                if parts <= CONFIG.PART_DETAIL_MAX then
                    rec.size = ser(safeCall(function() return node.Size end), 2)
                    rec.pos = ser(safeCall(function() return node.Position end), 2)
                    rec.anchored = safeCall(function() return node.Anchored end)
                    rec.canCollide = safeCall(function() return node.CanCollide end)
                    rec.material = safeCall(function() return tostring(node.Material) end)
                end
            elseif kind == "remote" then
                local r = {
                    path = path, class = node.ClassName, name = node.Name,
                    parent = node.Parent and node.Parent.Name,
                }
                if #S.remotes < CONFIG.REMOTE_MAX then
                    r.signals = {}
                    for _, sig in ipairs({ "OnServerEvent", "OnClientEvent" }) do
                        local s = safeCall(function() return node[sig] end)
                        if s then r.signals[#r.signals + 1] = sig end
                    end
                    S.remotes[#S.remotes + 1] = r
                end
            elseif kind == "script" or kind == "module" then
                local sc = { path = path, class = node.ClassName, name = node.Name }
                sc.enabled = safeCall(function() return node.Enabled end)
                sc.disabled = safeCall(function() return node.Disabled end)
                sc.runContext = safeCall(function() return tostring(node.RunContext) end)
                if kind == "module" and #S.modules < 4000 then
                    S.modules[#S.modules + 1] = sc
                end
                if #S.scripts < CONFIG.SCRIPT_MAX then
                    S.scripts[#S.scripts + 1] = sc
                end
            elseif kind == "humanoid" then
                local h = { path = path }
                h.health = safeCall(function() return node.Health end)
                h.maxHealth = safeCall(function() return node.MaxHealth end)
                h.walkSpeed = safeCall(function() return node.WalkSpeed end)
                h.jumpPower = safeCall(function() return node.JumpPower end)
                h.displayDistance = safeCall(function() return node.DisplayDistance end)
                h.healthDisplayDistance = safeCall(function() return node.HealthDisplayDistance end)
                h.state = safeCall(function() return tostring(node:GetState()) end)
                if #S.world.npcs < CONFIG.WORLD_MAX then
                    S.world.npcs[#S.world.npcs + 1] = h
                end
            elseif kind == "model" then
                if #S.world.models < CONFIG.WORLD_MAX then
                    S.world.models[#S.world.models + 1] = {
                        path = path, name = node.Name,
                        parts = safeCall(function() return #node:GetDescendants() end),
                    }
                end
            elseif kind == "mesh" or kind == "decal" or kind == "texture" then
                local id = safeCall(function() return node.TextureId end)
                    or safeCall(function() return node.MeshId end)
                    or safeCall(function() return node.Texture end)
                if id and #id > 3 and #S.world.assets < CONFIG.WORLD_MAX then
                    S.world.assets[#S.world.assets + 1] = { path = path, class = node.ClassName, id = tostring(id) }
                end
                if kind == "mesh" then
                    -- meshes are the visible geometry: they need geometry data too
                    meshes = meshes + 1
                    S.world.parts = S.world.parts + 1
                    if meshes <= CONFIG.PART_DETAIL_MAX then
                        rec.size = ser(safeCall(function() return node.Size end), 2)
                        rec.pos = ser(safeCall(function() return node.Position end), 2)
                        rec.anchored = safeCall(function() return node.Anchored end)
                        rec.canCollide = safeCall(function() return node.CanCollide end)
                    end
                end
            elseif kind == "spawn" then
                -- SpawnLocation is physical geometry: record it, not just the spawn config
                parts = parts + 1
                S.world.parts = S.world.parts + 1
                if parts <= CONFIG.PART_DETAIL_MAX then
                    rec.size = ser(safeCall(function() return node.Size end), 2)
                    rec.pos = ser(safeCall(function() return node.Position end), 2)
                    rec.anchored = safeCall(function() return node.Anchored end)
                    rec.canCollide = safeCall(function() return node.CanCollide end)
                end
                if #S.world.spawns < 200 then
                    S.world.spawns[#S.world.spawns + 1] = {
                        path = path,
                        teamColor = safeCall(function() return tostring(node.TeamColor) end),
                        duration = safeCall(function() return node.Duration end),
                        enabled = safeCall(function() return node.Enabled end),
                    }
                end
            elseif kind == "sound" then
                if #S.world.sounds < CONFIG.WORLD_MAX then
                    S.world.sounds[#S.world.sounds + 1] = {
                        path = path, name = node.Name,
                        soundId = safeCall(function() return tostring(node.SoundId) end),
                        playing = safeCall(function() return node.Playing end),
                    }
                end
            elseif kind == "anim" then
                if #S.world.anims < 2000 then
                    S.world.anims[#S.world.anims + 1] = {
                        path = path,
                        anims = safeCall(function() return #node:GetAnimationScripts() end),
                    }
                end
            end

            writer:put(rec)
            nodes = nodes + 1
            noteName(node.Name)
            if CONFIG.PROGRESS_EVERY > 0 and nodes % CONFIG.PROGRESS_EVERY == 0 then
                print(string.format("        ... %d nodes, queue %d", nodes, #queue - head + 1))
            end

            if kind == "player" then S.stats.players = (S.stats.players or 0) + 1 end

            local kids = safeCall(function() return node:GetChildren() end)
            if type(kids) == "table" then
                for _, kid in ipairs(kids) do
                    queue[#queue + 1] = { node = kid, path = path .. "." .. kid.Name }
                end
            end
        end
    end

    S.stats.nodes = nodes
    S.stats.parts = parts
    S.stats.meshes = meshes
    S.stats.queueRemaining = #queue - head + 1
    writer:finish()
    print(string.format("[tree] nodes=%d parts=%d meshes=%d truncated=%s leftoverQueue=%d",
        nodes, parts, meshes, tostring(truncated), #queue - head + 1))
end

----------------------------------------------------------------------
-- phase 2 — values (config tables live here)
----------------------------------------------------------------------
local function dumpValues()
    writeJson("02_values.json", {
        count = #S.values,
        note = "every Number/String/Bool/Int/ObjectValue found in the walk",
        values = S.values,
    })
    print("[values] " .. #S.values)
end

----------------------------------------------------------------------
-- phase 3 — remotes
----------------------------------------------------------------------
local function dumpRemotes()
    writeJson("03_remotes.json", {
        count = #S.remotes,
        capabilities = {
            hookmetamethod = type(hookmetamethod) == "function",
            hookfunction = type(hookfunction) == "function",
            getnamecallmethod = type(getnamecallmethod) == "function",
        },
        remotes = S.remotes,
    })
    print("[remotes] " .. #S.remotes)
end

----------------------------------------------------------------------
-- phase 4 — scripts
----------------------------------------------------------------------
local function dumpScripts()
    writeJson("04_scripts.json", {
        count = #S.scripts,
        debugInfo = HAS_DEBUG_INFO,
        scripts = S.scripts,
    })
    print("[scripts] " .. #S.scripts)
end

----------------------------------------------------------------------
-- phase 5 — modules (require, time budgeted)
----------------------------------------------------------------------
local function harvestModule(sc)
    local mod = nil
    -- walk the recorded path instead of trusting a stored instance reference
    for chunk in sc.path:gmatch("[^.]+") do
        if chunk == "game" then mod = game
        elseif mod then mod = safeCall(function() return mod:FindFirstChild(chunk, true) end) end
        if not mod then return { path = sc.path, name = sc.name, required = false, error = "not found" } end
    end
    local out = { path = sc.path, name = sc.name, class = sc.ClassName }
    local exported, err = nil, nil
    if type(require) == "function" then
        local ok
        ok, exported = pcall(require, mod)
        if not ok then err = tostring(exported); exported = nil end
    else
        err = "require unavailable"
    end
    if exported == nil then
        out.required = false
        out.error = err
        return out
    end
    out.required = true
    out.exports = {}
    local function addExports(tbl, target, depth)
        if type(tbl) ~= "table" or depth > 1 then return end
        for k, v in pairs(tbl) do
            if #target.exports >= 80 then return end
            local key = type(k) == "string" and k or ("[" .. tostring(k) .. "]")
            local vt = type(v)
            if vt == "function" then
                target.exports[#target.exports + 1] = { key = key, t = "fn", info = funcInfo(v) }
            elseif vt == "table" then
                local sub = { key = key, t = "table", fields = {} }
                local n = 0
                for k2, v2 in pairs(v) do
                    n = n + 1
                    if n > 60 then break end
                    local kk = type(k2) == "string" and k2 or ("[" .. tostring(k2) .. "]")
                    if type(v2) == "function" then
                        sub.fields[#sub.fields + 1] = { key = kk, t = "fn", info = funcInfo(v2) }
                    else
                        sub.fields[#sub.fields + 1] = { key = kk, t = type(v2), v = ser(v2, 2) }
                    end
                end
                target.exports[#target.exports + 1] = sub
            else
                target.exports[#target.exports + 1] = { key = key, t = vt, v = ser(v, 2) }
            end
        end
    end
    addExports(exported, out, 0)
    return out
end

local function dumpModules()
    if not CONFIG.REQUIRE_MODULES then
        print("[modules] skipped (REQUIRE_MODULES = false)")
        return
    end
    local t0 = now()
    local results, failed, stopped = {}, {}, false
    for i, sc in ipairs(S.modules) do
        if now() - t0 > CONFIG.MODULE_TIME_BUDGET then
            stopped = true
            print(string.format("[modules] time budget hit at %d/%d", i - 1, #S.modules))
            break
        end
        local r = harvestModule(sc)
        if r.required then
            results[#results + 1] = r
        else
            failed[#failed + 1] = r
        end
    end
    writeJson("05_modules.json", {
        harvested = #results,
        failed = #failed,
        stoppedByBudget = stopped,
        budgetSeconds = CONFIG.MODULE_TIME_BUDGET,
        modules = results,
        failures = failed,
    })
    print("[modules] harvested=" .. #results .. " failed=" .. #failed)
end

----------------------------------------------------------------------
-- phase 6/7 — netlog + summary
----------------------------------------------------------------------
local function hasKeyword(text, kw)
    if type(text) ~= "string" then return false end
    local hay = string.lower(text)
    for chunk in string.lower(tostring(kw)):gmatch("[^|]+") do
        if string.find(hay, chunk, 1, true) then return true end
    end
    return false
end

local function pushNet(entry)
    S.netCount = S.netCount + 1
    entry.id = S.netCount
    entry.t = now()
    entry.clock = os.time()
    if entry.dir == "out" then
        local plr = Players.LocalPlayer
        local tool = safeCall(function()
            return plr.Character and plr.Character:FindFirstChildOfClass("Tool")
        end)
        if tool then entry.tool = tool.Name end
        if hasKeyword(entry.remote, "fire|shoot|shot|ammo|clip|reload|weapon|gun|attack") then
            S.shotCount = S.shotCount + 1
            entry.shot = S.shotCount
        end
    end
    -- block trim: shifting element 1 on every event is O(n) per remote call and
    -- stalls the client during sustained fire
    if #S.netlog >= CONFIG.NETLOG_MAX then
        local drop = math.min(1000, #S.netlog)
        for _ = 1, drop do table.remove(S.netlog, 1) end
    end
    S.netlog[#S.netlog + 1] = entry
end

local function hookRemote(inst)
    if S.hookedRemotes[inst] then return end
    S.hookedRemotes[inst] = true
    if type(hookmetamethod) ~= "function" or type(getnamecallmethod) ~= "function" then return end
    pcall(function()
        hookmetamethod(inst, "__namecall", function(self, ...)
            local method = getnamecallmethod()
            local args = table.pack(...)
            if method == "FireServer" or method == "InvokeServer"
                or method == "FireAllClients" or method == "FireClient"
                or method == "InvokeClient" or method == "FireAll" then
                local head = {}
                for i = 1, math.min(args.n, 5) do head[i] = args[i] end
                pcall(pushNet, {
                    dir = "out", remote = inst.Name, class = inst.ClassName,
                    method = method, argc = args.n, args = ser(head, 4),
                })
            end
            return self[method](self, table.unpack(args, 1, args.n))
        end)
    end)
end

local function hookClientNet(player)
    if S.clientNetHooked[player] then return end
    S.clientNetHooked[player] = true
    pcall(function()
        player.OnClientEvent:Connect(function(...)
            local args = table.pack(...)
            local head = {}
            for i = 1, math.min(args.n, 5) do head[i] = args[i] end
            pcall(pushNet, {
                dir = "in", remote = "OnClientEvent", class = "RBXScriptSignal",
                method = "Fire", argc = args.n, from = player.Name, args = ser(head, 4),
            })
        end)
    end)
end

local function hookAllRemotes()
    for _, r in ipairs(S.remotes) do
        local inst = nil
        for chunk in r.path:gmatch("[^.]+") do
            inst = (inst == nil) and game or safeCall(function() return inst:FindFirstChild(chunk, true) end)
            if not inst then break end
        end
        if inst and (inst:IsA("RemoteEvent") or inst:IsA("RemoteFunction")) then
            hookRemote(inst)
        end
    end
    for _, p in ipairs(Players:GetPlayers()) do hookClientNet(p) end
end

local function sniff(seconds)
    seconds = tonumber(seconds) or CONFIG.NET_SECONDS
    hookAllRemotes()
    pcall(function() Players.PlayerAdded:Connect(hookClientNet) end)
    print("[sniff] listening " .. tostring(seconds) .. "s — fire, reload, move, open menus")
    local t0 = now()
    local lastPrint = t0
    while now() - t0 < seconds do
        task.wait(1)
        if now() - lastPrint >= 5 then
            lastPrint = now()
            local left = math.max(0, seconds - (now() - t0))
            print("[sniff] " .. tostring(math.floor(left + 0.5)) .. "s left, events="
                .. tostring(S.netCount) .. " shots=" .. tostring(S.shotCount))
        end
    end
    return S.netCount
end

-- summary per remote+method. one pass only: the old version re-scanned the whole
-- netlog per group (O(groups x events)) and mixed bursts into one mean.
local BURST_GAP_MS = 250   -- same rule ballistics.lua uses
local MAX_BURST_ROWS = 10

local function round2(x) return math.floor(x * 100 + 0.5) / 100 end

local function intervalStats(dts)
    if #dts == 0 then return { count = 0 } end
    local sorted = table.clone(dts)
    table.sort(sorted)
    local mid = math.ceil(#sorted / 2)
    local median = sorted[mid]
    local sum, mn = 0, math.huge
    for _, d in ipairs(dts) do sum = sum + d; if d < mn then mn = d end end
    return {
        count = #dts,
        mean = round2(sum / #dts),
        median = round2(median),
        min = round2(mn),
        rpmFromMedian = median > 0 and math.floor(60000 / median) or nil,
        rpmFromFastest = mn > 0 and math.floor(60000 / mn) or nil,
    }
end

local function dumpNetlog()
    local writer = makeWriter("06_netlog.ndjson", "ndjson", "one JSON object per remote call")
    for _, e in ipairs(S.netlog) do writer:put(e) end
    writer:finish()

    local groups, order = {}, {}
    for _, e in ipairs(S.netlog) do
        local key = (e.remote or "?") .. "|" .. (e.method or "?") .. "|" .. tostring(e.dir)
        local g = groups[key]
        if not g then
            g = { remote = e.remote, method = e.method, dir = e.dir, count = 0,
                  ts = {}, tools = {}, players = {}, argc = {}, samples = {} }
            groups[key] = g
            order[#order + 1] = key
        end
        g.count = g.count + 1
        g.ts[#g.ts + 1] = e.t
        g.argc[tostring(e.argc or 0)] = (g.argc[tostring(e.argc or 0)] or 0) + 1
        if e.tool then g.tools[e.tool] = true end
        if e.from then g.players[e.from] = true end
        if #g.samples < 3 then g.samples[#g.samples + 1] = e.args end
    end

    for _, key in ipairs(order) do
        local g = groups[key]
        local dts, bursts = {}, {}
        if #g.ts > 0 then bursts[1] = { from = 1, to = 1 } end
        for i = 2, #g.ts do
            local d = (g.ts[i] - g.ts[i - 1]) * 1000
            dts[#dts + 1] = d
            local cur = bursts[#bursts]
            if d <= BURST_GAP_MS then
                cur.to = i
            else
                cur.to = i - 1
                bursts[#bursts + 1] = { from = i, to = i }
            end
        end

        g.burstCount = #bursts
        g.overall = intervalStats(dts)
        g.longestBurst = 0
        for _, b in ipairs(bursts) do
            if (b.to - b.from + 1) > g.longestBurst then g.longestBurst = b.to - b.from + 1 end
        end
        g.bursts = {}
        for i, b in ipairs(bursts) do
            if i > MAX_BURST_ROWS then break end
            if (b.to - b.from + 1) >= 2 then
                local bd = {}
                for j = b.from + 1, b.to do bd[#bd + 1] = (g.ts[j] - g.ts[j - 1]) * 1000 end
                g.bursts[#g.bursts + 1] = { index = i, shots = b.to - b.from + 1, stats = intervalStats(bd) }
            end
        end

        local toolList, playerList = {}, {}
        for k in pairs(g.tools) do toolList[#toolList + 1] = k end
        for k in pairs(g.players) do playerList[#playerList + 1] = k end
        table.sort(toolList)
        table.sort(playerList)
        g.tools = toolList
        g.players = playerList
        g.ts = nil
    end

    local summary = {}
    for _, key in ipairs(order) do summary[#summary + 1] = groups[key] end
    table.sort(summary, function(a, b) return a.count > b.count end)

    writeJson("07_net_summary.json", {
        events = S.netCount,
        shots = S.shotCount,
        burstGapMs = BURST_GAP_MS,
        hookedRemotes = (function()
            local n = 0
            for _ in pairs(S.hookedRemotes) do n = n + 1 end
            return n
        end)(),
        groups = summary,
    })
    print("[net] events=" .. S.netCount .. " shots=" .. S.shotCount .. " groups=" .. #summary)
end

----------------------------------------------------------------------
-- phase 8 — connections
----------------------------------------------------------------------
local function dumpConnections()
    if not HAS_CONNECTIONS then
        writeJson("08_connections.json", {
            available = false,
            reason = "debug.getconnections missing in this executor",
        })
        print("[connections] unavailable (debug.getconnections missing)")
        return
    end
    local rows = {}
    local function scan(signal, owner)
        if type(signal) ~= "RBXScriptSignal" then return end
        if #rows >= CONFIG.CONNECTION_LIMIT then return end
        local ok, conns = pcall(debug.getconnections, signal)
        if not ok or type(conns) ~= "table" then return end
        for _, c in ipairs(conns) do
            if #rows >= CONFIG.CONNECTION_LIMIT then return end
            rows[#rows + 1] = {
                owner = owner,
                key = safeCall(function() return c.Key end) or "?",
                fn = safeCall(function() return tostring(c.Function) end),
                enabled = safeCall(function() return c.Enabled end),
                src = safeCall(function() return debug.info(c.Function, "s") end),
                line = safeCall(function() return debug.info(c.Function, "l") end),
                info = funcInfo(c.Function),
            }
        end
    end
    for _, r in ipairs(S.remotes) do
        local inst = nil
        for chunk in r.path:gmatch("[^.]+") do
            inst = (inst == nil) and game or safeCall(function() return inst:FindFirstChild(chunk, true) end)
            if not inst then break end
        end
        if inst then
            scan(safeCall(function() return inst.OnServerEvent end), r.path)
            scan(safeCall(function() return inst.OnClientEvent end), r.path)
        end
    end
    for _, p in ipairs(Players:GetPlayers()) do
        scan(safeCall(function() return p.OnClientEvent end), p.Name)
        local ch = safeCall(function() return p.Character end)
        local hum = ch and safeCall(function() return ch:FindFirstChildOfClass("Humanoid") end)
        if hum then
            scan(safeCall(function() return hum.Died end), p.Name .. "/Died")
            scan(safeCall(function() return hum.StateChanged end), p.Name .. "/StateChanged")
        end
    end
    for _, name in ipairs({ "CharacterAdded", "CharacterRemoving" }) do
        local p = Players.LocalPlayer
        scan(safeCall(function() return p[name] end), "LocalPlayer/" .. name)
    end
    writeJson("08_connections.json", { available = true, count = #rows, connections = rows })
    print("[connections] " .. #rows)
end

----------------------------------------------------------------------
-- phase 9 — gc
----------------------------------------------------------------------
local function dumpGc()
    local rows, seen = {}, {}
    if type(getgc) == "function" then
        local ok, gc = pcall(getgc, true)
        if ok and type(gc) == "table" then
            for _, f in ipairs(gc) do
                if type(f) == "function" and #rows < CONFIG.GC_LIMIT then
                    local addr = tostring(f)
                    if not seen[addr] then
                        seen[addr] = true
                        local info = funcInfo(f)
                        if info then rows[#rows + 1] = info end
                    end
                end
            end
        end
    end
    writeJson("09_gc.json", { count = #rows, functions = rows })
    print("[gc] " .. #rows)
end

----------------------------------------------------------------------
-- phase 10 — env
----------------------------------------------------------------------
local BASE = {}
do
    local base = {
        "game", "workspace", "script", "shared", "print", "warn", "error", "coroutine", "debug",
        "math", "string", "table", "task", "utf8", "bit32", "os", "io", "pcall", "xpcall",
        "select", "type", "next", "pairs", "ipairs", "rawget", "rawset", "setmetatable",
        "getmetatable", "tick", "time", "wait", "spawn", "delay", "elapsedTime", "loadstring",
        "require", "getgenv", "getgc", "getinstances", "getnilinstances", "hookfunction",
        "hookmetamethod", "newcclosure", "checkcaller", "getnamecallmethod", "getcallingscript",
        "identifyexecutor", "getexecutorname", "setclipboard", "userinput", "Fulldump",
        "FulldumpConfig", "gethui", "protect_gui", "Drawing", "CFrame", "Vector3", "Vector2",
        "Color3", "UDim", "UDim2", "RaycastParams", "Region3", "BrickColor", "TweenInfo",
        "NumberRange", "NumberSequence", "ColorSequence", "Rect", "Font", "Enum", "Random",
    }
    for _, k in ipairs(base) do BASE[k] = true end
end

local function envRows(tbl)
    local out, n = {}, 0
    if type(tbl) ~= "table" then return out end
    for k, v in pairs(tbl) do
        n = n + 1
        if n > 2000 then break end
        local key = type(k) == "string" and k or ("[" .. tostring(k) .. "]")
        if not BASE[key] then
            local vt = type(v)
            local row = { key = key, t = vt }
            if vt == "function" then
                row.info = funcInfo(v)
            elseif vt == "string" or vt == "number" or vt == "boolean" then
                row.v = ser(v)
            elseif vt == "table" then
                row.fields = {}
                local m = 0
                for k2, v2 in pairs(v) do
                    m = m + 1
                    if m > 80 then break end
                    local kk = type(k2) == "string" and k2 or ("[" .. tostring(k2) .. "]")
                    if type(v2) == "function" then
                        row.fields[#row.fields + 1] = { key = kk, t = "fn", info = funcInfo(v2) }
                    else
                        row.fields[#row.fields + 1] = { key = kk, t = type(v2), v = ser(v2, 2) }
                    end
                end
            end
            out[#out + 1] = row
        end
    end
    table.sort(out, function(a, b) return a.key < b.key end)
    return out
end

local function dumpEnv()
    writeJson("10_env.json", {
        _G = envRows(_G),
        genv = (type(getgenv) == "function") and envRows(getgenv()) or nil,
        shared = (type(shared) == "table") and envRows(shared) or nil,
        debugInfo = HAS_DEBUG_INFO,
        constants = HAS_CONSTANTS,
        upvalues = HAS_UPVALUES,
        connections = HAS_CONNECTIONS,
    })
    print("[env] done")
end

----------------------------------------------------------------------
-- phase 11 — players
----------------------------------------------------------------------
local function guiTree(root, depth, budget)
    if not root or depth > CONFIG.GUI_MAX_DEPTH then return nil end
    local entry = { name = root.Name, class = root.ClassName }
    local kids = safeCall(function() return root:GetChildren() end)
    if type(kids) == "table" then
        local kidsOut = {}
        for _, kid in ipairs(kids) do
            budget.n = budget.n - 1
            if budget.n <= 0 then break end
            local sub = guiTree(kid, depth + 1, budget)
            if sub then kidsOut[#kidsOut + 1] = sub end
        end
        if #kidsOut > 0 then entry.children = kidsOut end
    end
    return entry
end

local function dumpPlayers()
    local list, n = {}, 0
    for _, p in ipairs(Players:GetPlayers()) do
        n = n + 1
        if n > 80 then break end
        local row = { name = p.Name, displayName = p.DisplayName, userId = p.UserId }
        row.team = safeCall(function() return p.Team and p.Team.Name end)
        row.attrs = safeCall(function() return ser(p:GetAttributes(), 3) end)
        local char = safeCall(function() return p.Character end)
        if char then
            local hum = safeCall(function() return char:FindFirstChildOfClass("Humanoid") end)
            if hum then
                row.humanoid = {
                    health = safeCall(function() return hum.Health end),
                    maxHealth = safeCall(function() return hum.MaxHealth end),
                    walkSpeed = safeCall(function() return hum.WalkSpeed end),
                    jumpPower = safeCall(function() return hum.JumpPower end),
                    state = safeCall(function() return tostring(hum:GetState()) end),
                    displayDistance = safeCall(function() return hum.DisplayDistance end),
                }
            end
            local root = safeCall(function() return char:FindFirstChild("HumanoidRootPart") end)
            if root then
                row.root = {
                    pos = safeCall(function() return ser(root.Position, 2) end),
                    vel = safeCall(function() return ser(root.AssemblyLinearVelocity, 2) end),
                    cf = safeCall(function() return ser(root.CFrame, 2) end),
                }
            end
            row.tools = {}
            for _, d in ipairs(char:GetChildren()) do
                if d:IsA("Tool") then
                    local t = { name = d.Name, toolTip = safeCall(function() return d.ToolTip end),
                                requiresHandle = safeCall(function() return d.RequiresHandle end),
                                manualOnly = safeCall(function() return d.ManualActivationOnly end),
                                attrs = safeCall(function() return ser(d:GetAttributes(), 3) end),
                                values = {} }
                    for _, v in ipairs(d:GetChildren()) do
                        if v:IsA("IntValue") or v:IsA("NumberValue") or v:IsA("StringValue") or v:IsA("BoolValue") then
                            t.values[#t.values + 1] = { name = v.Name, value = safeCall(function() return ser(v.Value, 2) end) }
                        end
                    end
                    row.tools[#row.tools + 1] = t
                end
            end
            local backpack = safeCall(function() return p:FindFirstChildOfClass("Backpack") end)
            if backpack then
                row.backpack = {}
                for _, t in ipairs(backpack:GetChildren()) do
                    if t:IsA("Tool") then row.backpack[#row.backpack + 1] = t.Name end
                end
            end
        end
        local gui = safeCall(function() return p:FindFirstChild("PlayerGui") end)
        if gui then row.playerGui = guiTree(gui, 0, { n = 400 }) end
        list[#list + 1] = row
    end
    writeJson("11_players.json", { count = #list, players = list })
    print("[players] " .. #list)
end

----------------------------------------------------------------------
-- phase 12 — world
----------------------------------------------------------------------
local function dumpWorld()
    local Lighting = safeCall(function() return game:GetService("Lighting") end)
    local lighting = {}
    if Lighting then
        lighting.brightness = safeCall(function() return Lighting.Brightness end)
        lighting.clockTime = safeCall(function() return Lighting.ClockTime end)
        lighting.ambient = safeCall(function() return ser(Lighting.Ambient, 2) end)
        lighting.outdoorAmbient = safeCall(function() return ser(Lighting.OutdoorAmbient, 2) end)
        lighting.globalShadows = safeCall(function() return Lighting.GlobalShadows end)
        lighting.atmosphere = safeCall(function()
            local a = Lighting:FindFirstChildOfClass("Atmosphere")
            if not a then return nil end
            return { density = a.Density, haze = a.Haze, color = ser(a.Color, 2), name = a.Name }
        end)
        lighting.postEffects = safeCall(function()
            local out = {}
            for _, c in ipairs(Lighting:GetChildren()) do
                if c:IsA("PostEffect") or c:IsA("BloomEffect") or c:IsA("ColorCorrectionEffect")
                    or c:IsA("SunRaysEffect") or c:IsA("BlurEffect") or c:IsA("DepthOfFieldEffect") then
                    out[#out + 1] = { name = c.Name, class = c.ClassName, enabled = safeCall(function() return c.Enabled end) }
                end
            end
            return out
        end)
    end
    local terrain = safeCall(function()
        local t = Workspace:FindFirstChild("Terrain") or safeCall(function() return Workspace.Terrain end)
        if not t then return nil end
        return { size = ser(t.Size, 2), waterSize = ser(t.WaterSize, 2) }
    end)
    writeJson("12_world.json", {
        parts = S.world.parts,
        models = S.world.models,
        assets = S.world.assets,
        npcs = S.world.npcs,
        spawns = S.world.spawns,
        sounds = S.world.sounds,
        animationControllers = S.world.anims,
        terrain = terrain,
        lighting = lighting,
    })
    print("[world] models=" .. #S.world.models .. " assets=" .. #S.world.assets)
end

----------------------------------------------------------------------
-- names.txt
----------------------------------------------------------------------
local function dumpNames()
    local names = {}
    for name, rec in pairs(S.names) do names[#names + 1] = name end
    table.sort(names)
    local lines = {}
    for _, n in ipairs(names) do
        lines[#lines + 1] = string.format("%s\t%d", n, S.names[n].n)
    end
    writeOut("names.txt", "# name\tcount\n" .. table.concat(lines, "\n") .. "\n")
    registerFile("names.txt", "txt", "every unique instance name with a count")
    print("[names] " .. #names)
end

----------------------------------------------------------------------
-- index.html — offline browser for the dump
----------------------------------------------------------------------
local function htmlEscape(s)
    local out = tostring(s):gsub("&", "&amp;"):gsub("<", "&lt;"):gsub(">", "&gt;"):gsub('"', "&quot;")
    return (out)
end

local function dumpIndex()
    if not CONFIG.HTML_INDEX then return end
    registerFile("index.html")
    local rows = {}
    for _, name in ipairs(FILE_ORDER) do
        local rec = FILE_SEEN[name]
        local d = FILE_DESC[name] or { "", "" }
        rows[#rows + 1] = string.format(
            '<tr><td><a href="%s">%s</a></td><td class="k">%s</td><td>%s</td><td class="k">%s</td></tr>',
            name, name, htmlEscape((rec and rec.kind) or d[1]),
            htmlEscape((rec and rec.desc) or d[2]), htmlEscape((rec and rec.extra) or ""))
    end
    local names = {}
    local count = 0
    for name in pairs(S.names) do
        count = count + 1
        if count > 6000 then break end
        names[#names + 1] = htmlEscape(name)
    end
    table.sort(names)
    local html = [[<!doctype html><html><head><meta charset="utf-8">
<title>fulldump</title><style>
body{background:#141414;color:#ddd;font:13px/1.5 Consolas,monospace;margin:0;padding:24px}
h1{font-size:18px;color:#8fd67a}table{border-collapse:collapse;width:100%;margin:12px 0 28px}
td,th{border:1px solid #333;padding:4px 8px;text-align:left}
a{color:#7ab8ff;text-decoration:none}.k{color:#888}
input{background:#0d0d0d;border:1px solid #444;color:#eee;padding:6px;width:320px}
pre{background:#0d0d0d;border:1px solid #333;padding:10px;max-height:60vh;overflow:auto}
</style></head><body>
<h1>fulldump</h1>
<input id="q" placeholder="filter file name..." oninput="f(this.value)">
<table id="files"><tr><th>file</th><th>kind</th><th>contents</th><th>note</th></tr>
]] .. table.concat(rows, "\n") .. [[
</table>
<h1>instance names</h1>
<input id="q2" placeholder="filter names..." oninput="n(this.value)">
<pre id="names">
]] .. table.concat(names, "\n") .. [[
</pre>
<script>
function f(v){v=v.toLowerCase();document.querySelectorAll('#files tr').forEach(function(r){
 if(r.cells[0]&&r.cells[0].textContent.toLowerCase().indexOf(v)<0)r.style.display='none';else r.style.display='';});}
function n(v){v=v.toLowerCase();var out=[];
 document.getElementById('names').textContent.split('\n').forEach(function(l){
  if(!v||l.toLowerCase().indexOf(v)>=0)out.push(l);});
 document.getElementById('names').textContent=out.join('\n');}
</script></body></html>]]
    writeOut("index.html", html)
    registerFile("index.html", "html", "offline browser for this dump")
end

----------------------------------------------------------------------
-- 00_meta + orchestration
----------------------------------------------------------------------
local function writeMeta(extra)
    registerFile("00_meta.json")
    local meta = {
        generated = os.date("%Y-%m-%d %H:%M:%S"),
        executor = safeCall(function() return identifyexecutor and identifyexecutor() end),
        gameId = game.GameId,
        placeId = game.PlaceId,
        jobId = safeCall(function() return game.JobId end),
        localPlayer = safeCall(function() return Players.LocalPlayer and Players.LocalPlayer.Name end),
        playerCount = safeCall(function() return #Players:GetPlayers() end),
        fps = safeCall(function() return math.floor(Workspace:GetFPS()) end),
        nodes = S.stats.nodes,
        parts = S.stats.parts,
        values = #S.values,
        scripts = #S.scripts,
        remotes = #S.remotes,
        names = S.nameCount,
        netEvents = S.netCount,
        shots = S.shotCount,
        capabilities = {
            hookmetamethod = type(hookmetamethod) == "function",
            hookfunction = type(hookfunction) == "function",
            getgc = type(getgc) == "function",
            getgenv = type(getgenv) == "function",
            appendfile = type(appendfile) == "function",
            writefile = type(writefile) == "function",
            debugInfo = HAS_DEBUG_INFO,
            constants = HAS_CONSTANTS,
            upvalues = HAS_UPVALUES,
            connections = HAS_CONNECTIONS,
        },
        config = CONFIG,
        phases = S.timings,
        files = FILE_ORDER,
    }
    if extra then
        for k, v in pairs(extra) do meta[k] = v end
    end
    writeJson("00_meta.json", meta)
    return meta
end

local function printSummary(meta)
    print("")
    print("=== fulldump done ===")
    print(string.format("nodes=%s values=%d scripts=%d remotes=%d names=%d netEvents=%d shots=%d",
        tostring(meta.nodes), meta.values, meta.scripts, meta.remotes,
        meta.names, meta.netEvents, meta.shots))
    local failed = {}
    for name, t in pairs(S.timings) do
        if not t.ok then failed[#failed + 1] = name .. " -> " .. tostring(t.err) end
    end
    table.sort(failed)
    if #failed > 0 then
        print("FAILED phases (" .. #failed .. "):")
        for _, f in ipairs(failed) do print("  ! " .. f) end
    else
        print("all phases ok")
    end
    print("files in: " .. outPath("<name>"))
    for _, name in ipairs(FILE_ORDER) do
        local rec = FILE_SEEN[name]
        local d = FILE_DESC[name] or { "", "" }
        print(string.format("  %-22s %-8s %s%s", name, (rec and rec.kind) or d[1],
            (rec and rec.desc) or d[2], (rec and rec.extra) and (" [" .. rec.extra .. "]") or ""))
    end
    if CONFIG.HTML_INDEX then print("open: " .. outPath("index.html")) end
end

local function run(opts)
    opts = opts or {}
    local t0 = now()
    resolveOutput()

    phase("tree", walkTree)
    phase("values", dumpValues)
    phase("remotes", dumpRemotes)
    phase("scripts", dumpScripts)
    if opts.require ~= false then phase("modules", dumpModules) end
    phase("connections", dumpConnections)
    phase("gc", dumpGc)
    phase("env", dumpEnv)
    phase("players", dumpPlayers)
    phase("world", dumpWorld)
    phase("names", dumpNames)

    if opts.sniff ~= false then
        phase("sniff", function() sniff(opts.seconds or CONFIG.NET_SECONDS) end)
    end
    phase("netlog", dumpNetlog)

    dumpIndex()
    local meta = writeMeta({ totalSeconds = now() - t0 })
    printSummary(meta)
    return meta
end

----------------------------------------------------------------------
-- public surface
----------------------------------------------------------------------
local Fulldump = {
    run = run,
    quick = function() return run({ sniff = false, require = false }) end,
    sniff = function(sec) return sniff(sec or CONFIG.NET_SECONDS) end,
    dumpNetlog = dumpNetlog,   -- re-dump the buffer after a session of manual play
    remotes = function() return S.remotes end,
    report = function()
        local ok, meta = pcall(function()
            return Http:JSONDecode(readfile(outPath("00_meta.json")))
        end)
        if ok and meta then print(enc(meta)) else print("no meta yet — run Fulldump.run()") end
    end,
    state = S,
    config = CONFIG,
}

if type(getgenv) == "function" then
    getgenv().Fulldump = Fulldump
else
    _G.Fulldump = Fulldump
end

print("[fulldump] ready — getgenv().Fulldump.run()  (or .quick() / .sniff(60))")
return Fulldump
