# 新建一期悬赏。用法见 docs/how-to-write-a-bounty.md
require "date"
require "fileutils"
require "base64"
require "yaml"
require "time"
require "tmpdir"

DEADLINE_DAYS = 7

# 选项改这里就行。序号从 1 开始，最后一项「其它」会自动加。
REWARD_OPTIONS    = ["¥20", "¥50", "¥100", "¥200"]
REWARD_DEFAULT    = "¥50"
THRESHOLD_OPTIONS = [5, 8, 10, 15, 20]
THRESHOLD_DEFAULT = 10

# 读一行输入。空回车用默认值；非交互环境（管道、CI）直接返回默认值。
def ask(label, default = nil)
  prompt = default ? "#{label}（回车＝#{default}）：" : "#{label}："
  print prompt
  $stdout.flush
  line = $stdin.gets
  return default if line.nil?
  line = line.strip
  line.empty? ? default : line
end

# 多选一。回车用默认值；输入序号选对应项；输入其它内容当自定义值。
def choose(label, options, default)
  puts ""
  puts "#{label}："
  options.each_with_index { |o, i| puts "  #{i + 1}) #{o}" }
  puts "  #{options.size + 1}) 其它（手动输入）"
  print "选一个（回车＝#{default}）："
  $stdout.flush

  line = $stdin.gets
  return default if line.nil?
  line = line.strip
  return default if line.empty?

  if line =~ /\A\d+\z/
    n = line.to_i
    return options[n - 1] if n >= 1 && n <= options.size
    return ask("自定义值") if n == options.size + 1
  end

  line
end

# 跨平台找可执行文件
def which(cmd)
  exts = ENV["PATHEXT"] ? ENV["PATHEXT"].split(";") : [""]
  exts.each do |ext|
    ENV["PATH"].to_s.split(File::PATH_SEPARATOR).each do |dir|
      full = File.join(dir, "#{cmd}#{ext}")
      return full if File.file?(full) && File.executable?(full)
    end
  end
  nil
end

# 打开编辑器。终端编辑器要等它退出；IDE 的命令行工具不必等。
def open_in_editor(path)
  [ENV["EDITOR"], ENV["VISUAL"], "code", "cursor"].compact.uniq.each do |cmd|
    name, *flags = cmd.split(" ")
    exe = which(name)
    next unless exe
    system(exe, *flags, path)
    return true
  end
  false
end

# 每期悬赏的投稿模板：一个 .txt 页面，渲染成 /assets/submissions/<code>.md。
# 页面本身不存内容，只是把 _bounties 里的数据喂给 _includes/submission-template.md，
# 这样悬赏改了标题/奖励/截止/评审标准，下载的模板会自动跟着变。
SUBMISSION_STUB = <<~'LIQUID'
  ---
  layout: null
  permalink: /assets/submissions/CODE.md
  ---
  {% assign b = site.bounties | where: "code", "CODE" | first %}{% include submission-template.md b=b %}
LIQUID

def write_submission_stub(code)
  out = File.join("assets", "submissions", "#{code}.txt")
  FileUtils.mkdir_p(File.dirname(out))
  File.write(out, SUBMISSION_STUB.gsub("CODE", code))
  out
end

# 当前时间（+08:00）的日期，避免依赖系统时区。
def today_cn
  Time.now.getlocal("+08:00").strftime("%Y-%m-%d")
end

# 只保留小写字母、数字、连字符，用来生成 code。
def slugify(value)
  value.to_s.strip.downcase.gsub(/[^a-z0-9]+/, "-").gsub(/\A-+|-+\z/, "")
end

# 在 _bounties 下找下一个不冲突的 code。
def unique_bounty_code(base)
  code = base
  n = 2
  while File.exist?(File.join("_bounties", "#{code}.md"))
    code = "#{base}-#{n}"
    n += 1
  end
  code
end

# 统一系列名的空格/大小写，避免「Odoo 安装系列」和「Odoo  安装系列」匹配不上。
def normalize_series(value)
  value.to_s.strip.gsub(/\u3000/, " ").gsub(/\s+/, " ").downcase
end

# 现有系列名（去重、排序），用于菜单选择。
def existing_series_names
  Dir.glob("_bounties/*.md")
     .map { |path| share_front_matter(path)["series"].to_s.strip }
     .reject(&:empty?)
     .uniq
     .sort
end

