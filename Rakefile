# 新建一期悬赏。用法见 docs/how-to-write-a-bounty.md
require "date"

DAYS = 7

desc '新建一期悬赏：rake "bounty[2026-12-01,标题]"（日期和标题都可省略）'
task :bounty, [:date, :title] do |_t, args|
  raw = (args[:date] || Date.today.iso8601).to_s.strip
  begin
    d = Date.strptime(raw, "%Y-%m-%d")
  rescue ArgumentError
    abort "日期格式不对：#{raw}。要写成 2026-12-01 这样。"
  end

  code = d.iso8601
  path = File.join("_bounties", "#{code}.md")
  abort "#{path} 已经存在了。要重写就先删掉它，或者换一个日期。" if File.exist?(path)

  title = (args[:title] || "（待填标题）").to_s.strip
  opens    = "#{code}T09:00:00+08:00"
  deadline = "#{(d + DAYS).iso8601}T23:59:59+08:00"

  tpl = File.read("_templates/bounty.md")
  tpl = tpl.sub(/^code: .*$/,     %(code: "#{code}"))
  tpl = tpl.sub(/^title: .*$/,    %(title: "#{title}"))
  tpl = tpl.sub(/^opens: .*$/,    "opens: #{opens}")
  tpl = tpl.sub(/^deadline: .*$/, "deadline: #{deadline}")

  File.write(path, tpl)

  puts "已创建 #{path}"
  puts "  期号     #{code}"
  puts "  发布     #{opens}"
  puts "  截止     #{deadline}（#{DAYS} 天后）"
  puts ""
  puts "接下来打开这个文件，改这几处："
  puts "  summary  —— 一句话说清要做什么"
  puts "  reward   —— 奖励金额"
  puts "  threshold —— 成团需要几份投稿"
  puts "  正文      —— 题目 / 交付 / 评审标准 / 规则"
  puts ""
  puts "改完：git add -A && git commit -m \"第 #{code} 期\" && git push"
end
