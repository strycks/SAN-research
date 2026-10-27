# Cho hai cloud OpenStack dùng chung dung lượng SAN ("đầy thì tràn sang tủ khác")

**Liên quan:** `SAN.md` (tủ và driver Cinder) · `SAN_director.md` (director, zoning)
**Bản OpenStack:** Yoga, 2023.1 Antelope, 2024.1 Caracal. Các khẳng định về Cinder đã kiểm tra trên mã nguồn 24.0.0.
**Ngày:** 27/10/2026

---

## 1. Bài toán

```
Cloud 1:  3 compute ──► Switch A1, B1 ──► Tủ 1   (đã dùng 20%)
Cloud 2:  3 compute ──► Switch A2, B2 ──► Tủ 2   (đã dùng 80%)
```

- Có hai cụm OpenStack **độc lập**, mỗi cụm có Nova và Cinder riêng.
- Mỗi cụm chỉ có đường tới tủ của mình. Tủ 2 sắp đầy trong khi tủ 1 còn trống nhiều.
- Mong muốn: tủ nào đầy thì tự "tràn" sang tủ kia, không bị gắn cố định theo đường SAN.
- Ràng buộc của dự án: **bắt buộc dùng SAN director**.
- Giả định: "usage" ở đây là **dung lượng**. Nếu ý là hiệu năng (IOPS) thì Cinder mặc định không cân theo IOPS, phải dùng QoS hoặc `goodness_function` dựa trên số liệu mà driver báo lên.

---

## 2. Kết luận

1. **Director không tự cân bằng dung lượng.** Director là thiết bị mạng: nó cân bằng lưu lượng trên các đường nối, không quyết định dữ liệu nằm ở tủ nào.
2. Việc chọn volume nằm ở tủ nào là của **Cinder**. Hiện cloud 2 bị "cố định" vào tủ 2 vì **cấu hình Cinder 2 chỉ khai báo một backend là tủ 2**, không phải vì dây cáp.
3. Cách giải quyết:
   - nối cả hai cloud vào cả hai tủ qua director;
   - mỗi tủ là một pool, hai cloud dùng chung;
   - mỗi Cinder khai báo hai backend: tủ "nhà" và tủ kia.

   Khi đó Cinder tự đặt volume mới sang tủ còn chỗ.
4. Volume cũ trên tủ 2 không tự chuyển đi. Phải chạy `volume migrate` chủ động.
5. Hai cloud dùng chung toàn bộ chỗ trống của cả hai tủ, không phải chia cố định. Đổi lại, trên tủ **không có hạn mức riêng cho từng cloud**: một cloud có thể dùng hết chỗ của cloud kia. Kiểm soát bằng quota Cinder và `reserved_percentage` (mục 4).

---

## 3. Thiết kế

### 3.1 Mạng SAN

```
        Cloud 1                      Cloud 2
   compute 1, 2, 3              compute 4, 5, 6
          |                            |
          | cổng HBA 1                 | cổng HBA 1
          |                            |
          +------------+  +------------+
                       |  |
                  +----+--+----+
                  | Director A |   (Fabric A)
                  +----+--+----+
                       |  |
          +------------+  +------------+
          |                            |
        Tủ 1                         Tủ 2
```

Fabric B làm giống hệt: cổng HBA 2 của mọi compute nối vào Director B, Director B nối tới cả Tủ 1 và Tủ 2. **Director A và Director B không nối với nhau.**

- **Dùng 2 director, mỗi fabric một cái.** Một director dù có dự phòng bên trong vẫn chỉ là một fabric. Cấu hình sai zone hoặc lỗi firmware là mất cả fabric đó.
- **Các switch cũ:**
  - *Collapsed core* (khuyến nghị cho quy mô hiện tại): compute và tủ cắm thẳng vào director. Switch cũ để dành cho lab.
  - *Core-edge*: giữ switch cũ làm lớp biên, nối ISL lên director. Chọn cách này khi compute nằm ở các rack xa nhau. **Switch và director phải cùng hãng**, vì Brocade và Cisco không chung fabric được với nhau.
