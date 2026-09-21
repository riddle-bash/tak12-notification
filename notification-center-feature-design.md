# Notification Center — Feature Design

---

## 1. Overview

Học sinh nhận thông báo từ hai nguồn: nội dung MKT soạn thủ công (tin tức, khuyến mãi) và sự kiện hệ thống tự động (11 trigger: giao bài, deadline, đơn hàng, góp ý, danh hiệu, sao thưởng, thứ hạng — xem §5.2). Cả hai đổ vào một model campaign + fan-out dùng chung, để trang nhận thông báo của học sinh và bảng theo dõi của MKT chỉ cần viết một lần.

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

### 3.1 Enums

```csharp
namespace CTH.QuizSystem.Notifications
{
    public enum NotificationGroup
    {
        Study = 1,  // "Học tập" — chỉ dùng cho Automate
        Order = 2,  // "Đơn hàng" — chỉ dùng cho Automate
        News  = 3   // "Tin TAK12" — cố định cho Manual
    }

    public enum NotificationSource { Manual, Trigger }

    public enum NotificationTargetType { All, Segment, ImportedList, SingleUser }

    public enum CampaignStatus { Draft, Scheduled, Sending, Sent, Failed, Cancelled }

    public enum AccountType { Free, Trial, Pro }
}
```

### 3.2 Entities

**`NotificationCampaign`** — `FullAuditedEntity` (int PK). Nội dung + metadata gửi, dùng chung cho cả Manual và Trigger.

| Field | Type | Ghi chú |
|---|---|---|
| `Source` | `NotificationSource` | Manual hoặc Trigger |
| `TriggerCode` | `string` | null nếu Manual; với Trigger là mã cố định do hệ thống seed (`TRG-01`…`TRG-11`), không phải admin tự đặt |
| `Group` | `NotificationGroup` | Study/Order lấy từ trigger config; News nếu Manual (hard-coded) |
| `GroupKey` | `string` | khoá dedupe theo entity cụ thể, ví dụ `"TRG-04:4821"`; null với campaign Manual |
| `Title`, `Body` | `string` | Manual: Title ≤ 100, Body ≤ 2.000 ký tự |
| `LinkUrl` | `string` | dùng cho click tracking |
| `PayloadJson` | `string` | deep-link data (achievementId, orderId...) |
| `SearchText` | `string` | title+body đã chuẩn hoá (lowercase, bỏ dấu), tính lại mỗi khi Title/Body đổi |
| `TargetType` | `NotificationTargetType` | |
| `TargetConditionJson` | `string` | serialize từ `NotificationTargetCondition` (xem bên dưới) — chỉ dùng khi `TargetType = Segment` |
| `ImportedListFileUrl` | `string` | file gốc đã upload, lưu tham chiếu; danh sách user đã khớp nằm ở `NotificationCampaignImportedRecipient`, không resolve lại từ file |
| `ScheduledAt`, `SentAt` | `DateTime?` | |
| `Status` | `CampaignStatus` | |
| `RecipientCount`, `ReadCount`, `ClickCount` | `int` | denormalized, cập nhật định kỳ; `ClickCount` = tổng lượt click (đếm sự kiện, không phải unique user) |
| `IdempotencyKey` | `string` | tránh publish trùng khi job retry |

**`NotificationRecipient`** — `Entity<int>` với `CreationTime` thủ công (child/log row, theo convention `AchievementUser`/`GemShopOrderLog`). **Không đặt tên `UserNotification`** — trùng với `Abp.Notifications.UserNotification` đã dùng trong `GetNotificationsOutput.cs`.

| Field | Type | Ghi chú |
|---|---|---|
| `UserId` | `long` | khớp `IRepository<User, long>` |
| `CampaignId` | `int` | FK → `NotificationCampaign` |
| `Group` | `NotificationGroup` | denormalized từ Campaign — feed query không cần join |
| `IsRead` | `bool` | |
| `ReadAt` | `DateTime?` | |
| `ClickedAt` | `DateTime?` | mốc lần click gần nhất — dùng cho UI "đã từng click chưa"; tổng lượt click nằm ở `NotificationClickLog` |
| `CreationTime` | `DateTime` | |

