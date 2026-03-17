# 🗂️ Redis — Ôn tập hàng ngày

> Đọc mỗi sáng, hiểu rồi thì đọc lại để nhớ lâu hơn.

---

## 1. Redis `DECR` có rollback không?

**Không.** `DECR` là lệnh atomic — Redis thực thi xong là xong, không có khái niệm rollback.

Cái "rollback" trong code thực ra là **logic của app tự xử lý**:

```
value = DECR flash:iphone14:stock   ← Redis chỉ làm mỗi việc này

if value < 0:
    INCR flash:iphone14:stock       ← app tự cộng lại
    return "hết hàng"
else:
    # tiếp tục tạo đơn
```

> 💡 **Nhớ:** Redis trả về kết quả sau khi trừ. App đọc số đó, tự quyết định làm gì tiếp — Redis không biết và không quan tâm.

---

## 2. Redis có giống transaction của database không?

**Không giống hoàn toàn.** Redis có `MULTI/EXEC` để gom nhiều lệnh lại, nhưng khác database ở điểm quan trọng:

| | Database transaction | Redis MULTI/EXEC |
|---|---|---|
| Rollback khi lỗi | ✅ Có | ❌ Không |
| Đọc giữa chừng | ✅ Có thể | ❌ Không — lệnh chỉ chạy khi EXEC |
| Isolation | ✅ Đầy đủ | ⚠️ Hạn chế |

Redis `MULTI/EXEC` chỉ đảm bảo các lệnh **chạy liền nhau không bị xen ngang**, không phải transaction thật sự theo nghĩa ACID.

> 💡 **Nhớ:** Dùng Redis cho tốc độ và tính atomic của từng lệnh đơn lẻ, không phải để thay thế transaction của database.

---

## 3. Replica là gì?

**Replica là bản sao đồng bộ real-time của Master** — không phải backup theo lịch, mà là một Redis instance thật đang chạy song song, sync từng lệnh một với Master.

### Cách hoạt động

```
App gửi: DECR flash:iphone14:stock
         ↓
    Master thực thi → stock: 100 → 99
    Master ghi lệnh vào replication log
         ↓ propagate lệnh DECR
    Replica 1 nhận lệnh → tự thực thi lại → stock: 100 → 99
    Replica 2 nhận lệnh → tự thực thi lại → stock: 100 → 99
```

### ❗ Điểm quan trọng

Replica **không nhận kết quả** (99) từ Master — nó nhận **chính cái lệnh** (`DECR`) rồi tự chạy lại từ đầu. Đây là lý do Replica có thể promote lên làm Master bất cứ lúc nào mà không cần setup lại.

### Replica vs Backup

| | Backup | Replica |
|---|---|---|
| Khi nào tạo | Theo lịch (mỗi ngày, mỗi giờ) | Liên tục, real-time |
| Mất data không | Có — từ lúc backup đến lúc chết | Gần như không |
| Có thể dùng ngay | Phải restore trước | Promote ngay lập tức |
| Đang chạy không | Không | Luôn sống |

> 💡 **Nhớ:** Backup và Replica **dùng cùng nhau**, không thay thế nhau. Replica để failover nhanh, Backup để phục hồi khi cả cluster bị xóa nhầm.

---

## 4. Sentinel là gì?

**Sentinel là hệ thống giám sát và tự động failover** — không lưu data, chỉ có một nhiệm vụ: theo dõi Master, phát hiện khi Master chết, quyết định Replica nào lên thay.

### Ví dụ thực tế — Flash sale Shopee

```
12:00:00  Flash sale bắt đầu, 10,000 người bấm mua
12:00:03  Master Redis đột ngột chết, stock đang = 57
          Sentinel 1, 2, 3 đang ping Master...
          → không nhận được pong
          → S1 hỏi S2: "mày có thấy Master chết không?"
          → S2 đồng ý → quorum đạt (2/3)
          → kết luận: Master thật sự chết (ODOWN)
12:00:06  Sentinel bầu leader, promote Replica 1 → Master mới
          → stock = 57 (đã được replicate trước đó, không mất)
12:00:06  App hỏi Sentinel địa chỉ mới → tự reconnect
          Flash sale tiếp tục từ đúng số 57
```

