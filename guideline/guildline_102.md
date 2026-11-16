# Guideline gán nhãn —

- Loại nhãn: Bounding box
- Người viết: Nguyễn Hoàn Thiện
- Phiên bản: v0.1 
- Dữ liệu: 40 ảnh, cạnh dài ≤ 960 px. Công cụ CVAT, xuất COCO 1.0.
- Taxonomy hiện có (không đổi trong bản này): `car`, `truck`, `van`, `motorcycle`, `person`, `ignore_region`.

---

## 1. Mục tiêu

Mô hình **phát hiện vật 2D** trên ảnh đường phố (camera giao thông, hỗ trợ lái). Nhãn dạy mô hình:
**"có một người / xe thật, đang ở đây, và nó nằm ở hộp này"**.

Sai nguy hiểm nhất, theo thứ tự:
1. Box trên vật **không thật** (tượng, ảnh in, phản chiếu, đồ chơi) → mô hình học báo động giả (phanh gấp vì tượng).
2. **Bỏ sót** người / xe thật (đặc biệt người đi bộ, người đi xe máy).
3. Gán **sai lớp** (nhầm người vs vật; xe tải vs xe con).
4. Box lỏng / lệch (ảnh hưởng IoU; IoU ghép ≥ 0.5, ngưỡng đạt 0.85 với vật cao ≥ 50 px, 0.7 với vật nhỏ hơn).

**Phép thử chung** (dùng cho mọi ca chưa có quy tắc): *"Mô hình có phải đoán ra điều này không?"*
Nếu chỉ nhìn ảnh mà người gán không chắc chắn → không ép nhãn, đi theo R-12 (ignore / `can_review`).

---

## 2. Lớp và thuộc tính

### Lớp

| Lớp | Định nghĩa (một câu) | Ví dụ | Phản ví dụ / dễ nhầm |
|---|---|---|---|
| `person` | Người thật, đang đứng / đi / ngồi / đạp xe / ngồi trên xe, thấy được ít nhất đầu hoặc thân. | img_034 người băng đường; mỗi người trên xe máy ở img_006 | Tượng (img_017), ma-nơ-canh (img_033), ảnh người trên billboard (img_018), linh vật bông (img_019) |
| `motorcycle` | Xe hai bánh có động cơ, kể cả xe ga, xe có thùng bên hông (sidecar). | img_010 hai xe ga; img_022 xe có sidecar | Xe đạp (không có lớp), xích lô, tuk-tuk (ba bánh) |
| `car` | Ô tô con, SUV, taxi con, xe thể thao; **xe thật có động cơ chở người** thuộc nhóm này. | img_023 Honda đen; img_024 xe Pacer đen | Xe đồ chơi (img_038), mô hình đặt trên nóc xe (img_024) |
| `truck` | Xe có thùng hàng riêng: xe tải, **bán tải (pickup)**, xe đầu kéo + rơ-moóc, xe chở xe. | img_036 xe tải nhỏ xanh; img_012, img_037 pickup; img_002 xe chở xe | Xe SUV (car); xe van kín (van) |
| `van` | Xe thùng kín cỡ nhỏ, 1 khối (cabin liền thùng), ≤ ~15 chỗ. | img_005 và img_016 xe van | Xe buýt lớn / xe khách (ngoài taxonomy) |
| `ignore_region` | Vùng không gán từng vật; bỏ sót ở đây **không tính lỗi** (R-08, R-09, R-10). | cụm xe máy dày ở img_021; xe buýt ở img_009 | Dùng để "né" vật rõ ràng mà chỉ vì lười vẽ → bị coi là lỗi |