# 同一系列里现有最大 series_order + 1。
def next_series_order(series_name)
  return 1 if series_name.to_s.strip.empty?

  target = normalize_series(series_name)
  orders = Dir.glob("_bounties/*.md").map do |path|
    data = share_front_matter(path)
    normalize_series(data["series"]) == target ? data["series_order"].to_i : 0
  end
  (orders.max || 0) + 1
end

# 把 "2026-12-01" 或 "2026-12-01 09:00" 解析成 +08:00 的 Time。
def parse_plan_time(raw, default_hour, default_min, default_sec)
  str = raw.to_s.strip
  str = today_cn if str.empty?
  date_part, time_part = str.split(/[ T]/, 2)
  y, m, d = date_part.split("-").map(&:to_i)
  hh, mm = time_part.to_s.split(":").map(&:to_i)
  hh = default_hour if hh.nil?
  mm = default_min if mm.nil?
  Time.new(y, m, d, hh, mm, default_sec, "+08:00")
rescue ArgumentError, TypeError
  abort "时间格式不对：#{raw}。要写成 2026-12-01 或 \"2026-12-01 09:00\"。"
end

def iso_ts(time)
  time.strftime("%Y-%m-%dT%H:%M:%S+08:00")
end

def yaml_quote(value)
  value.to_s.inspect
end

