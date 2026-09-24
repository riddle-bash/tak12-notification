# Notification Center — Feature Design

---

## 1. Overview

Học sinh nhận thông báo từ hai nguồn: nội dung MKT soạn thủ công (tin tức, khuyến mãi) và hệ thống tự động thông báo (ví dụ Thưởng sao, Đạt danh hiệu, GV giao bài tập, ...). Cả hai đổ vào một model campaign + fan-out dùng chung, để trang nhận thông báo của học sinh.

So sánh giữa ABP built-in Notification và Custom notification

| Requirement                      | ABP built-in                          | Your own model |
| -------------------------------- | ------------------------------------- | -------------- |
| Specific users                   | ✅                                     | ✅              |
| Schedule for exact date/time     | ⚠️ Possible, but not ideal            | ✅              |
| Rich-text content                | ⚠️ Usually needs custom data/handling | ✅              |
| Auto/manual trigger              | ⚠️ Not a business concept             | ✅              |
| Achievement/Gem/Learning sources | ❌                                     | ✅              |
| Promotion/update campaigns       | ❌                                     | ✅              |
| Marketing tracking               | ❌                                     | ✅              |
| Sent/delivered/read/clicked      | Partially                             | ✅              |
| Retry/history                    | Limited                               | ✅              |
| Analytics                        | ❌                                     | ✅              |
| Audit campaign configuration     | ❌                                     | ✅              |


Hệ thống cố tình chạy **song song hai cơ chế** (theo yêu cầu thử nghiệm cả hai):

1. **Custom notification system** — bảng riêng, do CTH.QuizSystem tự quản lý, phục vụ toàn bộ nghiệp vụ (nhóm hiển thị, badge theo tab, tìm kiếm, lịch gửi, tracking).
2. **Abp built-in notification system** (`Abp.Notifications.*`) — giữ nguyên như hiện có, dùng cho realtime bell push qua `IAppNotifier`.

---

## 2. Business rules

- HS được xem thông báo trong 90 gần đây, riêng các thông báo cũ hơn vẫn được lưu để phục vụ cho MKT
- Thông báo **Trigger** là thông báo tự động theo cài đặt trong Admin (ví dụ Thưởng sao, Đạt danh hiệu, GV giao bài tập, ...), thuộc nhóm Học Tập hoặc Đơn Hàng, ...
- Thông báo **Manual** do team MKT quản lý, có thể tuỳ chỉnh nội dung HTML, thời điểm thông báo, danh sách HS được thông báo, thuộc nhóm Tin TAK12.
- Trigger chỉ tự sinh thông báo khi đang **Bật** *và* đã có nội dung mẫu (Title + Body) được thiết lập; thiếu một trong hai điều kiện thì không sinh, không lỗi.
- Form import danh sách HS được validate khi upload file và cho phép MKT đối chiếu
- Mỗi thông báo trùng lặp trên cùng một đối tượng (ví dụ nhiều lần cập nhật trạng thái của cùng một đơn hàng) nên gộp vào một dòng thay vì tạo nhiều thông báo rời rạc, miễn là dòng cũ chưa được đọc.
- Danh sách thông báo phía học sinh chỉ hiển thị trong một cửa sổ thời gian gần đây (mặc định 90 ngày, cấu hình được — xem §5.4); thông báo cũ hơn không hiển thị phía học sinh nhưng vẫn lưu để phục vụ báo cáo/thống kê phía MKT.
- Số chưa đọc trên mỗi tab (kể cả tab "Tất cả") = số thông báo chưa đọc **thuộc nhóm đó, của user hiện tại** — không tính tổng toàn hệ thống.
- Tìm kiếm từ khoá: không phân biệt hoa/thường, không dấu/có dấu.

---

## 3. Domain model

### 3.2 Entities

**`NotificationCampaign`** — `FullAuditedEntity` (int PK). Nội dung + metadata gửi, dùng chung cho cả Manual và Trigger.

| Field | Type | Ghi chú |
|---|---|---|
| `Type` | `smallint` | Trigger hoặc Manual |
| `Category` | `smallint` | Phân loại theo Học tập, Đổi Gems, ... |
| `Title`, `Body` | `string` | Manual: Title ≤ 100, Body ≤ 2.000 ký tự |
| `LinkUrl` | `string` | Link khi user ấn vào thông báo |
| `PayloadJson` | `string` | deep-link data (achievementId, orderId...) |
| `TargetType` | `smallint` | Loại HS: Theo DS import hoặc tất cả HS |
| `TargetFilterJson` | `string` | Chuỗi JSON cho các điều kiện để lọc các HS cần thông báo (ví dụ: lớp 5, học Ôn thi vào 6 UMS, ..) |
| `TargetUserIdsJson` | `string` | danh sách user đã đối chiếu khi MKT import danh sách HS |
| `ScheduledAt`, `SentAt` | `DateTime?` | |
| `IsActive` | `bit` | |
| `Status` | `smallint` | enums cho trạng thái |
| `RecipientCount`, `ReadCount`, `ClickCount` | `int` | Phục vụ cho business |

