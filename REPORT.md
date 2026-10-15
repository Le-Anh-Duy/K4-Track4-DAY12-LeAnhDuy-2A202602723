# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Lê Anh Duy |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602723 |
| GitHub | Le-Anh-Duy |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 4367 / 1138 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.257 | 21.5 | BLOCK |
| IDM | 1.526 | 0.1 | (baseline) |
| EgoMLP | 0.756 .. 0.767 | 14.3 .. 31.4 | BLOCK |
| PercMLP | 0.621 .. 0.625 | 0.5 .. 1.2 | BLOCK |
| PercMLP+ (sau khi sửa) | 0.804 .. 0.835 | 0.2 .. 0.3 | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Ở bước 0, quá trình kiểm tra phát hiện 3 vấn đề lớn trong tập dữ liệu Argoverse 2 (tổng cộng 992679 dòng thuộc 9993 quỹ đạo thô):
1. Cột gia tốc xe dẫn `Acc_LV` hoàn toàn không đáng tin cậy với hệ số tương quan tuyến tính so với đạo hàm vận tốc d(Speed_LV)/dt chỉ đạt xấp xỉ 0.0454 (gần như bằng 0), trong khi xe sau `Acc_FAV` có tương quan lên tới 0.9291. Do đó không thể dùng trực tiếp `Acc_LV` mà phải thay thế bằng gia tốc dẫn xuất `a_lead` được tính từ đạo hàm của chuỗi vận tốc đã làm mượt `v_lead_s`.
2. Có đúng 383 quỹ đạo bị trùng lặp hoàn toàn (giữ lại 9610 quỹ đạo sạch). Các quỹ đạo trùng nhau có ID cách xa nhau (do ghép log), nếu không loại bỏ sẽ dẫn đến rò rỉ dữ liệu nghiêm trọng giữa tập train và test.
3. Việc phân chia tập huấn luyện và kiểm thử bắt buộc phải chia theo quỹ đạo (trajectory) thay vì chia ngẫu nhiên theo khung hình (frame), bởi vì các khung hình trong cùng một quỹ đạo có tính tự tương quan thời gian cực cao; chia theo frame sẽ làm mô hình chỉ ghi nhớ thay vì khái quát hóa, tạo ra đánh giá sai lệch.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Khi kiểm thử phát lại trên 600 đoạn log (9.5 MB):
1. Bản tái cấu trúc mã nguồn `refactor_float32` bị khác bit ở 600/600 đoạn (100%), nhưng có 0/600 đoạn vượt dung sai cho phép (sai số tuyệt đối cực đại chỉ 2.05e-05 m/s²). Cần dung sai vì phép cộng số thực dấu phẩy động 32-bit không có tính kết hợp, ví dụ thực nghiệm cho thấy (0.1+0.2)+0.3 = 0.6000000000000001 trong khi 0.1+(0.2+0.3) = 0.6, và việc đảo thứ tự tính tổng 50000 phần tử gây lệch tương đối 6.94e-06. Do đó dung sai atol=1e-3, rtol=1e-3 là bắt buộc để tránh báo động giả.
2. Thay đổi thuật toán AEB làm lệch đúng 7/600 đoạn (các đoạn [65, 72, 142, 233, 234, 473, 548]). Khi soi chi tiết đoạn 142 hoặc 65, sự sai khác chỉ xuất hiện ở các khung hình có tình huống nguy hiểm khi khoảng cách gap co hẹp và vận tốc tiếp cận cao khiến TTC giảm xuống dưới 1.5 s. Khi đó luật AEB kích hoạt phanh khẩn cấp cưỡng bức cmd <= -6.0 m/s², tạo ra độ lệch gia tốc tối đa 1.556 m/s² so với lệnh IDM thông thường. Các khung hình còn lại có TTC an toàn thì AEB giữ nguyên lệnh nên không có sai khác.
3. Chế độ phát nhanh nhất (`fastest`) giả định xử lý tức thời không tốn thời gian nên che giấu toàn bộ vấn đề hạn chót (0 frame trễ, 0 frame rớt). Ngược lại, chế độ thời gian thực (`realtime`) với chu kỳ 100 ms và mô phỏng chi phí tính toán có spike (2% xác suất mất 140 ms > 100 ms) đã làm lộ ra 584 khung hình bị trễ (late) và 571 khung hình bị bỏ rơi (dropped) trên 59884 khung hình, khiến 369/600 đoạn bị biến đổi lệnh điều khiển do phải giữ lệnh cũ (held command).

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới kịch bản cắt vào được thiết lập với tốc độ cruise v_ego = 12.0 m/s, dải khoảng cách từ 2.76 m đến 59.31 m (60 mốc) và chênh lệch tốc độ dv từ 0.0 đến 12.0 m/s (45 mốc), tạo thành lưới 2700 ô:
1. Ở độ trễ phản ứng t_react = 0.0 s, có 2.22% số ô (60 ô) là va chạm bất khả kháng theo quy luật vật lý (unavoidable). Khi độ trễ phản ứng tăng lên t_react = 0.6 s, vùng va chạm bất khả kháng tăng vọt lên 7.44% (201 ô), tức tăng thêm 5.22 điểm % (gấp 3.35 lần diện tích ban đầu).
2. Tỷ lệ va chạm của IDM chuyển từ 2.22% (ở 0.0 s) lên đúng 7.44% (ở 0.6 s). Diện tích va chạm do lỗi bộ điều khiển (attributable area) của IDM bằng đúng 0.0% ở cả 2 độ trễ, nghĩa là IDM chỉ va chạm ở những ô mà vật lý không thể tránh khỏi.
3. Tập log thực tế hoàn toàn không phủ được các vùng tình huống khắc nghiệt: người lái trong log phanh mạnh nhất (phân vị 99.9%) chỉ đạt 3.62 m/s² (trung vị phanh gắt là -2.53 m/s² và p99 là -4.21 m/s²), trong khi cơ cấu chấp hành có thể phanh tới 9.0 m/s². Có tới 8.56% số ô trên lưới kịch bản đòi hỏi gia tốc phanh lớn hơn mức 3.62 m/s² của con người. 614 sự kiện phanh khai thác được từ log (chiếm 6.39% quỹ đạo) chỉ tập trung ở vùng khoảng cách an toàn và chênh lệch tốc độ nhỏ, bỏ trống hoàn toàn các góc nguy hiểm.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh giữa đánh giá vòng hở (L2 sai số quỹ đạo 3s) và vòng kín (tỷ lệ va chạm trên log):
| Bộ điều khiển | L2 vòng hở (m) | Thứ hạng L2 | Va chạm vòng kín (%) | Thứ hạng vòng kín |
|---|---|---|---|---|
| constant-velocity | 1.257 m | 3 | 21.5 % | 4 |
| IDM | 1.526 m | 4 | 0.1 % (0.07%) | 1 |
| EgoMLP | 0.760 m | 2 | 20.4 % (14.3..31.4%) | 3 |
| PercMLP | 0.623 m | 1 | 0.8 % (0.5..1.2%) | 2 |

