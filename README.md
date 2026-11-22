# docx-to-apkg

Agent skill biến **các từ được gạch chân trong file Word (.docx)** thành **một bảng từ vựng `.tsv` và một file Anki `.apkg`**.

Một bài đọc thành một bảng, một gói thẻ. Thẻ nằm trong deck **Default** có sẵn của Anki. Skill không tạo deck mới.

```
bai_doc.docx
    │
    ├─(1) lọc từ gạch chân──► extract/2026_11_23_01.txt
    ├─(2) bỏ từ đã có, viết TSV──► tsv/2026_11_23_01.tsv
    └─(3) đóng gói SpoonFed++──► output/2026_11_23_01.apkg
```

Chỉ cần Python 3 (thư viện chuẩn). Skill không mở collection Anki của bạn. Cột `Audio` và `SP` để trống, để HyperTTS điền sau trong Anki.

Cùng bộ với:

| Skill | Việc làm |
|---|---|
| [`docx-to-chinese-vocab`](https://github.com/vibeailabs-tech/docx-to-chinese-vocab) | Lọc từ gạch chân → TSV |
| [`chinese-vocab-tsv`](https://github.com/vibeailabs-tech/chinese-vocab-tsv) | Quy tắc bảng từ vựng 7 cột |
| [`anki-apkg-builder`](https://github.com/vibeailabs-tech/chinese/tree/main/anki-apkg-builder) | Build `.apkg` từ TSV, có cây deck |
| [Bộ skill `chinese`](https://github.com/vibeailabs-tech/chinese) | Gộp các skill tiếng Trung |

Skill này gộp bước lọc từ và bước đóng gói: **một file Word → một `.apkg`**, không tạo cây deck.

## Cài đặt

### Cursor

Clone repo rồi trỏ agent vào `docx-to-apkg/SKILL.md`, hoặc copy thư mục skill vào skills của Cursor:

```bash
git clone https://github.com/vibeailabs-tech/docx-to-apkg /tmp/docx-to-apkg
cp -R /tmp/docx-to-apkg/docx-to-apkg ~/.cursor/skills/
```

### Hermes Agent

```bash
hermes skills install vibeailabs-tech/docx-to-apkg/docx-to-apkg
```

Kiểm tra:

```bash
hermes skills list | grep docx-to-apkg
```

Skill được nạp từ phiên mới, hoặc `/skills install --now ...` trong phiên đang chạy. Cập nhật: `hermes skills update docx-to-apkg`. Gỡ: `hermes skills uninstall docx-to-apkg`.

### Devin / Claude Code

```bash
git clone https://github.com/vibeailabs-tech/docx-to-apkg /tmp/docx-to-apkg

# Devin (global)
cp -R /tmp/docx-to-apkg/docx-to-apkg ~/.config/devin/skills/
# Claude Code (global)
cp -R /tmp/docx-to-apkg/docx-to-apkg ~/.claude/skills/
```

## Yêu cầu

- `python3` — không cần `pip install`.
- File đầu vào là `.docx`. File `.doc` cũ phải chuyển trước:

```bash
textutil -convert docx FILE.doc
# hoặc
soffice --headless --convert-to docx FILE.doc
```

- Anki dùng note type **SpoonFed++**. Gói thẻ copy đúng id note type từ `docx-to-apkg/template/sample.apkg`, nên Anki tái sử dụng type sẵn có thay vì tạo bản trùng.

## Cách dùng với agent

Đưa đường dẫn file và nói, ví dụ:

- "Chạy docx-to-apkg với file `~/Desktop/bai_doc.docx`"
- "Lọc từ gạch chân rồi làm bộ thẻ Anki"
- "docx sang apkg: `/path/to/bai_doc.docx`"

Tuỳ chọn:

- Ngày đọc (không nói thì dùng hôm nay theo múi giờ của bạn).
- Chữ phồn thể (không nói thì viết giản thể).

Không cần đặt tên deck. Skill không tạo deck.

Agent sẽ:

1. Chọn mã bài `YYYY_MM_DD_STT` (số thứ tự trong ngày, không ghi đè file cũ).
2. Lọc từ gạch chân ra `docx-to-apkg/extract/<id>.txt`.
3. Bỏ từ đã có trong cột `Vocabulary` của mọi `tsv/*.tsv`.
4. Viết `docx-to-apkg/tsv/<id>.tsv` đúng 7 cột, rồi kiểm tra.
5. Chạy `build_apkg.py` → `output/<id>.apkg`.

Trả lời: số từ đã lọc, từ bị bỏ vì trùng (nếu có), đường dẫn extract / TSV / apkg, kết quả kiểm tra 7 cột, và một dòng hướng dẫn import.

Danh sách rỗng, hoặc mọi từ đều trùng: dừng. Không bịa từ, không viết TSV hay apkg rỗng.

## Chạy tay không cần agent

Bước 1 — lọc từ gạch chân:

```bash
python3 "docx-to-apkg/scripts/docx_underlined_words.py" "/path/to/file.docx" -o "docx-to-apkg/extract/2026_11_23_01.txt"
```

Bước 2 — tự viết `docx-to-apkg/tsv/2026_11_23_01.tsv` theo quy tắc dưới, rồi kiểm tra:

```bash
awk -F'\t' 'NF!=7{bad++} END{print NR" rows, "(bad+0)" bad"}' "docx-to-apkg/tsv/2026_11_23_01.tsv"
```

Bước 3 — đóng gói khi kiểm tra ra `0 bad`:

```bash
python3 "docx-to-apkg/scripts/build_apkg.py" "docx-to-apkg/tsv/2026_11_23_01.tsv"
```

File kết quả: `output/2026_11_23_01.apkg`.

## Kết quả từng bước

### 1. `extract/YYYY_MM_DD_STT.txt`

Mỗi dòng một từ/cụm từ. Quy tắc:

- Bỏ trùng, giữ lần xuất hiện đầu tiên (không phân biệt hoa/thường).
- Các đoạn gạch chân **liền nhau** (không có chữ chen giữa) được gộp thành một cụm.
- Bỏ dấu câu và khoảng trắng thừa ở hai đầu.
- Nhận cả gạch chân trực tiếp lẫn gạch chân qua character style của Word.
- Chỉ đọc phần thân tài liệu. Từ gạch chân trong header, footer, footnote, endnote, comment bị bỏ qua.

### 2. `tsv/YYYY_MM_DD_STT.tsv`

- Cùng mã với extract và apkg. `STT` bắt đầu `01` mỗi ngày.
- Quét `extract/`, `tsv/`, và `output/` cùng lúc khi chọn số; không ghi đè.
- UTF-8, phân tách bằng tab, có dòng tiêu đề, đúng 7 cột:

| Cột TSV | Field SpoonFed++ | Quy tắc |
|---|---|---|
| `Vocabulary` | `Hanzi` | Chỉ chữ Hán |
| `Pronunciation` | `Pinyin` | Pinyin có dấu thanh |
| `English` | `English` | Nghĩa tiếng Anh ngắn |
| `Vietnamese` | `Vietnamese` | Nghĩa tiếng Việt ngắn |
| `Example` | `Example` | Một câu dễ, phải chứa từ vựng |
| `Sentence Pinyin` | `Sentence Pinyin` | Pinyin cả câu, có dấu thanh |
| `Sentence Translation` | `Sentence Translation` | Dịch câu sang tiếng Anh |
| (không có) | `Audio`, `SP` | Để trống cho HyperTTS |

Trong một bảng, `Vocabulary` không được trùng. Ô không chứa ký tự tab.

### 3. `output/YYYY_MM_DD_STT.apkg`

- Một file `.apkg` cho một bài đọc.
- Mọi thẻ thuộc deck **Default**.
- Note type **SpoonFed++**, giữ nguyên id trong `template/sample.apkg`.
- GUID = `guid_for(note type id, Hanzi)`. Cùng một chữ Hán + cùng note type = cùng một note khi import.

## Nhập vào Anki

File → Import → chọn `.apkg`.

Anki nhận note theo note type cộng Hanzi:

| Muốn làm gì | Cách |
|---|---|
| Import từ mới | Import bình thường |
| Từ đã có trong Anki, note đó **đã có audio** | Chọn **Update notes: Never** |
| Rebuild rồi import lại với mặc định **If newer** | Anki ghi đè note, xoá audio HyperTTS |

Nếu `output/<id>.apkg` đã tồn tại, đừng build lại trừ khi bạn chủ động rebuild. Lúc import file rebuild, dùng **Update notes: Never** để audio cũ còn.

Cùng một `Hanzi` ở hai bài khác nhau vẫn là một note. Bài sau không tạo thẻ mới cho chữ đó — vì vậy bước 3 của skill đã bỏ từ trùng trước khi viết TSV.

## Cấu trúc repo

```
.
├── README.md
├── docx-to-apkg/                      skill cho agent
│   ├── SKILL.md
│   ├── scripts/
│   │   ├── docx_underlined_words.py   lọc từ gạch chân
│   │   └── build_apkg.py              đóng gói .apkg
│   ├── references/chinese-vocab-tsv.md
│   ├── template/sample.apkg           khuôn SpoonFed++, chỉ đọc
│   ├── extract/                       danh sách từ, một dòng một từ
│   └── tsv/                           bảng từ vựng 7 cột
└── output/                            chỉ nhận file .apkg
```

`<skill-dir>` là `docx-to-apkg/`. `<output-dir>` là `output/` cạnh skill, không nằm trong skill. Bảng TSV để trong skill để thư mục import không đầy file làm việc.

## Lưu ý

- Bước lọc từ là script, kết quả cố định. Pinyin, nghĩa và câu ví dụ do mô hình viết — xem lại trước khi học.
- Không xoá hay sửa `template/sample.apkg`. Đó là file export thật từ Anki. Nếu bạn đổi SpoonFed++ (thêm/bớt field, đổi template), export một deck chỉ dùng note type đó, tick **Support older Anki versions**, rồi thay file khuôn.
- File `.apkg` là ZIP kiểu legacy 2: `meta`, `collection.anki21`, `collection.anki2` (stub), `media` = `{}`. Không có audio.
- `evals/` trong thư mục skill là bộ kiểm tra nhỏ (bài ngắn, bài trùng từ, bài không gạch chân).
