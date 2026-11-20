# SE214-11 — Quyết định kiến trúc D-06

**Sprint 0 · Mốc hồ sơ 16/11/2026 · Bản review 0.1.0**

**Phương án được chọn để trình duyệt:** mở rộng `post-service` với các module Place, Proposal, Review, Rating và Attribute. Các module này có nhiều bất biến cần cập nhật cùng nhau, nên dùng một ranh giới giao dịch PostgreSQL. Các microservice hiện có tiếp tục sở hữu dữ liệu và vòng đời triển khai riêng. **Tech Lead Tú và người review Hoàng xác nhận bản này trước khi đóng ticket.** [SE214-11](https://edison001002019.atlassian.net/browse/SE214-11) · [SE214-12](https://edison001002019.atlassian.net/browse/SE214-12) · [SE214-13](https://edison001002019.atlassian.net/browse/SE214-13)

## 1. Kiến trúc và sở hữu dữ liệu

```text
Next.js ── HTTPS / JWT ──► API Gateway
                            ├─ /api/auth/** ─► auth-service ─► PostgreSQL sma_auth
                            ├─ /api/users/** ► user-service ─► PostgreSQL sma_user
                            ├─ /api/media/** ► media-service ► sma_media + object storage
                            ├─ /api/pages/** ► fanpage-service ► PostgreSQL sma_fanpage
                            ├─ /api/chat/** ─► chat-service ──► MongoDB
                            └─ Posts + Places + Proposals + Reviews + Shares
                                                 │
                                                 ▼
                                            post-service
                               Place · Proposal · Post · Review
                                      Rating · Attribute
                                                 │ transaction
                                                 ▼
                                   PostgreSQL sma_post + outbox
                                                 │ sau commit
                                                 ▼
                                               Kafka
                                        ├─ feed-service
                                        └─ notification-service
```

`sma_*` là schema trên profile Supabase hiện có. Mỗi service truy cập schema được cấp cho chính nó. Redis/Valkey hỗ trợ rate limiting và feed, không là nguồn dữ liệu gốc của Place, Review hoặc Rating. [Cấu hình Gateway](https://github.com/Social-media-sharing-location/SE214.R11_Social_Media_Real_Time_Multi_Platform/blob/af4b3aa795b572fec868a1230e67a6be6e71bdf7/infra/api-gateway/src/main/resources/application.yml) · [Profile post-service](https://github.com/Social-media-sharing-location/SE214.R11_Social_Media_Real_Time_Multi_Platform/blob/af4b3aa795b572fec868a1230e67a6be6e71bdf7/services/post-service/src/main/resources/application-cloud-free.yml) · [Schema và phân quyền](https://github.com/Social-media-sharing-location/SE214.R11_Social_Media_Real_Time_Multi_Platform/blob/af4b3aa795b572fec868a1230e67a6be6e71bdf7/infra/no-docker/config/supabase/init-schemas.sql)

| Chủ sở hữu | Bảng, dữ liệu hoặc hợp đồng |
| --- | --- |
| **`post-service` · PostgreSQL `sma_post`** | Place theo chi nhánh và trạng thái, Category, PlaceProposal và ảnh tham chiếu, Post, ReviewData/revision/cooldown ledger, CommunityRating, AttributeDefinition/Answer, PlaceRatingSummary và PlaceAttributeSummary. Hai summary tái dựng được từ dữ liệu gốc. |
| **`auth-service`, `user-service`** | Tài khoản/phiên/JWT do Auth sở hữu. User sở hữu hồ sơ, trạng thái Friend/Follow/Block. Post lấy quyền theo viewer qua hợp đồng dịch vụ; Follow không cấp quyền Friend. |
| **`media-service`, `fanpage-service`** | Media sở hữu ID và quyền cấp URL của object. Fanpage sở hữu vai trò quản trị và xác minh đại diện đúng Place. Post kiểm tra quyền trước khi trả danh tính/ảnh; lỗi dịch vụ quyền không cấp thêm quyền. |
| **`comment-service`, `reaction-service`, `feed-service`** | Comment/Reaction sở hữu tương tác của mình, hỏi Post về quyền đọc/ghi và trạng thái Place trước mutation. Feed dùng event để cập nhật danh sách rồi kiểm quyền khi đọc. |
| **`chat-service`, `notification-service`, `moderation-service`** | Chat quản lý yêu cầu/hội thoại. Notification quản lý thông báo. Moderation xử lý quyết định báo cáo và gọi hợp đồng Post khi có quyết định gỡ; Post cập nhật nguồn và phép tính Rating/Attribute. |

## 2. Các luồng và ranh giới giao dịch

| Luồng | Các bước và điểm commit |
| --- | --- |
| **Auth** | Next.js gửi login qua Gateway đến Auth. Auth cấp access JWT. Gateway xác minh JWT và gắn định danh phía server. Các dịch vụ vẫn kiểm role, quyền sở hữu và ngữ cảnh actor. Khi mở GET công khai, Gateway phải phân biệt theo **method + route** và xác minh JWT nếu client gửi. |
| **Tạo Post/Review** | Post xác minh actor, đúng một `placeId` hoặc `placeProposalId`, quyền Media, privacy và trạng thái Place. Với Place hợp lệ, một transaction ghi Post, ReviewData/ledger và dữ liệu tổng hợp cần đổi. Với Proposal chờ duyệt, ghi `PENDING_PLACE` và chưa đưa vào feed. |
| **Duyệt Proposal** | Một transaction khóa Proposal, ghi quyết định, liên kết đúng một Place, ghi thao tác xử lý bài chờ. Xử lý mỗi Post chờ bằng transaction riêng có guard trạng thái và khóa theo author/Place khi cần kiểm cooldown; từ chối hoặc vi phạm cooldown thì `DISCARDED`, lưu lý do và thông báo tác giả. Pending không thành bài công khai trước commit xuất bản. |
| **Sửa, chấm điểm và gỡ Review** | Một lần lưu Review cập nhật content/A/attributes, lịch sử, version và một lượt sửa chung. Rating upsert theo (reviewPostId, raterId). Khi dữ liệu hiệu lực đổi hoặc bài bị gỡ, tính lại summary trên nguồn đã commit; fallback về REVIEW hợp lệ trước đó. Ledger cooldown được giữ. |
| **Đọc Place/Review** | Service chọn projection theo quyền và ngữ cảnh đã xác minh. Stranger xem REVIEW FRIENDS chỉ tại Place dưới dạng ẩn danh, chỉ đọc. Quyền trả danh tính/media phụ thuộc Friend hoặc Fanpage chính thức đúng Place và kiểm Block. Place đóng vĩnh viễn giữ lịch sử đúng quyền và chặn tương tác mới. |

**Đồng thời và sự kiện.** Dùng khóa hàng hoặc version có điều kiện cho state transition và edit counter. Ràng buộc duy nhất cho phiếu hiệu lực và bản ghi xử lý idempotency theo actor, thao tác và key. Chuẩn hóa thứ tự xử lý các bài chờ theo thời điểm tạo rồi ID để quyết định cooldown có thể tái lập. Khi so REVIEW đã xuất bản, thứ tự là `publishedAt` rồi ID ổn định để xử lý đồng hạng. Tác vụ nội bộ ghi cùng transaction với dữ liệu nguồn. Event đưa ra Kafka qua bảng outbox và worker trong `post-service` sau commit, retry được. Consumer dedupe theo `eventId` và kiểm version/trạng thái khi nhận trễ. Dùng lại payload/topic Post legacy khi cần giữ consumer cũ. Event mới cho Proposal quyết định, Review xuất bản/sửa/gỡ và Rating thay đổi chỉ mang ID nội bộ, thời gian và version cần thiết. Feed/Notification xác minh quyền khi dựng nội dung. Xem SRS §8.3, §9.8 và phụ lục A.3.

## 3. Migration và tương thích Post legacy

1. **Chuẩn bị.** SE214-13 xác minh bảng `posts`, kiểu ID `String` ở Java, kết quả SQL schema thực và profile đang chạy. Snapshot schema/dữ liệu ở môi trường kiểm thử, ghi số bản ghi và mẫu trạng thái privacy. Dùng migration PostgreSQL có phiên bản.
2. **Mở rộng.** Thêm bảng mới và cột `post_type`, `place_id`, `place_proposal_id`, `publish_status` tương thích dữ liệu cũ. Giữ cột mới nullable cho hàng legacy, giữ ID/counter/media/privacy cũ. Thêm unique/check có điều kiện cho dữ liệu mới sau khi kiểm tra. Không tự gắn Place, không mặc định chuyển `PRIVATE/CUSTOM` sang `PUBLIC/FRIENDS`.
3. **Triển khai N/N−1.** Cập nhật backend, consumer và FE đọc được hai kiểu dữ liệu trước khi bật ghi Place-linked. Trong giai đoạn tương thích, server quản lý thời điểm còn nhận POST legacy. Khi cutover sau khi FE/BE và consumer sẵn sàng, POST bài người dùng mới bắt buộc đi qua hợp đồng có Place hoặc Proposal. Bài legacy đang tồn tại giữ đường đọc và PUT theo quyền cũ. Với bài có loại mới, dùng PATCH chuyên biệt để PUT cũ không vượt bộ đếm.
4. **Kiểm thử trước cutover.** Chạy duy nhất một bộ migration trên DB sạch và DB có Post legacy. So số bản ghi, ID, quyền truy cập và media. Chạy AT-39 với Post PUBLIC, FRIENDS, PRIVATE, CUSTOM và truy cập trái quyền. Kiểm retry/đồng thời ở approve, edit counter và Rating. So producer/consumer ở hai phiên bản và kiểm cache/URL phương tiện ngoài quyền.
5. **Quay lui.** Tắt ghi Place-linked, tạm dừng worker theo quy trình, giữ cột/bảng mở rộng. Chỉ chạy binary N−1 nếu đầu đọc và quyền legacy đã qua kiểm tra với dữ liệu mới, nếu không giữ phiên bản đọc tương thích và vá tiếp. Restore snapshot chỉ khi cần phục hồi dữ liệu, có đối soát thay đổi sau snapshot và kiểm tra feed/outbox trước khi phát lại. Không drop bảng để rollback ứng dụng. Owner migration: Hoàng theo SE214-13.

| Rủi ro | Cách giảm và tiêu chí xác nhận |
| --- | --- |
| Post cũ bị ép có Place hoặc thay quyền | Cột nullable cho legacy. AT-39 so dữ liệu và quyền trước/sau. |
| Cập nhật qua PUT cũ vượt giới hạn | PUT legacy chỉ sửa hàng legacy. PATCH mới dùng một transaction và guard chung. AT-19/47 chạy retry và race. |
| Duyệt lặp hoặc consumer chạy lại | Guard state, idempotency, khóa theo author/Place và dedupe event. AT-03/22 kiểm Place, Post, notice không trùng. |
| Mất quyền riêng tư qua media/cache/feed | DTO ẩn danh riêng, URL theo quyền, response phân biệt viewer và không cache dùng chung. AT-06–09/30/49 kiểm cả truy cập gián tiếp. |
| Rating đổi số hoặc fallback sai | Giữ công thức 50/50, trọng số log₂ và thứ tự newest valid. AT-10–14/24–29 đối chiếu điểm và dữ liệu gốc. |

## 4. Mục chốt tại review

- **Tú (Tech Lead)** xác nhận ownership, protocol nội bộ, migration và rollback với Hoàng. Ghi link/comment xác nhận đúng phiên bản 0.1.0 tại SE214-11.
- **Hoàng (Backend)** đối chiếu API, PostgreSQL và consumer Kafka với SE214-12/13. Xác nhận guard quyền khi G2 tích hợp.
- **FR/BR áp dụng theo SRS v1.4**, gồm BR-09 về quyền, D-03 về cooldown, BR-44/45 về số lượt, BR-08/D-02 về fallback, FR-21 về đóng cửa. Hồ sơ này chốt đề xuất tổ chức kỹ thuật để review.

**Nguồn hồ sơ** — `Baocaotiendo.pdf`, tr. 9 bảng 1.5 và tr. 29 bảng 1.15. `SE214_SRS_Hoan_Thien.docx`, §6, §8.1–8.6, §9.1–9.9, §11, phụ lục A. [Entity Post hiện hành](https://github.com/Social-media-sharing-location/SE214.R11_Social_Media_Real_Time_Multi_Platform/blob/af4b3aa795b572fec868a1230e67a6be6e71bdf7/services/post-service/src/main/java/com/socialapp/post/entity/Post.java) · [Bộ lọc JWT hiện hành](https://github.com/Social-media-sharing-location/SE214.R11_Social_Media_Real_Time_Multi_Platform/blob/af4b3aa795b572fec868a1230e67a6be6e71bdf7/infra/api-gateway/src/main/java/com/socialapp/gateway/filter/JwtAuthenticationFilter.java)
