# Git Branch & Merge Guide

Tài liệu nhanh để làm việc với **Git Branch và Merge**.

---

# 1. Cấu hình Git lần đầu

```bash
git config --global user.name "Jane Doe"
git config --global user.email "jane@example.com"
```

Kiểm tra cấu hình:

```bash
git config --list
```

---

# 2. Làm việc với Branch

## Tạo branch mới

```bash
git branch <ten_nhanh>
```

Ví dụ:

```bash
git branch feature/login
```

---

## Tạo branch và chuyển sang branch đó

```bash
git checkout -b <ten_nhanh>
```

hoặc (Git mới):

```bash
git switch -c <ten_nhanh>
```

---

## Xem danh sách branch

```bash
git branch
```

Xem tất cả branch (bao gồm remote):

```bash
git branch -a
```

---

## Chuyển branch

```bash
git checkout <ten_nhanh>
```

hoặc

```bash
git switch <ten_nhanh>
```

---

# 3. Push branch lên Remote

```bash
git push origin <ten_nhanh>
```

Ví dụ:

```bash
git push origin feature/login
```

---

# 4. Merge Branch

Ví dụ: merge `feature/login` vào `main`.

## Bước 1: cập nhật branch cần merge

```bash
git checkout feature/login
git pull origin feature/login
```

---

## Bước 2: chuyển về branch chính

```bash
git checkout main
git pull origin main
```

---

## Bước 3: merge branch

```bash
git merge feature/login
```

---

## Bước 4: push lên remote

```bash
git push origin main
```

---

# 5. Nếu xảy ra Conflict

Git sẽ báo:

```
CONFLICT (content)
```

### Ví dụ conflict

```
<<<<<<< HEAD
code ở main
=======
code ở feature/login
>>>>>>> feature/login
```

### Cách xử lý

1. Mở file bị conflict  
2. Chỉnh sửa code cho đúng  
3. Sau đó chạy:

```bash
git add .
git commit
```

---

# 6. Hủy Merge

Nếu merge bị lỗi và muốn quay lại trạng thái trước:

```bash
git merge --abort
```

---

# 7. Xóa Branch

## Xóa branch local

```bash
git branch -d <ten_nhanh>
```

## Xóa branch remote

```bash
git push origin --delete <ten_nhanh>
```

---

# 8. Kiểm tra trạng thái project

```bash
git status
```

---

# 9. Xem lịch sử commit

```bash
git log
```

Xem dạng gọn:

```bash
git log --oneline --graph
```

---

# 10. Flow làm việc phổ biến

```bash
# tạo branch mới
git checkout -b feature/login

# code xong
git add .
git commit -m "create login feature"

# push branch
git push origin feature/login

# merge vào main
git checkout main
git pull
git merge feature/login
git push origin main
```

---

# 11. Một số lệnh hữu ích

Fetch code mới từ remote:

```bash
git fetch
```

Pull code:

```bash
git pull
```

Push code:

```bash
git push
```

---

# 12. Quy tắc quan trọng khi Merge

Luôn nhớ:

```
Đứng ở nhánh muốn nhận code → merge nhánh kia vào
```

Ví dụ:

```
main nhận code từ feature/login
```

```bash
git checkout main
git merge feature/login
```