**Vật không thuộc lớp nào (ngoài taxonomy):** xe buýt / xe khách, tuk-tuk / xe lam / auto-rickshaw, xích lô (cyclo), xe đạp, xe ba gác.
- Không ép vào lớp gần nhất (xe lam ≠ `car` ≠ `motorcycle`).
- Vẽ `ignore_region` ôm phương tiện đó. **Người ngồi/lái trên đó vẫn vẽ `person`** (box `person` nằm trong vùng ignore, cho phép chồng).
- Ghi vào danh sách ca họp nhóm, **đề xuất** thêm lớp `other_vehicle` ở Pha 2 (chưa phải quyết định). Ca bị ảnh hưởng: img_004, 009, 016, 023, 035, 037, 040, 006 (xích lô, xe đạp).

### Thuộc tính (mức cố định, ngưỡng đo được)

**`occluded_level`** — đo theo **phần diện tích nhìn thấy của vật** so với toàn bộ vật (ước lượng phần bị che từ hình dạng quen thuộc).

| Mức | Phần nhìn thấy | Cách nhận biết nhanh |
|---|---|---|
| `0` | ≥ 90% | Thấy trọn vật |
| `1` | 60% – < 90% | Che một góc / một phần nhỏ |
| `2` | 25% – < 60% | Che quá nửa nhưng còn rõ dạng |
| `3` | < 25% | Chỉ thấy mảnh nhỏ nhưng **vẫn nhận ra lớp** (nếu không nhận ra lớp: R-12) |

- Vật che **mảnh** (hàng rào lưới, dây điện, cành cây thưa, kính xe): nếu vẫn nhận ra vật thì **không tính là che** (img_018 xe sau hàng rào: mức 0–1).
- Người ngồi sau người khác trên xe máy (img_005): thường mức 2–3.
- Cờ `occluded` có sẵn của CVAT (phím `Q`) **bỏ qua**, script không đọc.

**`truncated`** — `yes` khi vật bị **cắt bởi mép ảnh** (cạnh của box chạm mép ảnh, sai số ≤ 2 px) và phần ngoài ảnh là phần thân thật của vật. Không dùng cho vật bị che bởi vật khác. Box **dừng đúng ở mép ảnh**, không vẽ ra ngoài.

**`can_review`** — tick khi **thật sự phân vân giữa hai quyết định có quy tắc khác nhau**. Không tick cho box chỉ hơi lỏng. Tick cho: lớp ranh giới (xe thể thao/props), nghi vật giả, mức che sát ngưỡng. Ghi lý do vào danh sách ca (ảnh, box, vì sao) để họp.

---

## 3. Quy tắc vẽ box

