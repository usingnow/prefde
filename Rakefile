# 新建一期悬赏。用法见 docs/how-to-write-a-bounty.md
require "date"

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

  full = File.expand_path(path)
  puts ""
  puts "已创建 #{path}"

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
          font-family: sans-serif; font-size: 9.5pt; break-inside: avoid; }
  th { text-align: left; padding: 6pt 8pt; border-top: 1px solid #121212;
       border-bottom: 1px solid #121212; font-size: 8pt; font-weight: 600;
       letter-spacing: .06em; text-transform: uppercase; color: #6b6b6b; }
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