Thực nghiệm ghi nhận 3 cặp đảo chiều thứ hạng hoàn toàn: ('const-vel', 'IDM'), ('IDM', 'EgoMLP'), và ('IDM', 'PercMLP').
- IDM có điểm L2 kém nhất (1.526 m, hạng 4) vì IDM là mô hình quy tắc phản hồi động học, nó kiểm soát khoảng cách an toàn chứ không tối ưu hóa hàm mất mát để bắt chước từng sai số vận tốc nhỏ của con người trong log. Tuy nhiên trong vòng kín, IDM luôn chủ động giữ khoảng cách và phanh đúng lúc, đạt tỷ lệ va chạm chỉ 0.07% (gần như an toàn tuyệt đối).
- Ngược lại, EgoMLP có L2 rất ấn tượng (0.760 m, vượt trội so với const-vel 1.257 m) nhưng lại đâm tới 20.4% trên toàn bộ log và đâm 67.1% ở lát cắt xe dẫn phanh. Nguyên nhân là EgoMLP bị "mù" nhận thức (hoàn toàn không được cung cấp khoảng cách gap và tốc độ xe dẫn v_lead). Trong vòng hở xe đi theo quỹ đạo có sẵn của con người nên EgoMLP dự đoán vận tốc có vẻ mượt mà. Nhưng khi lái vòng kín, các sai số tích lũy dần (covariate shift), đẩy xe vào trạng thái lạ; khi xe trước phanh gấp, EgoMLP không hề nhận biết được nên tiếp tục lao về phía trước gây tai nạn.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Khi huấn luyện với 5 seed ngẫu nhiên (1138, 1139, 1140, 1141, 1142):
1. Tỷ lệ va chạm của EgoMLP dao động cực kỳ lớn từ 14.33% (seed 1140, 413 va chạm/2883 log) đến 31.43% (seed 1139, 906 va chạm/2883 log), độ phân tán trải rộng tới 17.10 điểm %!
2. Khoảng tin cậy Wilson 95% thể hiện rõ sự bất định này:
   - Seed tốt nhất (1140): [13.09%, 15.65%]
   - Seed xấu nhất (1139): [29.76%, 33.14%]
   Hai khoảng tin cậy này cách xa nhau và hoàn toàn không có vùng giao thoa (chồng lấn).
