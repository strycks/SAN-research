# Tính năng tủ SAN và mức hỗ trợ trên OpenStack Cinder

**Tủ khảo sát:** Dell Unity XT 880 · IBM FlashSystem 7300 · Hitachi VSP G700 · Hitachi VSP E590H
**Bản OpenStack:** Yoga (03/2022, Cinder 20) · 2023.1 Antelope (03/2023, Cinder 22) · 2024.1 Caracal (04/2024, Cinder 24)

---

## 1. Tóm tắt

1. **Chức năng bắt buộc của Cinder:** cả ba driver đều có đủ ở cả ba bản.
2. **Dell Unity và IBM FlashSystem 7300:** driver hỗ trợ gần hết chức năng tùy chọn (QoS, nhân bản, di chuyển volume do tủ thực hiện), ổn định qua ba bản.
3. **Hitachi:** bước thay đổi lớn là từ Yoga lên Antelope. Antelope thêm GAD, nén/dedup, di chuyển volume do tủ thực hiện, retype, nhiều pool. QoS và nhân bản kèm failover thì cả ba bản đều chưa có.
4. **Antelope so với Caracal:** với ba driver này không có tính năng mới, chỉ có sửa lỗi.
5. **VSP E590H:** sổ tay của Hitachi xác nhận tủ có các tính năng trong bảng. Tài liệu driver upstream ở cả ba bản không ghi tên model này, nhưng Product Compatibility Guide của Hitachi ghi E590H được hỗ trợ trên Yoga, Antelope, Caracal. Còn thiếu thông tin firmware tối thiểu (mục 7).

---

## 2. Cách đọc bảng

- Bốn cột **Tủ**: tủ có tính năng đó hay không.
- Ba cột **Driver**: dùng được tính năng đó từ OpenStack hay không.
- Cột **Cinder yêu cầu**: "Bắt buộc" là Cinder quy định mọi driver phải có; "Tùy chọn" là mỗi hãng tự quyết định có làm trong driver hay không.

| Ký hiệu | Nghĩa |
|:---:|---|
| ✔ | Có |
| ✖ | Không |
| ✔ từ Antelope | Driver Hitachi có từ bản 2023.1 Antelope trở đi; Yoga không có |
| ◐ | Một phần, hoặc có điều kiện (xem ghi chú) |
| – | Tài liệu không nêu, hoặc chưa xác nhận được |
| ¹ ² ³ | Số ghi chú ở mục 4 |

---

## 3. Bảng so sánh tính năng của tủ và mức hỗ trợ của driver

Áp dụng cho cả ba bản Yoga, Antelope, Caracal, trừ các ô ghi "từ Antelope". Cột driver của Hitachi áp dụng cho VSP G700 (được ghi tên cả trong tài liệu upstream lẫn Product Compatibility Guide của Hitachi); VSP E590H được hỗ trợ theo Product Compatibility Guide của Hitachi (mục 7).