**`NotificationRecipient`** — Bổ sung Title, Body để personalize cho user

| Field | Type | Ghi chú |
|---|---|---|
| `UserId` | `long` | khớp `IRepository<User, long>` |
| `FK: CampaignId` | `int` | |
| `Title` | `string` | |
| `Body` | `string` | |
| `IsRead` | `bool` | |
| `ReadAt` | `DateTime?` | |
| `ClickedAt` | `DateTime?` |  |
| `CreationTime` | `DateTime` | |
| `IdempotencyKey` | `string` | khoá dedupe để tránh insert trùng dữ liệu khi có nhiều worker |

**`LeaderboardRankSnapshot`** — `Entity<int>`. Lưu rank kỳ trước để so sánh, phục vụ riêng TRG-09 (vào Top 10) và TRG-10 (rời Top 10).

| Field | Type | Ghi chú |
|---|---|---|
| `UserId` | `long` | |
| `Rank` | `int` | |
| `SnapshotAt` | `DateTime` | |
| `ExamId` | `int` | Global có examId = 0 |

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/5e1c18b9-2a62-430c-bfc7-8f95ffed478f" />

<img width="975" height="627" alt="image" src="https://github.com/user-attachments/assets/af3e3127-5e3f-497f-a10a-2280bc260799" />


### 3.3 Target condition schema (thay cho JSON tự do)

`TargetFilterJson` : điều kiện để filter user được thông báo trong campaign

```csharp
public class TargetFilterJson
{
    public List<AccountType> AccountTypes { get; set; }      
    public List<int> BirthYears { get; set; }              
    public List<int> PurchasedEditionIds { get; set; }       
    public List<int> InterestedExamIds { get; set; }       
    public int? ActiveWithinDays { get; set; }             
}
```

Toàn bộ field có giá trị được kết hợp **AND** khi resolve

---

## 5. Flows

### 5.1 Luồng A — MKT gửi thông báo Manual

1. **Tạo** — `CustomNotificationAppService.CreateManualCampaignAsync`: soạn nội dung (rich text ⇄ HTML thô, đồng bộ 2 chiều), chọn đối tượng, chọn thời gian gửi. Validate: Title bắt buộc ≤ 100 ký tự, Body bắt buộc ≤ 2.000 ký tự, đối tượng bắt buộc chọn đúng 1 trong 3 cách (không kết hợp).
   - `TargetType = All` — không resolve gì lúc tạo, job gửi tự query toàn bộ user active.
   - `TargetType = Filtered` — lưu `TargetFilterJson` (§3.3); resolve lại tại thời điểm gửi.
   - `TargetType = SelectedUsers` — xem quy trình import riêng bên dưới; đối tượng **chốt tại lúc import**, không resolve lại khi gửi.
2. **Import danh sách (khi chọn ImportedList)** — endpoint riêng, chạy đối chiếu ngay lúc upload, không đợi tới lúc gửi:
   ```csharp
   Task<ImportPreviewResultDto> PreviewImportAsync(int campaignId, IFormFile file)
   ```
3. **"Xem trước"** — đọc thuần từ form state phía FE (Title/Body hiện tại), không gọi API, không tạo bản ghi nào.
4. **Gửi**:
   - Nếu `ScheduledAt == null` → enqueue Hangfire job gửi ngay.
   - Nếu có lịch → `RecurringJobRegistrar` (recurring, DB-driven, chạy mỗi 3h) quét campaign `Status = Scheduled` và `ScheduledAt <= now`, enqueue `NotificationCampaignApplyJob`.
   - `NotificationCampaignApplyJob`: nếu `TargetType = Filtered` thì resolve audience ngay lúc này; nếu `TargetType = ImportedList` thì parse từ `TargetUserIdsJson` (đã chốt từ bước 2); nếu `All` thì query toàn bộ user active. Bulk insert `NotificationRecipient` theo batch, cập nhật `RecipientCount`, `Status = Sent`.
5. **Sửa/hủy trước khi gửi**:
   ```csharp
   Task UpdateScheduledCampaignAsync(int campaignId, UpdateCampaignInput input); // chỉ khi Status == Scheduled
   Task CancelScheduledCampaignAsync(int campaignId);                             // chỉ khi Status == Scheduled
   // Status == Sent → throw UserFriendlyException, lịch sử không được sửa
   ```
6. **Học sinh nhận và đọc** — xem Luồng C.
7. **Theo dõi** — `ReadCount` = `COUNT(NotificationRecipient WHERE IsRead)`; Link URL trong notification được inject thêm param để kiểm soát lượng truy cập

### 5.2 Luồng B — Trigger tự động