| ID | Quy tắc |
|---|---|
| **R-01** | **Chỉ vẽ phần NHÌN THẤY**. Không đoán phần khuất (không vẽ box "suy diễn đủ vật"). Với vật bị che, box ôm đúng phần thấy. |
| **R-02** | **Ôm sát** thân vật (người: đầu → chân hoặc phần thấy; xe: thân + bánh + gương). Sai số mục tiêu ≤ 2–3 px hoặc ≤ 5% cạnh. **Không gồm**: bóng đổ, phản chiếu mặt đường ướt (img_013, img_015, img_031), ăng-ten, dây, vật nhô ra xa. |
| **R-03** | **Người đi xe máy = nhiều box**: mỗi người là một `person` (kể cả người ngồi sau, ngồi sidecar img_022) **+** một `motorcycle` ôm riêng phần xe (bánh, tay lái, yên). Các box được **chồng nhau**. Thử trên img_006, img_026 (hai người, một xe), img_008, img_010. Không dùng một box chung "người+xe". |
| **R-04** | **Xe đạp, xích lô, xe ba bánh**: chỉ vẽ `person` cho người đạp/lái. Phương tiện đó theo ngoài taxonomy ở mục 2. |
| **R-05** | **Hàng hóa và vật nhô ra không thuộc box vật mang**: ống dài (img_029), gánh bánh trái (img_003), hàng chất cao trên xe lam (img_035), thùng/bao trên xe bán tải (img_012). Box xe chỉ ôm thân xe; cụm gắn cố định như mui che (canopy, img_037), thùng xe, nắp thùng **thuộc xe**. |
| **R-06** | **Xe chở xe** (img_002): xe đầu kéo + rơ-moóc đã ghép = **một** box `truck`. Từng xe con đang được chở là xe thật nhìn thấy → **mỗi xe vẽ một box `car`** (chồng trong box `truck`), mức che theo thực tế. Nếu nhóm đổi quy tắc này, phải kiểm lại ca chở hàng khác (img_001, img_012, img_035). |
| **R-07** | **Người trong xe kín/có kính** (taxi, ô tô) **không vẽ** `person`. Người ở xe **hở** (xe máy, tuk-tuk, xích lô, xe mui trần img_019) **vẽ** nếu thấy rõ. |
| **R-08** | **Vật nhỏ**: chiều cao box < 20 px (`min_box_height_px`) → **không** vẽ box; nếu có nhiều vật, phủ `ignore_region`. Đo bằng pixel trên ảnh gốc trong CVAT. Ví dụ: người xa ở img_011, img_013, img_025, img_039. |
| **R-09** | **Đám đông / cụm dày**: vẽ box riêng cho vật thỏa **cả ba**: (a) nhìn thấy ≥ 25% (mức ≤ 2), (b) chiều cao ≥ 20 px, (c) tách được bằng mắt khỏi vật kề. Phần cụm còn lại phủ **`ignore_region`**. Ca: img_021 (xe máy ùn), img_030 (xe máy đậu), img_019 và img_032 (đám đông), img_008, img_036 (hàng xe máy đậu). |
| **R-10** | **Mờ nhòe chuyển động** (img_014, img_015): vẽ nếu còn nhận ra lớp **và** ranh giới đủ rõ để ôm sát; vệt mờ không nhận ra → `ignore_region`. |
| **R-11** | **Vật KHÔNG thật → không box (không ignore)**: tượng (img_017 người+ngựa, img_027 bia đá hình xe, img_028 tượng ngựa trên thùng xe), ma-nơ-canh (img_033), người/xe trong ảnh in hoặc biển (img_018, img_026 tranh tường cảng), xe và đồ chơi (img_038, nóc xe img_024), hình phản chiếu (img_020 gương chiếu hậu, img_013 / 031 mặt đường), tác phẩm điêu khắc kim loại (img_011: xe máy làm bằng sắt vụn và tượng Predator), linh vật bông (img_019). Đây là **mẫu âm (hard negative)**: để mô hình học "không phải người / xe". Tick `can_review` ở vật liền kề nếu cần. Ghi vào danh sách ca. |
| **R-12** | **Không chắc lớp / chỉ thấy mảnh nhận không ra** (ví dụ chỉ thấy gương xe máy ở img_040): không đoán lớp; phủ `ignore_region` (nếu đủ lớn) hoặc bỏ nếu < 20 px. Có thể tick `can_review` trên box lân cận. |
| **R-13** | **Xe chủ (ego) / vật của người chụp** (thân xe ở img_020, nắp capô): **không** gán. |
| **R-14** | **Xe đậu, xe không người lái** vẫn gán như xe đang chạy (img_003, img_010, img_012, img_030, img_036). Xe trong **nhà kho / bảo tàng** (img_007, img_024) vẫn gán nếu là xe thật. |
| **R-15** | **Phân lớp xe**: bán tải (cabin + thùng hở) → `truck` (taxonomy v0: "xe có thùng hàng riêng"); SUV, crossover, wagon → `car`; xe van kín nhỏ → `van`; xe khách/minibus lớn (img_009) → ngoài taxonomy (mục 2, `ignore_region`). Xe tải nhỏ có mui bạt (img_036) → `truck`. |
| **R-16** | **Xe giả nhưng chạy được trên đường** (img_019 xe Danger Mouse có biển số, có tài xế thật): xem là xe thật → `car`, tick `can_review`; người lái → `person`. |
| **R-17** | **Ảnh không có vật theo phạm vi** (img_017, img_033, img_038, img_027 nếu không có vật thật): vẫn nằm trong Task, **không box**. Không vẽ để "cho có". |