| Tính năng | Mô tả | Cinder yêu cầu | Tủ Dell Unity XT 880 | Tủ IBM FlashSystem 7300 | Tủ Hitachi VSP G700 | Tủ Hitachi VSP E590H | Driver Cinder Dell | Driver Cinder IBM | Driver Cinder Hitachi |
|---|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Tạo/xóa LUN, map/unmap host | Cấp phát volume từ pool và gán cho host | Required | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Mở rộng LUN | Tăng dung lượng volume, không cần tạo lại | Required | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ ¹ | ✔ ¹ |
| Mở rộng LUN đang attach | Tăng dung lượng khi volume vẫn đang gắn vào máy ảo | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ ¹ | ✔ ¹ |
| Snapshot | Bản chụp volume tại một thời điểm | Required | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Clone, tạo volume từ snapshot | Tạo volume mới độc lập từ volume hoặc snapshot | Required | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Giao thức FC, iSCSI | Kết nối host tới tủ qua Fibre Channel hoặc iSCSI | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Thin provisioning | Cấp dung lượng logic lớn hơn vật lý, chỉ chiếm chỗ khi ghi | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Thick provisioning | Cấp phát đủ dung lượng vật lý ngay khi tạo | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Khôi phục LUN về snapshot | Đưa volume về đúng trạng thái của snapshot | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Chia sẻ LUN cho nhiều host | Một volume gán đồng thời cho nhiều host (cluster) | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| NVMe phía host | Giao thức NVMe qua FC/Ethernet, độ trễ thấp hơn SCSI | Optional | ✖ | ✔ | ✖ | ✖ ² | ✖ | ✖ | ✖ |
| QoS giới hạn IOPS/băng thông | Đặt trần IOPS hoặc MB/s cho từng volume | Optional | ✔ | ✔ | ✔ | ✔ ³ | ✔ | ✔ | ✖ |
| Nén dữ liệu | Nén inline để giảm dung lượng vật lý | Optional | ✔ | ✔ | ✔ ⁴ | ✔ | ✔ ⁵ | ✔ | ✔ từ Antelope ⁶ |
| Dedup | Loại bỏ các block dữ liệu trùng lặp | Optional | ✔ | ✔ | ✔ ⁴ | ✔ | ◐ ¹⁷ | ✖ ¹⁸ | ✔ từ Antelope ⁶ |
| Auto-tiering | Tự chuyển dữ liệu nóng/lạnh giữa các tầng đĩa | Optional | ✔ ⁷ | ✔ | ✔ | ✔ ² | ✔ | ✔ | ◐ ⁸ |
| Consistency group, snapshot nhóm | Snapshot nhất quán cho một nhóm volume | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Replication đồng bộ | Nhân bản sang tủ khác, RPO = 0, khoảng cách ngắn | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Replication bất đồng bộ | Nhân bản sang site xa, RPO tính bằng giây/phút | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Replication theo consistency group | Nhân bản cả nhóm volume, giữ nhất quán | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Failover/failback sang site DR | Chuyển vai trò chính sang tủ DR và chuyển ngược lại | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✖ |
| Active-active metro cluster | Một volume đọc/ghi đồng thời ở 2 tủ, failover tự động | Optional | ◐ ⁹ | ✔ | ✔ | ✔ | ✖ | ✔ | ✔ từ Antelope ¹⁰ ¹¹ |
| Mirror volume giữa 2 pool | Hai bản sao của volume ở 2 pool trong cùng tủ | Optional | – | ✔ | ✔ | ✔ | ✖ | ✔ | ✖ |
| Replication 3 site | Bản đồng bộ gần kết hợp bản bất đồng bộ xa | Optional | – | ✔ | ✔ | ✔ | ✖ | ✖ | ✖ |
| Snapshot bất biến chống ransomware | Snapshot không thể sửa/xóa trước hạn lưu giữ | Optional | ✖ | ✔ | ◐ ¹² | ✔ ¹² | ✖ | ✖ | ✖ |
| Backup volume không gián đoạn | Backup từ snapshot, không phải dừng volume | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Mã hóa dữ liệu trên tủ | Mã hóa dữ liệu lưu trên đĩa (data-at-rest) | Optional | ✔ | ✔ | ✔ | ✔ | ◐ ¹³ | ◐ ¹³ | ◐ ¹³ |
| Di chuyển LUN giữa pool do tủ thực hiện | Tủ tự chuyển dữ liệu sang pool khác khi host vẫn I/O | Optional | ✔ | ✔ | ✔ | ✔ ³ | ✔ | ✔ | ✔ từ Antelope ¹⁴ |
| Đổi thuộc tính LUN online (retype) | Đổi thin/thick, nén, tier của volume đang dùng | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ◐ ¹⁵ |
| Import LUN có sẵn (manage/unmanage) | Đưa LUN đã tồn tại trên tủ vào Cinder, không copy dữ liệu | Optional | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| Ảo hóa tủ ngoài | Dùng LUN của tủ hãng khác làm dung lượng cho tủ này | Optional | ✖ | ✔ | ✔ | ✔ | ✖ | ◐ ¹³ | ◐ ¹³ |
| Model được driver ghi tên | Tủ có trong danh sách hỗ trợ của tài liệu driver | – | | | | | ✔ | ✔ | G700 ✔ · E590H ◐ ¹⁶ |
| Nhiều pool trên một backend | Một backend Cinder quản lý nhiều pool của tủ | Optional | | | | | ✔ | ✔ | ✔ từ Antelope |
| FC auto-zoning | Tự tạo/xóa zone trên SAN switch khi attach/detach | Optional | | | | | ✔ | ✔ | ✔ |
| Cinder active/active HA | Chạy nhiều cinder-volume song song cho một backend | Optional | | | | | ✖ | ✖ | ✖ |

---

## 4. Ghi chú