### Tại sao cần 3 Sentinel, không phải 1?

Nếu chỉ có 1 Sentinel mà chính nó bị mất mạng tạm thời → nó tưởng Master chết → promote Replica lên → giờ có **2 Master cùng lúc** → data vỡ (split-brain).

Với 3 Sentinel cần quorum 2: dù 1 Sentinel mất mạng, 2 Sentinel còn lại không đủ để một mình ra quyết định sai.

> 💡 **Ví von:** Sentinel giống **Hội đồng quản trị** của công ty.
> - **Master** = Giám đốc — người duy nhất có quyền ký (ghi data)
> - **Replica** = Trợ lý — ghi chép mọi quyết định của sếp, sẵn sàng lên thay
> - **Sentinel** = Hội đồng quản trị — không điều hành, chỉ giám sát và bổ nhiệm khẩn cấp khi sếp倒
> - **Quorum** = Điều lệ công ty — phải đủ số thành viên đồng ý thì nghị quyết mới có hiệu lực

---

## 5. Redis Cluster là gì?

**Cluster giải quyết vấn đề Sentinel không giải quyết được:** data quá lớn, vượt RAM của 1 máy.

Thay vì 1 Master giữ tất cả, Cluster **chia nhỏ data ra nhiều Master** theo cơ chế **16384 hash slot**.

### Cơ chế hash slot

```
CRC16("flash:iphone14:stock") % 16384 = slot 3200  → Master A
CRC16("flash:samsung:stock")  % 16384 = slot 9100  → Master B
CRC16("flash:macbook:stock")  % 16384 = slot 14500 → Master C
```

App không cần biết logic này — Redis client tự tính và route đúng chỗ.

### Ví dụ flash sale Shopee

```
Shopee có 1,000,000 sản phẩm flash sale
→ 1 Master 16GB RAM không chứa hết

Cluster chia ra:
  Master A: slot 0–5460      (iphone, airpods, ...)
  Master B: slot 5461–10922  (samsung, xiaomi, ...)
  Master C: slot 10923–16383 (macbook, laptop, ...)

Mỗi Master có Replica riêng để HA.

Khi Master A chết:
  → chỉ shard A tạm ngưng vài giây
  → Replica A promote lên
  → Master B và C không hề biết, vẫn chạy bình thường
  → Shopee vẫn bán được Samsung và MacBook trong lúc iPhone đang failover
```

### Cluster vs Sentinel

| | Sentinel | Cluster |
|---|---|---|
| Giải quyết | Master chết thì ai lên? | Data quá lớn thì làm sao? |
| Số Master | 1 | Nhiều (tối thiểu 3) |
| Failover | Cả hệ thống chờ | Chỉ shard bị ảnh hưởng |
| Độ phức tạp | Thấp | Cao |
| Khi nào dùng | 99% dự án | Data vượt RAM 1 máy |

### Cái giá phải trả khi dùng Cluster

**Multi-key command bị giới hạn:** `MGET key1 key2` chỉ chạy được nếu cả 2 key cùng shard. Fix bằng **hash tag**:

```
# Không dùng được nếu khác shard:
MGET flash:iphone:stock flash:samsung:stock

# Fix: dùng hash tag {} để ép cùng shard:
MGET {flash}:iphone:stock {flash}:samsung:stock
#     ^^^^^^ phần này dùng để hash, 2 key sẽ cùng slot
```

> 💡 **Nhớ:** Sentinel + Replica là đủ cho 99% dự án. Chỉ nghĩ đến Cluster khi data thật sự vượt RAM của 1 máy.

---

*Cập nhật lần cuối: 2026*