- **Số cổng:** mỗi fabric cần khoảng 6 cổng cho compute, cộng khoảng 4 cổng mỗi tủ × 2 tủ, tức khoảng 14 cổng. Director nhỏ nhất (Brocade X7-4, Cisco MDS 9706) có hàng trăm cổng. Về kỹ thuật, hai switch thường cũng đáp ứng được bài toán này. Director mang lại thêm độ bền (linh kiện hỏng không làm mất fabric) và chỗ để mở rộng (xem `SAN_director.md`). Khi viết đề xuất nên ghi rõ lý do dự án bắt buộc dùng director.

### 3.2 Zoning (làm bằng tay trên director, cả fabric A và B)

| Zone | Mục đích |
|---|---|
| Compute cloud 1 ↔ cổng tủ 1 | Như hiện tại |
| Compute cloud 2 ↔ cổng tủ 2 | Như hiện tại |
| **Compute cloud 2 ↔ cổng tủ 1** | Mới: cho cloud 2 tràn sang tủ 1 |
| Compute cloud 1 ↔ cổng tủ 2 | Tùy chọn, làm khi cần đối xứng |

Không dùng auto-zoning của Cinder: driver Brocade đã bị đánh dấu unsupported, driver Cisco gần như không còn ai bảo trì (`SAN_director.md` mục 5).

### 3.3 Pool trên tủ

```
Tủ 1 = Pool 1 (Cinder 1 là tủ nhà, Cinder 2 tràn sang)
Tủ 2 = Pool 2 (Cinder 2 là tủ nhà, Cinder 1 tràn sang nếu cần)
```

**Mỗi tủ một pool, hai Cinder cùng dùng.** Dung lượng không bị tính sai: scheduler lấy `free_capacity_gb` và `provisioned_capacity_gb` từ số driver đọc trên tủ, đã gồm cả phần cloud kia đang dùng. Hitachi, IBM và Unity đều báo hai số này. Chỉ `allocated_capacity_gb` là Cinder tự cộng từ volume của mình, và nó chỉ dùng thay khi driver không báo `provisioned_capacity_gb`. Tên volume trên tủ có UUID nên không trùng. Những chỗ có thể đụng nhau được liệt kê ở mục 4.

### 3.4 Cinder chọn tủ thế nào

Cinder 2 khai báo hai backend (tủ 2 là tủ nhà, tủ 1 để tràn) dưới cùng một volume type. Khi có yêu cầu tạo volume, scheduler của Cinder làm hai bước:

1. **Lọc:** loại các pool không còn đủ chỗ. Có thể đặt ngưỡng "coi như đầy" bằng `reserved_percentage` trên từng backend, ví dụ chừa lại 15% dung lượng.
2. **Chấm điểm và chọn:**
   - Mặc định: chọn pool **còn trống nhiều nhất**. Hai pool được lấp đều, không ưu tiên tủ nhà.
   - Nếu muốn **dùng tủ nhà trước, đầy mới tràn sang**: bật thêm cơ chế chấm điểm theo công thức (`GoodnessWeigher`) và cho tủ nhà điểm cao hơn. Cơ chế này không bật sẵn.

Cinder 1 làm tương tự nếu cần tràn sang tủ 2.

### 3.5 Xử lý tủ 2 đang ở mức 80%

Scheduler chỉ quyết định chỗ đặt cho volume **mới**. Volume cũ phải chuyển chủ động bằng lệnh migrate của Cinder. Vì tủ 1 đã là backend của chính Cinder 2, đây là chuyển volume giữa hai backend trong cùng một cloud.

| Trạng thái volume | Ai copy dữ liệu | Lưu ý |
|---|---|---|
| Chưa gắn vào máy ảo | Cinder (qua host chạy cinder-volume) | Host đó cần có đường tới cả hai tủ |
| Đang gắn vào máy ảo | Nova/libvirt copy trong lúc máy ảo vẫn chạy | Chậm và tốn băng thông, nên làm giờ thấp điểm |