3. Nếu kỹ sư chỉ chạy đúng một lần (1 seed duy nhất):
   - Nếu tình cờ chạy trúng seed 1140, người đó có thể kết luận lạc quan sai lầm rằng tỷ lệ va chạm chỉ ~14.3% và vội vã đánh giá mô hình đủ điều kiện.
   - Nếu trúng seed 1139, kết quả lại là thảm họa với hơn 31.4% số ca bị đâm.
   Điều này chứng minh rằng việc đánh giá trên một lần chạy duy nhất là vô cùng nguy hiểm trong kỹ thuật xe tự hành, vì nó che giấu bản chất phi lồi (non-convex) và sự mất ổn định của mạng nơ-ron sâu. Báo cáo bắt buộc phải nêu rõ dải phân tán qua nhiều seed kèm khoảng tin cậy thống kê để tránh đưa ra quyết định sai lầm.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Bảng so sánh tỷ lệ va chạm trên lưới kịch bản cắt vào khi t_react = 0.6 s:
| Bộ điều khiển | Xe sau phát lại (%) | Xe sau phản ứng (%) | Tỷ lệ va chạm đuôi (Phát lại / Phản ứng) |
|---|---|---|---|
| const-vel | 77.6 % | 77.6 % | 0.0 % / 0.0 % |
| IDM | 75.7 % | 7.4 % | 68.3 % / 0.0 % |
| EgoMLP | 83.6 % | 83.6 % | 0.0 % / 0.0 % |
| PercMLP | 82.5 % | 23.9 % | 59.1 % / 0.0 % |
| PercMLP+ | 75.5 % | 7.6 % | 67.9 % / 0.0 % |

1. Hiệu chỉnh tham số khoảng cách thời gian: giá trị tối ưu tìm được là T = 1.4 s với khoảng cách Wasserstein W1 = 0.346 (gần nhất với phân bố headway trung vị 1.85 s của con người trong log). Xe sau phản ứng với T = 1.4 s chân thực hơn rất nhiều so với tác nhân không bao giờ phản ứng (có W1 lên tới 0.877).
2. Va chạm giả phía sau: Trong môi trường xe sau phát lại (replay), xe phía sau di chuyển một cách thụ động theo dữ liệu lịch sử mà không biết rằng xe thử nghiệm (ego) ở phía trước vừa phải phanh gấp để xử lý xe tạt đầu. Hậu quả là xe sau đâm thẳng vào đuôi xe thử nghiệm, khiến IDM bị ghi nhận tới 68.3% va chạm từ phía sau (tổng va chạm 75.7%).
3. Khi thay bằng xe sau có phản ứng (reactive IDM), va chạm đuôi của IDM lập tức giảm triệt để về 0.0%, và tổng va chạm giảm xuống đúng 7.4% (toàn bộ là va chạm trước bất khả kháng do vật lý). Điều này đảo lộn hoàn toàn kết luận: nếu chỉ dùng xe sau phát lại, người kiểm thử sẽ kết luận sai lầm rằng IDM là bộ điều khiển nguy hiểm (75.7% va chạm, gần bằng const-vel 77.6%), trong khi thực tế đó là lỗi của môi trường mô phỏng không tương tác chứ không phải lỗi của IDM.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

1. PercMLP bị chặn (BLOCK) bởi 4 quy tắc an toàn cốt lõi:
   - R2_closed_loop_logs: Tỷ lệ va chạm trên log là 1.2% (cận trên 95% là 1.7%), vượt quá ngưỡng baseline 0.1% + 1.0% = 1.1%.
   - R3_braking_slice: Trên lát cắt phanh gấp (n=170), tỷ lệ va chạm là 4.1% với cận trên 8.3%, vượt quá baseline 0.0% + 5.0% = 5.0%.
   - R4_scenario_grid: Diện tích va chạm tránh được trên lưới kịch bản cắt vào lên tới 16.4%, vượt xa ngưỡng baseline 0.0% + 2.0% = 2.0%.
   - R6_reactive_world: Trong thế giới phản ứng, diện tích va chạm là 23.9%, vượt quá baseline 7.4% + 2.0% = 9.4%.
   Nguyên nhân gốc gồm hai yếu tố:
   (a) Dữ liệu huấn luyện thiếu kịch bản cắt vào khắc nghiệt (log Argoverse chỉ gồm xe bám đuôi êm dịu, phanh tối đa 3.62 m/s²).
   (b) Sai số tích lũy khi tự lái trong vòng kín (covariate shift) đưa mô hình vào các trạng thái lạ mà nó không biết cách phục hồi.
