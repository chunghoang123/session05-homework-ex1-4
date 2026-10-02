# Bài 1: Khôi phục commit đã mất bằng Git Reflog

## 1. Mục tiêu
- Hiểu Reflog lưu lịch sử hoạt động cục bộ (di chuyển HEAD, commit, reset).
- Dùng `git reflog` để truy vết commit bị `reset --hard` làm biến mất.
- Khôi phục commit mà không viết lại code thủ công.

## 2. Môi trường thực hành
- Git 2.51.0.windows.1
- Repo thử nghiệm cục bộ, nhánh `main`
- Thực hiện trong thư mục demo riêng trước khi nộp bài.

## 3. Các bước thực hiện

### Bước 1 — Khởi tạo dự án và tạo commit quan trọng
```bash
git init -b main
echo "# demo ex1" > README.md
git add .
git commit -m "init project"

echo "Day la tinh nang quan trong" > feature.txt
git add .
git commit -m "Them tinh nang quan trong"
```

Kiểm tra:
```bash
git log --oneline
```
Kết quả (hash ví dụ, máy bạn sẽ khác):
```
4a4e884 Them tinh nang quan trong
7e714af init project
```

### Bước 2 — Giả lập lỗi: reset hard làm mất commit
```bash
git reset --hard HEAD~1
```

Kiểm tra:
```bash
git log --oneline
# 7e714af init project
# -> commit "Them tinh nang quan trong" đã biến mất

ls
# feature.txt không còn tồn tại
Test-Path feature.txt  # False
```

Đây chính là hiện tượng đề bài mô tả: `git log` không còn thấy commit mới nhất.

### Bước 3 — Phát hiện nguyên nhân và tra cứu Reflog
`git log` chỉ hiển thị lịch sử *có thể tới được* từ HEAD hiện tại.
Reflog lưu mọi lần HEAD di chuyển, kể cả commit đã bị bỏ rơi:

```bash
git reflog
```
Kết quả thực tế:
```
7e714af HEAD@{0}: reset: moving to HEAD~1
4a4e884 HEAD@{1}: commit: Them tinh nang quan trong
7e714af HEAD@{2}: commit (initial): init project
```

Giải thích:
- `HEAD@{0}`: thao tác `reset` vừa gây lỗi.
- `HEAD@{1}`: chính là commit bị mất `4a4e884 Them tinh nang quan trong`.
- Dạng hiển thị đúng như gợi ý đề bài: `<hash> HEAD@{1}: commit: Them tinh nang quan trong`.

### Bước 4 — Khôi phục commit
Có 2 cách tương đương, đều dùng hash tìm được từ reflog:

Cách A (dùng trong demo, đưa HEAD về đúng commit):
```bash
git reset --hard 4a4e884
```

Cách B (nếu chỉ muốn xem trước, không di chuyển nhánh):
```bash
git checkout 4a4e884
# sau đó nếu ưng ý thì: git switch -c recovery && git switch main && git merge recovery
# hoặc: git reset --hard 4a4e884
```

> Thay `4a4e884` bằng hash thật trong `git reflog` của bạn.

### Bước 5 — Kiểm tra khôi phục thành công
```bash
git log --oneline
```
```
4a4e884 Them tinh nang quan trong
7e714af init project
```

```bash
cat feature.txt
# Day la tinh nang quan trong

git reflog
```
```
4a4e884 HEAD@{0}: reset: moving to 4a4e884
7e714af HEAD@{1}: reset: moving to HEAD~1
4a4e884 HEAD@{2}: commit: Them tinh nang quan trong
7e714af HEAD@{3}: commit (initial): init project
```

## 4. Kết quả đối chiếu yêu cầu
- [x] `git log --oneline` hiển thị lại đúng commit `Them tinh nang quan trong` với nội dung `feature.txt` ban đầu.
- [x] Không viết lại code thủ công, chỉ dùng Reflog + `reset --hard <hash>`.
- [x] `git reflog` chứng minh được quá trình truy vết.

## 5. Bài học rút ra
- `git reset --hard` không xóa object commit ngay, chỉ di chuyển HEAD. Object vẫn nằm trong `.git/objects` và Reflog giữ tham chiếu tới nó.
- Reflog là nhật ký *cục bộ*, mặc định giữ ~90 ngày. Muốn cứu dữ liệu phải làm sớm trước khi `git gc` dọn dẹp.
- Thói quen an toàn: trước `reset --hard`, chạy `git log --oneline -5` và `git status` để xác nhận.