Cinder không migrate được volume đang có snapshot, nằm trong group hoặc có replication (`volume/api.py`, `migrate_volume`). Các volume này phải xóa snapshot hoặc gỡ khỏi group trước, hoặc giữ lại trên tủ 2.

Tính năng di chuyển volume do tủ tự làm **chỉ chạy được trong cùng một tủ**, không chuyển được sang tủ khác.

---

## 4. Những điểm dễ gây sự cố khi hai cloud dùng chung một tủ

| Vấn đề | Áp dụng cho | Chi tiết | Cách tránh |
|---|---|---|---|
| **Trùng hostname compute** | **Dell Unity** | Driver Unity tìm và tạo host object trên tủ theo `connector['host']`, tức hostname của compute (`unity/client.py`, `create_host_wo_lock`). Hai cloud cùng có máy tên `compute-1` thì sẽ dùng chung một host object, dẫn đến map LUN nhầm máy | Đặt hostname khác nhau giữa hai cloud, ví dụ `c1-compute-1` và `c2-compute-1` |
| Trùng hostname compute | IBM FlashSystem | **Không bị.** Driver tìm host theo WWPN và tạo tên host kèm hậu tố ngẫu nhiên 8 chữ số | Vẫn nên đặt hostname khác nhau để dễ đọc trên tủ |
| Trùng số LDEV | Hitachi | Hai Cinder cùng tạo volume trên một tủ có thể cùng chọn một số LDEV còn trống | Chia `hitachi_ldev_range` riêng cho mỗi cloud, ví dụ `0-4999` và `5000-9999` |
| Snapshot pool | Hitachi | Nếu để trống `hitachi_snap_pool`, snapshot nằm chung pool với volume gốc (`hbsd_rest.py`, `_get_snap_pool_id`) | Để mặc định là được. LDEV của snapshot cũng nằm trong `hitachi_ldev_range` nên không trùng |
| Một cloud dùng hết chỗ | Mọi hãng | Pool dùng chung nên không có hạn mức riêng cho từng cloud trên tủ | Đặt quota Cinder cho project. `reserved_percentage` đặt riêng ở từng Cinder: Cinder 2 khai tủ 1 với mức cao hơn Cinder 1 (ví dụ 30% so với 10%), để cloud 2 dừng tràn sớm, chừa chỗ cho cloud 1 |
| Nhận nhầm volume của cloud kia | Hitachi, IBM, Unity | Ba driver này không hỗ trợ `cinder manageable-list`, nhưng có `cinder manage` theo tên hoặc ID. Với Hitachi, `manage` theo số LDEV không bị giới hạn trong `hitachi_ldev_range` (`hbsd_common.py`, `manage_existing`). Nhận nhầm thì volume bị đổi tên và hai cloud cùng quản lý một volume | Chỉ `manage` volume đã đối chiếu: tên volume chứa UUID, kiểm tra UUID đó không có trong DB của cloud kia |
| Tài khoản quản trị tủ | Mọi hãng | Dùng chung tài khoản thì khó truy vết khi có sự cố | Mỗi cloud một tài khoản riêng |
| Availability Zone | Nova | Nếu đặt `[cinder] cross_az_attach = False` thì máy ảo chỉ gắn được volume cùng AZ | Mặc định là `True`. Nếu đã đổi thì kiểm tra lại AZ của các backend |

Muốn tách biệt chặt hơn: chia cổng tủ cho từng cloud (ví dụ cổng 1–2 cho cloud 1, cổng 3–4 cho cloud 2), hoặc dùng Virtual Fabrics/VSAN trên director. Cách này phức tạp hơn. Với quy mô hiện tại, zoning cẩn thận là đủ.

Nếu tủ 1 và tủ 2 khác hãng (`SAN.md` có 4 tủ thuộc 3 hãng), mỗi Cinder sẽ chạy hai driver khác nhau. Khi đó phải áp dụng bảng trên theo từng hãng.

---

## 5. Các phương án đã cân nhắc