---

## 4. Ca khó

| Ảnh | Tình huống | Rule | Quyết định | Lý do |
|---|---|---|---|---|
| img_017 | Tượng người cưỡi ngựa, có nón cone, có chim | R-11, R-17 | Không box, ảnh rỗng | Tượng không phải người; tượng là ca IAA cao mà vẫn sai nếu hai người cùng vẽ `person` |
| img_002 | Xe chở xe | R-06 | 1 `truck` (đầu kéo + rơ-moóc) + từng `car` nhìn thấy | Mô hình phải biết cả hai là vật thật; tránh hai cách vẽ khác nhau |
| img_005 | Cận cảnh người đi xe máy, nhiều người bị che, 1 xe van nền | R-01, R-03, R-09, R-12 | `person` cho từng đầu/thân thấy ≥ 20 px; xe máy chỉ khi thấy tay lái/gương rõ; phần chen chúc → `ignore_region` | Che nhiều (mức 2–3); chỉ vẽ phần thấy |
| img_026 | Hai người trên xe máy, xe bị cắt mép dưới | R-03, `truncated` | 2 `person` + 1 `motorcycle` (`truncated=yes`) | Mép dưới cắt xe; tranh tường nền không thật |
| img_004 | Hai tuk-tuk, tài xế ngồi trong | Ngoài taxonomy, R-07 | `ignore_region` ôm mỗi tuk-tuk; `person` cho tài xế nếu thấy rõ | Xe ba bánh không có lớp |
| img_016 | Xe van, xe ga có người, hai tuk-tuk, người cắt mép phải | R-03, R-15 | `van`, `motorcycle`+`person`, `ignore_region` tuk-tuk, `person` (`truncated=yes`) | |
| img_023 | Xích lô + ô tô Honda | R-04 | `car` Honda; `person` người đạp; `ignore_region` ôm xích lô | Xích lô không có lớp |
| img_028 | Xe pickup cổ có tượng ngựa trên thùng | R-11, R-15 | `truck` ôm thân xe; không box cho ngựa | Ngựa là tượng |
| img_027 | Bia mộ đá hình xe pickup | R-11, R-17 | Không box | Không phải xe thật |
| img_024 | Xe Pacer đen, mô hình xe vàng trên nóc, nhiều xe cổ nền | R-11, R-14 | `car` Pacer (box không gồm vật trên nóc); xe nền thật thì gán nếu thỏa R-08/R-09 | Mô hình vàng không phải xe thật |
| img_019 | Xe Danger Mouse (diễu hành) + đám đông sau rào | R-16, R-09 | `car` (`can_review`), `person` tài xế, người hàng đầu gán, đám sau `ignore_region`; linh vật bông không box | |
| img_020 | Gương chiếu hậu có hình phản chiếu | R-11, R-13 | Không box | Phản chiếu và xe chủ |
| img_033 | Ma-nơ-canh trong cửa kính | R-11, R-17 | Không box | |
| img_018 | Biển quảng cáo có mặt người; ô tô sau hàng rào | R-11 | Không `person` cho biển; `car` cho xe thật thấy rõ (hàng rào lưới là vật che mảnh, xem `occluded_level` ở mục 2) | |
| img_038 | Tủ trưng bày xe và thú đồ chơi | R-11, R-17 | Không box | |
| img_011 | Tượng Predator + xe máy sắt, người rất xa | R-11, R-08 | Không box điêu khắc; người xa < 20 px không box | |
| img_029 | Xe ga chở ống dài + xe đạp | R-05, R-04 | `motorcycle`+`person`, `person` người đạp; ống không vào box | |
| img_003 | Gánh bán hàng, xe đạp, xe máy đậu | R-05, R-14 | `person` (gánh không vào box); xe máy đậu gán | |
| img_035 | Xe lam chất hàng cao, xe máy có người, ô tô | R-05, ngoài taxonomy | `ignore_region` xe lam; `motorcycle`+`person`; `car` | |
| img_022 | Ảnh trắng đen, xe có sidecar, người ngồi sidecar | R-03 | 1 `motorcycle` (gồm sidecar) + `person` ngồi sidecar (che nhiều) | Sidecar là phần của xe |
| img_021 / 030 | Cụm xe máy ùn tắc / đậu dày | R-09 | Box riêng cho vật tách được; còn lại `ignore_region` | Không tự hứa đếm hết |
| img_014 | Nền mờ nhòe, hai người rõ | R-10 | `person` 2 người rõ; xe nền mờ không nhận ra → `ignore_region` | |
| img_013, 031 | Ban đêm, đường ướt phản chiếu đèn | R-02, R-08 | Không box ở phản chiếu; vật thật gán nếu ≥ 20 px | |
| img_040 | Cận cảnh người đội nón bảo hiểm; thấy gương xe | R-12 | `person` (thấy đầu/thân); gương không đủ nhận ra xe → không box; taxi xanh `car` (cắt mép phải); tuk-tuk nền → ignore | |

