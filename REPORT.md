# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Đào Quang Thái Anh |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602987 |
| GitHub | masao1112 |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split_seed: 9767 / model_seed: 2536 (các seed mô hình: 2536, 2537, 2538, 2539, 2540) |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 cho các bước cơ sở (2536..2540), 3 cho bước sửa chữa DAgger (2536..2538) |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.243 m | 20.0 % (CI95: [18.6 %, 21.5 %]) | BLOCK |
| IDM | 1.535 m | 0.1 % (CI95: [0.1 %, 0.4 %]) | RELEASE (baseline) |
| EgoMLP | 0.758 ± 0.005 m | 13.6 .. 34.9 % (worst seed: 34.9 %, CI95: [33.2 %, 36.7 %]) | BLOCK |
| PercMLP | 0.618 ± 0.002 m | 0.6 .. 1.8 % (worst seed: 1.8 %, CI95: [1.4 %, 2.4 %]) | BLOCK |
| PercMLP+ (sau khi sửa) | 0.801 ± 0.006 m | 0.3 .. 0.4 % (worst seed: 0.4 %, CI95: [0.2 %, 0.7 %]) | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Trong bước 0 kiểm tra dữ liệu (`evidence/data_audit.json`), từ 992.679 dòng thuộc 9.993 quỹ đạo thô ban đầu, tôi phát hiện 383 quỹ đạo bị trùng lặp hoàn toàn (exact duplicates) dù có mã ID cách xa nhau. Các quỹ đạo trùng này đã bị loại bỏ để còn lại 9.610 quỹ đạo sạch (trajectories_clean). Nếu không lọc bỏ, khi chia ngẫu nhiên sẽ dẫn tới hiện tượng rò rỉ dữ liệu (data leakage) nghiêm trọng giữa tập train và test. Vấn đề lớn thứ hai là cột gia tốc xe dẫn `Acc_LV` có hệ số tương quan tuyến tính với đạo hàm vận tốc d(Speed_LV)/dt xấp xỉ bằng 0 (`acc_lead_column_corr_with_dv_dt` = 0.0454), trong khi xe sau `Acc_FAV` có độ tương quan rất cao đạt 0.9291. Do `Acc_LV` hoàn toàn không đáng tin cậy, ta không dùng nó mà thay thế bằng gia tốc `a_lead` được tính thông qua việc lấy đạo hàm của chuỗi vận tốc xe dẫn đã được làm mượt (`v_lead_s`, thực hiện qua hàm `data.derive_lead`). Cuối cùng, việc chia train/test bắt buộc phải thực hiện theo từng quỹ đạo (trajectory) thay vì theo từng khung hình (frame), bởi vì các khung hình liên tiếp trong cùng một quỹ đạo có tính tự tương quan chuỗi thời gian (temporal correlation) cực kỳ mạnh. Nếu chia theo khung hình, mô hình chỉ việc nội suy hoặc học vẹt các khung hình lân cận đã thấy trong tập huấn luyện, làm mất đi khả năng đánh giá độ tổng quát hóa trên dữ liệu thực tế chưa từng gặp.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Từ kết quả bước phát lại log (`evidence/replay.json` với 600 đoạn và 59.576 khung hình): Bản refactor float32 (thay đổi thứ tự các phép tính và dùng float32) có đúng 600/600 đoạn không giống từng bit (not_bit_identical = 600), nhưng có 0/600 đoạn vượt dung sai cho phép (differ_beyond_tol = 0), với độ lệch cực đại `max_abs_diff` chỉ khoảng 1.30e-4 m/s² (nhỏ hơn nhiều so với atol = 0.001, rtol = 0.001). Điều này giải thích tại sao so sánh bằng bit là bài kiểm tra sai lầm khi làm việc với số thực dấu phẩy động (do phép cộng float không có tính kết hợp: `(0.1+0.2)+0.3 = 0.6000000000000001` khác `0.1+(0.2+0.3) = 0.6`). Khi thêm luật phanh khẩn cấp AEB (`aeb_ttc = 1.5`), có 6/600 đoạn bị lệch (các đoạn: 76, 141, 187, 301, 331, 442) với độ lệch gia tốc cực đại đạt 1.84 m/s². Mở đoạn 76 hoặc 141 ra kiểm tra, ta thấy độ lệch xuất hiện chính xác tại các khung hình mà xe sau tiếp cận xe dẫn với vận tốc tiến gần lớn (closing > 0) và khoảng cách hẹp, khiến thời gian va chạm TTC = gap / closing sụt xuống dưới ngưỡng 1.5 s. Tại đây, luật mới can thiệp phanh khẩn cấp ở mức -6.0 m/s² thay vì phanh êm theo IDM gốc. Chế độ phát nhanh nhất ("fastest") thực thi ngay khi có thể mà không quan tâm đến chu kỳ cảm biến, che giấu hoàn toàn áp lực thời gian thực nên báo cáo 0 khung hình trễ và 0 khung hình rớt (0 late, 0 dropped). Ngược lại, phát theo thời gian thực ("realtime") mô phỏng sát thực tế với độ trễ xử lý (base 6 ms, 2% đột biến spike 140 ms vượt deadline 100 ms), làm lộ ra 581 khung hình bị trễ (late) và 578 khung hình bị rớt (dropped), khiến 369/600 đoạn bị thay đổi lệnh điều khiển do phải giữ lệnh cũ (held command).

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Theo `evidence/scenarios.json`, lưới kịch bản cắt vào được thiết lập với tốc độ hành trình v_ego = 10.0 m/s, dải khoảng cách từ 2.76 m đến 59.31 m (60 mức gap) và dải vận tốc tiếp cận từ 0.0 đến 10.0 m/s (45 mức dv), tạo thành tổng cộng 2.700 ô kiểm thử. Ở độ trễ phản ứng t_react = 0.0 s, có 0.96 % (khoảng 1.0 %, tức 26/2.700 ô) là vùng va chạm vật lý không thể tránh khỏi (unavoidable) ngay cả khi phanh hết cỡ ở b_max = 9.0 m/s². Khi độ trễ phản ứng tăng lên t_react = 0.6 s, tỷ lệ ô bất khả thi tăng vọt lên 5.15 % (khoảng 5.1 %, tức 139/2.700 ô), do xe bị mất một quãng đường trượt tự do `dv * t_react` trước khi kích hoạt phanh. Bộ điều khiển IDM bám sát hoàn hảo biên giới hạn vật lý này: ở cả hai độ trễ 0.0 s và 0.6 s, số ô lệch khỏi biên vật lý đều bằng 0 (mismatching cells away from the frontier = 0), diện tích va chạm của IDM tăng đúng từ 1.0 % lên 5.1 %. Dữ liệu log thực tế KHÔNG phủ được vùng khó: mức phanh gấp nhất mà người lái thực tế từng đạp (phân vị 99.9%) chỉ đạt 3.62 m/s² (`human_brake_limit`), trong khi cơ cấu chấp hành có thể phanh tới 9.0 m/s². Trên lưới kịch bản, có tới 5.22 % (khoảng 5 %) số ô đòi hỏi gia tốc phanh lớn hơn 3.62 m/s² để không đâm. Toàn bộ 614 sự kiện xe dẫn phanh khai thác được từ log (chiếm 6.39 % tổng số quỹ đạo) chỉ có trung vị gia tốc giảm tốc p50 là -2.53 m/s² và p99 là -4.21 m/s², nằm hoàn toàn ở vùng an toàn và bỏ trống các ca nguy hiểm.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng đối chiếu thứ hạng giữa đánh giá vòng hở (L2 position error) và vòng kín (Closed-loop collision rate trên 2.883 quỹ đạo test) từ `evidence/openloop.json` và `evidence/closedloop.json`:
- PercMLP: L2 vòng hở 0.618 ± 0.002 m (Hạng 1) | Va chạm vòng kín 0.9 % (worst seed: 1.8 %) (Hạng 2)
- EgoMLP: L2 vòng hở 0.758 ± 0.005 m (Hạng 2) | Va chạm vòng kín 24.2 % (worst seed: 34.9 %) (Hạng 4)
- const-vel: L2 vòng hở 1.243 ± 0.000 m (Hạng 3) | Va chạm vòng kín 20.0 % (Hạng 3)
- IDM: L2 vòng hở 1.535 ± 0.000 m (Hạng 4) | Va chạm vòng kín 0.1 % (Hạng 1)

