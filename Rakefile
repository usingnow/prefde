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

desc '新建一期悬赏（交互式，直接 rake bounty）'
task :bounty, [:date, :title] do |_t, args|
  interactive = $stdin.tty? && args[:date].nil?

  puts ""
  date_raw = interactive ? ask("期号", Date.today.iso8601) : (args[:date] || Date.today.iso8601)
  begin
    date = Date.strptime(date_raw.to_s.strip, "%Y-%m-%d")
  rescue ArgumentError
    abort "日期格式不对：#{date_raw}。要写成 2026-12-01 这样。"
  end

  title     = interactive ? ask("标题") : (args[:title] || "（待填标题）")
  title     = "（待填标题）" if title.nil? || title.to_s.empty?
  reward    = interactive ? choose("奖励", REWARD_OPTIONS, REWARD_DEFAULT) : REWARD_DEFAULT
  reward    = "¥#{reward}" if reward.to_s =~ /\A\d+\z/   # 手输数字自动补 ¥
  threshold = interactive ? choose("成团门槛", THRESHOLD_OPTIONS.map { |t| "#{t} 份" }, "#{THRESHOLD_DEFAULT} 份") : THRESHOLD_DEFAULT

  th = threshold.to_s.gsub(/[^0-9]/, "")
  th = "10" if th.empty?

  code     = date.iso8601
  path     = File.join("_bounties", "#{code}.md")
  opens    = "#{code}T09:00:00+08:00"
  deadline = "#{(date + DEADLINE_DAYS).iso8601}T23:59:59+08:00"

  abort "\n#{path} 已经存在了。要重写就先删掉它，或者换一个日期。" if File.exist?(path)

  if interactive
    puts ""
    puts "要创建 #{path}："
    puts "  期号   #{code}"
    puts "  标题   #{title}"
    puts "  奖励   #{reward}"
    puts "  门槛   #{th} 份"
    puts "  发布   #{opens}"
    puts "  截止   #{deadline}（#{DEADLINE_DAYS} 天后）"
    puts ""
    confirm = ask("确认", "Y")
    abort "已取消，什么都没有写。" unless confirm.to_s.downcase.start_with?("y")
  end

  tpl = File.read("_templates/bounty.md")
  tpl = tpl.sub(/^code: .*$/,      %(code: "#{code}"))
  tpl = tpl.sub(/^title: .*$/,     %(title: "#{title}"))
  tpl = tpl.sub(/^reward: .*$/,    %(reward: "#{reward}"))
  tpl = tpl.sub(/^threshold: .*$/, "threshold: #{th}")
  tpl = tpl.sub(/^opens: .*$/,     "opens: #{opens}")
  tpl = tpl.sub(/^deadline: .*$/,  "deadline: #{deadline}")
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
    puts "改 summary、正文，保存后复制这一行到终端："
    puts %(  git add -A && git commit -m "第 #{code} 期：#{title}" && git push)
  else
    puts "接下来改 summary / 正文，然后 git add -A && git commit && git push"
  end
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

desc '生成每期悬赏的分享图（1080×1440 PNG）：rake share 或 rake "share[2026-11-19-b1]"'
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