| # | Nội dung |
|:---:|---|
| ¹ | Driver IBM và Hitachi không extend được volume đang có snapshot. |
| ² | NVMe phía host: E590H không có. Family Matrix ghi cổng host của E590H gồm FC 16/32 Gb/s và iSCSI 10/25 Gb/s; FC-NVMe chỉ có trên E1090 (hỗ trợ từ firmware 93-06-61). Dynamic Tiering: có trên E590H, mô tả trong System Construction Guide (sổ tay ghi rõ áp dụng cho E590H). Cần license Dynamic Tiering theo dung lượng pool bật tiering; license này không nằm trong gói Base (Phụ lục B). |
| ³ | Đã xác nhận trong sổ tay Hitachi ghi rõ áp dụng cho E590H. QoS (Performance Manager User Guide, 4060-1J-U08-61): đặt trần theo từng volume, tính bằng IOPS, MB/s hoặc cả hai; tự cài sẵn từ firmware 88-06-01 trở lên; cấu hình qua Storage Navigator hoặc `raidcom`. Di chuyển volume (Volume Migration User Guide, 4060-1J-U18-80): host vẫn đọc/ghi được trong lúc chuyển. |
| ⁴ | G700: nén/dedup (capacity saving) chỉ áp dụng trên ổ flash nội bộ. |
| ⁵ | Driver Unity: chỉ tạo được volume nén trên pool all-flash. Mã nguồn báo `compression_support = pool.is_all_flash`, nên dù từ Unity OE 5.2 tủ Unity XT đã có Data Reduction trên pool hybrid, Cinder vẫn không tạo được volume nén trên pool hybrid. |
| ⁶ | Driver Hitachi từ Antelope: nén và dedup bật cùng nhau qua `hbsd:capacity_saving=deduplication_compression`. Cần bật trước trên pool và có license dedup/compression. |
| ⁷ | Unity: FAST VP và FAST Cache chỉ có trên model hybrid (880, không phải 880F). |
| ⁸ | Tiering của Hitachi chạy theo pool trên tủ; Cinder không đặt được chính sách tier cho từng volume. |
| ⁹ | Unity không có active-active gốc cho block; cần mua thêm metro node hoặc VPLEX. |
| ¹⁰ | Release notes 2023.1 ghi "Support Global-Active Device (GAD) volume"; mã nguồn bản 22.0.0 (Antelope) đã có `hbsd_replication.py` và các tùy chọn `hitachi_mirror_*`. Tài liệu driver bản 2023.1 chưa có hướng dẫn GAD; hướng dẫn đầy đủ có từ tài liệu Caracal. |
| ¹¹ | GAD qua Cinder (từ Antelope): dùng `hbsd:topology=active_active_mirror_volume`. Giới hạn: không dùng ALUA; volume GAD không bật được nén/dedup, không migrate bằng tủ, không manage/unmanage; G700 cần firmware 88-03-21 trở lên. |
| ¹² | E590H: Thin Image Advanced (dòng E, firmware 93-07-21 trở lên) có "thời gian bảo vệ dữ liệu snapshot" (firmware 93-07-24 trở lên). Trong thời gian này, snapshot không bị ghi đè, resync hay xóa; thời gian chỉ được kéo dài, không rút ngắn hay tắt được. Đặt được 1 đến 12.288 giờ (khoảng 512 ngày), theo từng pair, snapshot group hoặc consistency group, bằng RAID Manager (`-retention`, `raidcom modify snapshot`). Volume phải là volume dùng chung có giảm dữ liệu (cần Adaptive Data Reduction). G700: Thin Image Advanced chỉ có trên dòng E; chỉ có Data Retention Utility, đặt thuộc tính Read Only/Protect cho volume, không phải cơ chế snapshot bất biến riêng. |
| ¹³ | Trong suốt với Cinder: bật trên tủ thì volume được hưởng, nhưng Cinder không điều khiển. Riêng mã hóa, Cinder có cơ chế mã hóa volume riêng (LUKS). |
| ¹⁴ | Chỉ migrate trong cùng một tủ. Release notes và mã nguồn có từ Antelope, nhưng ma trận hỗ trợ của Cinder đến Caracal vẫn ghi "missing" cho Hitachi; nên thử trước trên môi trường test. |
| ¹⁵ | Yoga: driver Hitachi không có retype riêng, Cinder đổi loại volume bằng cách copy qua host. Từ Antelope: đổi được nén/dedup ngay trên volume; đổi sang pool khác thì dùng migrate do tủ thực hiện. Không đổi được tier hay thin/thick. |
| ¹⁶ | Tài liệu driver upstream (từ Antelope) ghi E590, E790, E1090, E1090H nhưng không có E590H. Product Compatibility Guide của Hitachi thì ghi E590H được hỗ trợ trên Yoga, Antelope, Caracal. "◐" nghĩa là hãng xác nhận nhưng upstream chưa ghi, và chưa rõ firmware tối thiểu. Xem mục 7. |
| ¹⁷ | Driver Unity chỉ có một tùy chọn `provisioning:type=compressed`. Trên Unity OE 4.3 trở lên, thư viện storops gửi tùy chọn này thành `isDataReductionEnabled`; theo Dell, Data Reduction gồm dedup, nén và phát hiện block rỗng. Advanced Deduplication (OE 4.5 trở lên, bật riêng cho từng LUN) thì driver không bật được: storops có tham số `is_advanced_dedup_enabled`, nhưng driver Cinder không truyền tham số này. Qua Cinder chỉ áp dụng được trên pool all-flash (ghi chú ⁵). |
| ¹⁸ | Trên FlashSystem, dedup phải bật riêng cho từng volume (`mkvolume -deduplicated`) và chỉ có trong data reduction pool. Mã nguồn driver ở cả ba bản không có tùy chọn hay lệnh nào cho dedup, chỉ có nén (`compression`). Volume tạo từ Cinder vì vậy không được dedup. |

---

## 5. So sánh ba hãng

Phạm vi: tính năng có trên tủ và mức hỗ trợ của driver. Không đánh giá giá, hiệu năng, chất lượng hỗ trợ.