---

## 5. QC và phiên bản

**Ngưỡng QC áp dụng** (từ `taxonomy.json`, không sửa): IoU ghép ≥ 0.5; đạt IoU 0.85 (vật cao ≥ 50 px) / 0.7 (vật nhỏ); IAA: IoU TB ≥ 0.85, cùng lớp ≥ 95%; vật < 20 px → `ignore_region`; Release khi lỗi theo ảnh ≤ 2% **trên toàn lô**.

**Ai quyết ca mơ hồ:** ở bản cá nhân này, tôi chỉ ghi ca vào danh sách; quyết định chính thức do nhóm ở Pha 2 / 5, mỗi ca một mã quy tắc `R-xx` và một người chốt.

**Việc nên làm trước khi gán:** thử R-03 (người đi xe máy) trên **một ảnh thứ hai** (img_010 hoặc img_008) để chắc quy tắc dùng được ngoài ảnh gốc.

| Phiên bản | Thay đổi | Lý do (ca) |
|---|---|---|
| v0 | Bản khởi đầu của giảng viên | |
| v0.1 | Thêm R-01…R-17, ngưỡng `occluded_level`, quy tắc `truncated`, bảng ca khó | Xe chở xe (img_002), người đi xe máy (img_005, 006, 026), tượng và vật giả (img_017, 027, 028, 033, 038), vật ngoài taxonomy (img_004, 016, 023, 035) |

---

## Danh sách tự đối chiếu (so với guideline của bạn)

1. Người đi xe máy: bạn vẽ mấy box, phần nào vào box nào? (bản tham khảo: 1 box `motorcycle` + 1 box `person` mỗi người, chồng nhau)
2. Xe chở xe (img_002): 1 box hay nhiều box?
3. Tượng, ma-nơ-canh, ảnh in, phản chiếu, đồ chơi: không box hay có box? (nên **không** box; ghi rõ là mẫu âm)
4. Xe bán tải: `truck` hay `car`? Bạn có nêu căn cứ không?
5. Tuk-tuk, xích lô, xe buýt, xe đạp: bạn xử lý thế nào, và có nói rõ người ngồi/lái thì sao?
6. Ngưỡng `occluded_level`: bạn có **con số %** cho từng mức chưa?
7. `truncated`: định nghĩa có phân biệt với "bị vật khác che" không?
8. Vật nhỏ: bạn dùng ngưỡng 20 px của `rules` chưa, đo bằng gì?
9. Đám đông: có tiêu chí **đo được** để quyết định vẽ riêng hay phủ `ignore_region` không?
10. Mờ nhòe, ban đêm, phản chiếu mặt đường ướt?
11. Hàng hóa nhô ra, vật mang theo: có nằm trong box vật mang không?
12. Bạn có dẫn ID ảnh (`img_0xx`) cho từng ca không? (chiếm 6/15 điểm mục 1 của rubric)
