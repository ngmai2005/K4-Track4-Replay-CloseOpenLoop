# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Hồ Ngọc Mai |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `02509` |
| GitHub | `ngmai2005` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 8380 / 6583, 6584, 6585 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.257 | 19.7 | Chặn |
| IDM | 1.533 | 0.1 | baseline |
| EgoMLP | 0.764 ± 0.001 | 19.8–23.9 | Chặn |
| PercMLP | 0.620 ± 0.002 | 0.7–1.1 | Chặn |
| PercMLP+ (sau khi sửa) | 0.808 ± 0.017 | 0.2–0.3 | Phát hành |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Dữ liệu thô có 992.679 hàng và 9.993 quỹ đạo, nhưng 383 quỹ đạo là bản sao chính xác nên đã được loại, còn 9.610 quỹ đạo sạch. Cột `Acc_LV` không đáng tin vì tương quan với đạo hàm tốc độ xe dẫn chỉ là 0,045; trong khi tương quan tương ứng của xe theo sau là 0,929. Vì vậy tôi suy ra gia tốc xe dẫn từ `Speed_LV` đã làm mượt rồi lấy sai phân trung tâm, thay vì dùng `Acc_LV`.

Chia theo từng quỹ đạo tránh việc các khung gần như giống nhau của cùng một tình huống xuất hiện đồng thời ở train và test. Nếu chia theo frame, mô hình có thể nhìn thấy phần lớn diễn biến của một quỹ đạo ở train rồi được chấm trên phần còn lại, làm L2 và va chạm đánh giá quá lạc quan. Việc loại trùng trước khi chia cũng ngăn rò rỉ một quỹ đạo nguyên vẹn qua hai tập.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Phát lại đúng cùng mã trên 600 đoạn cho 0 đoạn khác biệt và 0 đoạn khác bit. Bản refactor float32 thay đổi thứ tự tính toán nên cả 600 đoạn không đồng nhất từng bit, nhưng sai khác lớn nhất chỉ 1,81e-05 m/s² và không đoạn nào vượt dung sai `atol=rtol=1e-3`. Vì số thực dấu chấm động không kết hợp được, kiểm tra theo dung sai là đúng hơn kiểm tra bằng tuyệt đối.

Khi thêm AEB, 9/600 đoạn vượt dung sai (10 đoạn khác bit), vì luật này chỉ tác động khi TTC thấp hơn ngưỡng 1,5 s; do đó lệnh chỉ đổi tại những frame nguy hiểm. Chạy thời gian thực làm lộ 582 frame trễ và 579 frame rơi, khiến 372 đoạn đổi kết quả, còn chế độ nhanh nhất có cả hai số bằng 0 vì không mô phỏng deadline và tải xử lý.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới cho MSSV này dùng vận tốc cruise 10 m/s, gap 2,76–59,31 m và closing speed 0–10 m/s, tổng cộng 2.700 ô. Khi phản ứng ngay, 1,0% ô là bất khả thi; với trễ 0,6 s, tỷ lệ tăng lên 5,1%. FullBrake và công thức vật lý khớp hoàn toàn: 0 ô sai khác ngoài vùng biên ở cả hai độ trễ.

Từ log khai thác được 614 sự kiện xe dẫn phanh. Giới hạn phanh mạnh nhất của người lái theo percentile 99,9 là 3,62 m/s², trong khi 5,2% lưới đòi hỏi lớn hơn mức đó. Các event log chỉ phủ một vùng hẹp hơn các cảnh cut-in cực đoan; vì vậy cần tách “ai cũng đâm” khỏi vùng vẫn tránh được mà controller thất bại.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Theo L2 vòng hở, thứ hạng là PercMLP (0,620 m), EgoMLP (0,764 m), constant-velocity (1,257 m), rồi IDM (1,533 m). Theo va chạm vòng kín, thứ hạng đảo thành IDM (0,1%), PercMLP (trung bình 0,95%), constant-velocity (19,7%), và EgoMLP (trung bình 22,3%). Các cặp đảo thứ tự gồm constant-velocity/IDM, constant-velocity/EgoMLP, IDM/EgoMLP và IDM/PercMLP.

IDM có L2 kém vì nó không cố bắt chước chính xác tốc độ người lái trong log; nó ưu tiên khoảng cách và hãm an toàn nên gần như không va chạm. EgoMLP học diễn biến tốc độ nên L2 tốt hơn constant-velocity, nhưng không quan sát gap hay tốc độ xe dẫn. Khi xe dẫn giảm tốc, lỗi dự báo tích lũy trong vòng kín và nó không có tín hiệu để phản ứng kịp.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

