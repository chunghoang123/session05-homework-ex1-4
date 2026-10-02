# Bài 3: Xử lý xung đột phức tạp trong quá trình Rebase

## 1. Mục tiêu
- Phân biệt giải quyết xung đột khi Merge (1 lần, 1 merge commit) vs khi Rebase (từng commit, replay từng bước).
- Thực hành `rebase --continue` qua nhiều chặng conflict.
- Đưa lịch sử `feature-api` lên thẳng trên đầu `main`, không merge commit phụ.

## 2. Dựng kịch bản (đúng gợi ý đề bài)

### Bước 1 — Tạo `config.json` ban đầu trên `main`
```bash
git init -b main
# config.json:
# {
#   "port": 8080,
#   "debug": false
# }
git add config.json && git commit -m "init config"   # 4173008
```

### Bước 2 — Tạo nhánh `feature-api` và commit 2 lần
```bash
git checkout -b feature-api
# sửa "port": 9000
git add config.json && git commit -m "feat: change port"  # 6f8855d
# sửa "debug": true
git add config.json && git commit -m "feat: enable debug" # 7e942bb
# lúc này feature-api: { "port": 9000, "debug": true }
```

### Bước 3 — Quay lại `main`, tạo 2 commit mới song song
```bash
git checkout main
# sửa "port": 8081
git add config.json && git commit -m "update port on main" # 4559cee
# thêm "env": "production"
# => { "port": 8081, "debug": false, "env": "production" }
git add config.json && git commit -m "add env config"      # 9f2c73b
```

Sơ đồ trước rebase:
```
* 7e942bb (feature-api) feat: enable debug
* 6f8855d feat: change port
| * 9f2c73b (main) add env config
| * 4559cee update port on main
|/
* 4173008 init config
```
Cả hai nhánh sửa cùng dòng `port` → chắc chắn conflict khi rebase.

## 3. Tiến hành rebase
```bash
git checkout feature-api
git rebase main
```

### Chặng conflict 1 — commit `feat: change port` (6f8855d)
Lỗi:
```
Auto-merging config.json
CONFLICT (content): Merge conflict in config.json
error: could not apply 6f8855d... feat: change port
```

`git status`:
```
interactive rebase in progress; onto 9f2c73b
Last command done (1 command done):
   pick 6f8855d # feat: change port
Next command to do (1 remaining command):
   pick 7e942bb # feat: enable debug
You are currently rebasing branch 'feature-api' on '9f2c73b'.
Unmerged paths:
  both modified: config.json
```

Mở `config.json` thấy:
```json
{
<<<<<<< HEAD
  "port": 8081,
  "debug": false,
  "env": "production"
=======
  "port": 9000,
  "debug": false
>>>>>>> 6f8855d (feat: change port)
}
```

**Cách giải quyết:** giữ ý định của feature (`port: 9000`), đồng thời không phá code main (`env: production`):
```json
{
  "port": 9000,
  "debug": false,
  "env": "production"
}
```

Tiếp tục:
```bash
git add config.json
GIT_EDITOR=true git rebase --continue
# [detached HEAD 62d0fab] feat: change port
```
> Dùng `GIT_EDITOR=true` để bỏ qua popup vim xác nhận message trong môi trường demo. Thao tác thủ công chỉ cần `git rebase --continue`.

### Chặng conflict 2 — commit `feat: enable debug` (7e942bb)
Ngay sau chặng 1, Git replay commit tiếp theo và conflict tiếp:
```
Auto-merging config.json
CONFLICT (content): Merge conflict in config.json
error: could not apply 7e942bb... feat: enable debug
```

`config.json` lúc này:
```json
{
  "port": 9000,
<<<<<<< HEAD
  "debug": false,
  "env": "production"
=======
  "debug": true
>>>>>>> 7e942bb (feat: enable debug)
}
```

**Cách giải quyết:** giữ `debug: true` của feature, giữ `port: 9000` và `env: production` đã hòa giải ở chặng 1:
```json
{
  "port": 9000,
  "debug": true,
  "env": "production"
}
```

Tiếp tục:
```bash
git add config.json
GIT_EDITOR=true git rebase --continue
# [detached HEAD 03c213e] feat: enable debug
# Successfully rebased and updated refs/heads/feature-api.
```

## 4. Kiểm tra sau rebase
```bash
git status
# On branch feature-api
# nothing to commit, working tree clean
```

```bash
git log --graph --oneline --all
```
```
* 03c213e (feature-api) feat: enable debug
* 62d0fab feat: change port
* 9f2c73b (main) add env config
* 4559cee update port on main
* 4173008 init config
```
(Hình chụp cần nộp: đồ thị thẳng tắp trên, không có nhánh rẽ hay merge commit.)

```bash
cat config.json
# {
#   "port": 9000,
#   "debug": true,
#   "env": "production"
# }
```

## 5. Đối chiếu yêu cầu
- [x] Giải quyết xung đột thủ công ở từng chặng (2 lần), không dùng merge thường.
- [x] Không phá code main: giữ `port` mới của feature + `env: production` của main.
- [x] Lịch sử thẳng, không merge commit; 2 commit `feature-api` nằm nối tiếp sau commit mới nhất của `main`.

## 6. So sánh Merge vs Rebase
| Tiêu chí | Merge | Rebase (bài này) |
|---|---|---|
| Số lần giải conflict | 1 lần duy nhất | Từng commit replay (2 lần) |
| Lịch sử | Giữ nguyên + thêm merge commit, có nhánh rẽ | Viết lại hash (62d0fab, 03c213e), tuyến tính |
| Khi nào dùng | Nhánh public/shared, cần giữ ngữ cảnh | Nhánh feature cục bộ, muốn lịch sử sạch trước khi merge/PR |
| Lệnh tiếp tục | `git commit` sau khi sửa | `git add` + `git rebase --continue` (hoặc `--skip` / `--abort`) |

Lệnh cứu hộ khi làm sai giữa chừng:
```bash
git rebase --abort   # quay về trước khi rebase
git rebase --skip    # bỏ qua commit hiện tại (cẩn trọng, dễ mất code)
```