Có tới 4 cặp bộ điều khiển bị đảo ngược thứ tự xếp hạng (order flips) giữa vòng hở và vòng kín: (`const-vel`, `IDM`), (`const-vel`, `EgoMLP`), (`IDM`, `EgoMLP`), (`IDM`, `PercMLP`). IDM có L2 vòng hở tệ nhất (1.535 m, hạng 4) vì nó là luật vi phân quy tắc chứ không tối ưu để bắt chước từng bước lái xe cụ thể của người trong log, nhưng lại đạt tỷ lệ va chạm vòng kín thấp nhất (0.1%, hạng 1). Đó là vì trong vòng kín, IDM liên tục nhận phản hồi khoảng cách tức thời để điều chỉnh gia tốc, tự sửa sai sau mỗi chu kỳ điều khiển. Ngược lại, EgoMLP có L2 vòng hở (0.758 m) vượt trội so với constant-velocity (1.243 m) nhờ học được phân bố vận tốc trung bình của tập dữ liệu. Tuy nhiên khi vào vòng kín, EgoMLP đâm tới 24.2% (trên lát cắt xe dẫn phanh đâm tới 73.1%, worst seed đâm 34.9%), tệ hơn cả constant-velocity (20.0%). Nguyên nhân là EgoMLP chỉ nhìn thấy lịch sử vận tốc của chính nó mà không hề nhận tín hiệu cảm biến về khoảng cách (`gap`) hay vận tốc xe dẫn (`v_lead`). Trong vòng kín, các sai số tích lũy nhỏ đưa xe vào trạng thái lạ (distribution drift) mà mô hình không biết cách phanh để tránh xe phía trước.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Từ `evidence/closedloop.json`, kết quả của EgoMLP qua 5 seed huấn luyện ngẫu nhiên dao động rất lớn: tỷ lệ va chạm trải dài từ 13.6 % đến 34.9 % (chênh lệch tới 21.3 điểm phần trăm giữa seed tốt nhất và seed xấu nhất). Cụ thể, 5 seed có tỷ lệ va chạm lần lượt là: 34.9 %, 24.2 %, 13.6 %, 16.5 %, 31.9 %. Khoảng tin cậy Wilson 95% trên 2.883 quỹ đạo kiểm thử đối với seed xấu nhất (seed 2536) là [33.2 %, 36.7 %] (tương ứng k = 1005 ca va chạm), hoàn toàn nằm tách biệt và không hề giao thoa với khoảng tin cậy của seed tốt nhất (seed 2538 có k = 393 ca, rate 13.6%, CI95 khoảng [12.4 %, 15.0 %]). Nếu một kỹ sư chỉ chạy đúng một lần duy nhất với một seed may mắn (ví dụ seed 2538 cho ra 13.6%), họ có thể lạc quan báo cáo rằng EgoMLP hoạt động khá tốt và vượt trội so với baseline vận tốc không đổi (20.0%). Nhưng nếu hệ thống được triển khai ra thực tế, mô hình hoàn toàn có thể rơi vào hành vi của seed 2536 gây va chạm tới 34.9% và trên lát cắt xe dẫn phanh đâm tới 73.1%. Đánh giá trên một seed duy nhất là cực kỳ nguy hiểm vì nó che giấu sự bất ổn định ngẫu nhiên trong khởi tạo trọng số và thứ tự xáo trộn dữ liệu của mạng nơ-ron.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng đối chiếu tỷ lệ va chạm tổng hợp (ANY collision = va chạm trước hoặc sau) và va chạm phía sau trên lưới kịch bản ở độ trễ phản ứng t_react = 0.6 s từ `evidence/agents.json`:
- const-vel: Xe sau phát lại 73.4 % | Xe sau phản ứng 73.4 % (Va chạm sau: 0.0 % / 0.0 %)
- IDM: Xe sau phát lại 70.1 % | Xe sau phản ứng 5.1 % (Va chạm sau: 64.9 % / 0.0 %)
- EgoMLP: Xe sau phát lại 71.8 % | Xe sau phản ứng 71.8 % (Va chạm sau: 0.0 % / 0.0 %)
- PercMLP: Xe sau phát lại 79.0 % | Xe sau phản ứng 17.3 % (Va chạm sau: 62.0 % / 0.0 %)
- PercMLP+: Xe sau phát lại 69.6 % | Xe sau phản ứng 5.3 % (Va chạm sau: 64.3 % / 0.0 %)

