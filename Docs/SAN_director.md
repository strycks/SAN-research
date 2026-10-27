# SAN director (switch FC loại lớn) và tích hợp với OpenStack Cinder

**Hãng:** Broadcom Brocade, Cisco MDS
**OpenStack:** Yoga (Cinder 20.0.0), 2023.1 Antelope (22.0.0), 2024.1 Caracal (24.0.0)
**Liên quan:** `SAN.md` (tủ SAN và driver Cinder)
**Kiểm tra lại nguồn:** 27/10/2026

---

## 1. Kết luận nhanh

1. **Director** là switch FC loại lớn, dạng chassis: cắm thêm card để có thêm cổng, mọi linh kiện đều có bản dự phòng. Thường đặt ở trung tâm (core) của mạng SAN. **Switch cố định** nhỏ hơn và rẻ hơn, phù hợp với SAN nhỏ hoặc đặt ở đầu rack.
2. Yếu tố quan trọng nhất cho độ sẵn sàng là **có hai fabric tách rời (A/B)**, hơn là mua director. Hai switch thường chia làm A/B vẫn an toàn hơn một director duy nhất.
3. **Auto-zoning qua Cinder chạy được nhưng không ai bảo trì:**
   - Driver **Brocade** bị đánh dấu *unsupported* từ 2019 vì Broadcom bỏ hỗ trợ. Muốn chạy phải bật `enable_unsupported_driver = True` trong section `[fc-zone-manager]`.
   - Driver **Cisco** vẫn ghi là *supported*, nhưng không thấy CI của Cisco chạy từ khoảng năm 2019.
   - Mã nguồn của cả hai driver giống hệt nhau ở Yoga, Antelope và Caracal (đã so sánh từng file).
4. **Đề xuất:** zone bằng tay (pre-zoning) mỗi khi thêm compute node, và không bật auto-zoning. Nếu buộc phải bật auto-zoning thì xem mục 5.4 và thử trên lab trước.

---

## 2. Thuật ngữ tối thiểu

| Thuật ngữ | Nghĩa |
|---|---|
| **Fabric** | Một mạng FC gồm một hoặc nhiều switch nối với nhau, dùng chung cấu hình zone. |
| **WWPN** | Địa chỉ cố định của một cổng FC, tương tự địa chỉ MAC. |
| **Zone** | Danh sách các WWPN được phép nhìn thấy nhau, ví dụ cổng HBA của host và cổng tủ. Không có zone thì host không thấy tủ. |
| **Zoneset (Brocade gọi là cfg)** | Tập hợp các zone. Mỗi fabric chỉ có **một** zoneset đang chạy (active). |
| **LUN masking** | Việc cấp quyền trên **tủ** cho host thấy volume nào. Cinder luôn làm việc này. Zoning là việc làm trên **switch**. |
| **ISL / ICL** | Đường nối giữa hai switch. ICL là cổng riêng của Brocade dùng để nối director với director, không chiếm cổng dành cho thiết bị. |
| **VF / VSAN** | Chia một switch vật lý thành nhiều fabric logic. Brocade gọi là Virtual Fabrics, Cisco gọi là VSAN. |
| **Gen 6 / 7 / 8** | Thế hệ FC, lần lượt 32G / 64G / 128G. |
| **POD** | Ports on Demand: switch giao với một số cổng đã bật, muốn dùng thêm cổng thì mua license. |

---

## 3. Director hay switch cố định?

| | Director | Switch cố định |
|---|---|---|
| Số cổng | Hàng trăm (Brocade X7-8: 512; Cisco MDS 9718: 768) | 48–128 |
| Dự phòng bên trong | 2 card điều khiển, nhiều nguồn và quạt, nâng cấp firmware không gián đoạn | 2 nguồn. Cisco 9148V/9396V có nâng cấp không gián đoạn (ISSU) |
| Hãng công bố | Thiết kế cho 99.999% (đây là mục tiêu thiết kế, không phải SLA) | Cisco 9148V/9396V cũng công bố 99.999% |
| Giá | Cao | Thấp hơn nhiều |

**Nên chọn gì:**

| Tình huống | Lựa chọn |
|---|---|
| SAN nhỏ (dưới ~100 cổng mỗi fabric) | 2 switch cố định, một cho fabric A, một cho fabric B |
| Cần mở rộng dần | Switch 2U có POD (Brocade G730, Cisco 9396V), sau này ghép thêm switch |
| SAN lớn, nhiều tủ | Mỗi fabric một director ở core, kèm switch ở đầu rack (edge) |
| Cần FICON (mainframe) hoặc nhân bản qua WAN (FCIP) | Director có card mở rộng |

