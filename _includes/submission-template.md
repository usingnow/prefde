{%- assign b = include.b -%}
# {{ b.code }} 投稿 · 你的GitHub用户名

- 期号：{{ b.code }}
- 题目：{{ b.title }}
- 奖励：{{ b.reward }}{% if b.reward_note %}（{{ b.reward_note }}）{% endif %}
- 截止：{{ b.deadline | date: "%Y-%m-%d %H:%M" }}（+08:00）
- 作品仓库：https://github.com/你的GitHub用户名/你的仓库名
- 演示地址：（可选，没有就写「无」）
- 一句话说明：

## 我做了什么

（一两段，讲清你做了什么、和题目要求怎么对应。）

## 怎么跑起来 / 怎么验证

```bash
git clone https://github.com/你的GitHub用户名/你的仓库名
cd 你的仓库名
# 把题目要求的验证命令写在这里
```

## 对照评审标准

{% for c in b.criteria -%}
- [ ] {{ c }}
{% endfor %}
{%- if b.submission_checklist %}
## 本期交付清单

{% for item in b.submission_checklist -%}
- [ ] {{ item }}
{% endfor %}
{%- endif %}
## 复盘

- 踩了哪些坑：
- 怎么解决的：
- 如果再做一次，会怎么改：

## 自查

- [ ] 文件名是 `_submissions/{{ b.code }}/你的GitHub用户名.md`
- [ ] 仓库是 public，链接点得开
- [ ] 没有提交二进制文件
- [ ] 作品为本人原创