Tham số thời gian khoảng cách hiệu chỉnh tối ưu cho xe sau IDM là T = 1.4 s, đạt khoảng cách Wasserstein 1-D là W1 = 0.3545 so với phân bố khoảng cách của người lái thực tế trong log (người lái có trung vị thời gian giãn cách là 1.86 s). Con số này chân thực hơn rất nhiều so với một xe sau không bao giờ phản ứng (never-reacting agent có W1 = 0.8643). Khi xe sau chỉ phát lại (replayed rear car), IDM bị tính tới 64.9 % va chạm phía sau ("va chạm giả"). Nguyên nhân là khi có xe cắt vào đầu, IDM phản ứng phanh an toàn để tránh va chạm phía trước; tuy nhiên xe phía sau do phát lại mù quáng từ log quá khứ nên không hề biết xe thử nghiệm vừa phanh, dẫn tới việc giữ nguyên tốc độ và tông vào đuôi xe IDM. Điều này làm đảo ngược hoàn toàn kết luận: trong môi trường phát lại, IDM trông rất tệ với 70.1% va chạm (ngang ngửa const-vel 73.4%). Nhưng trong môi trường phản ứng chân thực (reactive world), va chạm của IDM giảm mạnh từ 70.1% xuống chỉ còn 5.1% (toàn bộ 64.9% va chạm đuôi biến mất hoàn toàn). Điều này khẳng định IDM là bộ điều khiển an toàn, và môi trường kiểm thử phải có tính tương tác phản ứng để không phạt oan các bộ điều khiển biết phanh an toàn.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

