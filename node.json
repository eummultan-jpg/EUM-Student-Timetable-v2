gl.setup(NATIVE_WIDTH, NATIVE_HEIGHT)

local font = resource.load_font("font.ttf")
local bold = resource.load_font("font-bold.ttf")

local data = {}
local cfg = { family = "ALL", rotation_seconds = 12 }
local sections = {}
local section_index = 1
local rotation_started = sys.now()

local days = {"Sunday","Monday","Tuesday","Wednesday","Thursday","Friday","Saturday"}

local function clamp(v, lo, hi)
    if v < lo then return lo end
    if v > hi then return hi end
    return v
end

local function trim(s)
    s = tostring(s or "")
    return (s:gsub("^%s+", ""):gsub("%s+$", ""))
end

local function words(text)
    local out = {}
    for w in tostring(text or ""):gmatch("%S+") do out[#out + 1] = w end
    return out
end

local function fit(text, f, size, maxw)
    text = tostring(text or "")
    if f:width(text, size) <= maxw then return text end
    local s = ""
    for _, w in ipairs(words(text)) do
        local n = (s == "" and w or s .. " " .. w)
        if f:width(n .. "…", size) > maxw then break end
        s = n
    end
    if s ~= "" then return s .. "…" end
    while #text > 1 and f:width(text .. "…", size) > maxw do
        text = text:sub(1, -2)
    end
    return text .. "…"
end

local function parse_clock(part, suffix, evening_hint)
    local h, m = part:match("^(%d%d?):(%d%d)$")
    h, m = tonumber(h), tonumber(m)
    if not h or not m then return nil end
    suffix = (suffix or ""):lower()
    if suffix == "pm" then
        if h < 12 then h = h + 12 end
    elseif suffix == "am" then
        if h == 12 then h = 0 end
    elseif evening_hint and h < 8 then
        h = h + 12
    elseif h == 5 then
        -- Source PDFs contain a 05:40–06:30 Quran period in a daytime PDF.
        h = h + 12
    end
    return h * 60 + m
end

local function parse_range(s, evening_hint)
    s = tostring(s or ""):gsub("–", "-"):gsub("—", "-")
    local a, b, suf = s:match("^%s*(%d%d?:%d%d)%s*%-%s*(%d%d?:%d%d)%s*([ap]m)?%s*$")
    if not a then return nil end
    local st = parse_clock(a, suf, evening_hint)
    local en = parse_clock(b, suf, evening_hint)
    if not st or not en then return nil end
    return st, en
end

local function rebuild_sections()
    sections = {}
    for _, family in ipairs(data.families or {}) do
        if cfg.family == "ALL" or cfg.family == "" or cfg.family == family.name then
            for _, section in ipairs(family.sections or {}) do
                section.family = family.name
                sections[#sections + 1] = section
            end
        end
    end
    if #sections == 0 then
        for _, family in ipairs(data.families or {}) do
            for _, section in ipairs(family.sections or {}) do
                section.family = family.name
                sections[#sections + 1] = section
            end
        end
    end
    if #sections > 0 then
        section_index = clamp(section_index, 1, #sections)
    else
        section_index = 1
    end
    rotation_started = sys.now()
end

util.json_watch("timetable.json", function(v)
    data = v or {}
    rebuild_sections()
end)

util.json_watch("config.json", function(v)
    v = v or {}
    cfg.family = v.family or "ALL"
    cfg.rotation_seconds = tonumber(v.rotation_seconds or 12) or 12
    cfg.rotation_seconds = clamp(cfg.rotation_seconds, 5, 120)
    rebuild_sections()
end)

local function current_section()
    if #sections == 0 then return nil end
    if sys.now() - rotation_started >= cfg.rotation_seconds then
        section_index = section_index + 1
        if section_index > #sections then section_index = 1 end
        rotation_started = sys.now()
    end
    return sections[section_index]
end

local function today_index()
    return os.date("*t").wday - 1
end

local function day_events(section, di)
    local events = {}
    local evening = section.shift and section.shift:find("Evening") ~= nil
    for _, row in ipairs(section.rows or {}) do
        local cell = row.days and row.days[di]
        if cell then
            local is_break = cell["break"] == true
            if is_break then
                local s, e = parse_range(row.time, evening)
                events[#events + 1] = {
                    start=s, finish=e, time=row.time,
                    is_break=true, course="JUMMAH BREAK", instructor="", room=""
                }
            elseif cell.course then
                local s, e = parse_range(row.time, evening)
                events[#events + 1] = {
                    start=s, finish=e, time=row.time,
                    is_break=false, course=cell.course or "",
                    instructor=cell.instructor or "", room=cell.room or ""
                }
            end
        end
    end
    table.sort(events, function(a,b) return (a.start or 9999) < (b.start or 9999) end)
    return events
end

local function status_for(events)
    local t = os.date("*t")
    local now = t.hour * 60 + t.min
    local cur, nxt = nil, nil
    for _, e in ipairs(events) do
        if e.start and e.finish then
            if now >= e.start and now < e.finish then
                cur = e
            elseif e.start > now and not nxt then
                nxt = e
            end
        end
    end
    return cur, nxt
end

local function rect(x, y, w, h, r, g, b, a)
    gl.rect(x, y, x + w, y + h, r, g, b, a or 1)
end

local function line(x1, y1, x2, y2, w, r, g, b, a)
    gl.line(x1, y1, x2, y2, w, r, g, b, a or 1)
end

local function center(f, x, y, w, text, size, r, g, b, a)
    text = tostring(text or "")
    f:write(x + (w - f:width(text, size)) / 2, y, text, size, r, g, b, a or 1)
end

local function pill(x, y, w, h, label, bg, fg, f, size)
    rect(x, y, w, h, bg[1], bg[2], bg[3], bg[4] or 1)
    center(f, x, y + (h-size)*0.45, w, label, size, fg[1], fg[2], fg[3], fg[4] or 1)
end

local function draw_status_card(x, y, w, h, title, color, e, is_now)
    rect(x, y, w, h, 0.96, 0.98, 1.00, 1)
    rect(x, y, w, 62, color[1], color[2], color[3], 1)
    rect(x + 22, y + 16, 30, 30, 0.92, 0.96, 0.92, 1)
    bold:write(x + 70, y + 17, title, 28, 1, 1, 1, 1)
    if not e then
        bold:write(x + 30, y + 90, is_now and "NO CLASS NOW" or "NO MORE CLASSES", 27, 0.12, 0.20, 0.32, 1)
        return
    end
    if e.is_break then
        bold:write(x + 30, y + 92, "JUMMAH BREAK", 29, 0.90, 0.52, 0.10, 1)
    else
        bold:write(x + 30, y + 88, fit(e.course, bold, 30, w - 60), 30, 0.06, 0.12, 0.22, 1)
        font:write(x + 30, y + 132, fit(e.instructor, font, 21, w - 60), 21, 0.18, 0.27, 0.38, 1)
        font:write(x + 30, y + 168, "ROOM  " .. (e.room ~= "" and e.room or "—"), 20, 0.06, 0.32, 0.55, 1)
    end
    font:write(x + 30, y + h - 36, e.time or "", 20, 0.22, 0.30, 0.40, 1)
end

function node.render()
    local W, H = NATIVE_WIDTH, NATIVE_HEIGHT
    gl.clear(0.94, 0.96, 0.98, 1)

    local section = current_section()
    local t = os.date("*t")
    local day = days[t.wday]
    local date_str = os.date("%A\n%d %B %Y")
    local time_str = os.date("%I:%M:%S %p")

    -- ===== HEADER: white EUM branding + blue date/time panel =====
    rect(0, 0, W, 165, 0.98, 0.99, 1.00, 1)
    rect(0, 160, W, 7, 0.98, 0.73, 0.10, 1)

    -- Simple EUM crest-style mark (keeps package self-contained/offline).
    rect(35, 28, 105, 105, 0.04, 0.20, 0.40, 1)
    rect(43, 36, 89, 89, 0.98, 0.99, 1.00, 1)
    rect(51, 44, 73, 73, 0.04, 0.20, 0.40, 1)
    center(bold, 51, 61, 73, "EUM", 27, 1, 1, 1, 1)
    center(font, 51, 94, 73, "MULTAN", 11, 0.95, 0.78, 0.18, 1)

    bold:write(165, 31, "EMERSON UNIVERSITY MULTAN", 43, 0.03, 0.15, 0.30, 1)
    font:write(166, 88, "FACULTY OF COMPUTING & EMERGING TECHNOLOGIES", 25, 0.08, 0.24, 0.42, 1)

    local date_x = W - 570
    rect(date_x, 0, 570, 160, 0.04, 0.23, 0.46, 1)
    bold:write(date_x + 38, 27, day, 27, 1, 1, 1, 1)
    bold:write(date_x + 38, 61, os.date("%d %B %Y"), 25, 1, 1, 1, 1)
    line(date_x + 285, 27, date_x + 285, 132, 2, 0.55, 0.70, 0.88, 1)
    bold:write(date_x + 315, 44, time_str, 31, 1, 1, 1, 1)
    font:write(date_x + 315, 91, "LIVE CLOCK", 17, 0.70, 0.88, 1, 1)

    if not section then
        center(bold, 0, H/2, W, "Loading timetable...", 42, 0.03, 0.15, 0.30, 1)
        return
    end

    local di = today_index()
    local events = (di >= 1 and di <= 5) and day_events(section, di) or {}
    local cur, nxt = status_for(events)

    -- ===== PROGRAM TITLE =====
    bold:write(45, 195, fit(section.name or "", bold, 39, W - 650), 39, 0.03, 0.12, 0.24, 1)
    font:write(47, 241, (section.semester or "") .. "  •  " .. (section.shift or "") .. "  •  FALL 2026", 23, 0.13, 0.25, 0.39, 1)

    -- ===== LEFT TABLE =====
    local leftX, leftY, leftW = 35, 285, math.floor(W * 0.665)
    local rightX, rightW = leftX + leftW + 25, W - (leftX + leftW + 25) - 35

    bold:write(leftX + 10, 268, "TODAY'S CLASS TIMETABLE", 24, 0.05, 0.18, 0.34, 1)
    font:write(leftX + 455, 269, day, 22, 0.05, 0.45, 0.70, 1)

    rect(leftX, leftY, leftW, H - leftY - 105, 0.98, 0.99, 1.00, 1)
    rect(leftX, leftY, leftW, 58, 0.04, 0.24, 0.47, 1)

    local c1 = 205
    local c2 = math.floor(leftW * 0.39)
    local c3 = math.floor(leftW * 0.27)
    local c4 = leftW - c1 - c2 - c3
    local tableY = leftY + 58
    local availableH = H - tableY - 105
    local maxRows = math.min(#events, 7)
    local rowH = maxRows > 0 and math.floor(availableH / maxRows) or 70
    rowH = clamp(rowH, 62, 92)

    center(bold, leftX, leftY + 17, c1, "TIME", 22, 1, 1, 1, 1)
    center(bold, leftX + c1, leftY + 17, c2, "SUBJECT", 22, 1, 1, 1, 1)
    center(bold, leftX + c1 + c2, leftY + 17, c3, "INSTRUCTOR", 20, 1, 1, 1, 1)
    center(bold, leftX + c1 + c2 + c3, leftY + 17, c4, "ROOM", 20, 1, 1, 1, 1)

    if #events == 0 then
        center(bold, leftX, tableY + 75, leftW, "NO SCHEDULED CLASSES TODAY", 29, 0.22, 0.31, 0.42, 1)
    else
        for i = 1, maxRows do
            local e = events[i]
            local y = tableY + (i-1) * rowH
            local is_cur = (cur == e)
            local is_nxt = (nxt == e)
            local bg = (i % 2 == 0) and {0.95,0.97,0.99} or {0.99,0.995,1.0}
            if is_cur then bg = {1.0, 0.94, 0.68}
            elseif is_nxt then bg = {0.91, 0.96, 1.0} end
            if e.is_break then bg = {0.90,0.92,0.95} end
            rect(leftX, y, leftW, rowH, bg[1], bg[2], bg[3], 1)
            if is_cur then rect(leftX, y, 7, rowH, 0.98, 0.72, 0.10, 1)
            elseif is_nxt then rect(leftX, y, 7, rowH, 0.05, 0.43, 0.76, 1)
            elseif e.is_break then rect(leftX, y, 7, rowH, 0.90, 0.53, 0.12, 1) end

            line(leftX, y + rowH, leftX + leftW, y + rowH, 1, 0.78, 0.84, 0.90, 1)
            line(leftX + c1, y, leftX + c1, y + rowH, 1, 0.80, 0.86, 0.92, 1)
            line(leftX + c1 + c2, y, leftX + c1 + c2, y + rowH, 1, 0.80, 0.86, 0.92, 1)
            line(leftX + c1 + c2 + c3, y, leftX + c1 + c2 + c3, y + rowH, 1, 0.80, 0.86, 0.92, 1)

            center(font, leftX, y + rowH/2 - 10, c1, e.time, 20, 0.05, 0.12, 0.22, 1)
            if e.is_break then
                center(bold, leftX + c1, y + rowH/2 - 12, leftW - c1, "JUMMAH BREAK", 23, 0.75, 0.42, 0.08, 1)
            else
                bold:write(leftX + c1 + 14, y + 14, fit(e.course, bold, 20, c2 - 28), 20, 0.04, 0.12, 0.22, 1)
                font:write(leftX + c1 + c2 + 12, y + 14, fit(e.instructor, font, 17, c3 - 24), 17, 0.08, 0.17, 0.28, 1)
                center(font, leftX + c1 + c2 + c3, y + rowH/2 - 9, c4, fit(e.room, font, 17, c4 - 12), 17, 0.05, 0.27, 0.48, 1)
            end
        end
        if #events > maxRows then
            font:write(leftX + 15, H - 124, "+ " .. tostring(#events - maxRows) .. " more class(es) — rotating timetable", 15, 0.32, 0.42, 0.53, 1)
        end
    end

    -- ===== RIGHT NOW / NEXT =====
    draw_status_card(rightX, 285, rightW, 235, "NOW  •  CURRENT CLASS", {0.02,0.48,0.29}, cur, true)
    draw_status_card(rightX, 535, rightW, 235, "NEXT  •  UPCOMING CLASS", {0.03,0.38,0.72}, nxt, false)

    -- ===== FOOTER / ROTATION =====
    rect(0, H - 82, W, 82, 0.02, 0.16, 0.32, 1)
    bold:write(35, H - 57, "UPCOMING SECTIONS", 21, 1, 1, 1, 1)
    local bx = 260
    local shown = math.min(#sections, 6)
    for i = 1, shown do
        local idx = section_index + i - 1
        while idx > #sections do idx = idx - #sections end
        local s = sections[idx]
        local label = fit(s.id or s.name or "", bold, 17, 120)
        local bw = 135
        local active = (idx == section_index)
        pill(bx, H - 66, bw, 45, label,
            active and {0.99,0.72,0.10,1} or {0.05,0.25,0.45,1},
            active and {0.03,0.12,0.22,1} or {1,1,1,1}, bold, 17)
        bx = bx + bw + 12
    end
    font:write(W - 285, H - 57, "POWERED BY IT CENTER", 18, 1, 1, 1, 1)
end