2. Giải pháp khắc phục:
   - Bổ sung 196960 cửa sổ cắt vào nhân tạo mô phỏng từ 20000 kịch bản cắt vào đa dạng khoảng cách và tốc độ, gắn nhãn bằng phản hồi chuẩn của IDM.
   - Áp dụng thuật toán DAgger qua 2 vòng: cho chính mô hình tự lái trên log, thu thập 895000 cửa sổ tại các trạng thái nó tự viếng thăm và gắn nhãn bằng dự báo của chuyên gia IDM.
   Giải pháp này chữa đúng bệnh vì nó vừa lấp đầy khoảng trống dữ liệu kịch bản biên, vừa dạy mạng nơ-ron cách tự sửa sai khi bị chệch quỹ đạo.
3. Kết quả PercMLP+: Đạt quyết định RELEASE khi vượt qua toàn bộ 7 quy tắc: va chạm trên log giảm còn 0.2..0.3% (cận trên 0.5% <= 1.1%), va chạm trên lưới cắt vào giảm ngoạn mục từ 16.4% xuống 0.2%, va chạm lát cắt phanh về 0.0% (cận trên 2.2% <= 5.0%).
4. PercMLP+ có tốt hơn IDM không? KHÔNG. PercMLP+ học hoàn toàn từ chuyên gia IDM nên trần chất lượng của nó bị chặn bởi IDM (trên lưới cắt vào IDM có 0.0% va chạm tránh được, PercMLP+ có 0.2%). Sai số L2 vòng hở của PercMLP+ thậm chí tăng từ 0.623 m lên 0.820 m do phải đánh đổi độ khớp trên log thông thường để học các ca biên.
5. Khuyến nghị phát hành: KHÔNG nên vội vàng phát hành thương mại ngay. Cần thực hiện các bước kiểm tra tiếp theo gồm: thử nghiệm trên mô phỏng không gian 2D/3D đầy đủ (đánh lái, chuyển làn), kiểm thử với nhiều loại nhiễu cảm biến động và trễ mạng, chạy chế độ quan sát ngầm (shadow mode) trên xe thực tế trước khi cấp phép can thiệp vào hệ thống lái.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Để đưa ra quyết định phát hành một hệ thống tự hành ra đường thật, bài lab này tồn tại 3 giới hạn mang tính quyết định:
1. Giới hạn chuyển động một chiều dọc (1-D longitudinal assumption): Hệ thống giả định mọi chuyển động chỉ diễn ra trên một đường thẳng dọc, bỏ qua hoàn toàn chuyển động ngang (lateral motion). Trong thực tế, khi gặp nguy hiểm xe có thể đánh lái né tránh (evasive steering) hoặc chuyển làn; ngược lại, việc đánh lái gấp có thể gây mất ổn định thân xe hoặc trượt bánh. Đánh giá thuần 1-D không thể phản ánh đầy đủ năng lực an toàn thực tế.
2. Giả định cảm biến và môi trường quá lý tưởng (Idealized perception and physics): Bài lab giả định khoảng cách và tốc độ đo được tức thời (chỉ có nhiễu Gaussian nhẹ 1 m và 0.5 m/s), không tính đến các hiện tượng thực tế như vật cản che khuất tầm nhìn (occlusion), thời tiết xấu (mưa, sương mù làm mờ camera/LiDAR), ma sát mặt đường thay đổi (đường ướt, băng tuyết làm giảm gia tốc phanh từ 9 m/s² xuống 2-3 m/s²).
3. Xe dẫn trong log không phản ứng (Non-interactive lead vehicle): Xe dẫn trong log Argoverse là một đoạn băng phát lại cố định, không hề phản ứng với xe thử nghiệm. Nếu xe thử nghiệm bấm còi, nháy đèn hoặc tiến quá gần, xe dẫn thật có thể tăng tốc hoặc chuyển làn nhường đường. Mô phỏng tĩnh này biến thế giới thành các tác nhân vô cảm, làm sai lệch phân bố tương tác giao thông.
4. Bước đi tiếp theo để thu hẹp khoảng cách: Cần tích hợp bộ điều khiển vào các nền tảng mô phỏng 3D chân thực như CARLA, CARSim hoặc NuPlan; kiểm thử phần cứng trong vòng lặp (Hardware-in-the-Loop - HIL) để đo độ trễ bus CAN thực tế; thử nghiệm trên đường thử kín (closed proving ground); và cuối cùng là chạy thử nghiệm song song không can thiệp (Shadow Mode) trên hàng triệu km thực tế để thu thập bằng chứng an toàn trước khi kích hoạt chính thức.
