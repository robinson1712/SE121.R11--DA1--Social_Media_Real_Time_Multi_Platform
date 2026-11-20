# PB-02 (#3) — Biên bản rà soát API Contract | SE121.R11 — Đồ án 1

**Sprint 0 · Phiên bản tài liệu 0.2.0 (20/11/2026) · Chờ review/merge**

**Tệp đi kèm:** `PB-02_OpenAPI.json` · OpenAPI 3.1.1, 15 path, 18 thao tác. D-06 mở rộng `post-service`, giữ REST/JSON qua API Gateway hiện có. [PB-02 #3](https://github.com/robinson1712/SE121.R11--DA1--Social_Media_Real_Time_Multi_Platform/issues/3) · [PB-01 #2](https://github.com/robinson1712/SE121.R11--DA1--Social_Media_Real_Time_Multi_Platform/issues/2)

## Đối chiếu AC

| Tiêu chí | Nội dung bàn giao |
| --- | --- |
| Request/response và enum | Tìm/xem/lọc Place, bài tại Place, điểm/thuộc tính, tạo/duyệt Proposal, đăng/sửa SHARE và REVIEW, đọc và upsert điểm. Có đường dẫn login để chỉ rõ nguồn JWT. Response dùng envelope `success/message/data/timestamp`. |
| Phân quyền và lỗi | Guest hoặc JWT tùy method/route, backend kiểm User/Admin và quyền trên tài nguyên. `Error.code` là trường bổ sung của API mới. Client cũ có thể tiếp tục đọc `message`. Error schema và ví dụ gắn theo HTTP status. |
| Privacy và tình huống biên | REVIEW FRIENDS ẩn danh chỉ tại Place. Điểm nguyên 1–5. Place tạm đóng cho SHARE và từ chối REVIEW mới. Place đóng vĩnh viễn cho đọc lịch sử theo quyền và từ chối tạo/sửa/chấm điểm/tương tác mới. Payload sai cấu trúc nằm ở `x-schema-test-cases`, hành vi theo actor/trạng thái ở `x-runtime-acceptance`. |
| Tương thích Post legacy | Hồ sơ nguồn có đối chiếu DTO/Entity/Controller tại commit của repo khác; cần đối chiếu lại DTO hiện hữu và commit SE121 trước khi xác nhận tương thích. Ví dụ Post PRIVATE không có Place vượt kiểm tra schema cả chiều ghi cũ và đọc. BG-13 chạy AT-39 trên DB đã migration. |
| Kiểm tra hợp đồng | Ngày 20/11/2026, đối chiếu lại OpenAPI, HTTP 409/422 và fixture tĩnh trong gói phát hành. **Kết quả kiểm tra tĩnh cũ chưa tái xác minh tại repo SE121**. Kết quả kiểm tra trên file JSON cập nhật được ghi trong `KIEM_DINH_TRUOC_PUSH.md`; contract test FE↔BE runtime chưa được cung cấp. Các ca backend tích hợp nằm trong kế hoạch nghiệm thu của ticket liên quan. |

## Ví dụ cần FE/BE dùng chung

| Kịch bản | Payload hoặc trạng thái | Kỳ vọng |
| --- | --- | --- |
| Privacy | Stranger xem REVIEW FRIENDS bằng `GET /api/places/{id}/posts` | `200 ANONYMOUS_READ_ONLY`, author null, quyền tương tác false, không kèm user ID hoặc URL media riêng tư. Cùng actor gọi Post trực tiếp nhận 404. |
| Privacy | Owner yêu cầu PUBLIC → FRIENDS sau khi xuất bản | `409 VISIBILITY_DOWNGRADE_NOT_ALLOWED`. FRIENDS → PUBLIC hợp lệ nếu quyền và trạng thái Place cho phép. |
| Rating | User có quyền gửi `{"score":5}`, sau đó đổi thành 3 | Upsert một bản ghi, số người chấm giữ nguyên ở lần đổi. Điểm 6 hoặc 4.5 bị từ chối. Tác giả tự chấm nhận `403 SELF_RATING_NOT_ALLOWED`. |
| Place | Place `TEMPORARILY_CLOSED`, gửi cùng địa điểm với hai loại bài | SHARE được tạo, REVIEW mới nhận `409 PLACE_CLOSED`. Với `PERMANENTLY_CLOSED`, đọc lịch sử vẫn được theo quyền, ghi mới bị chặn. |
| Legacy | Post cũ có `id`, `privacy: PRIVATE`, `mediaUrls`, không có `placeId` | Owner đọc lại bằng GET và JWT với dữ liệu/ID/quyền cũ. Người ngoài vẫn bị từ chối. Dùng AT-39 xác nhận sau migration. |

## Các thay đổi cần đối chiếu trước khi merge

- **Gateway:** route mới cho Places/Proposals/Reviews/Shares về `post-service`. Allowlist GET có thể đọc công khai theo method. JWT hợp lệ được giữ context khi đọc tùy người xem. Cho phép `Idempotency-Key` trong CORS cho các mutation mới. *(đường dẫn mã nguồn cũ; chưa xác minh trên repo SE121)* · *(đường dẫn mã nguồn cũ; chưa xác minh trên repo SE121)*
- **Frontend:** adapter phân biệt `PostFull`, `AnonymousReview`, trạng thái của bài chờ và `LegacyPost`. Nhánh legacy giữ `id/authorId/privacy/mediaUrls`. Nhánh mới dùng `postId/author/visibility` và tham chiếu `mediaIds` lấy từ kết quả upload. *(đường dẫn mã nguồn cũ; chưa xác minh trên repo SE121)* · *(đường dẫn mã nguồn cũ; chưa xác minh trên repo SE121)*
- **Backend:** giữ POST/PUT legacy trong giai đoạn tương thích. PATCH dành cho bài mới, không đi vòng bộ đếm BR-44/45 qua PUT. Khi có `attributes` ở PATCH REVIEW, thay toàn bộ tập câu trả lời hiện hành; bỏ field để giữ nguyên. Kiểm trùng theo `attributeId`, tính phù hợp category và trạng thái với DB. *(đường dẫn mã nguồn cũ; chưa xác minh trên repo SE121)*
- **Quy ước đề xuất để review:** `Idempotency-Key`, `expectedVersion`, `accessMode`, tên query filter, thao tác quản trị `PATCH /api/places/{id}` và map HTTP status. Nhóm dùng 409 cho xung đột trạng thái và 422 cho payload sai quy tắc. Giữ FR/BR/AT của SRS khi chỉnh hình dạng wire.

**Xác nhận trước merge:** Repo SE121 chỉ có Nguyễn Tiến Sơn (`Jandz01`) và Nguyễn Hữu Trường Sơn (`robinson1712`). Hai thành viên kiểm tra chéo API contract, backward compatibility, migration và mapping DTO; người tạo PR không tự duyệt PR của mình. Việc phân công người chạy FE/BE contract test và ký chốt D-06 ở PB-01 #2 cần ghi lại tại PB-57 #9, PB-02 #3 hoặc PR. Không coi phần rà soát này là đã chạy test thực tế.

**Nguồn tham khảo lịch sử SE214 (không phải tài liệu SE121)** — `Baocaotiendo.pdf`, bảng 1.5 tr. 9 và bảng 1.15 tr. 29 (chưa có bản PDF gốc để đối chiếu); `SE214_SRS_Hoan_Thien.docx`, §8.1–8.6, §9.1–9.9, §11 và phụ lục A. [OpenAPI 3.1.1](https://spec.openapis.org/oas/v3.1.1.html) · [Schema OpenAPI 3.1](https://spec.openapis.org/oas/3.1/schema/2026-11-05)


## Trạng thái AC3 và quy ước HTTP sau rà soát 20/11

- `400`: JSON/query/header sai cú pháp; `422`: payload có cấu trúc hợp lệ nhưng sai dữ liệu (ví dụ `VALIDATION_ERROR`, `INVALID_REVIEW_SCORE`, `INVALID_ATTRIBUTE`).
- `409`: xung đột trạng thái/version/idempotency/cooldown, kể cả `PLACE_CLOSED`, `REVIEW_NOT_RATEABLE` và `CONCURRENT_MODIFICATION`; `403`: quyền bị từ chối, gồm `SELF_RATING_NOT_ALLOWED`.
- `VALIDATION_ERROR` phải có `details.fieldErrors` khi biết trường lỗi. SRS §9.7 và checklist PB-62 thống nhất mã này; API Auth register chưa nằm trong 18 thao tác của gói PB-02 nên AT-50 chưa được tính là contract test đã chạy.
- **AC3 chưa hoàn tất** nếu chưa có kết quả chạy contract test với FE client và BE thật, ghi SHA, môi trường, người chạy và trạng thái. `tools/validate_contract.py` trong gói chỉ kiểm schema/fixture tĩnh; không được ghi runtime PASS.

| Hạng mục | Trạng thái xác nhận | Bằng chứng yêu cầu |
| --- | --- | --- |
| OpenAPI + fixture tĩnh | Có kết quả trong `KIEM_DINH_TRUOC_PUSH.md` | Output script kiểm định |
| FE parse DTO success/error | Chưa có log thực thi | Commit FE, report test client |
| BE response và idempotency | Chưa có log thực thi | Commit BE, test report |
| Review và merge | Chờ review | PR/link GitHub |
