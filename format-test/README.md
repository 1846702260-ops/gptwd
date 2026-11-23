# 格式可读性测试 · 样本索引

本目录用于测试外部模型（如 ChatGPT）能否读取 GitHub 仓库中不同格式的文档。
全部内容为格式测试样本，**不含生产地址、内部路径、凭证或个人数据**。

生成时间：2026-11-24 01:35 (UTC+8)

## 一、样本清单

| 格式 | 文件 | 校验码 | 指纹短语 |
| --- | --- | --- | --- |
| PLAIN TEXT | `fmt-test-01.txt` | `ZY-TXT-20261124-A1F3` | 午夜蓝的第七盏灯 |
| MARKDOWN | `fmt-test-02.md` | `ZY-MD-20261124-B2E7` | 铁皮屋顶上的第三只猫 |
| JSON | `fmt-test-03.json` | `ZY-JSON-20261124-C9D4` | 编号为零的备用钥匙 |
| HTML | `fmt-test-04.html` | `ZY-HTML-20261124-D5B8` | 半旧的铜制罗盘 |
| CSV | `fmt-test-05.csv` | `ZY-CSV-20261124-E6A2` | 盐渍柠檬的第四片 |
| PDF | `fmt-test-06.pdf` | `ZY-PDF-20261124-F7C1` | 逆光里的灰鸽子 |
| WORD (docx) | `fmt-test-07.docx` | `ZY-DOCX-20261124-G8E5` | 停摆的第廿三分钟 |

## 二、三种访问方式

GitHub 页面是前端渲染的，部分抓取工具拿不到正文；原始链接和 CDN 直出文件内容，成功率通常更高。

### PLAIN TEXT

- 网页：https://github.com/1846702260-ops/gptwd/blob/main/format-test/fmt-test-01.txt
- 原始：https://raw.githubusercontent.com/1846702260-ops/gptwd/main/format-test/fmt-test-01.txt
- CDN：https://cdn.jsdelivr.net/gh/1846702260-ops/gptwd@main/format-test/fmt-test-01.txt

### MARKDOWN

- 网页：https://github.com/1846702260-ops/gptwd/blob/main/format-test/fmt-test-02.md
- 原始：https://raw.githubusercontent.com/1846702260-ops/gptwd/main/format-test/fmt-test-02.md
- CDN：https://cdn.jsdelivr.net/gh/1846702260-ops/gptwd@main/format-test/fmt-test-02.md

### JSON

- 网页：https://github.com/1846702260-ops/gptwd/blob/main/format-test/fmt-test-03.json
- 原始：https://raw.githubusercontent.com/1846702260-ops/gptwd/main/format-test/fmt-test-03.json
- CDN：https://cdn.jsdelivr.net/gh/1846702260-ops/gptwd@main/format-test/fmt-test-03.json

### HTML

- 网页：https://github.com/1846702260-ops/gptwd/blob/main/format-test/fmt-test-04.html
- 原始：https://raw.githubusercontent.com/1846702260-ops/gptwd/main/format-test/fmt-test-04.html
- CDN：https://cdn.jsdelivr.net/gh/1846702260-ops/gptwd@main/format-test/fmt-test-04.html

### CSV

- 网页：https://github.com/1846702260-ops/gptwd/blob/main/format-test/fmt-test-05.csv
- 原始：https://raw.githubusercontent.com/1846702260-ops/gptwd/main/format-test/fmt-test-05.csv
- CDN：https://cdn.jsdelivr.net/gh/1846702260-ops/gptwd@main/format-test/fmt-test-05.csv

### PDF

- 网页：https://github.com/1846702260-ops/gptwd/blob/main/format-test/fmt-test-06.pdf
- 原始：https://raw.githubusercontent.com/1846702260-ops/gptwd/main/format-test/fmt-test-06.pdf
- CDN：https://cdn.jsdelivr.net/gh/1846702260-ops/gptwd@main/format-test/fmt-test-06.pdf

### WORD (docx)

- 网页：https://github.com/1846702260-ops/gptwd/blob/main/format-test/fmt-test-07.docx
- 原始：https://raw.githubusercontent.com/1846702260-ops/gptwd/main/format-test/fmt-test-07.docx
- CDN：https://cdn.jsdelivr.net/gh/1846702260-ops/gptwd@main/format-test/fmt-test-07.docx

## 三、怎么判断模型有没有真读到

每份文档都埋了两处可验证信息：

1. **校验码**：位于文档开头的元信息里，形如 `ZY-TXT-20261124-A1F3`。
2. **指纹短语**：埋在正文第四节的句子里，与报告主题完全无关，摘要不会提到它。

分别向模型提这三个问题即可：

- 这份文档的校验码是什么？
- 第四节那句"无关紧要的话"里，提到的是什么？
- 三条固定要点中的第二条是什么？

能同时答对校验码和指纹短语，说明它读到了完整正文；只答对标题或要点，通常是从搜索摘要或页面片段拼出来的。

## 四、已知差异

- `.pdf` 与 `.docx` 为二进制格式，网页上无法预览正文，只能下载或由模型直接解析文件，适合验证模型的原生文档解析能力。
- `.csv` 用 `section,item,value` 三列承载全部内容，表格语义最弱，但机器解析最稳。
- `.html` 含样式标签，纯文本抓取时会有少量噪音。