| Name | Category | Sự kiện | Ghi chú xử lý |
|---|---|---|---|
| TRG-01 | Học tập | Giao bài (School/phụ huynh/giáo viên) | `entityKey` = assignmentId; `{{nguoi_giao}}` để phân biệt nguồn giao |
| TRG-02 | Học tập | Bài đến hạn chưa làm | Không sinh từ action tức thời — cần job quét định kỳ, xem bên dưới |
| TRG-03 | Đơn hàng | Đặt đơn hàng | `entityKey` = orderId; không cam kết quyền lợi đã kích hoạt |
| TRG-04 | Đơn hàng | Đơn hàng được kích hoạt | Tách biệt TRG-03 vì đơn có thể đặt nhưng chưa từng kích hoạt |
| TRG-05 | Đơn hàng | Gửi góp ý | Chỉ xác nhận đã ghi nhận |
| TRG-06 | Đơn hàng | Góp ý được phản hồi | Chỉ bắn khi xử lý xong, không bắn ở trạng thái trung gian |
| TRG-07 | Học tập | Đạt danh hiệu | `entityKey` = achievementId |
| TRG-08 | Học tập | Được thưởng sao | `entityKey` = giao dịch thưởng |
| TRG-09 | Học tập | Vào Top 10 BXH | Cần `LeaderboardRankSnapshot`, xem bên dưới |
| TRG-10 | Học tập | Rời Top 10 BXH | Loại trừ lẫn nhau với TRG-09 trong cùng lần cập nhật |
| TRG-11 | Đơn hàng | Đơn đổi quà hoàn tất | Tách biệt TRG-03/04 (đổi quà bằng sao, không phải mua gói) |

Seed 11 dòng này qua SQL khi migration chạy;

1. **Admin cấu hình** — Thiết lập `Subject`/`Body` (≤ 100 / ≤ 500 ký tự) cho TriggerTemplate
2. **Backend phát sinh** — gọi trực tiếp từ đúng chỗ trong domain logic, qua `NotificationTriggerService.PublishAsync(triggerCode, userId, entityKey, placeholders)`:
   ```csharp
   var trigger = await _notificationTriggerRepository.FirstOrDefaultAsync(x => x.Name == triggerCode && x.IsActive);
   if (trigger == null || string.IsNullOrWhiteSpace(trigger.Subject) || string.IsNullOrWhiteSpace(trigger.Body))
       return; // Bật nhưng chưa có nội dung mẫu → im lặng bỏ qua, đúng business rule US-06
   ```
   - `IdempotencyKey` (ví dụ `"TRG-07:{userId}:{achievementId}"`) chặn publish trùng khi Hangfire job retry.
3. **TRG-02 (bài quá hạn)** — cần `AssignmentOverdueCheckJob` (recurring, DB-driven) quét bài đã hết hạn mà chưa hoàn thành, gọi `PublishAsync("TRG-02", userId, assignmentId.ToString(), ...)`; `IdempotencyKey` unique index tự chặn việc job chạy lại nhiều lần bắn trùng cho cùng bài. Số lần nhắc lại nếu học sinh vẫn chưa làm: để mở, xem §9.
4. **TRG-09/TRG-10 (Top 10 BXH)** — mỗi lần BXH cập nhật, so `Rank` mới với `LeaderboardRankSnapshot` gần nhất cùng `UserId + ExamId`:
5. **Học sinh nhận và đọc** — xem Luồng C.

### 5.3 Luồng C — Học sinh nhận thông báo

1. **Báo ngay** — SignalR/Abp realtime cập nhật badge chưa đọc (bao gồm tab "Tất cả"); banner/toast riêng (subscribe cùng event, tự ẩn sau 5-6 giây, xếp chồng theo chiều dọc nếu có nhiều banner mới, tách state khỏi dropdown) nếu app đang mở. Banner chỉ hiện tại đúng thời điểm phát sinh — mở lại app sau đó không hiện lại banner, nhưng badge/danh sách vẫn phản ánh đúng.
2. **Mở xem** — dropdown fetch top 10 thông báo gần nhất, có butotn để hiển thị thông báo cũ hơn
3. **Đọc** — User ấn vào link hoặc mark as read, post API markNotificationAsRead(Id)

### 5.4 Retention window (US-01)

Feed phía học sinh chỉ trả về thông báo trong N ngày gần đây (mặc định 90, để trong config chứ không hard-code); **không** áp dụng cutoff này cho query thống kê của MKT (US-05) — lịch sử gửi luôn xem được đầy đủ.

```csharp
// GetFeedAsync (học sinh) — có cutoff
var cutoff = Clock.Now.AddDays(-_settings.NotificationRetentionDays);
query = query.Where(x => x.CreationTime >= cutoff);

// GetCampaignHistoryAsync (MKT, US-05) — không cutoff
```


## 6. Open questions

- Số ngày retention hiển thị phía học sinh — đề xuất mặc định 90 ngày, cần chốt.
- Số lần nhắc lại tối đa cho TRG-02 nếu học sinh vẫn chưa làm bài sau khi đã nhắc.