| Phương án | Kết quả |
|---|---|
| Chỉ mua director, hoặc nối switch tới cả hai tủ | **Chưa đủ.** Mới có đường đi; nếu Cinder 2 chưa khai báo tủ 1 thì vẫn đổ hết vào tủ 2 |
| Director "auto balance" | Không có. Director chỉ cân bằng lưu lượng trên các đường nối (trunking, port channel) |
| Gộp hai cụm thành **một** OpenStack | Một Cinder thấy cả hai tủ, một nơi quản lý quota và volume. Về dung lượng thì không hơn phương án được chọn, mà lại là thay đổi kiến trúc lớn |
| Ảo hóa storage (IBM External Virtualization, Hitachi Universal Volume Manager) | Gom các tủ thành một kho. Không cần thiết, vì cách được chọn đã cho hai Cinder dùng chung cả hai tủ. Tủ phía trước thành điểm tập trung I/O, tốn thêm license. Dell Unity không có tính năng này |
| **Mỗi Cinder khai cả hai tủ làm backend, mỗi tủ một pool dùng chung** (tài liệu này chọn) | Giải được bài toán mà không thay đổi kiến trúc OpenStack. Giới hạn: không có hạn mức dung lượng riêng cho từng cloud trên tủ |

---

## 6. Các việc cần làm để giải quyết bài toán

| # | Bước | Làm ở đâu | Nếu bỏ qua |
|:---:|---|---|---|
| 1 | Lắp director A và B, nối compute của cả hai cloud và cả hai tủ vào | Phần cứng | Không có đường đi |
| 2 | Zone compute cloud 2 với cổng tủ 1, trên cả fabric A và B | Director | Có dây nhưng bị chặn, compute không thấy tủ 1 |
| 3 | Đổi hostname compute nếu bị trùng (với Unity) | Compute | Map LUN nhầm máy |
| 4 | Tạo tài khoản riêng cho Cinder 2 trên tủ 1. Với Hitachi, chia `hitachi_ldev_range` cho hai cloud | Tủ 1 | Khó truy vết; Hitachi có thể trùng LDEV |
| 5 | Khai tủ 1 làm backend thứ hai trong Cinder 2, chung volume type với tủ 2, đặt `reserved_percentage` | Cinder 2 | Cinder 2 vẫn chỉ biết tủ 2 |
| 6 | Thử tạo volume, gắn vào máy ảo, migrate, trên lab hoặc với volume test | OpenStack | – |
| 7 | Migrate dần volume cũ từ tủ 2 sang tủ 1, vào giờ thấp điểm. Volume có snapshot, thuộc group hoặc có replication không migrate được (mục 3.5) | OpenStack | Tủ 2 vẫn ở 80% |

---

## Nguồn (mã nguồn Cinder 24.0.0, Nova 29.0.0 = 2024.1)

- Điều kiện migrate: [volume/api.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/volume/api.py) (`migrate_volume`)
- Scheduler: [host_manager.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/scheduler/host_manager.py) (filter/weigher mặc định) · [weights/capacity.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/scheduler/weights/capacity.py) · [weights/goodness.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/scheduler/weights/goodness.py) · [filters/capacity_filter.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/scheduler/filters/capacity_filter.py)
- Unity, đặt tên host: [unity/client.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/volume/drivers/dell_emc/unity/client.py) · [unity/adapter.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/volume/drivers/dell_emc/unity/adapter.py)
- IBM, tìm host theo WWPN và hậu tố ngẫu nhiên: [storwize_svc_common.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/volume/drivers/ibm/storwize_svc/storwize_svc_common.py)
- Hitachi `hitachi_ldev_range`, `hitachi_snap_pool`, báo dung lượng pool, `manage_existing`: [hbsd_common.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/volume/drivers/hitachi/hbsd_common.py) · snapshot pool mặc định: [hbsd_rest.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/volume/drivers/hitachi/hbsd_rest.py)
- Nova `cross_az_attach`: [nova/conf/cinder.py](https://raw.githubusercontent.com/openstack/nova/29.0.0/nova/conf/cinder.py)
