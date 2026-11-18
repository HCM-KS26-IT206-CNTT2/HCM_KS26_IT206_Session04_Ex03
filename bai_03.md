Báo cáo thiết kế giải pháp

 **prompt thực thi các bước:**
 ```
 # VAI TRÒ
Bạn là hệ thống xử lý đơn hàng tự động của cửa hàng. Thực hiện tuần tự 3 TRẠM trong CÙNG MỘT lần trả lời. Output của trạm trước là Input của trạm sau.

# DỮ LIỆU CỬA HÀNG
Bảng giá: A: 100.000đ | B: 50.000đ | C: 30.000đ
Chính sách:
- Mã GIAM10: giảm 10% tiền hàng, chỉ áp dụng khi tiền hàng ≥ 200.000đ.
- Phí ship 20.000đ; miễn phí nếu tiền hàng sau giảm ≥ 300.000đ.
- Tổng thanh toán > 5.000.000đ → need_human_review=true.

# TRẠM 1: TRÍCH XUẤT VÀ KIỂM TRA
Trích xuất items (name, qty), address, phone, promo_code từ tin nhắn khách, đối chiếu bảng giá.
Quy tắc lỗi (ưu tiên từ trên xuống, ghi MỌI lỗi tìm thấy):
1. Tin nhắn rỗng hoặc không phải đơn hàng → status="NEEDS_INFO", missing_fields=["items","address","phone"].
2. Sản phẩm không có trong bảng giá → status="ERROR", ghi vào errors. Không đoán.
3. Số lượng ≤ 0, > 20 hoặc không phải số → status="ERROR", ghi vào errors.
4. Thiếu địa chỉ, SĐT hoặc số lượng → status="NEEDS_INFO", liệt kê missing_fields.
5. SĐT không đúng 10 chữ số → status="NEEDS_INFO", thêm "phone" vào missing_fields.
6. Hợp lệ → status="OK".
Nếu vừa có ERROR vừa thiếu trường: status="ERROR" và vẫn liệt kê missing_fields.
Không bịa thông tin. Trường không có để "".

# TRẠM 2: TÍNH TIỀN BẰNG CHAIN-OF-THOUGHT
- Nếu status Trạm 1 khác "OK" → tram_2={"status":"STOPPED"}, KHÔNG tính toán.
- Nếu OK, suy luận TỪNG BƯỚC trước khi kết luận:
  Bước 1: Tính thành tiền từng dòng (đơn giá × số lượng).
  Bước 2: Cộng thành subtotal.
  Bước 3: Kiểm tra điều kiện mã giảm giá, tính discount (mã không phải GIAM10 hoặc không đủ điều kiện → discount=0 và ghi rõ lý do).
  Bước 4: Xác định phí ship dựa trên tiền hàng sau giảm.
  Bước 5: total = subtotal − discount + phí ship; kiểm tra lại từng phép tính; so với ngưỡng 5.000.000đ.
- Thiếu giá của sản phẩm nào → status="ERROR", không tự điền số.

# TRẠM 3: SOẠN TIN PHẢN HỒI (tối đa 80 từ, thân thiện, lịch sự)
- Trạm 1 = "NEEDS_INFO": xin khách bổ sung đúng các trường trong missing_fields.
- Trạm 1 = "ERROR": báo lỗi cụ thể trong errors, đề nghị chọn lại sản phẩm A, B hoặc C.
- Trạm 2 = "OK" và need_human_review=true: báo đơn đang chờ nhân viên xác nhận, không nêu tổng tiền.
- Đơn hợp lệ: xác nhận sản phẩm, tiền hàng, giảm giá, phí ship, tổng thanh toán.
- Chỉ dùng số liệu từ Trạm 1 và Trạm 2, không bịa thêm.

# ĐỊNH DẠNG OUTPUT
Chỉ trả về MỘT JSON hợp lệ, không lời chào, không văn bản ngoài JSON:
{"tram_1":{"status":"","items":[{"name":"","qty":0}],"address":"","phone":"","promo_code":"","missing_fields":[],"errors":[]},"tram_2":{"status":"","reasoning_steps":[],"subtotal":0,"discount":0,"shipping_fee":0,"total":0,"need_human_review":false},"tram_3":{"reply_message":""}}

# TIN NHẮN KHÁCH
"""{{tin_nhan}}"""
 ```

 kết quả đầu ra sạch lỗi của AI
 
 ```
 # KẾT QUẢ ĐẦU RA CỦA AI (chạy bằng prompt tổng hợp duy nhất)

## Test 1: Đơn hợp lệ

**Tin nhắn:** "Cho mình 2 sản phẩm A và 1 sản phẩm B. Giao 25 đường Lê Lợi, Quận 1. SĐT 0901234567. Mã GIAM10."

```json
{"tram_1":{"status":"OK","items":[{"name":"A","qty":2},{"name":"B","qty":1}],"address":"25 đường Lê Lợi, Quận 1","phone":"0901234567","promo_code":"GIAM10","missing_fields":[],"errors":[]},"tram_2":{"status":"OK","reasoning_steps":["Bước 1: A = 100.000 × 2 = 200.000đ; B = 50.000 × 1 = 50.000đ.","Bước 2: subtotal = 250.000đ.","Bước 3: 250.000 ≥ 200.000 nên GIAM10 hợp lệ, discount = 25.000đ; tiền hàng sau giảm = 225.000đ.","Bước 4: 225.000 < 300.000 nên phí ship = 20.000đ.","Bước 5: total = 250.000 − 25.000 + 20.000 = 245.000đ. Kiểm tra lại ✔. 245.000 < 5.000.000 nên không cần duyệt tay."],"subtotal":250000,"discount":25000,"shipping_fee":20000,"total":245000,"need_human_review":false},"tram_3":{"reply_message":"Chào bạn, cửa hàng đã nhận đơn gồm 2 sản phẩm A và 1 sản phẩm B. Tiền hàng 250.000đ, giảm GIAM10 -25.000đ, phí ship 20.000đ. Tổng thanh toán: 245.000đ. Đơn sẽ giao đến 25 đường Lê Lợi, Quận 1. Cảm ơn bạn!"}}
```

## Test 2: Đơn hợp lệ, được miễn phí ship

**Tin nhắn:** "Đặt 3 sản phẩm A và 1 sản phẩm B, giao 10 đường Hai Bà Trưng, Q3. SĐT 0912345678. Mã GIAM10."

```json
{"tram_1":{"status":"OK","items":[{"name":"A","qty":3},{"name":"B","qty":1}],"address":"10 đường Hai Bà Trưng, Q3","phone":"0912345678","promo_code":"GIAM10","missing_fields":[],"errors":[]},"tram_2":{"status":"OK","reasoning_steps":["Bước 1: A = 100.000 × 3 = 300.000đ; B = 50.000 × 1 = 50.000đ.","Bước 2: subtotal = 350.000đ.","Bước 3: 350.000 ≥ 200.000 nên GIAM10 hợp lệ, discount = 35.000đ; tiền hàng sau giảm = 315.000đ.","Bước 4: 315.000 ≥ 300.000 nên miễn phí ship, shipping_fee = 0.","Bước 5: total = 350.000 − 35.000 + 0 = 315.000đ. Kiểm tra lại ✔. 315.000 < 5.000.000 nên không cần duyệt tay."],"subtotal":350000,"discount":35000,"shipping_fee":0,"total":315000,"need_human_review":false},"tram_3":{"reply_message":"Chào bạn, cửa hàng đã nhận đơn gồm 3 sản phẩm A và 1 sản phẩm B. Tiền hàng 350.000đ, giảm GIAM10 -35.000đ, miễn phí ship. Tổng thanh toán: 315.000đ. Đơn sẽ giao đến 10 đường Hai Bà Trưng, Q3. Cảm ơn bạn!"}}
```

## Test 3: Dữ liệu dị biệt (sản phẩm lạ, số lượng âm, thiếu địa chỉ/SĐT, mã sai)

**Tin nhắn:** "Lấy 3 sản phẩm D và -2 sản phẩm A, giao về nhà. Mã GIAM50."

```json
{"tram_1":{"status":"ERROR","items":[],"address":"","phone":"","promo_code":"GIAM50","missing_fields":["address","phone"],"errors":["Sản phẩm 'D' không có trong bảng giá","Số lượng sản phẩm A không hợp lệ (-2)"]},"tram_2":{"status":"STOPPED"},"tram_3":{"reply_message":"Chào bạn, rất tiếc sản phẩm D hiện không có trong cửa hàng và số lượng sản phẩm A chưa hợp lệ. Bạn vui lòng chọn lại sản phẩm A, B hoặc C với số lượng đúng, đồng thời cho mình xin địa chỉ và số điện thoại nhé. Cảm ơn bạn!"}}
```

## Test 4: Thiếu thông tin (SĐT sai định dạng)

**Tin nhắn:** "Cho mình 1 sản phẩm C, giao 5 đường Nguyễn Trãi, Q5. SĐT 09123."

```json
{"tram_1":{"status":"NEEDS_INFO","items":[{"name":"C","qty":1}],"address":"5 đường Nguyễn Trãi, Q5","phone":"09123","promo_code":"","missing_fields":["phone"],"errors":[]},"tram_2":{"status":"STOPPED"},"tram_3":{"reply_message":"Chào bạn, số điện thoại bạn cung cấp chưa đủ 10 chữ số. Bạn vui lòng gửi lại số điện thoại đầy đủ để cửa hàng hoàn tất đơn hàng nhé. Cảm ơn bạn!"}}
```

## Test 5: Tin nhắn vô nghĩa

**Tin nhắn:** "asdfgh ???"

```json
{"tram_1":{"status":"NEEDS_INFO","items":[],"address":"","phone":"","promo_code":"","missing_fields":["items","address","phone"],"errors":[]},"tram_2":{"status":"STOPPED"},"tram_3":{"reply_message":"Chào bạn, mình chưa nhận được thông tin đơn hàng. Bạn vui lòng cho mình biết sản phẩm, số lượng, địa chỉ và số điện thoại nhé. Cảm ơn bạn!"}}
```

## Bảng tổng kết

| Test | Dữ liệu | Trạm 1 | Trạm 2 | Kết quả |
|---|---|---|---|---|
| 1 | Hợp lệ | OK | OK | Tổng 245.000đ |
| 2 | Hợp lệ, miễn ship | OK | OK | Tổng 315.000đ |
| 3 | Sản phẩm lạ, số lượng âm, thiếu thông tin | ERROR | STOPPED | Báo lỗi cụ thể |
| 4 | SĐT sai định dạng | NEEDS_INFO | STOPPED | Xin SĐT đầy đủ |
| 5 | Tin nhắn vô nghĩa | NEEDS_INFO | STOPPED | Xin bổ sung thông tin |

**Nhận xét:** Cả 5 trường hợp đều trả về JSON hợp lệ, không văn bản thừa, không bịa dữ liệu. Khi Trạm 1 lỗi thì Trạm 2 tự dừng (`STOPPED`) nên không tính tiền sai và tiết kiệm token, và Trạm 3 luôn có phản hồi cho khách.
 ```
 