**Fabric A/B:** dựng hai mạng FC độc lập. Mỗi host có ít nhất 2 cổng HBA, một cổng vào A, một cổng vào B. Mỗi controller của tủ cũng nối vào cả hai. Khi một fabric hỏng hoặc đang nâng cấp, multipath trên host tự chuyển sang fabric còn lại. Một director dù dự phòng tốt đến đâu vẫn chỉ là **một** fabric: chỉ cần kích hoạt nhầm zoneset là mất cả fabric.

---

## 4. Các dòng sản phẩm

### 4.1 Brocade (hệ điều hành FOS, phần mềm quản lý SANnav)

| | X6 | X7 | X8 | G620 | G720 | G730 |
|---|---|---|---|---|---|---|
| Loại | Director | Director | Director | Switch | Switch | Switch |
| Thế hệ | Gen 6 (32G) | Gen 7 (64G) | Gen 8 (128G) | Gen 6 (32G) | Gen 7 (64G) | Gen 7 (64G) |
| Kích thước | X6-4: 8U, X6-8: 14U | X7-4: 8U, X7-8: 14U | X8-4: 9U, X8-8: 14U | 1U | 1U | 2U |
| Cổng tối đa | 256 / 512 | 256 / 512 (với card FC64-64) | 192 / 384 | 64 | 64 (24 bật sẵn) | 128 (48 bật sẵn) |
| License | – | Gói Enterprise cài sẵn, chỉ bán thêm ICL POD | Mọi tính năng bật sẵn | Fabric Vision tính riêng | POD | POD |

- X6 lên X7: dùng lại chassis, chỉ thay card lõi. Port blade 32G cũ dùng tiếp được.
- X7 lên X8: **không nâng cấp được**, phải mua chassis mới. X8 nối với X7 qua ICL được, nối với X6 thì chỉ qua ISL.
- Nối director bằng ICL: tối đa 12 chassis theo mô hình core-edge, hoặc 9 chassis theo mô hình full-mesh.

### 4.2 Cisco MDS (hệ điều hành NX-OS, phần mềm quản lý DCNM, nay đổi tên thành NDFC)

| | MDS 9706 | MDS 9710 | MDS 9718 | MDS 9148V | MDS 9396V |
|---|---|---|---|---|---|
| Loại | Director | Director | Director | Switch | Switch |
| Kích thước | 9U | 14U | 26U | 1U | 2U |
| Cổng tối đa (64G) | 192 | 384 | 768 | 48 (24 bật sẵn, mua thêm 8 cổng mỗi lần) | 96 (48 bật sẵn, mua thêm 16 cổng mỗi lần) |
| NX-OS tối thiểu | 9.3(1) để dùng quang 64G | như 9706 | như 9706 | 9.3(1) | 9.4(1) |
| License | Enterprise | như 9706 | như 9706 | Thuê bao Advantage, hoặc Premier nếu cần SAN Analytics | như 9148V |

- Cả 3 director dùng chung một dòng chassis 9700. Muốn lên 32G hay 64G thì thay module, không thay chassis.

### 4.3 So sánh nhanh hai hãng

| | Brocade | Cisco |
|---|---|---|
| Tốc độ cao nhất | 128G (X8) | 64G |
| Chia fabric logic | Virtual Fabrics | VSAN (dùng phổ biến trên MDS) |
| Zoning gọn | Peer Zoning | Smart Zoning |
| API | REST | NX-API |
| Zone driver Cinder | Unsupported, giao thức REST/HTTP/SSH | Supported trên giấy, chỉ dùng SSH |

Về phần cứng, hai hãng ngang nhau ở 64G. Cả hai đều có zone driver Cinder yếu, nên **không nên lấy khả năng auto-zoning làm tiêu chí chọn hãng**. Nên chọn theo hạ tầng sẵn có, kỹ năng của đội vận hành và hợp đồng hỗ trợ.

---

## 5. Cinder FC Zone Manager (auto-zoning)

### 5.1 Hoạt động thế nào

Khi attach volume, driver tủ báo cho Cinder biết WWPN host nào cần nhìn thấy WWPN tủ nào. Nếu auto-zoning đang bật, Zone Manager đăng nhập vào switch (SSH hoặc REST), **thêm zone mới vào zoneset đang chạy rồi kích hoạt lại cả zoneset**. Khi detach thì xóa zone đó đi. Nếu tắt auto-zoning, Cinder chỉ làm LUN masking trên tủ, còn zone phải có sẵn trên switch.