**`NotificationCampaignImportedRecipient`** — `Entity<int>`. Snapshot danh sách user đã khớp được khi MKT import file, chốt tại thời điểm import (không re-evaluate khi gửi).

| Field | Type | Ghi chú |
|---|---|---|
| `CampaignId` | `int` | FK → `NotificationCampaign`, chỉ áp dụng khi `TargetType = ImportedList` |
| `UserId` | `long` | user đã đối chiếu khớp từ file (ưu tiên theo Email nếu file có cả Email và SĐT) |

**`NotificationClickLog`** — `Entity<int>`. Log từng sự kiện click, phục vụ "tổng lượt click" (US-05) thay vì chỉ đếm unique user.

| Field | Type | Ghi chú |
|---|---|---|
| `RecipientId` | `int` | FK → `NotificationRecipient` |
| `ClickedAt` | `DateTime` | |

**`NotificationTriggerDefinition`** — `Entity<int>`. Cấu hình admin cho từng trigger tự động; **11 dòng khởi điểm được seed sẵn** (xem §5.2), Admin chỉnh metadata (tên, nhóm, bật/tắt, nội dung mẫu), không tự tạo `TriggerCode` mới qua UI.

| Field | Type | Ghi chú |
|---|---|---|
| `TriggerCode` | `string` | unique, cố định do hệ thống sinh (`TRG-01`…`TRG-11`), không sửa được sau khi tạo |
| `Name` | `string` | tên hiển thị trên Admin Panel, MKT/Admin có thể đổi, ≤ 100 ký tự |
| `Group` | `NotificationGroup` | chỉ nhận `Study` hoặc `Order` — enforce ở app service + CHECK constraint |
| `TitleTemplate` | `string` | ≤ 100 ký tự, hỗ trợ placeholder `{{...}}` |
| `BodyTemplate` | `string` | ≤ 500 ký tự, hỗ trợ placeholder + link |
| `AllowedPlaceholders` | `string` | CSV các biến động hợp lệ cho trigger này, ví dụ `"ten_hoc_sinh,ten_danh_hieu,ngay_dat"` — validate khi lưu template |
| `LinkUrlTemplate` | `string` | |
| `IsActive` | `bool` | |

**`LeaderboardRankSnapshot`** — `Entity<int>`. Lưu rank kỳ trước để so sánh, phục vụ riêng TRG-09 (vào Top 10) và TRG-10 (rời Top 10).

| Field | Type | Ghi chú |
|---|---|---|
| `LeaderboardId` | `int` | loại BXH (theo lớp/khối/trường...) — phạm vi cụ thể để mở (xem §9) |
| `UserId` | `long` | |
| `Rank` | `int` | |
| `SnapshotAt` | `DateTime` | |

### 3.3 Target condition schema (thay cho JSON tự do)

`TargetConditionJson` serialize từ class có schema rõ, không phải object tự do — 3 field đầu là điều kiện mặc định khi chọn "Theo điều kiện", 3 field sau chỉ hiện khi MKT bấm "+ Thêm điều kiện":

```csharp
public class NotificationTargetCondition
{
    public List<AccountType> AccountTypes { get; set; }       // mặc định hiện
    public List<int> BirthYears { get; set; }                  // mặc định hiện
    public List<int> PurchasedPackageIds { get; set; }         // "+ Thêm điều kiện"
    public List<int> InterestedExamIds { get; set; }           // "+ Thêm điều kiện"
    public int? ActiveWithinDays { get; set; }                  // "+ Thêm điều kiện"
}
```

Toàn bộ field có giá trị được kết hợp **AND** khi resolve — không hỗ trợ OR giữa các nhóm điều kiện. `ITargetAudienceResolver.ResolveAsync(condition)` dịch từng field thành điều kiện SQL tương ứng, chạy tại thời điểm gửi thực tế (không phải lúc tạo campaign).