| Tiêu chí | Dell Unity XT 880 | IBM FlashSystem 7300 | Hitachi VSP G700, VSP E590H |
|---|---|---|---|
| Chức năng bắt buộc của Cinder | Đủ | Đủ | Đủ |
| QoS | Có, driver hỗ trợ | Có, driver hỗ trợ | Có, driver chưa hỗ trợ |
| Nhân bản sang tủ dự phòng | Có, driver hỗ trợ | Có, driver hỗ trợ | Có (gói Advanced), driver chưa hỗ trợ |
| Active-active | Phải mua thêm thiết bị, driver không hỗ trợ | Có, driver hỗ trợ | Có (gói Advanced), driver hỗ trợ từ Antelope |
| Nén, dedup | Có; driver bật Data Reduction (gồm dedup cơ bản), không bật được Advanced Deduplication | Có; driver hỗ trợ nén, không có dedup | Có; driver hỗ trợ cả hai từ Antelope |
| Thick provisioning, tiering | Có, driver hỗ trợ | Có, driver hỗ trợ | Có; driver không hỗ trợ thick, tiering chỉ theo pool trên tủ |
| License | Gần như trọn gói | Trọn gói, trừ mã hóa và ảo hóa tủ ngoài | Chia gói cơ bản và gói Advanced |
| Tích hợp OpenStack | Đầy đủ, ổn định | Đầy đủ nhất | Hạn chế hơn; đủ dùng từ Antelope, thiếu QoS và nhân bản |

**Nhận định:** IBM có bộ tính năng và mức tích hợp OpenStack đầy đủ nhất. Dell Unity trọn gói, tích hợp ổn định, không có active-active sẵn. Hitachi mạnh trên tủ; qua OpenStack thì từ Antelope đã có phần lớn tính năng, trừ QoS và nhân bản.

---

## 6. Khác nhau giữa ba bản OpenStack

- Unity và FS7300: ba bản Yoga, Antelope, Caracal hỗ trợ như nhau.
- Hitachi: Antelope là bản đầu tiên điều khiển được active-active (GAD), nén/dedup, di chuyển volume do tủ thực hiện, retype và nhiều pool; cũng là bản đầu tiên tài liệu upstream ghi tên VSP E590 (trang tương thích của Hitachi thì ghi E590, E590H từ Yoga). Caracal so với Antelope chỉ có sửa lỗi.
- Hitachi ở cả ba bản: QoS và nhân bản vẫn phải cấu hình trực tiếp trên tủ.

---

## 7. Riêng về Hitachi VSP E590H

### E590 và E590H khác nhau thế nào

| | VSP E590 | VSP E590H |
|---|---|---|
| Loại | All-flash | Hybrid (lai) |
| Ổ đĩa | Chỉ SSD: NVMe và SAS SSD | SSD kết hợp HDD, lắp qua các khay mở rộng |
| HDD tối đa | Không có | 480 ổ 3.5 inch SAS |
| Hệ điều hành tủ | SVOS RF | SVOS RF |
| Cổng kết nối máy chủ | FC, iSCSI | FC, iSCSI |
| Phù hợp | Ứng dụng cần độ trễ thấp | Dung lượng lớn, chi phí thấp hơn cho dữ liệu ít truy xuất |

Hitachi mô tả hai model trong cùng một bộ tài liệu phần cứng. Khác biệt chính là loại ổ đĩa lắp được, không phải phần mềm điều khiển.

### Tài liệu driver OpenStack ghi gì

Đã kiểm tra lại trực tiếp bảng "Supported storages" trên trang tài liệu driver Hitachi của từng bản:

| Bản OpenStack | Dòng E được ghi tên | Có "E590H" |
|---|---|:---:|
| Yoga | E990 | Không |
| 2023.1 Antelope | E590, E790, E990, E1090, E1090H | Không |
| 2024.1 Caracal | E590, E790, E990, E1090, E1090H | Không |
| Bản mới nhất (kiểm tra thêm, 10/2026) | E590, E790, E990, E1090, E1090H | Không |

Tài liệu ghi tên không nhất quán giữa các model: có ghi cả "E1090, E1090H", nhưng chỉ ghi "E590, E790" mà không có bản H. Đối chiếu với Product Compatibility Guide của Hitachi (bên dưới) thì E590H, E790H đã được hãng hỗ trợ, nên nhiều khả năng tài liệu upstream chưa cập nhật.

### Đã tra thêm phía Hitachi

| Nguồn | Kết quả |
|---|---|
| Trang Product Compatibility Guide cho driver OpenStack | Có ghi tên E590H cho Yoga, Antelope và Caracal (chi tiết bên dưới) |
| Hướng dẫn cài driver cho Red Hat OpenStack Platform 16.2 và 17.1 (09/2024) | Không có danh sách model |
| Hướng dẫn cài driver cho Red Hat OpenStack Services on OpenShift 18.0 (06/2025) | Không có danh sách model |
| User guide driver riêng của Hitachi | Chỉ tìm được bản cũ (Train, Queens), trước thời E590H |
| Red Hat Ecosystem Catalog (trang chứng nhận driver) | Chứng nhận theo bản Red Hat OpenStack Platform, không ghi model tủ hay firmware (chi tiết bên dưới) |

