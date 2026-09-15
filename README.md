# BookArchive · 个人知识库

> 书籍 / PPT / 论文 / 讲义 / 视频 / 链接 的统一笔记仓库。
> 大文件（书籍、视频源文件）不入库（见 .gitignore），通过 MD5 与本地文件关联。
> **规则与总索引的唯一权威来源：[说明文档.md](说明文档.md)**｜执行计划：[0-计划/](0-计划/README.md)

## 目录结构

```
BookArchive/
├── README.md            # 本文件：总导航
├── 书单总目.md           # 书籍全量清单（含ISBN/MD5）
├── 专题/                # 跨分类聚合（MOC）
├── 模板/笔记.md          # 新笔记模板
└── {分类}/
    ├── INDEX.md         # 分类总索引（所有形态统一表）
    ├── links.md         # 轻收藏链接
    └── notes/{资料名}/  # 完整笔记（笔记.md + 摘录.md + assets/）
```

## 分类导航

| 分类 | 资料数 | 入口 |
|---|---|---|
| 儿童读物 | 4 有效 | [INDEX](儿童读物/INDEX.md) |
| 计算机&机器学习 | 待收录 2 | [INDEX](计算机&机器学习/INDEX.md) |
| 其他 | 在库 12 ｜ 待收录 6 | [INDEX](其他/INDEX.md) |
| 数学&物理 | 在库 9 ｜ 待收录 4 | [INDEX](数学&物理/INDEX.md) |
| 文明&历史 | 在库 1 ｜ 待收录 10 | [INDEX](文明&历史/INDEX.md) |
| 育儿&哲学 | 在库 16（含综述）｜ 待迁入 5 | [INDEX](育儿&哲学/INDEX.md) |

## 形态枚举（front-matter `kind`）

📚 book ｜ 📊 slides ｜ 📄 paper ｜ 📝 lecture ｜ 🎬 video ｜ 🔗 link

## 工作流

1. 新资料 → 复制 [模板/笔记.md](模板/笔记.md) 到 `{分类}/notes/{资料名}/笔记.md`，填写 front-matter
2. 轻收藏 → 各分类 `links.md` 一行一条；重要者升级为完整笔记
3. 在 `{分类}/INDEX.md` 登记一行，笔记列将 ☐ 替换为笔记链接
4. 跨分类主题 → `专题/` 新建 MOC 页
5. 视频笔记附时间戳表，截图按 `HH-MM-SS.png` 存 `assets/`
