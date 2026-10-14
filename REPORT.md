MSSV: 202602429

# Lab 11 · Scenario Forge — báo cáo

Thị trấn A (`evidence/meta.json: _run.town`). Toàn bộ `evidence/` được tạo bằng ngân sách đầy đủ (`_run.quick = false`). Các xác suất dưới đây thuộc **thế giới mô phỏng và phân phối dạy học**, không phải tần suất tai nạn trên đường thật.

## 1. Phát lại log có trung thực không?

`evidence/replay.json`: 400 cảnh, loại 2 log mà người lái gây va chạm, còn 398. Planner trên log phát lại va chạm 112 lần (`rear` 100, `side` 12), còn trong thế giới phản ứng là 0; `crash_in_both = 0`. Cảnh `replay_only[0]` là kịch bản số 0, va chạm `rear`; `min_ttc_reactive = 2,318 s`. Chạy riêng tham số trong `replay_only[0].params` cho thấy TG tông đuôi ego ở 16,75 s khi phát lại. Log cố định quỹ đạo TG theo **người lái thật**, nên TG không phanh đáp lại việc planner giảm tốc sớm hơn. Đây là va chạm giả do thiếu phản hồi, không phải bằng chứng planner sẽ bị tông trong thế giới phản ứng.

## 2. Kịch bản bất khả thi

`lab11/feasible.py` loại cảnh khi gia tốc ngang đỉnh của chuyển làn cosine `1,75(π/cutDur)² > μg`, khi `cutBrake > μg`, khi `pedV > 2,5 m/s`, hoặc `cutV` ngoài `(0, 22] m/s`. Hai luật đầu giới hạn bởi độ bám lốp–đường; luật người đi bộ giữ đúng giả định đi bộ thay vì chạy; luật tốc độ loại xe đứng/đi lùi và tốc độ ngoài miền bài toán. `evidence/hunt.json: rear_ended.infeasible = 9` là nhiều nhất (so với 6, 2, 7 ở ba cuộc săn khác). Đối chiếu điểm `infeasible` trong `evidence/cloud.json`, cả 9 vi phạm giới hạn gia tốc ngang. CEM thích chuyển làn rất gấp vì nó làm điểm nguy hiểm tăng; nếu chạy rồi tính các cảnh không có bám đường ấy là va chạm, xác suất bị thổi phồng.

## 3. Monte Carlo cho biết gì?

`evidence/mc.json`: `n = 3000`, `crashes = 4`, `p = 0,001333`; Wilson 95% `[0,000519; 0,003424]`. Cả 4 thuộc `bus_bypass`. Ước tính cần `n ≈ 96/p ≈ 72.000` lần để sai số 95% khoảng ±20% của p̂. `cut_in`, `crosswalk_block`, `rear_ended` và `other` chưa xuất hiện trong 3.000 lần; mỗi kiểu có cận Wilson trên `mc.json: per_mode.*.hi = 0,001279`, **không phải xác suất 0**. Có 14 bộ tham số bất khả thi (`mc.json: infeasible`); chúng không được chạy hoặc tính là va chạm.

## 4. Importance sampling, ESS và độ suy biến

`evidence/estimate.json`; mỗi ô là `p / hits / ess / top_share`:

| Kiểu | Seed 0 | Seed 1 | Seed 2 |
|---|---:|---:|---:|
| `bus_bypass` | `5,83e-4 / 40 / 145 / 0,509` | `9,32e-4 / 39 / 139 / 0,519` | `1,68e-3 / 43 / 118 / 0,439` |
| `cut_in` | `chưa thấy / 0 / 124 / 0` | `chưa thấy / 0 / 112 / 0` | `chưa thấy / 0 / 120 / 0` |
| `crosswalk_block` | `1,15e-6 / 76 / 141 / 0,512` | `1,46e-5 / 66 / 130 / 0,958` | `5,14e-6 / 79 / 140 / 0,384` |
| `rear_ended` | `chưa thấy / 0 / 111 / 0` | `chưa thấy / 0 / 119 / 0` | `chưa thấy / 0 / 114 / 0` |