### Chứng nhận của Red Hat

Tab "Certifications" trên Red Hat Ecosystem Catalog, xem ngày 27/10/2026:

| Red Hat OpenStack Platform | Tương ứng bản upstream | Hệ điều hành | Bản driver | Giao thức được chứng nhận |
|---|---|---|---|---|
| 17.1 | Wallaby | RHEL 9.2, x86_64 | 2.0.0 | FC, iSCSI (Base, Backup) |
| 16.2 | Train | RHEL 8.4, x86_64 | Community Edition v2.0.0 | FC, iSCSI (Base, Backup) |
| 13.0 | Queens | RHEL 7.5, x86_64 | HBSD 6.1 | Không ghi |
| 10.0 | Newton | RHEL 7.3, x86_64 | HBSD 3.1 | Không ghi |

- Trang chứng nhận driver chứ không chứng nhận từng model tủ, nên không có thông tin firmware.
- Chưa có dòng nào cho RHOSO 18.0 (dựa trên Antelope), dù Product Compatibility Guide của Hitachi có ghi RHOSO 18.0 và 18.0.17.
- Các bản RHOSP được chứng nhận đều cũ hơn Yoga, nên trang này không áp dụng cho Yoga, Antelope, Caracal bản cộng đồng. Chỉ cần quan tâm nếu triển khai bằng bản phân phối của Red Hat.

### Product Compatibility Guide của Hitachi

Tra cứu ngày 27/10/2026 tại mục "VSP One Block Storage → Block Storage Driver for OpenStack", lọc Storage = E590 và E590H, Release Name = Yoga, Antelope, Caracal. Kết quả 21 dòng.

| Bản OpenStack | Bản phân phối | Hệ điều hành | Loại driver | Cổng kết nối |
|---|---|---|---|---|
| 2023.1 Antelope | RH OSO 18.0 và 18.0.17 | Red Hat EL 9.4 (x64) | RHOSO in-tree driver | FC 16 Gbps, FC 32 Gbps, iSCSI 10 Gbps |
| 2023.1 Antelope | Other (Community) | Other (Community) | Community driver | FC 16 Gbps, FC 32 Gbps, iSCSI 10 Gbps |
| 2024.1 Caracal | 2024.1 Sunbeam, 2024.1 Juju Charmed | Ubuntu 24.04 LTS | Community driver | FC 16 Gbps, FC 32 Gbps, iSCSI 10 Gbps |
| 2024.1 Caracal | Other (Community) | Other (Community) | Community driver | FC 16 Gbps, FC 32 Gbps, iSCSI 10 Gbps |
| Yoga | Other (Community) | Other (Community) | Community driver | FC 16 Gbps, FC 32 Gbps, iSCSI 10 Gbps |

- Mọi dòng đều ghi cùng một danh sách tủ: VSP E590, E590H, E790, E790H, E990, E1090, E1090H.
- Ghi chú ở dòng community driver: "Supported on the Operating Systems which community Openstack supports", tức chạy trên các hệ điều hành mà OpenStack cộng đồng hỗ trợ.
- Trang không ghi yêu cầu firmware.
- Với iSCSI, trang chỉ ghi 10 Gbps, dù E590H có cả cổng iSCSI 25 Gb/s (ghi chú ²). Nếu định dùng cổng 25 Gb/s thì nên hỏi Hitachi.
- Trang của Hitachi rộng hơn tài liệu upstream: upstream bản Yoga chỉ ghi E990, còn trang Hitachi ghi cả E590 và E590H cho Yoga.

Như vậy đã có nguồn chính thức của hãng ghi tên E590H với driver OpenStack ở cả ba bản.

Tra cùng cách cho G700 (lọc Storage = G700) cũng ra 21 dòng, cùng các tổ hợp bản OpenStack, bản phân phối và cổng kết nối như trên. Các dòng này ghi chung nhóm VSP F350, F370, F700, F900, G350, G370, G700, G900.

### Đánh giá

Cần phân biệt hai mức:

| Mức | Nghĩa là | Tình trạng với E590H |
|---|---|---|
| Chạy được về kỹ thuật | Driver gửi lệnh và tủ thực hiện được | Khả năng cao là có. Driver làm việc với hệ điều hành của tủ qua REST API, không phụ thuộc loại ổ đĩa; E590 cùng hệ điều hành đã được ghi tên từ Antelope. Đây là suy luận, chưa kiểm chứng. |
| Được hỗ trợ chính thức | Hãng đã kiểm thử, ghi tên trong tài liệu, chịu trách nhiệm khi có lỗi | Có. Product Compatibility Guide của Hitachi ghi tên E590H cho Yoga, Antelope, Caracal, qua FC 16/32 Gbps và iSCSI 10 Gbps. Chưa rõ yêu cầu firmware. |

Điểm còn vênh: tài liệu upstream của OpenStack chưa ghi E590H, và bản Yoga upstream chỉ ghi E990. Khi có sự cố, bảng của Hitachi là căn cứ để yêu cầu hãng hỗ trợ.

### Đề xuất