---

## 5. Flows

### 5.1 Luồng A — MKT gửi thông báo Manual

1. **Tạo** — `CustomNotificationAppService.CreateManualCampaignAsync`: soạn nội dung (rich text ⇄ HTML thô, đồng bộ 2 chiều), chọn đối tượng, chọn thời gian gửi. `Group` bị ép cứng = `News`. Validate: Title bắt buộc ≤ 100 ký tự, Body bắt buộc ≤ 2.000 ký tự, đối tượng bắt buộc chọn đúng 1 trong 3 cách (không kết hợp).
   - `TargetType = All` — không resolve gì lúc tạo, job gửi tự query toàn bộ user active.
   - `TargetType = Segment` — lưu `NotificationTargetCondition` (§3.3); resolve lại tại thời điểm gửi.
   - `TargetType = ImportedList` — xem quy trình import riêng bên dưới; đối tượng **chốt tại lúc import**, không resolve lại khi gửi.
2. **Import danh sách (khi chọn ImportedList)** — endpoint riêng, chạy đối chiếu ngay lúc upload, không đợi tới lúc gửi:
   ```csharp
   Task<ImportPreviewResultDto> PreviewImportAsync(int campaignId, IFormFile file);
   // đối chiếu theo Email (ưu tiên) hoặc SĐT; trả matchedCount + danh sách dòng không khớp để MKT xem
   // đồng thời ghi matched userIds vào NotificationCampaignImportedRecipient gắn với campaign (Draft)
   ```
   Giới hạn số dòng / kích thước file: để mở, xem §9.
3. **"Xem trước"** — đọc thuần từ form state phía FE (Title/Body hiện tại), không gọi API, không tạo bản ghi nào.
4. **Gửi**:
   - Nếu `ScheduledAt == null` → enqueue Hangfire job gửi ngay.
   - Nếu có lịch → `NotificationCampaignSchedulerJob` (recurring, DB-driven, chạy mỗi phút) quét campaign `Status = Scheduled` và `ScheduledAt <= now`, enqueue `NotificationBroadcastJob`.
   - `NotificationBroadcastJob`: nếu `Segment` thì resolve audience ngay lúc này; nếu `ImportedList` thì đọc thẳng từ `NotificationCampaignImportedRecipient` (đã chốt từ bước 2); nếu `All` thì query toàn bộ user active. Bulk insert `NotificationRecipient` theo batch, cập nhật `RecipientCount`, `Status = Sent`.
5. **Sửa/hủy trước khi gửi**:
   ```csharp
   Task UpdateScheduledCampaignAsync(int campaignId, UpdateCampaignInput input); // chỉ khi Status == Scheduled
   Task CancelScheduledCampaignAsync(int campaignId);                             // chỉ khi Status == Scheduled
   // Status == Sent → throw UserFriendlyException, lịch sử không được sửa
   ```
6. **Học sinh nhận và đọc** — xem Luồng C.
7. **Theo dõi** — `ReadCount` = `COUNT(NotificationRecipient WHERE IsRead)`; `ClickCount` = `COUNT(NotificationClickLog)` (tổng lượt click, không phải unique user) — cả hai denormalized trên `NotificationCampaign`, cập nhật bằng job định kỳ. Click tracking qua redirect endpoint `GET /n/{recipientId}/click`: ghi 1 dòng vào `NotificationClickLog`, set `NotificationRecipient.ClickedAt` nếu đang null, rồi 302 sang `LinkUrl` thật.

### 5.2 Luồng B — Trigger tự động

**Danh sách trigger khởi điểm — seed sẵn, không phải admin tự tạo:**

| Mã | Nhóm | Sự kiện | Ghi chú xử lý |
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

Seed 11 dòng này qua SQL script riêng (`20260918_SeedNotificationTriggers.sql`) khi migration chạy; Admin Panel chỉ CRUD `Name`, `Group`, `IsActive`, template — không tạo/xoá `TriggerCode`.

