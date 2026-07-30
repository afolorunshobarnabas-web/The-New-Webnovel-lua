-- Minimal m.webnovel.com mobile scraper
-- Requires: LuaSocket/luasec (ssl.https), ltn12, and a JSON library (dkjson or cjson)
-- Usage:
--   local mw = require("m_webnovel")
--   local info = mw.get_book_info("https://m.webnovel.com/book/12345")
--   local chapters = mw.get_chapter_list_from_book_url("https://m.webnovel.com/book/12345")
--   local body = mw.get_chapter_body("https://m.webnovel.com/book/12345/67890")

local https = require("ssl.https")
local ltn12 = require("ltn12")
local json -- will try dkjson then cjson
local ok, dkjson = pcall(require, "dkjson")
if ok then json = dkjson end
if not json then
  ok, json = pcall(require, "cjson")
  if not ok then
    error("Please install dkjson or cjson for JSON parsing")
  end
end

local M = {}

local default_headers = {
  ["User-Agent"] = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/115.0 Safari/537.36",
  ["Accept"] = "application/json, text/javascript, */*; q=0.01",
  ["Accept-Language"] = "en-US,en;q=0.9",
}

local function fetch(url, headers)
  headers = headers or {}
  for k,v in pairs(default_headers) do
    if not headers[k] then headers[k] = v end
  end
  local resp = {}
  local req = {
    url = url,
    method = "GET",
    headers = headers,
    sink = ltn12.sink.table(resp),
  }
  local r, code, res_headers, status = https.request(req)
  if not r and code == nil then
    return nil, ("request failed: %s"):format(tostring(status or r))
  end
  local body = table.concat(resp)
  return body, code, res_headers
end

local function urlencode(s)
  if s == nil then return "" end
  s = tostring(s)
  s = s:gsub("([^%w%-_%.~])", function(c) return string.format("%%%02X", string.byte(c)) end)
  s = s:gsub(" ", "+")
  return s
end

-- Extract numeric bookId or path pieces from a typical m.webnovel URL
-- Examples:
--  https://m.webnovel.com/book/12345
--  https://m.webnovel.com/book/12345/67890
local function parse_book_and_chapter_from_url(url)
  if not url then return nil end
  local bookId, chapterId = url:match("/book/(%d+)/?(%d*)")
  if bookId == "" then bookId = nil end
  if chapterId == "" then chapterId = nil end
  return bookId, chapterId
end

-- Try to get chapter list via mobile AJAX archive endpoint (common pattern).
-- If that fails, fall back to fetching book page HTML and parsing links.
function M.get_chapter_list_from_book_url(bookUrl)
  local bookId = parse_book_and_chapter_from_url(bookUrl)
  if not bookId then
    -- Try to extract numeric id from elsewhere
    bookId = bookUrl:match("bookId=(%d+)")
  end
  local result = {}

  if bookId then
    -- first attempt: known-ish AJAX endpoint (may change on site)
    local ajax = ("https://m.webnovel.com/ajax/chapter-archive?bookId=%s&page=1"):format(urlencode(bookId))
    local body, code = fetch(ajax, { ["X-Requested-With"] = "XMLHttpRequest" })
    if body and code == 200 then
      -- try parse JSON if any
      local success, decoded = pcall(function() return json.decode(body) end)
      if success and type(decoded) == "table" then
        -- JSON structure varies; try common fields
        local list = decoded.data or decoded.chapterList or decoded.chapters or decoded
        if list and type(list) == "table" then
          for _, item in ipairs(list) do
            -- item may have id/title/url
            local title = item.title or item.name or item.chapterName
            local id = item.chapterId or item.id or item.cid or tostring(item)
            local url = item.url or item.chapterUrl
            if not url and id then
              url = ("/book/%s/%s"):format(bookId, id)
            end
            if title and url then
              table.insert(result, { title = title, url = url, id = id })
            end
          end
          if #result > 0 then return result end
        end
      else
        -- If not JSON, maybe HTML; fall through to HTML parsing below
      end
    end
  end

  -- Fallback: fetch book page HTML and parse <a> links
  local htmlUrl = bookUrl
  local html, code = fetch(htmlUrl)
  if not html or code ~= 200 then
    return nil, "failed to fetch book page"
  end

  -- Simple heuristic: find chapter links like /book/<bookId>/<chapterId>
  for href, text in html:gmatch('href=("?/book/%d+/%d+[^"]-)"[^>]*>(.-)</a') do
    local u = href:gsub('"', "")
    if not u:match("^https?://") then
      u = "https://m.webnovel.com" .. u
    end
    local title = text:gsub("<.->", ""):gsub("^%s+", ""):gsub("%s+$", "")
    table.insert(result, { title = title, url = u })
  end

  -- Another common pattern: <a class="chapter-item" href="...">title</a>
  if #result == 0 then
    for href, text in html:gmatch('<a[^>]-class="[^"]-chapter[^"]-"[^>]-href="([^"]-)"[^>]->(.-)</a') do
      local u = href
      if not u:match("^https?://") then
        u = "https://m.webnovel.com" .. u
      end
      local title = text:gsub("<.->", ""):gsub("^%s+", ""):gsub("%s+$", "")
      table.insert(result, { title = title, url = u })
    end
  end

  return result