1. Lưu lại bản PDF/CSV xuất từ Product Compatibility Guide (trang có sẵn nút xuất) làm bằng chứng hỗ trợ.
2. Hỏi Hitachi hoặc đối tác cung cấp tủ về firmware tối thiểu cho E590H và phiên bản driver cụ thể ứng với từng bản OpenStack.
3. Với Yoga, tính năng driver vẫn giới hạn theo bản Yoga (mục 5, 6); muốn có GAD, nén/dedup qua OpenStack thì cần Antelope trở lên.
4. Nếu có sẵn tủ và môi trường thử nghiệm, test các thao tác cơ bản (tạo, gán, mở rộng, snapshot, clone) trước khi dùng thật.

---

## Phụ lục A. Mô tả tính năng và tên gọi theo hãng

| Tính năng | Mô tả | Dell Unity | IBM FS7300 | Hitachi |
|---|---|---|---|---|
| Thin provisioning | Cấp dung lượng logic lớn hơn vật lý, chỉ chiếm chỗ khi ghi | Thin LUN | Thin volume | Dynamic Provisioning |
| Snapshot | Bản chụp volume tại một thời điểm, tiết kiệm dung lượng | Unified Snapshots | FlashCopy | Thin Image |
| Clone | Volume mới độc lập tạo từ volume/snapshot | Thin Clone | FlashCopy (có background copy) | ShadowImage, Thin Image |
| QoS | Đặt trần IOPS hoặc MB/s cho volume/host | Quality of Service (Host I/O Limits) | Throttling | QoS controls trong SVOS RF |
| Nén, dedup | Giảm dung lượng vật lý bằng nén và loại block trùng | Inline Data Reduction | Data Reduction Pool, nén phần cứng trên FCM | Adaptive Data Reduction (capacity saving) |
| Auto-tiering | Tự chuyển dữ liệu nóng/lạnh giữa các tầng đĩa | FAST VP | Easy Tier | Dynamic Tiering |
| SSD cache | Dùng SSD tăng tốc cho pool đĩa quay | FAST Cache | Không có | active flash |
| Replication đồng bộ | Nhân bản sang tủ khác, RPO = 0 | Native Sync Replication | Metro Mirror | TrueCopy |
| Replication bất đồng bộ | Nhân bản sang site xa, RPO giây/phút | Native Async Replication | Global Mirror, GMCV, policy-based replication | Universal Replicator |
| Active-active metro | Một volume đọc/ghi đồng thời ở 2 tủ, failover tự động | metro node, VPLEX (mua thêm) | HyperSwap | global-active device (GAD) |
| Mirror giữa 2 pool | Hai bản sao của volume trong cùng một tủ | Không có | Volume Mirroring | ShadowImage |
| Replication 3 site | Bản đồng bộ gần kết hợp bản bất đồng bộ xa | Không nêu | 3-site replication | 3DC (GAD + UR, TC + UR) |
| Snapshot bất biến | Snapshot không thể sửa/xóa trước hạn | Không có cho block | Safeguarded Copy | Thin Image Advanced với thời gian bảo vệ snapshot (dòng E); Data Retention Utility (G700) |
| Mã hóa | Mã hóa dữ liệu trên đĩa | D@RE | AES-XTS 256 | AES-256-XTS (cần back-end mã hóa) |
| Ảo hóa tủ ngoài | Dùng LUN của tủ hãng khác làm dung lượng | Không có | External virtualization | Universal Volume Manager |
| Consistency group | Gom nhiều volume để snapshot/replicate nhất quán | Consistency Group | Consistency group, volume group | Consistency group |

## Phụ lục B. License phía tủ

| Tủ | Kèm sẵn | Phải mua thêm hoặc tùy chọn |
|---|---|---|
| Dell Unity XT 880 | Thin, data reduction, QoS, snapshot, thin clone, replication đồng bộ và bất đồng bộ, FAST VP và FAST Cache (hybrid) | metro node, VPLEX, RecoverPoint Advanced, AppSync Advanced, PowerPath; mã hóa D@RE chọn lúc đặt hàng |
| IBM FS7300 | Mọi tính năng trừ hai mục bên phải | Ảo hóa tủ ngoài (theo dung lượng), mã hóa (feature code). Cần hỏi IBM về license remote mirroring cho HyperSwap |
| Hitachi G700 | Gói Foundation: SVOS RF, Universal Volume Manager, Local Replication (Thin Image, ShadowImage), Data Mobility (Dynamic Tiering) | Gói Advanced: TrueCopy, Universal Replicator, GAD. Mã hóa: license và phần cứng riêng |
| Hitachi E590H | Gói Base: Adaptive Data Reduction, Storage Virtualization, In-System Replication, Non-disruptive Migration | Gói Advanced: remote replication, GAD. Dynamic Tiering: license theo dung lượng pool |

License thực tế trên từng tủ cần kiểm tra trực tiếp: Unisphere (Unity), lệnh `lslicense` (IBM), màn hình License trong Storage Navigator (Hitachi).