Tổng ba seed (`seeds[*].total`) là `5,84e-4`, `9,47e-4`, `1,69e-3`: cùng bậc với MC `1,33e-3`, nhưng chênh gần 3 lần; riêng `crosswalk_block` chênh hơn 12 lần. Bus seed 0–1 và crosswalk seed 0–1 có `top_share > 0,5`. Giá trị 0,958 của crosswalk seed 1 nghĩa là **một mẫu** mang gần 96% ước lượng dù có 66 lần trúng; khoảng ±1,96·se lúc này chỉ là xấp xỉ dễ lạc quan. `top_share` thấp cũng không chứng minh q phủ hết vùng va chạm.

Không trúng `cut_in` hay `rear_ended` trong 600 mẫu/seed thì dùng `upper_zero_hits = 3/(0,2×600) = 0,025` (`estimate.json: seeds[*].per.*.upper_zero_hits`), không báo 0 và không dùng `3/ESS`. Phần `other` không có mục tiêu săn; lấy từ MC (`other.k = 0`, `other.n = 3000`, `other.hi = 0,001279`) rồi cộng vào tổng/cận. Ngân sách `estimate.json: budget = 16.800` lượt so với khoảng 72.000 lượt MC cho mục tiêu ±20% tại p̂; IS dùng ít lượt hơn, nhưng ba seed và trọng số đỉnh cho thấy chưa đạt độ ổn định tương đương.

## 5. Các kiểu thất bại

- `bus_bypass`: `evidence/hunt.json: bus_bypass.found.bus_bypass = 155`. C2 chờ khe rồi lách qua xe buýt, chậm và lấn vào hành lang của ego; ego nhận C2 làm xe dẫn quá muộn nên đầu xe chạm C2. Cảnh B-01 ở 12,65 s: [ảnh va chạm](evidence/shots/bus_bypass.png).
- `crosswalk_block`: `evidence/hunt.json: crosswalk_block.found.crosswalk_block = 38`. Xe buýt che P1; ego phanh, dừng với thân xe trên vạch và P1 phải đứng lại. Cảnh C-04 ở 18,25 s có `kind = block`, không phải tiếp xúc vật lý: [ảnh chặn lối](evidence/shots/crosswalk_block.png).

Cuộc săn `cut_in` chỉ thấy 28 ca `bus_bypass`; cuộc săn `rear_ended` thấy 2 `bus_bypass` và 5 `crosswalk_block`, không tìm được kiểu đích (`hunt.json: *.found`). Điều đó không chứng minh hai kiểu ấy không tồn tại. Nếu dùng một CEM chung, đề xuất dễ co vào kiểu có điểm nguy hiểm tăng nhanh nhất rồi lặp kiểu ấy; các cuộc săn riêng cho thấy rõ kiểu nào vẫn chưa được khảo sát đủ.

## 6. Sửa planner, kiểm tra trung thực

`evidence/fix.json`: chọn `{"cutinLat": 3.0}` **chỉ từ TRAIN**; policy gốc có `cutinLat = 2.15`. Ngưỡng ngang lớn hơn khiến ego coi C2 đang lấn là xe dẫn sớm hơn.

| Tập và policy | `fail` | `fail_hi_seen` | `fail_hi` | Quãng đường | Số khung phanh gấp |
|---|---:|---:|---:|---:|---:|
| TRAIN gốc (`baseline_train`) | `2,378e-3` | `4,998e-3` | `4,998e-3` | `213,835 m` | `8,098` |
| TRAIN chọn (`chosen`) | `2,926e-7` | `6,544e-7` | `6,544e-7` | `213,815 m` | `7,844` |
| TEST gốc (`baseline_test`) | `8,359e-4` | `1,521e-3` | `2,800e-3` | `214,453 m` | `8,123` |
| TEST chọn (`chosen_test`) | `5,942e-7` | `1,005e-6` | `3,878e-2` | `214,329 m` | `7,807` |