Từ `evidence/gate.json`, PercMLP bị chặn bởi 5 quy tắc:
1. R2_closed_loop_logs: Tỷ lệ va chạm worst seed là 1.8 %, cận trên 95% là 2.4 %, vượt ngưỡng baseline IDM (0.1 % + 1.0 % = 1.1 %).
2. R3_braking_slice: Trên lát cắt xe dẫn phanh, PercMLP va chạm 8.0 % (cận trên 13.0 %), vượt baseline IDM (1.1 % + 5.0 % = 6.1 %).
3. R4_scenario_grid: Diện tích va chạm có thể tránh được trên lưới cắt vào là 12.1 %, vượt baseline IDM (0.0 % + 2.0 % = 2.0 %).
4. R5_seed_stability: Tỷ lệ va chạm dao động từ 0.6 % đến 1.8 % qua 5 seed (chênh lệch 1.2 điểm phần trăm > ngưỡng 1.0 điểm phần trăm).
5. R6_reactive_world: Va chạm trong thế giới phản ứng là 17.3 %, vượt baseline IDM (5.1 % + 2.0 % = 7.1 %).

Nguyên nhân gốc gồm hai yếu tố: (a) Không gian trạng thái chưa từng thấy (OOD): dữ liệu log Argoverse 2 chỉ chứa các tình huống bám xe êm ái, hoàn toàn thiếu các ca cắt vào đột ngột với khoảng cách nhỏ và vận tốc tiếp cận cao; (b) Sai số tích lũy (covariate shift): sai số dự báo nhỏ của MLP khi tự lái làm xe chệch khỏi quỹ đạo huấn luyện, mô hình không biết cách phục hồi. Phương pháp sửa kết hợp hai kỹ thuật: (1) Bổ sung 197.180 cửa sổ từ 20.000 kịch bản cắt vào mô phỏng gán nhãn bởi IDM để bao phủ vùng OOD; (2) Thuật toán DAgger qua 2 vòng (hàm `dagger_windows`) cho mô hình tự lái trên tập huấn luyện, thu thập chính các trạng thái trôi dạt nó tự đi vào và gắn nhãn bằng dự báo của chuyên gia IDM để học cách sửa sai. Nhờ đó, PercMLP+ giảm tỷ lệ va chạm trên log xuống 0.3..0.4% (upper 0.7% <= 1.1%), trên lát cắt phanh xuống 1.7% (upper 4.9% <= 6.1%), diện tích đâm trên lưới cắt vào giảm từ 12.1% xuống 0.1%, và vượt qua toàn bộ 7 quy tắc của cổng phát hành (RELEASE). Tuy nhiên, PercMLP+ KHÔNG tốt hơn IDM: bản chất nó chỉ là một học trò bắt chước luật IDM (thông qua nhãn chuyên gia). Điểm L2 vòng hở của nó thậm chí tăng từ 0.618 m lên 0.801 m do phải học thêm các kịch bản khó. Ta chưa nên phát hành PercMLP+ ngay để thay thế IDM vì mô hình học máy tiêu tốn tài nguyên tính toán hơn, kém minh bạch hơn luật toán học, và khi thử nghiệm ở vận tốc hành trình ngoài miền huấn luyện 20 m/s vẫn có tỷ lệ đâm 0.1..0.7% (trong khi IDM giữ vững 0.0%). Đề xuất: cần kiểm thử thêm trong mô phỏng đa làn có cảm biến nhiễu, thử nghiệm ngầm (shadow mode) trên xe thật và duy trì bộ giám sát an toàn (safety monitor / fallback) bằng IDM trước khi đưa vào sản xuất.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Ba giới hạn quan trọng nhất của bài lab đối với quyết định phát hành hệ thống tự lái trong thực tế:
1. Mô hình động học 1-D thuần túy, bỏ qua hoàn toàn chuyển động ngang và trễ cơ cấu chấp hành: Bài lab giả định xe di chuyển trên một đường thẳng, không có góc lái, trượt bánh, điều kiện mặt đường (mưa, băng tuyết) hay độ trễ thủy lực phanh thực tế (actuator latency). Trong thực tế, xe có thể đánh lái tránh chướng ngại vật thay vì chỉ phanh dọc, và giới hạn gia tốc phanh không thể đạt tức thời 9.0 m/s².
2. Giả định cảm biến hoàn hảo, không có nhận thức từ dữ liệu thô (sensor-less perception): Đầu vào của mô hình là các biến trạng thái hoàn hảo (`gap`, `v_lead`, `v_ego`). Trong xe tự lái thật, các giá trị này được ước lượng từ camera, LiDAR, radar với nhiễu đo đạc, độ trễ nhận diện vật thể, hiện tượng che khuất (occlusion), và dương tính/âm tính giả (false positive/negative). Một sai lệch nhỏ trong nhận diện khoảng cách ở khoảng cách 10 m có thể biến một cú phanh êm thành một vụ va chạm thảm khốc.
3. Tương tác đa tác nhân và hành vi con người phức tạp bị đơn giản hóa: Xe dẫn trong log hoàn toàn là phát lại tĩnh, không hề phản ứng với xe thử nghiệm (open-loop lead vehicle). Trong giao thông thật, hành vi là một quá trình đàm phán hai chiều (game-theoretic interaction): xe dẫn có thể quan sát gương chiếu hậu, nhường đường, hoặc phanh gấp khi thấy xe sau bám quá sát.

Các bước tiếp theo cần thực hiện để thu hẹp khoảng cách đến thế giới thực:
- Mở rộng mô phỏng sang 2-D/3-D hoàn chỉnh với mô hình động lực học xe đầy đủ (full vehicle dynamics) và mô phỏng cảm biến vật lý (CARLA, nuPlan).
- Đánh giá khả năng chống chịu nhiễu (noise robustness) với mô hình mất dấu đối tượng (sensor dropout) và trễ truyền thông CAN.
- Triển khai chạy chế độ ngầm (shadow mode) trên xe thử nghiệm thực tế trên đường thật qua hàng triệu km để so sánh quyết định của mạng nơ-ron với tài xế an toàn con người trước khi cấp phép can thiệp vào vô lăng và bàn đạp phanh.