end

-- Get chapter body. Tries AJAX chapter endpoint first then fallback to HTML selector heuristics.
function M.get_chapter_body(chapterUrl)
  local bookId, chapterId = parse_book_and_chapter_from_url(chapterUrl)
  local tried = {}

  if bookId and chapterId then
    -- Try AJAX chapter endpoint (common pattern)
    local ajax = ("https://m.webnovel.com/ajax/chapter?bookId=%s&chapterId=%s"):format(urlencode(bookId), urlencode(chapterId))
    local body, code = fetch(ajax, { ["X-Requested-With"] = "XMLHttpRequest" })
    if body and code == 200 then
      -- if JSON, decode
      local success, decoded = pcall(function() return json.decode(body) end)
      if success and type(decoded) == "table" then
        -- common fields: content, title
        local content = decoded.data and decoded.data.content or decoded.content or decoded.chapterContent
        local title = decoded.data and decoded.data.title or decoded.title or decoded.chapterName
        if content then
          return { title = title or ("Chapter "..chapterId), content = content, source = "ajax" }
        end
      else
        -- sometimes response is HTML; try to find content directly
        local cont = body:match('<div[^>]-class="[^"]*chapter__content[^"]*"[^>]->(.-)</div>') or
                     body:match('<div[^>]-class="[^"]*cha%-words[^"]*"[^>]->(.-)</div>')
        if cont then
          return { title = "Chapter "..chapterId, content = cont, source = "ajax-html" }
        end
      end
    end
    table.insert(tried, ajax)
  end

  -- Fallback to fetching chapter page and parsing HTML
  local html, code = fetch(chapterUrl)
  if not html or code ~= 200 then
    return nil, ("failed to fetch chapter page (tried %d endpoints)"):format(#tried)
  end

  -- Look for common content selectors
  local content = html:match('<div[^>]-class="[^"]*chapter__content[^"]*"[^>]->(.-)</div>')
               or html:match('<div[^>]-class="[^"]*cha%-words[^"]*"[^>]->(.-)</div>')
               or html:match('<div[^>]-id="content"[^>]->(.-)</div>')
               or html:match('<div[^>]-class="txt%-wrap"[^>]->(.-)</div>')

  -- Try to extract title
  local title = html:match('<h?%d[^>]-class="[^"]*cha%-tit[^"]*"[^>]->(.-)</h?%d>') or
                html:match('<title[^>]->(.-)</title>')

  if content then
    -- Clean up some wrappers if needed
    return { title = (title and title:gsub("<.->","") or "Chapter"), content = content, source = "html" }
  end

  return nil, "unable to extract chapter content; update selectors or check AJAX endpoints"
end

-- Small helper: get book basic info (title, author) from book page
function M.get_book_info(bookUrl)
  local html, code = fetch(bookUrl)
  if not html or code ~= 200 then return nil, "failed to fetch book page" end
  local title = html:match('<h1[^>]->(.-)</h1>') or html:match('<title[^>]->(.-)</title>')
  if title then title = title:gsub("<.->",""):gsub("^%s+",""):gsub("%s+$","") end
  local author = html:match('Author:%s*</span>%s*<a[^>]->(.-)</a>') or html:match('authorName":"(.-)"')
  return { title = title, author = author }
end

return M