TEST dùng seed mới, săn lại theo từng policy. `chosen_test.unseen = ["bus_bypass"]`; dù chưa thấy kiểu này, `fail_hi` vẫn cộng cận `3/(0,2×400) = 0,0375` cùng cận cho va chạm khác. Vì thế cận hợp lệ của bản sửa lớn hơn cả điểm ước lượng của bản gốc, dù dữ liệu thấy ít thất bại hơn. Quãng đường TEST chỉ giảm `0,124 m` (khoảng 0,06%), dưới mức 5%. [Ảnh bóng policy gốc](evidence/shots/policy_ghost.png) dùng đúng tham số B-01 từ `viewer.json: runs[0]`, phát lại dưới policy chọn; nhãn F-01 là bản xem tạm của cùng cảnh. Bản chọn tránh C2, bóng gốc chạm C2 ở 12,7 s.

Núm thận trọng không đơn điệu: TRAIN với `cautionV = 6,0` cho `fail = 4,70e-4`, cao hơn `1,75e-4` ở `cautionV = 6,5` (`fix.json: table[2]` so với `table[1]`), dù mục tiêu tốc độ thấp hơn. Ego đi đến xe buýt/vạch vào thời điểm khác, làm xe cắt làn và người đi bộ phản ứng khác; đây là giải thích theo cơ chế mô phỏng, không phải quan hệ nhân quả đã kiểm chứng ngoài đời.

## 7. Từ mô phỏng ra đường thật

| Giả định `avsim/space.py` | Dữ liệu thật để thay | Nếu giả định đang lạc quan |
|---|---|---|
| Trung vị `cutGap = 12 m` ở thị trấn A | Quỹ đạo xe nhập làn từ camera/radar đội xe thử nghiệm, phân tầng theo tốc độ và mật độ | Khe thật nhỏ hơn sẽ khiến mô hình đánh giá thiếu nguy cơ C2 lấn sát ego. |
| `pedLook = 0,85` | Video người chờ ở vạch có xe buýt che; ghi liếc nhìn, thời điểm bước xuống và khoảng cách xe | Ít người nhìn qua xe buýt hơn có thể làm mô hình đánh giá thiếu băng ngang bất ngờ và `crosswalk_block`. |
| `wet = 0,20`, `μ = 0,5` khi ướt | Cảm biến mưa và ước lượng ma sát lốp–đường trên các tuyến, theo mùa/loại mặt đường | Nếu ướt thường hơn hoặc μ thấp hơn, mô hình cho khoảng phanh quá ngắn và đánh giá thiếu rủi ro. |

## 8. Giới hạn

Xác suất trên không bao trùm (1) mạng đường/nút giao và chuyến dài hơn một phố trong 24 s; (2) hành vi khác IDM như mất tập trung, đánh lái bất ngờ và thương lượng không theo luật khe; (3) nhiễu hoặc độ trễ cảm biến ngoài phần bị xe buýt che. Nếu có thêm một tuần, tôi sẽ lấy video/quỹ đạo đa tuyến để khớp các phân phối câu 7, thêm độ trễ cảm biến, và kiểm định trên holdout lớn theo từng ODD/kiểu thất bại.

## 9. Rắc rối của tôi: xe đạp né chướng ngại và lạng vào làn ego

`hazard.json` là thẻ `bike` H0 ở làn E2, bắt đầu lạng khi ego còn cách 20 m. Cảnh gốc khả thi; `evidence/mine.json: exact.planner = [(ego, H0, 12,8 s)]`. Planner hiện dự báo xe lấn là xe dẫn qua ngưỡng `cutinLat`; một xe đạp vừa chậm vừa lạng có thể vào hành lang phanh trước khi planner phản ứng đủ. [Ảnh GB-01 ở 13,4 s](evidence/shots/bike_swerve.png) cho thấy bản sửa va chạm H0 trong một biến thể quét, dù policy gốc của riêng biến thể này không va chạm; cải thiện trung bình không có nghĩa cải thiện từng cảnh.