Cả ba driver tủ trong `SAN.md` đều hỗ trợ việc này. Riêng Hitachi phải bật thêm `hitachi_zoning_request = True`. Tính năng *Port Scheduler* của Hitachi (`hitachi_port_scheduler`, có từ bản Zed) cũng cần Zone Manager.

### 5.2 Tình trạng driver (giống nhau ở Yoga, Antelope, Caracal)

| | Brocade | Cisco |
|---|---|---|
| Trạng thái | **Unsupported** từ Ussuri (12/2019). Commit ghi: *"Brocade has indicated they will no longer be supporting their zone manager driver"* | Supported. Từng bị đánh dấu unsupported vào 09/2017, đến 01/2018 bỏ đánh dấu |
| Cần `enable_unsupported_driver = True` (section `[fc-zone-manager]`) | Có | Không |
| CI kiểm thử của hãng | Không có | Không thấy hoạt động từ khoảng 2019 |
| Giao thức tới switch | `REST_HTTP` (mặc định, không mã hóa), `REST_HTTPS`, `HTTP`, `HTTPS`, `SSH` | Chỉ SSH |
| Hỗ trợ VF / VSAN | VF chỉ qua REST/HTTP, **không** qua SSH | VSAN, bắt buộc khai báo. Tên zoneset đang chạy phải trùng tên fabric trong `fc_fabric_names` |
| Firmware tài liệu ghi | FOS ≥ 6.4 (chưa cập nhật cho FOS 9/10) | NX-OS ≥ 6.2(9) (chưa cập nhật cho 9.x) |

Cả hai driver đều tạo một zone cho mỗi cặp (WWPN host, WWPN tủ), không dùng Peer Zoning hay Smart Zoning.

### 5.3 Rủi ro (đã kiểm tra trên mã nguồn)

| Rủi ro | Driver | Hậu quả |
|---|---|---|
| **Lỗi không báo:** quên `enable_unsupported_driver`, hoặc đặt `zoning_mode = fabric` sai section (mã nguồn chỉ đọc trong `[DEFAULT]`, dù tài liệu Cinder bảo đặt trong section backend) | Brocade / cả hai | Attach báo thành công nhưng không có zone nào được tạo, host không thấy LUN. Lỗi chỉ hiện trong log |
| **Lỗi không báo:** có người khác đang mở phiên sửa zone trên switch | Cisco | Driver bỏ qua, chỉ ghi log mức debug "Zoning session exists" |
| Kích hoạt lại toàn bộ zoneset | Cả hai | Nếu admin đã sửa zoneset nhưng chưa kích hoạt, các thay đổi dở dang đó cũng bị kích hoạt theo |
| Hủy thao tác của người khác | Brocade (SSH) | Khi gặp lỗi, driver chạy `cfgtransabort` |
| Mã cũ, chưa cập nhật cho firmware mới | Cả hai | Ví dụ hàm kiểm tra phiên bản FOS đọc sai các bản 10.x. Hiện hàm này chỉ chạy khi đã có lỗi khác nên ít ảnh hưởng |
| Mật khẩu switch nằm trong `cinder.conf` | Cả hai | Lỗi bên Cinder có thể làm hỏng zone của mọi host trên fabric, kể cả host không thuộc OpenStack |
| Số zone tăng nhanh | Cả hai | Mỗi cặp WWPN là một zone, dễ chạm giới hạn của zone database |

### 5.4 Đề xuất

1. **Mặc định zone bằng tay:** không đặt `zoning_mode`. Khi thêm compute node, zone các WWPN HBA của node với cổng tủ trên cả fabric A và B (nên dùng Peer Zone hoặc Smart Zone).
2. **Nếu buộc phải dùng auto-zoning:**
   - Thử trên lab với đúng phiên bản FOS hoặc NX-OS đang chạy.
   - Brocade: đặt `fc_southbound_protocol = REST_HTTPS` (mặc định `REST_HTTP` gửi mật khẩu không mã hóa) và dùng một tài khoản `zoneadmin` riêng cho Cinder.
   - Theo dõi log `cinder-volume`, bắt các câu: `Unsupported drivers are disabled`, `Zoning session exists`, `FC Zone Manager not enabled` (hai câu sau chỉ ghi ở mức debug, phải bật `debug = True` mới thấy).
   - Đặt quy định: không ai sửa zoneset trên switch mà để dở chưa kích hoạt.
3. Nếu cần Hitachi Port Scheduler thì phải dùng auto-zoning. Khi đó cần cân nhắc lại điểm 1.

---

## 6. Câu hỏi khi tiếp quản hệ thống