1. **Admin cấu hình** — CRUD `NotificationTriggerDefinition`: sửa tên/nhóm/bật-tắt, và thiết lập `TitleTemplate`/`BodyTemplate` (≤ 100 / ≤ 500 ký tự). Khi lưu template, validate mọi `{{placeholder}}` xuất hiện phải nằm trong `AllowedPlaceholders` của đúng trigger đó.
2. **Backend phát sinh** — gọi trực tiếp từ đúng chỗ trong domain logic, qua `NotificationTriggerService.PublishAsync(triggerCode, userId, entityKey, placeholders)`:
   ```csharp
   var def = await _triggerDefRepo.FirstOrDefaultAsync(x => x.TriggerCode == triggerCode && x.IsActive);
   if (def == null || string.IsNullOrWhiteSpace(def.TitleTemplate) || string.IsNullOrWhiteSpace(def.BodyTemplate))
       return; // Bật nhưng chưa có nội dung mẫu → im lặng bỏ qua, đúng business rule US-06
   ```
   - `GroupKey = $"{triggerCode}:{entityKey}"` — dedupe theo entity cụ thể, không theo cả trigger.
   - Nếu đã có `NotificationRecipient` chưa đọc cùng `GroupKey` → update title/body của campaign cũ thay vì tạo dòng mới.
   - `IdempotencyKey` (ví dụ `"TRG-07:{userId}:{achievementId}"`) chặn publish trùng khi Hangfire job retry.
   - `RenderTemplate`: placeholder không có trong `placeholders` được thay bằng chuỗi rỗng, không throw — tránh chặn cả thông báo chỉ vì thiếu 1 biến động.
   - Sau khi ghi campaign + recipient, gọi `IAppNotifier.SendMessageAsync` để đẩy realtime qua Abp built-in.
3. **TRG-02 (bài quá hạn)** — cần `AssignmentOverdueCheckJob` (recurring, DB-driven) quét bài đã hết hạn mà chưa hoàn thành, gọi `PublishAsync("TRG-02", userId, assignmentId.ToString(), ...)`; `IdempotencyKey` unique index tự chặn việc job chạy lại nhiều lần bắn trùng cho cùng bài. Số lần nhắc lại nếu học sinh vẫn chưa làm: để mở, xem §9.
4. **TRG-09/TRG-10 (Top 10 BXH)** — mỗi lần BXH cập nhật, so `Rank` mới với `LeaderboardRankSnapshot` gần nhất cùng `UserId + LeaderboardId`:
   ```csharp
   var prev = await _snapshotRepo.GetLatestAsync(leaderboardId, userId); // null nếu chưa từng có snapshot
   if ((prev == null || prev.Rank > 10) && newRank <= 10)
       await _triggerService.PublishAsync("TRG-09", userId, $"{leaderboardId}:{periodKey}", placeholders);
   else if (prev != null && prev.Rank <= 10 && newRank > 10)
       await _triggerService.PublishAsync("TRG-10", userId, $"{leaderboardId}:{periodKey}", placeholders);
   await _snapshotRepo.InsertAsync(new LeaderboardRankSnapshot { LeaderboardId = leaderboardId, UserId = userId, Rank = newRank, SnapshotAt = Clock.Now });
   ```
   `if/else if` loại trừ lẫn nhau tự nhiên đáp ứng đúng rule "không bắn đồng thời cả hai". Phạm vi `LeaderboardId` cụ thể (theo lớp/khối/trường) để mở, xem §9.
5. **Học sinh nhận và đọc** — xem Luồng C.

### 5.3 Luồng C — Học sinh nhận thông báo

