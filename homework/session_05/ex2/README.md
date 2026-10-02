# Bài 2: Tái cấu trúc lịch sử commit bằng Interactive Rebase

## 1. Mục tiêu
- Dùng Interactive Rebase để sửa lịch sử cục bộ.
- Gộp (squash) 3 commit nhỏ thành 1 commit duy nhất.
- Đổi thông điệp thành `feat: hoan thien module authentication`.
- Xóa (drop) commit chứa file rác `temp.txt`.

## 2. Chuẩn bị: tạo 4 commit thử nghiệm
```bash
git init -b main
echo "# base" > README.md
git add . && git commit -m "init base"

echo "// khoi tao module auth" > auth.js
git add auth.js && git commit -m "feat: khoi tao module auth"

echo "// fix typo" >> auth.js
git add auth.js && git commit -m "fix typo"

echo "// utility functions" >> auth.js
git add auth.js && git commit -m "adds utility functions"

echo "debug temp" > temp.txt
git add temp.txt && git commit -m "add temp file for debug"
```

Lịch sử trước rebase (`git log --oneline`):
```
cd7b02b add temp file for debug
2e8ead8 adds utility functions
a35d0af fix typo
0a01a50 feat: khoi tao module auth
4ea3da3 init base
```

Trong đó 4 commit cuối (`HEAD~4`) là đối tượng cần tái cấu trúc.

## 3. Chạy Interactive Rebase
```bash
git rebase -i HEAD~4
```

### 3.1. Màn hình soạn thảo lần 1 — cấu hình pick / squash / drop

Git mở trình soạn thảo (vim / VS Code / Notepad) với nội dung gốc:
```text
pick 0a01a50 feat: khoi tao module auth
pick a35d0af fix typo
pick 2e8ead8 adds utility functions
pick cd7b02b add temp file for debug

# Rebase 4ea3da3..cd7b02b onto 4ea3da3 (4 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup [-C | -c] <commit> = like "squash" but keep only the previous
#                    commit's log message, unless -C is used, in which case
#                    keep only this commit's message; -c is same as -C but
#                    opens the editor
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
# l, label <label> = label current HEAD with a name
# t, reset <label> = reset HEAD to a label
# m, merge [-C <commit> | -c <commit>] <label> [# <oneline>]
```

Sửa thành (đúng yêu cầu đề bài):
```text
pick 0a01a50 feat: khoi tao module auth
squash a35d0af fix typo
squash 2e8ead8 adds utility functions
drop cd7b02b add temp file for debug
```
- Commit 1 giữ `pick`.
- Commit 2, 3 đổi `pick` → `squash` (viết tắt `s`).
- Commit 4 đổi `pick` → `drop` (viết tắt `d`).

> Ảnh chụp màn hình cần nộp: chụp đúng giao diện trên sau khi sửa xong, trước khi Save/Close.
> Nếu dùng VS Code, tiêu đề tab thường là `git-rebase-todo`.

Lưu và đóng editor. Git bắt đầu áp dụng:
```
Rebasing (2/4)
Rebasing (3/4)
```

### 3.2. Màn hình soạn thảo lần 2 — gộp thông điệp commit

Vì có `squash`, Git mở tiếp editor gộp message:
```text
# This is a combination of 3 commits.
# This is the 1st commit message:

feat: khoi tao module auth

# This is the commit message #2:

fix typo

# This is the commit message #3:

adds utility functions
```

Xóa hết và thay bằng đúng 1 dòng theo yêu cầu:
```text
feat: hoan thien module authentication
```

Lưu và đóng. Git hoàn tất:
```
[detached HEAD 81a25f6] feat: hoan thien module authentication
Successfully rebased and updated refs/heads/main.
```

> Trong demo tự động, bước này được thực hiện bằng `GIT_EDITOR` ghi đè message, kết quả tương đương thao tác thủ công.

## 4. Kiểm tra sau rebase
```bash
git log --oneline
```
```
81a25f6 feat: hoan thien module authentication
4ea3da3 init base
```

```bash
git show --stat HEAD
# commit 81a25f64510d609f6438d77d34242e30bd352a35
# feat: hoan thien module authentication
#  auth.js | 3 +++
```

```bash
ls
# auth.js, README.md  (không còn temp.txt)
Test-Path temp.txt  # False

cat auth.js
# // khoi tao module auth
# // fix typo
# // utility functions
```

## 5. Đối chiếu yêu cầu
- [x] Gộp 3 commit nhỏ (`feat: khoi tao...`, `fix typo`, `adds utility...`) thành 1 commit.
- [x] Thông điệp sau gộp: `feat: hoan thien module authentication`.
- [x] Commit `add temp file for debug` bị `drop`, file `temp.txt` biến mất.
- [x] Lịch sử sạch, chỉ còn 1 commit tính năng (+ commit base `init base` có từ trước).

## 6. Lưu ý
- Chỉ rebase lịch sử *chưa push*. Nếu đã push, phải `push --force-with-lease` và thông báo cho cả nhóm.
- `squash` giữ và cho phép sửa tất cả message; `fixup` vứt message của commit gộp (dùng khi message phụ không quan trọng).
- `drop` xóa hẳn commit; muốn an toàn có thể dùng `git rebase --abort` để thoát giữa chừng.