EgoMLP có tỷ lệ va chạm theo seed là 19,8%, 23,3% và 23,9%, tức dải 4,2 điểm phần trăm. Cận trên Wilson của seed xấu nhất là 25,5% với 2.883 quỹ đạo test, nên kết luận không chỉ là nhiễu hiển thị. PercMLP ổn định hơn nhiều, 0,7–1,1%, nhưng vẫn cần so với baseline và lưới kịch bản thay vì chỉ nhìn trung bình.

Nếu chỉ chọn seed 6583 của EgoMLP, con số 19,8% sẽ che việc seed khác lên 23,9% và có thể dẫn đến quyết định sai về độ ổn định. Nhiều seed tách biến thiên do tối ưu hoá khỏi hiệu quả thực, còn Wilson diễn đạt bất định do số quỹ đạo hữu hạn. Vì vậy báo cáo dùng dải seed và cận tin cậy, không dùng một lần chạy may mắn.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Hiệu chỉnh tác nhân sau cho T = 1,4 s: Wasserstein-1 với phân bố headway người lái là 0,355, thấp hơn rõ so với xe không phản ứng là 0,852; median headway người thật là 1,88 s. Tác nhân sau phản ứng mô tả việc tài xế sau quan sát xe ego đã phanh, thay vì tiếp tục chạy theo quỹ đạo log bất biến.

Ở trễ 0,6 s, IDM có va chạm bất kỳ 70,1% trong thế giới rear replay, trong đó va chạm sau là 64,9%; khi rear reactive, va chạm sau về 0 và tổng chỉ còn 5,1%. Đây là “va chạm giả” do xe sau bị buộc không phản ứng, không phải bằng chứng IDM nguy hiểm. Kết luận về controller do đó phải dựa vào thế giới có tác nhân phản ứng.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP ban đầu bị chặn bởi R2, R3, R4 và R6. Ví dụ, cận trên va chạm log là 1,6% so với IDM 0,1% + 1 điểm; trên lưới cut-in, vùng tránh được nhưng vẫn đâm là 12,0% so với baseline 0,0% + 2 điểm. Điều này kết hợp hai nguyên nhân: log không phủ đủ cut-in mạnh và lỗi bắt chước nhỏ tích luỹ khi mô hình tự lái.

Tôi bổ sung 20.000 kịch bản cut-in gán nhãn IDM và hai vòng DAgger trên các trạng thái mô hình tự đi tới. PercMLP+ qua cả bảy quy tắc: va chạm log 0,2–0,3%, R4 chỉ 0,1%, và R6 là 5,3% so với IDM 5,1%. Tuy nhiên không thể gọi nó tốt hơn IDM: nó học nhãn IDM và chỉ mới được chứng minh không kém hơn trong phạm vi kiểm thử; trước phát hành thực tế cần thử nghiệm rộng hơn và giám sát an toàn.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Bài lab chỉ mô phỏng chuyển động một chiều; nó chưa bao quát đổi làn, giao lộ, người đi bộ, thời tiết, cảm biến hay hành vi đa tác nhân phức tạp. Ví dụ, 614 sự kiện phanh khai thác được chỉ chiếm 6,4% trong 9.610 quỹ đạo sạch, trong khi lưới vẫn có 5,2% ô đòi phanh mạnh hơn bất kỳ người lái nào trong log từng thực hiện. Vì vậy một kết quả tốt trên gap và tốc độ dọc không suy ra an toàn cho môi trường giao thông thật. Cần đánh giá đa chiều, dữ liệu đa dạng và các kịch bản có tương tác thực hơn.

Xe dẫn trong log được phát lại và không phản ứng với ego, còn tác nhân sau chỉ là một IDM hiệu chỉnh theo headway. Minh hoạ rõ nhất là IDM có 70,1% va chạm khi xe sau bị replay, nhưng chỉ còn 5,1% khi xe sau phản ứng; mô hình tác nhân này vẫn là giả định. Kết luận về va chạm có thể thay đổi khi tài xế thật phản ứng khác hoặc hệ thống có độ trễ cảm biến/điều khiển biến thiên. Cần kiểm thử với nhiều mô hình hành vi, dữ liệu can thiệp và mô phỏng xác thực hơn.

Cuối cùng, các nhãn sửa và một phần đánh giá đều dựa vào IDM, nên PercMLP+ được chứng minh bắt chước thầy tốt trong phạm vi đã chọn chứ không chứng minh vượt thầy. Kết quả ở vận tốc cruise giữ lại 20 m/s đã tốt hơn PercMLP nhưng vẫn là mô phỏng. Trước khi phát hành cần xác nhận trên tập holdout độc lập, kiểm thử biên có hệ thống và vận hành thử có người giám sát.