**Giả định:** `hazard.json: space` cho `x = 40–90 m`, `v = 4,5–6 m/s`, lạng ngang `3,6–4,0 m` trong `2,7–3,3 s`, kích hoạt ở `18–30 m`. Khoảng lạng lấy từ khoảng cách tâm làn E1/E2 3,5 m của bản đồ cộng vị trí lệch ban đầu; khoảng thời gian giữ gia tốc ngang đỉnh `π²·swerve/(2·swDur²)` dưới 3 m/s². [Nghiên cứu quan sát ở Metro Manila, bảng 5–6](https://www.jstage.jst.go.jp/article/easts/16/0/16_PP4133/_pdf/-char/en) ghi nhận 12/954 và 21/1.124 xe đạp chuyển làn khi qua các vị trí có hư hỏng mặt đường, gộp 33/2.078 ≈ 1,59% mỗi lượt qua vị trí **đã chọn sẵn**. Để có `exposure = 0,005/chuyến`, tôi giả định 0,3 lượt gặp xe đạp ở vị trí hư hỏng/chuyến × 33/2.078 ≈ 0,0048 rồi làm tròn. Hệ số 0,3 chưa được đo, và nghiên cứu không đo biên độ lạng 3,6–4,0 m; cần video hành trình địa phương có xe đạp, mặt đường và mẫu số chuyến để hiệu chỉnh. Do đó tần suất/chuyến dưới đây chỉ là phân tích nhạy cảm với giả định ấy.

`mine.json: accepted = 1496/1500 = 99,73%`; không có cảnh chồng lấn lúc bắt đầu. Trên **1.496 cảnh hợp lệ**:

| Policy | Va chạm | `avoidable` | `unavoidable` | `unattributed` | `low_speed` | `avoidable_foreknown` | P(lỗi planner \| rắc rối), Wilson 95% |
|---|---:|---:|---:|---:|---:|---:|---:|
| Gốc (`mine.json: baseline`) | 209 | 207 | 1 | 1 | 0 | 208 | `0,13837 [0,12180; 0,15679]` |
| Chọn (`mine.json: chosen`) | 20 | 20 | 0 | 0 | 3 | 20 | `0,01337 [0,00867; 0,02056]` |

`avoidable` nghĩa là **phanh hết lực từ khung đầu tiên xe ta thấy rắc rối** thì tránh được; `unavoidable` là vẫn đâm, còn `unattributed` là va chạm thường chưa quy cho H0. `low_speed` là cờ chồng lên loại, ở bản chọn có 3 ca ego dưới 1 m/s. Chênh 208 so với 207 ở `avoidable_foreknown` của bản gốc là một ca chỉ tránh nếu phanh từ lúc H0 bắt đầu dù chưa thấy. Chuẩn “tránh được” lạc quan vì oracle phanh tức thì, tối đa và biết chính xác khi rắc rối nhìn thấy; song chỉ thử phanh, không thử đánh lái nên cũng có thể bỏ sót phương án tránh khác.

Nhân giả định exposure, `mine.json: baseline.per_trip = 6,918e-4` và `chosen.per_trip = 6,684e-5` lỗi planner/chuyến. Giá trị gốc bằng khoảng 0,52 lần xác suất `bus_bypass` từ MC (`mc.json: p = 1,333e-3`, chỉ chứa kiểu này ở lần chạy này), nhưng so sánh phụ thuộc mạnh vào exposure tự đặt. Bản sửa `cutinLat = 3.0` giảm 207 xuống 20 ca tránh được nhưng chưa loại hết. Cần logic theo dõi **quỹ đạo ngang và TTC của xe đạp**, dự báo nó cắt vào làn rồi phanh sớm theo giới hạn bám đường, thay vì chỉ chờ vượt ngưỡng ngang tĩnh. Tôi không tự lái trong chương 10 nên không tuyên bố đã thắng planner bằng lái tay.