1. **Báo ngay** — SignalR/Abp realtime cập nhật badge chưa đọc (bao gồm tab "Tất cả"); banner/toast riêng (subscribe cùng event, tự ẩn sau 5-6 giây, xếp chồng theo chiều dọc nếu có nhiều banner mới, tách state khỏi dropdown) nếu app đang mở. Banner chỉ hiện tại đúng thời điểm phát sinh — mở lại app sau đó không hiện lại banner, nhưng badge/danh sách vẫn phản ánh đúng.
2. **Mở xem** — dropdown xem nhanh (4 tab: Tất cả / Tin TAK12 / Học tập / Đơn hàng, fetch trang đầu khi mở) hoặc trang "Tất cả thông báo" (layout master-detail, phân trang theo cursor).
3. **Đọc** — mark-as-read khi mở (popup hoặc khung chi tiết); lọc theo nhóm hoặc tìm theo từ khoá (chỉ áp dụng ở trang "Tất cả thông báo"). Khi đổi bộ lọc mà thông báo đang xem không còn thuộc nhóm đang chọn, tự chọn lại thông báo đầu danh sách mới.

### 5.4 Retention window (US-01)

Feed phía học sinh chỉ trả về thông báo trong N ngày gần đây (mặc định 90, để trong config chứ không hard-code); **không** áp dụng cutoff này cho query thống kê của MKT (US-05) — lịch sử gửi luôn xem được đầy đủ.

```csharp
// GetFeedAsync (học sinh) — có cutoff
var cutoff = Clock.Now.AddDays(-_settings.NotificationRetentionDays);
query = query.Where(x => x.CreationTime >= cutoff);

// GetCampaignHistoryAsync (MKT, US-05) — không cutoff
```


## 8. Recurring jobs (DB-driven)

Theo convention hiện có của repo:

1. Thêm key vào `CTH.QuizSystem.Core/BackgroundJobs/RecurringJobScheduleKeys.cs`.
2. Thêm case trong `CTH.QuizSystem.Application/BackgroundJobs/RecurringJobRegistrar.cs` (switch `ApplyRecurringSchedule`).
3. Seed một dòng vào `dbo.RecurringJobSchedules` qua SQL script trong `Migrations/Scripts/` (theo mẫu `20260916_NotificationUpdate.sql`).
4. Job class là `ITransientDependency` thuần với `ExecuteAsync()` — **không** dùng `IAsyncBackgroundJob<T>` (pattern đó dành cho job enqueue qua Hangfire như `AchievementApplyBatchJob`).

Jobs cần thiết:
- `NotificationCampaignSchedulerJob` — quét campaign Manual đến hạn gửi, enqueue broadcast.
- `NotificationStatsRefreshJob` — cập nhật `ReadCount`/`ClickCount` denormalized định kỳ (từ `NotificationClickLog`).
- `AssignmentOverdueCheckJob` — quét bài đã hết hạn chưa hoàn thành, bắn TRG-02.
- `LeaderboardRankUpdateJob` (đã tồn tại theo logic BXH hiện có, cần bổ sung bước) — sau khi tính rank mới, ghi `LeaderboardRankSnapshot` và so với snapshot trước để bắn TRG-09/TRG-10.

---

## 9. Open questions

- Số ngày retention hiển thị phía học sinh — đề xuất mặc định 90 ngày, cần chốt.
- Giới hạn số dòng / kích thước file khi MKT import danh sách.
- Ngưỡng thời gian cho việc gộp dedupe (cùng `GroupKey`, chưa đọc) — gộp vô thời hạn cho đến khi đọc, hay giới hạn cửa sổ thời gian (ví dụ 24h)?
- MKT có cần override `Group` hiển thị cho từng campaign Manual (khác "Tin TAK12") trong tương lai không, hay giữ cố định vĩnh viễn theo rule hiện tại?
- Số lần nhắc lại tối đa cho TRG-02 nếu học sinh vẫn chưa làm bài sau khi đã nhắc.
- Phạm vi cụ thể của `LeaderboardId` cho TRG-09/TRG-10 — theo lớp, khối, trường, hay toàn hệ thống; có nhiều BXH của học sinh cùng lúc không.