| # | Câu hỏi | Kiểm tra ở đâu |
|:---:|---|---|
| 1 | Đang dùng director hay switch? Hãng, model, bao nhiêu cổng đã có license? | Đội SAN; SANnav hoặc NDFC |
| 2 | Phiên bản FOS/NX-OS? Có nằm trong danh sách tương thích của hãng tủ không? | Brocade: lệnh `version`. Cisco: `show version` |
| 3 | Đã có fabric A/B chưa? Mỗi host và mỗi controller tủ có cổng ở cả hai fabric chưa? | Sơ đồ đấu nối; Brocade `nsshow`, Cisco `show fcns database vsan <id>` |
| 4 | Có dùng Virtual Fabrics hoặc nhiều VSAN không? | Cấu hình switch |
| 5 | Đang zone bằng tay hay auto-zoning? | `cinder.conf` có `zoning_mode = fabric` không; trên switch có zone tên bắt đầu bằng `openstack` không |
| 6 | Nếu đang auto-zoning: driver nào, log có cảnh báo "unsupported" không? | `cinder.conf`, log `cinder-volume` |
| 7 | Nếu zone bằng tay: quy trình khi thêm node hoặc thay HBA là gì? | Tài liệu vận hành, đội SAN |
| 8 | Ai có quyền sửa zone? | Danh sách user trên switch |
| 9 | Hợp đồng hỗ trợ phần cứng và phần mềm còn hạn không? | Bộ phận mua sắm |
| 10 | Có lab để thử trước không? | Trưởng dự án |

---

## Nguồn

**Mã nguồn Cinder** (thay `24.0.0` bằng `20.0.0` hoặc `22.0.0` để xem bản khác)
- [zonemanager/utils.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/zonemanager/utils.py): nơi đọc `zoning_mode` từ `[DEFAULT]` · [volume/configuration.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/volume/configuration.py)
- [fc_zone_manager.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/zonemanager/fc_zone_manager.py) (`enable_unsupported_driver` thuộc `[fc-zone-manager]`) · [brcd_fc_zone_driver.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/zonemanager/drivers/brocade/brcd_fc_zone_driver.py) · [brcd_fabric_opts.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/zonemanager/drivers/brocade/brcd_fabric_opts.py) (`fc_southbound_protocol` mặc định `REST_HTTP`) · [brcd_rest_fc_zone_client.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/zonemanager/drivers/brocade/brcd_rest_fc_zone_client.py) · [cisco_fc_zone_driver.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/zonemanager/drivers/cisco/cisco_fc_zone_driver.py)
- [hbsd_rest_fc.py](https://raw.githubusercontent.com/openstack/cinder/24.0.0/cinder/volume/drivers/hitachi/hbsd_rest_fc.py) (Hitachi `hitachi_zoning_request`)
- Commit: [Brocade unsupported (8c2e3cd)](https://github.com/openstack/cinder/commit/8c2e3cd66b) · [Cisco unsupported (c92c428)](https://github.com/openstack/cinder/commit/c92c428233) · [Cisco bỏ đánh dấu (47b2c53)](https://github.com/openstack/cinder/commit/47b2c53952)
- Tài liệu: [FC Zone Manager (2024.1)](https://docs.openstack.org/cinder/2024.1/configuration/block-storage/fc-zoning.html) · [Danh sách driver](https://docs.openstack.org/cinder/2024.1/drivers.html)
- CI Cisco: [Gerrit, các bình luận của tài khoản 22804](https://review.opendev.org/q/commentby:22804)

**Brocade:** [X8 Product Brief](https://docs.broadcom.com/doc/X8-Director-PB) · [X7 Product Brief](https://docs.broadcom.com/doc/X7-Director-PB) · [X6 Data Sheet](https://docs.broadcom.com/doc/GA-DS-5721) · [G720](https://docs.broadcom.com/doc/G720-Switch-PB) · [G730](https://docs.broadcom.com/doc/G730-Switch-PB) · [G620 FAQ](https://docs.broadcom.com/doc/GA-FAQ-6312)

**Cisco:** [MDS 9700 so sánh model](https://www.cisco.com/c/en/us/products/storage-networking/mds-9700-series-multilayer-directors/models-comparison.html) · [Module 48 cổng 64G](https://www.cisco.com/c/en/us/products/collateral/storage-networking/mds-9700-series-multilayer-directors/datasheet-c78-2466559.html) · [9148V](https://www.cisco.com/c/en/us/products/collateral/storage-networking/mds-9100-series-multilayer-fabric-switches/mds-9148v-fibre-channel-switch-ds.html) · [9396V](https://www.cisco.com/c/en/us/products/collateral/storage-networking/mds-9300-series-multilayer-fabric-switches/nb-06-mds-9396v-64-gbps-96-port-fiber-ds-cte-en.html)