desc '新建一期悬赏（交互式，默认保存为草稿）'
task :bounty, [:date, :title] do |_t, args|
  interactive = $stdin.tty? && args[:date].nil?

  puts ""
  date_raw = interactive ? ask("日期", today_cn) : (args[:date] || today_cn)
  begin
    date = Date.strptime(date_raw.to_s.strip, "%Y-%m-%d")
  rescue ArgumentError
    abort "日期格式不对：#{date_raw}。要写成 2026-12-01 这样。"
  end

  # 系列 / 单期
  series_slug = interactive ? ask("系列代号（小写英文，如 odoo；留空＝单期）", "") : ""
  series_name = ""
  series_order = nil

  if series_slug.to_s.strip.empty?
    short_slug = interactive ? ask("短名（小写英文，如 rss-reader）", "new-bounty") : "new-bounty"
    short_slug = slugify(short_slug)
    abort "短名只能用小写字母、数字、连字符" if short_slug.empty?
    code_base = "#{date.iso8601}-#{short_slug}"
  else
    series_slug = slugify(series_slug)
    abort "系列代号只能用小写字母、数字、连字符，例如 odoo" if series_slug.empty?
    names = existing_series_names
    if interactive && names.any?
      series_name = choose("系列显示名", names, names.first)
    elsif interactive
      series_name = ask("系列显示名（如 Odoo 安装系列）", "")
    else
      series_name = ""
    end
    abort "系列显示名不能为空" if series_name.to_s.strip.empty?
    suggested_order = next_series_order(series_name)
    order_raw = interactive ? ask("系列序号", suggested_order) : suggested_order
    series_order = order_raw.to_s.gsub(/[^0-9]/, "").to_i
    series_order = suggested_order if series_order <= 0
    code_base = "#{date.iso8601}-#{series_slug}-#{format('%02d', series_order)}"
  end

  title = interactive ? ask("标题") : (args[:title] || "（待填标题）")
  title = "（待填标题）" if title.nil? || title.to_s.empty?

  suggested_code = unique_bounty_code(code_base)
  code = interactive ? ask("代号 code", suggested_code) : suggested_code
  code = code.to_s.strip.downcase
  abort "代号只能用小写字母、数字、连字符，例如 2026-12-01-odoo-07" unless code =~ /\A[a-z0-9][a-z0-9-]*\z/

  path = File.join("_bounties", "#{code}.md")
  abort "\n#{path} 已经存在了。换一个代号，或者先删掉旧文件。" if File.exist?(path)

  level = interactive ? choose("难度等级", ["1 基础", "2 进阶", "3 挑战"], "2 进阶") : "2 进阶"
  lv = (level.to_s[/[0-9]/] || "2").to_i

  reward = interactive ? choose("奖励", REWARD_OPTIONS, REWARD_DEFAULT) : REWARD_DEFAULT
  reward = "¥#{reward}" if reward.to_s =~ /\A\d+\z/

  threshold = interactive ? choose("成团门槛", THRESHOLD_OPTIONS.map { |t| "#{t} 份" }, "#{THRESHOLD_DEFAULT} 份") : THRESHOLD_DEFAULT
  th = threshold.to_s.gsub(/[^0-9]/, "")
  th = "10" if th.empty?

  planned_publish = date + DEADLINE_DAYS
  opens_raw = interactive ? ask("计划发布日期", "#{planned_publish.iso8601} 09:00") : "#{planned_publish.iso8601} 09:00"
  opens = parse_plan_time(opens_raw, 9, 0, 0)

  planned_deadline = planned_publish + DEADLINE_DAYS
  deadline_raw = interactive ? ask("计划截止日期", "#{planned_deadline.iso8601} 23:59") : "#{planned_deadline.iso8601} 23:59"
  deadline = parse_plan_time(deadline_raw, 23, 59, 59)

  abort "截止时间必须晚于计划发布时间。" if deadline <= opens

  created = Time.now.getlocal("+08:00")

  if interactive
    series_label = series_name.empty? ? "（单期）" : "#{series_name} 第 #{series_order} 题"
    puts ""
    puts "要创建 #{path}："
    puts "  code     #{code}"
    puts "  标题     #{title}"
    puts "  系列     #{series_label}"
    puts "  难度     L#{lv}"
    puts "  奖励     #{reward}"
    puts "  门槛     #{th} 份"
    puts "  计划发布 #{iso_ts(opens)}"
    puts "  计划截止 #{iso_ts(deadline)}"
    puts "  状态     draft（尚未发布）"
    puts ""
    confirm = ask("确认", "Y")
    abort "已取消，什么都没有写。" unless confirm.to_s.downcase.start_with?("y")
  end

  tpl = File.read("_templates/bounty.md")
  series_order_line = series_order.nil? ? "series_order:" : "series_order: #{series_order}"
  replacements = {
    /^code:.*$/         => %(code: #{yaml_quote(code)}),
    /^title:.*$/        => %(title: #{yaml_quote(title)}),
    /^series:.*$/       => %(series: #{yaml_quote(series_name)}),
    /^series_order:.*$/ => series_order_line,
    /^created:.*$/      => %(created: #{iso_ts(created)}),
    /^level:.*$/        => %(level: #{lv}),
    /^reward:.*$/       => %(reward: #{yaml_quote(reward)}),
    /^threshold:.*$/    => %(threshold: #{th}),
    /^opens:.*$/        => %(opens: #{iso_ts(opens)}),
    /^deadline:.*$/     => %(deadline: #{iso_ts(deadline)}),
    /^status:.*$/       => %(status: draft)
  }
  replacements.each { |pattern, value| tpl = tpl.sub(pattern, value) }
  File.write(path, tpl)

  stub = write_submission_stub(code)

  full = File.expand_path(path)
  puts ""
  puts "已创建 #{path}"
  puts "已生成 #{stub}（投稿模板下载）"

  if interactive
    if open_in_editor(path)
      puts "已在编辑器里打开。"
    else
      puts "没找到编辑器，请手动打开："
      puts "  #{full}"
    end
    puts ""
    puts "改 summary、正文、submission_checklist，保存后："
    puts "  1. 本地预览：bundle exec jekyll serve"
    puts "  2. 发布：rake \"publish[#{code}]\""
    puts "  3. 提交：git add -A && git commit -m \"第 #{code} 期：#{title}\" && git push"
  else
    puts "接下来改 summary / 正文，然后："
    puts "  rake \"publish[#{code}]\""
    puts "  git add -A && git commit && git push"
  end
end

desc '发布一期草稿：rake "publish[2026-12-01-odoo-07]"'
task :publish, [:code] do |_t, args|
  code = args[:code].to_s.strip
  abort '用法：rake "publish[期号]"' if code.empty?

  path = File.join("_bounties", "#{code}.md")
  abort "找不到 #{path}" unless File.exist?(path)

  data = share_front_matter(path)
  unless data["status"] == "draft"
    abort "这一期状态是 #{data['status']}，只有 draft 可以发布。"
  end

  now = Time.now.getlocal("+08:00")
  raw = File.read(path)
  raw = raw.sub(/^status:.*$/, "status: open")
  raw = raw.sub(/^opens:.*$/, "opens: #{iso_ts(now)}")

  deadline = data["deadline"]
  if deadline.nil? || deadline.to_s.empty? || Time.parse(deadline.to_s) <= now
    new_deadline = Time.new(now.year, now.month, now.day, 23, 59, 59, "+08:00") + DEADLINE_DAYS * 86_400
    raw = raw.sub(/^deadline:.*$/, "deadline: #{iso_ts(new_deadline)}")
    puts "截止时间已过或为空，自动改为 #{iso_ts(new_deadline)}"
  end

  File.write(path, raw)
  stub = write_submission_stub(code)
  poster = build_share_poster(code)

  puts ""
  puts "已发布 #{code}"
  puts "  状态   open"
  puts "  发布   #{iso_ts(now)}"
  puts "  模板   #{stub}"
  puts "  分享图 #{poster}"
  puts ""
  puts "提交：git add -A && git commit -m \"发布 #{code}\" && git push"
end

desc '为每一期悬赏生成投稿模板下载文件'
task :templates do
  codes = Dir.glob("_bounties/*.md").map { |p| File.basename(p, ".md") }.sort
  abort "没有找到 _bounties/*.md" if codes.empty?

  codes.each do |code|
    puts "已生成 #{write_submission_stub(code)}"
  end
end

# ---------------------------------------------------------------------------
# 分享图：每期一张 1080×1440 的 PNG，带群二维码
# ---------------------------------------------------------------------------

# 只从文件顶部的 front matter 读 YAML；没有 front matter 就返回空 Hash。
def share_front_matter(path)
  raw = File.read(path)
  front = raw.start_with?("---") ? (raw.split(/^---\s*$/, 3)[1] || "") : ""
  YAML.safe_load(front, permitted_classes: [Time, Date]) || {}
rescue StandardError
  {}
end

# 单次替换 {{TOKEN}}，避免替换值里又含 token 被二次替换。
def share_replace(template, values)
  template.gsub(/\{\{[A-Z_]+\}\}/) { |m| values.key?(m) ? values[m].to_s : m }
end

# 文字字段进 HTML 前转义；logo SVG 和二维码 data URI 不要过这里。
def share_escape(value)
  value.to_s
       .gsub("&", "&amp;")
       .gsub("<", "&lt;")
       .gsub(">", "&gt;")
       .gsub('"', "&quot;")
end

def build_share_poster(code)
  bounty_path = File.join("_bounties", "#{code}.md")
  abort "找不到 #{bounty_path}" unless File.exist?(bounty_path)

  b = share_front_matter(bounty_path)
  config = YAML.safe_load(File.read("_config.yml"), permitted_classes: [Time, Date]) || {}
  share_cfg = config["share"] || {}

  qr_rel = (share_cfg["qr_image"] || "/assets/img/wechat-group.png").sub(%r{\A/}, "")
  qr_path = File.expand_path(qr_rel)
  abort "找不到二维码图片：#{qr_rel}" unless File.exist?(qr_path)

  logo_path = File.expand_path("assets/img/logo.svg")
  abort "找不到站点 logo：#{logo_path}" unless File.exist?(logo_path)

  qr_data = Base64.strict_encode64(File.binread(qr_path))
  logo_svg = File.read(logo_path)

  deadline_str =
    begin
      Time.parse(b["deadline"].to_s).strftime("%m.%d %H:%M")
    rescue StandardError
      b["deadline"].to_s
    end

  level_str = case b["level"].to_i
              when 1 then "L1 基础"
              when 2 then "L2 进阶"
              when 3 then "L3 挑战"
              else "未分级"
              end

  subs = b["submissions_shown"].to_i
  threshold = b["threshold"].to_i
  todo = threshold - subs
  progress = todo.positive? ? "还差 #{todo} 份" : "已满 #{threshold} 份"

  list = Array(b["submission_checklist"])
  list_title = "本期交付"
  if list.empty?
    list = Array(b["criteria"])
    list_title = "评审看什么"
  end
  deliverables = list.first(4).map { |item| "<li>#{share_escape(item)}</li>" }.join

  title = b["title"].to_s
  title_class = title.length > 18 ? "long" : ""
  site_url = config["url"].to_s.sub(%r{\Ahttps?://}, "")

  template = File.read("_templates/share-poster.html")
  html = share_replace(template, {
    "{{LOGO_SVG}}"    => logo_svg,
    "{{KICKER}}"      => share_escape(b["kicker"] || "本期悬赏"),
    "{{LEVEL}}"       => share_escape(level_str),
    "{{SERIES}}"      => share_escape(b["series"]),
    "{{TITLE_CLASS}}" => title_class,
    "{{TITLE}}"       => share_escape(title),
    "{{SUMMARY}}"     => share_escape(b["summary"]),
    "{{REWARD}}"      => share_escape(b["reward"]),
    "{{DEADLINE}}"    => share_escape(deadline_str),
    "{{PROGRESS}}"    => share_escape(progress),
    "{{DELIVERABLES_TITLE}}" => share_escape(list_title),
    "{{DELIVERABLES}}" => deliverables,
    "{{QR_IMAGE}}"    => "data:image/png;base64,#{qr_data}",
    "{{QR_CAPTION}}"  => share_escape(share_cfg["qr_caption"] || "扫码加群"),
    "{{URL}}"         => share_escape(site_url),
    "{{CODE}}"        => share_escape(b["code"] || code)
  })

  out = File.join("assets", "img", "share", "#{code}.png")
  FileUtils.mkdir_p(File.dirname(out))

  chrome = which("google-chrome") || which("chromium") || which("chromium-browser")
  abort "没找到 Chrome/Chromium。分享图用 headless 浏览器截图，请先装一个。" unless chrome

  Dir.mktmpdir do |dir|
    tmp = File.join(dir, "poster.html")
    File.write(tmp, html)
    ok = system(
      chrome,
      "--headless", "--no-sandbox", "--disable-gpu", "--hide-scrollbars",
      "--window-size=1080,1440",
      "--screenshot=#{File.expand_path(out)}",
      "file://#{tmp}",
      out: File::NULL, err: File::NULL
    )
    abort "Chrome 截图失败：#{code}" unless ok
  end

  out
end

desc '生成每期悬赏的分享图（1080×1440 PNG）：rake share 或 rake "share[2026-11-19-odoo-01]"'
task :share, [:code] do |_t, args|
  codes =
    if args[:code].to_s.strip.empty?
      Dir.glob("_bounties/*.md").map { |p| File.basename(p, ".md") }.sort
    else
      [args[:code].to_s.strip]
    end

  abort "没有找到 _bounties/*.md" if codes.empty?

  codes.each do |code|
    puts "已生成 #{build_share_poster(code)}"
  end
end

task :default do
  puts ""
  puts "用法：rake <任务名>。当前可用："
  puts ""
  system("rake", "-T")
  puts ""
end


# ---------------------------------------------------------------------------
# markdown → PDF（给同事传阅用）
# ---------------------------------------------------------------------------

PDF_CSS = <<~CSS
  @page {
    size: A4;
    margin: 20mm 18mm 18mm;
    @bottom-left  { content: "PreFDE 加油站"; font-family: sans-serif; font-size: 8pt; color: #6b6b6b; }
    @bottom-right { content: counter(page) " / " counter(pages); font-family: sans-serif; font-size: 8pt; color: #6b6b6b; }
  }
  @page :first { @bottom-left { content: ""; } }
  * { box-sizing: border-box; }
  body {
    font-family: "Noto Serif CJK SC", "Songti SC", serif;
    font-size: 11pt; line-height: 1.75; color: #121212; margin: 0;
    font-feature-settings: "kern" 1, "liga" 1;
  }
  .kicker { font-family: sans-serif; font-size: 8pt; font-weight: 600;
            letter-spacing: .09em; text-transform: uppercase; color: #e3120b; margin: 0 0 6pt; }
  h1.title { font-size: 21pt; line-height: 1.25; font-weight: 600; margin: 0 0 8pt; }
  .meta { font-family: sans-serif; font-size: 9pt; color: #6b6b6b; margin: 0 0 14pt; }
  .rule { border: 0; border-top: 3px solid #e3120b; margin: 0 0 16pt; }
  .lede { font-size: 12.5pt; line-height: 1.65; color: #3d3d3d; margin: 0 0 18pt; }
  h2 { font-size: 15pt; font-weight: 600; margin: 22pt 0 8pt; padding-top: 10pt;
       border-top: 1px solid #d7d3cc; break-after: avoid; }
  h3 { font-size: 11.5pt; font-weight: 600; margin: 14pt 0 6pt; break-after: avoid; }
  p, li { orphans: 2; widows: 2; }
  ul, ol { margin: 0 0 10pt; padding-left: 18pt; }
  li { margin-bottom: 3pt; }
  strong { font-weight: 600; }
  em.hl { font-style: normal; border-bottom: 2px solid #e3120b; }
  blockquote { margin: 12pt 0; padding-left: 10pt; border-left: 3px solid #e3120b; font-weight: 600; }
  blockquote p { margin: 0; }
  code { font-family: "DejaVu Sans Mono", monospace; font-size: 9.5pt;
         background: #f7f5f0; border: 1px solid #d7d3cc; padding: 0 3pt; }
  pre { background: #f7f5f0; border: 1px solid #d7d3cc; padding: 10pt; margin: 10pt 0;
        font-size: 9pt; line-height: 1.55; white-space: pre-wrap; break-inside: avoid; }
  pre code { border: 0; background: none; padding: 0; font-size: 9pt; }
  table { width: 100%; border-collapse: collapse; margin: 10pt 0 14pt;
          font-family: sans-serif; font-size: 9.5pt; }
  tr { break-inside: avoid; }
  th { text-align: left; padding: 6pt 8pt; border-top: 1px solid #121212;
       border-bottom: 1px solid #121212; font-size: 8pt; font-weight: 600;
       letter-spacing: .06em; text-transform: uppercase; color: #6b6b6b;
       white-space: nowrap; }
  td { padding: 6pt 8pt; border-bottom: 1px solid #d7d3cc; vertical-align: top; }
  hr { border: 0; border-top: 1px solid #d7d3cc; margin: 16pt 0; }
  @media screen {
    body { max-width: 820px; margin: 0 auto; padding: 40px 28px 80px; }
    h1.title { font-size: 28px; }
  }
CSS

def build_doc_html(path)
  require "kramdown"
  require "yaml"

  raw = File.read(path)
  front, body = raw.start_with?("---") ? (raw.split(/^---\s*$/, 3)[1..2] || [nil, raw]) : [nil, raw]
  data = (YAML.safe_load(front.to_s, permitted_classes: [Time, Date]) || {}) rescue {}

  html_body = Kramdown::Document.new(body.to_s, input: "GFM", hard_wrap: false).to_html

  meta = [
    data["reward"]   && "奖励 #{data['reward']}",
    data["threshold"] && "门槛 #{data['threshold']} 份投稿",
    data["deadline"] && "截止 #{data['deadline'].to_s[0, 10].tr('-', '.')}"
  ].compact.join("　·　")

  html = <<~HTML
    <!doctype html>
    <html lang="zh-CN"><head><meta charset="utf-8">
    <title>#{data['title'] || File.basename(path, '.md')}</title>
    <style>#{PDF_CSS}</style></head><body>
    <p class="kicker">#{data['kicker'] || '悬赏'}</p>
    <h1 class="title">#{data['title']}</h1>
    #{meta.empty? ? "" : "<p class=\"meta\">#{meta}</p>"}
    <hr class="rule">
    #{data['summary'] ? "<p class=\"lede\">#{data['summary']}</p>" : ''}
    #{html_body}
    </body></html>
  HTML
  [html, path.sub(/\.md\z/, "")]
end

desc '把 markdown 转成 PDF：rake "pdf[docs/xxx.md]"'
task :pdf, [:path] do |_t, args|
  require "tmpdir"
  path = args[:path].to_s.strip
  abort '用法：rake "pdf[docs/xxx.md]"' if path.empty?
  abort "找不到文件：#{path}" unless File.exist?(path)

  html, base = build_doc_html(path)
  out = "#{base}.pdf"
  Dir.mktmpdir do |dir|
    tmp = File.join(dir, "page.html")
    File.write(tmp, html)
    ok = system("weasyprint", tmp, out)
    abort "weasyprint 失败，检查上面的输出" unless ok
  end
  puts "已生成 #{out}（#{File.size(out) / 1024} KB）"
end

desc '同时生成 HTML 和 PDF：rake "doc[docs/xxx.md]"'
task :doc, [:path] do |_t, args|
  require "tmpdir"
  path = args[:path].to_s.strip
  abort '用法：rake "doc[docs/xxx.md]"' if path.empty?
  abort "找不到文件：#{path}" unless File.exist?(path)

  html, base = build_doc_html(path)
  html_path = "#{base}.html"
  File.write(html_path, html)

  out = "#{base}.pdf"
  Dir.mktmpdir do |dir|
    tmp = File.join(dir, "page.html")
    File.write(tmp, html)
    ok = system("weasyprint", tmp, out)
    abort "weasyprint 失败" unless ok
  end
  puts "已生成 #{html_path}（#{File.size(html_path) / 1024} KB）"
  puts "已生成 #{out}（#{File.size(out) / 1024} KB）"
end