## Phụ lục C. Driver Cinder tương ứng

| Tủ | Driver Cinder | Yêu cầu |
|---|---|---|
| Dell Unity 880 | Dell Unity driver (`cinder.volume.drivers.dell_emc.unity.Driver`) | Unity OE 4.1.X trở lên, storops 1.2.3 trở lên |
| IBM FS7300 | IBM Storage Virtualize driver (`StorwizeSVCFCDriver`, `StorwizeSVCISCSIDriver`) | Phủ họ FlashSystem 5xxx, 7xxx, 9xxx |
| Hitachi G700, E590H | Hitachi VSP driver (`HBSDFCDriver`, `HBSDISCSIDriver`) | G700: firmware 88-01-04 trở lên. E590 (được ghi tên upstream từ Antelope): firmware 93-03-22 trở lên. E590H chưa được ghi tên upstream, nhưng có trong Product Compatibility Guide của Hitachi cho Yoga, Antelope, Caracal. License SVOS, Dynamic Provisioning, Thin Image |

Bảng dựa trên tài liệu upstream của OpenStack. Driver do Hitachi phát hành riêng có thể hỗ trợ rộng hơn.

## Nguồn

**Quy định và danh mục chức năng của Cinder**
- [Cinder Support Matrix, bản Yoga](https://docs.openstack.org/cinder/yoga/reference/support-matrix.html)
- [Cinder support-matrix.ini, bản 2023.1](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2023.1/doc/source/reference/support-matrix.ini)
- [Cinder support-matrix.ini, bản 2024.1](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/doc/source/reference/support-matrix.ini)
- [Cinder: tài liệu cho người viết driver (tính năng tối thiểu)](https://docs.openstack.org/cinder/2024.1/contributor/drivers.html)

**Release notes của Cinder**
- [Yoga](https://docs.openstack.org/releasenotes/cinder/yoga.html) · [Zed](https://docs.openstack.org/releasenotes/cinder/zed.html) · [2023.1 Antelope](https://docs.openstack.org/releasenotes/cinder/2023.1.html) · [2023.2 Bobcat](https://docs.openstack.org/releasenotes/cinder/2023.2.html) · [2024.1 Caracal](https://docs.openstack.org/releasenotes/cinder/2024.1.html)

**Tài liệu driver**
- Dell Unity: [Yoga](https://docs.openstack.org/cinder/yoga/configuration/block-storage/drivers/dell-emc-unity-driver.html) · [2023.1](https://docs.openstack.org/cinder/2023.1/configuration/block-storage/drivers/dell-emc-unity-driver.html) · [2024.1](https://docs.openstack.org/cinder/2024.1/configuration/block-storage/drivers/dell-emc-unity-driver.html)
- IBM Storage Virtualize: [Yoga](https://docs.openstack.org/cinder/yoga/configuration/block-storage/drivers/ibm-storwize-svc-driver.html) · [2023.1](https://docs.openstack.org/cinder/2023.1/configuration/block-storage/drivers/ibm-storwize-svc-driver.html) · [2024.1](https://docs.openstack.org/cinder/2024.1/configuration/block-storage/drivers/ibm-storwize-svc-driver.html)
- Hitachi VSP: [Yoga](https://docs.openstack.org/cinder/yoga/configuration/block-storage/drivers/hitachi-vsp-driver.html) · [2023.1](https://docs.openstack.org/cinder/2023.1/configuration/block-storage/drivers/hitachi-vsp-driver.html) · [2024.1](https://docs.openstack.org/cinder/2024.1/configuration/block-storage/drivers/hitachi-vsp-driver.html) · [mới nhất](https://docs.openstack.org/cinder/latest/configuration/block-storage/drivers/hitachi-vsp-driver.html)

**Mã nguồn driver (tác giả, phiên bản, chức năng)**
- Dell Unity: [driver.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/dell_emc/unity/driver.py) · [adapter.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/dell_emc/unity/adapter.py) · [client.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/dell_emc/unity/client.py)
- IBM Storage Virtualize: [storwize_svc_fc.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/ibm/storwize_svc/storwize_svc_fc.py) · [storwize_svc_common.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/ibm/storwize_svc/storwize_svc_common.py)
- Hitachi VSP: [hbsd_fc.py bản Yoga](https://raw.githubusercontent.com/openstack/cinder/unmaintained/yoga/cinder/volume/drivers/hitachi/hbsd_fc.py) · [bản 2023.1](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2023.1/cinder/volume/drivers/hitachi/hbsd_fc.py) · [bản 2024.1](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/hitachi/hbsd_fc.py) · [hbsd_common.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/hitachi/hbsd_common.py) · [hbsd_rest.py](https://raw.githubusercontent.com/openstack/cinder/unmaintained/2024.1/cinder/volume/drivers/hitachi/hbsd_rest.py)

**Tài liệu hãng**
- [Dell Unity XT Series Specification Sheet](https://www.delltechnologies.com/asset/en-my/products/storage/technical-support/h17713_dell_emc_unity_xt_series_ss.pdf)
- [Dell Unity: Data Reduction Considerations and Best Practices](https://www.dell.com/support/kbdoc/en-us/000022795/unity-how-to-use-compression-considerations-and-best-practices)
- [Dell Unity: About LUN move sessions](https://www.dell.com/support/manuals/en-us/unity-480/unity_p_config_luns/about-lun-move-sessions?guid=guid-e8d2a258-d7fe-4ce1-ba66-afc8f36b61f5&lang=en-us)
- [Dell Unity: Changing LUN Provisioning Between Thin and Thick](https://www.dell.com/support/kbdoc/en-us/000021267/dell-emc-unity-is-it-possible-to-change-unity-luns-from-thin-to-thick-and-vice-versa-user-correctable)
- [Dell Unity: Consistency Groups with Replication Configured](https://www.dell.com/support/kbdoc/en-us/000202243/dell-unity-best-practices-to-manage-consistency-groups-with-replication-configured-dell-correctable)
- [Dell EMC Unity: Unisphere Overview (white paper H15085)](https://resources.r2ut.com/hubfs/dell-emc-unity-unisphere-overview.pdf)
- [Dell Unity XT HFA Top Reasons (metro node)](https://www.delltechnologies.com/asset/he-il/products/storage/briefs-summaries/dell-unity-xt-hfa-top-reasons.pdf)
- [IBM Storage FlashSystem 7300 Product Guide](https://redbooks.ibm.com/redpapers/pdfs/redp5668.pdf)
- [Hitachi VSP G/F series: Software components and features](https://knowledge.hitachivantara.com/Documents/Storage/VSP_G130_GF350_GF370_GF700_GF900/88-01-0x/About_Your_System/Product_Overview/Software_components_and_features)
- [Hitachi: Base and Advanced Software Packages for VSP Midrange Storage](https://www.hitachivantara.com/en-us/pdf/datasheet/base-advanced-software-packages-for-vsp-midrange-storage-datasheet.pdf)
- [Hitachi SVOS RF Provisioning Guide cho VSP Gx00/Fx00](https://knowledge.hitachivantara.com/@api/deki/files/50438/SVOS_RF_v8_3_Provisioning_Guide_VSP_Gx00_Fx00_MK-97HM85026-02.pdf?revision=1)
- Sổ tay Hitachi SVOS RF 9.8 (tiếng Nhật, ghi rõ áp dụng cho E590H và G700): [Document Map](https://itpfdoc.hitachi.co.jp/manuals/4049/40491JU04_SVOSRF98/40491JU04.pdf) · [System Construction Guide](https://itpfdoc.hitachi.co.jp/manuals/4049/40491JU09_SVOSRF985/40491JU09.pdf) · [Thin Image User Guide](https://itpfdoc.hitachi.co.jp/manuals/4049/40491JU13_SVOSRF986/40491JU13.pdf) · [Universal Replicator User Guide](https://itpfdoc.hitachi.co.jp/manuals/4046/40461JU15_SVOSRF96/40461JU15.pdf)
- [Hitachi Thin Image: Restoring Thin Image pairs](https://docs.hitachivantara.com/r/en-us/svos/9.8.6/mk-98rd9020/managing-thin-image-pairs/restoring-thin-image-pairs)
- [Hitachi VSP E Series Family Matrix](https://www.hitachivantara.com/en-us/pdf/specifications/virtual-storage-platform-e-series-family-matrix.pdf)
- [Hitachi VSP E590/E790 Hardware Reference](https://docs.hitachivantara.com/r/en-us/mk-97hm85050/latest/introduction)
- [Hitachi Product Compatibility Guide: Block Storage Driver for OpenStack](https://compatibility.hitachivantara.com/products/openstack)
- [Red Hat Ecosystem Catalog: Hitachi Block Storage Driver for OpenStack](https://catalog.redhat.com/en/software/openstack/detail/60466)
- Sổ tay Hitachi (tiếng Nhật, ghi rõ áp dụng cho E590H): [Danh sách tính năng theo phiên bản firmware dòng E](https://itpfdoc.hitachi.co.jp/manuals/4049/40491JU91_SVOSRF987/40491JU91.pdf) · [Thin Image Advanced User Guide](https://itpfdoc.hitachi.co.jp/manuals/4049/40491JU23_SVOSRF987/40491JU23.pdf) · [Performance Manager (Performance Monitor, Server Priority Manager, QoS) User Guide](https://itpfdoc.hitachi.co.jp/manuals/4049/40491JU08_SVOSRF96/40491JU08.pdf) · [Volume Migration User Guide](https://itpfdoc.hitachi.co.jp/manuals/4049/40491JU18_SVOSRF98/40491JU18.pdf)
- [IBM: lệnh mkvolume (tùy chọn -deduplicated)](https://www.ibm.com/docs/en/SSGBLEL_9.1.0/storageflashsystem/cli/svc_mkvolume.html)
- [storops: tham số tạo LUN của Unity (lun.py)](https://github.com/emc-openstack/storops/blob/master/storops/unity/resource/lun.py)