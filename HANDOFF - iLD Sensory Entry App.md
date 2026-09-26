# HANDOFF — iLD Sensory Entry App

> Tài liệu bàn giao để tiếp tục phát triển ứng dụng nhập và tổng hợp kết quả nếm cảm quan của ILD Coffee Vietnam.
> Đọc toàn bộ file này trước, sau đó mở `index.html` để xem code.
>
> Cập nhật: 2026-09-23
> Chủ dự án: Đặng Thanh Bình — QA, ILD Coffee Vietnam

---

## 1. Trạng thái hiện tại

- File ứng dụng duy nhất: `C:\Apps\Sensory App\index.html`.
- HTML/CSS/JavaScript thuần trong một file, responsive cho PC và iPad.
- Dữ liệu hiển thị từ cache LocalStorage và đồng bộ lâu dài lên Firebase/Cloud Firestore v2.
- Sync v2 có outbox offline, revision/transaction chống ghi đè, conflict UI và khóa một tab chỉnh sửa.
- Đã bổ sung tab **Items Code** quản lý quan hệ `Nhóm · Items Code · Tên sản phẩm · Recipe`.
- Tab Items Code hỗ trợ thêm/xóa/tìm kiếm, tải template, nhập `.xlsx/.xls/.csv`, xuất `.xls`.
- Firebase Authentication đã bật Email/Password và Google Sign-In.
- Authorized domain: `ngocthanhthien.github.io`.
- Firestore Rules đã Publish, chỉ email `binhdangthanh94@gmail.com` được đọc/ghi document dùng chung.
- Bản local đã có Firebase. Tại thời điểm viết handoff, GitHub Pages vẫn chạy HTML cũ, chưa có khối đồng bộ Firebase. Cần thay `index.html` trên repository và chờ Pages deploy.

URL:

- App public: `https://ngocthanhthien.github.io/Sensory-App/`
- Firebase project: `sensory-app-72a44`
- Firebase Console: `https://console.firebase.google.com/u/1/project/sensory-app-72a44/overview`

---

## 2. Mục tiêu nghiệp vụ

Số hóa quy trình đánh giá cảm quan In/Just In/Out của nhà máy, thay việc nhập và tổng hợp thủ công từ:

- `QA.F.071`: phiếu chấm cá nhân của panellist.
- `QA.F.072`: bảng tổng hợp kết quả và phần trăm IN.

App cần giảm sai sót tính toán, lưu hồ sơ QA, hỗ trợ tra cứu và hoạt động khi mất mạng. Firebase là lớp đồng bộ nhiều thiết bị; LocalStorage vẫn là bản dự phòng offline.

---

## 3. Nguyên tắc không được phá vỡ

1. Giữ app dạng **một file `index.html`** trừ khi chủ dự án yêu cầu đổi kiến trúc.
2. Không làm mất dữ liệu LocalStorage với key `ild_sensory_v1`.
3. Nếu đổi data model, phải có migration tương thích ngược.
4. Không đổi công thức QA ở mục 6 nếu chưa được phê duyệt nghiệp vụ.
5. Không đưa mật khẩu, service-account private key hoặc token Firebase vào HTML.
6. Firebase Web API key trong HTML là định danh client, không thay thế Security Rules.
7. Tính năng mới phải hoạt động hợp lý khi offline; lỗi tải Firebase SDK không được làm hỏng phần local.
8. Trước thao tác thay thế dữ liệu (restore JSON/cloud pull/reset), phải có xác nhận.

---

## 4. Các màn hình hiện có

App có 10 tab:

```text
Tác nghiệp
  1. Chấm điểm nếm — QA.F.071
  2. Tạo Set nếm — wizard 2 bước
  3. Tổng hợp & Báo cáo — QA.F.072

Tra cứu
  4. Truy xuất kết quả

Master Data
  5. Quản lý Panellist
  6. SP · Recipe · PO · Batch
  7. Items Code

Kiến thức
  8. Sensory Guide

Hệ thống
  9. Hướng dẫn sử dụng
 10. Cài đặt
```

Tab **Cài đặt** gồm:

- Trạng thái và thao tác đăng nhập Google.
- Đẩy dữ liệu lên Firebase.
- Tải dữ liệu từ Firebase.
- Bật/tắt tự động đồng bộ realtime.
- Sao lưu/phục hồi JSON.
- Audit log.
- Reset toàn bộ dữ liệu.

---

## 5. Data model

Object toàn cục `STATE`:

```js
{
  panellists: [{ id, code, name, func, dept }],   // code = Employee code, func = Function (tùy chọn)
  products:   [{ id, name }],
  recipes:    [{ id, name }],
  pos:        [{ id, name }],
  batches:    [{ id, name }],
  itemCodes:  [{ id, group, item, name, recipe }],
  sets: [{
    id, name, date, shift, sampleType, creator, status,
    samples: [{ no, code, name, sscc, batch }],   // code = Tên mã hóa (mẫu mù); thiếu → dùng no
    scores: {
      [panellistId]: {
        [sampleNo]: { result, appearance: [], taste: [], note }
      }
    },
    attendees: [panellistId]
  }],
  currentSetId,
  currentPanellistId,
  audit: [{ t, role, action }]
}
```

Quy ước:

- `shift`: `AM | PM`; `status`: `open | closed`; `result`: `IN | JUSTIN | OUT`.
- `currentPanellistId` chỉ phục vụ phiên thao tác; không nằm trong payload backup/cloud hiện tại.
- Phần trăm kết quả không lưu cứng. `calcSample(set, no)` luôn tính lại từ `scores`.
- `itemCodes` là dữ liệu liên đới độc lập; chưa tự động ràng buộc với `products` hoặc mẫu trong Set.

---

## 6. Công thức QA bất biến

```text
Điểm: In = 0 · Just In = 0.5 · Out = 3

Defect  = (Out × 3 + Just In × 0.5) / (Total × 3)
Result% = (1 − Defect) × 100

Passed  khi Result% ≥ 80
Failed  khi Result% < 80
Cảnh báo khi Total < 3 panellist
```

Hằng số trong `CONFIG`:

```js
PASS_THRESHOLD: 80
SCORE: { OUT: 3, JUSTIN: 0.5, IN: 0 }
MIN_PANELLIST: 3
MAX_IMPORT_MB: 3
```

---

## 7. Items Code và Excel

### Cấu trúc chuẩn

| Group | Items Code | Product Name | Recipe |
|---|---|---|---|
| FG | 10100001 | Freeze Dried Instant Coffee PCAF561B1 25kg | 405 |

### Chức năng

- `renderItems()` hiển thị form nhập, tìm kiếm và bảng dữ liệu.
- `addItem()` thêm liên kết; `delItem(id)` xóa sau xác nhận.
- `downloadItemsTemplate()` xuất `iLD_ItemsCode_Template.xls`.
- `exportItemsExcel()` xuất toàn bộ dữ liệu thành `.xls` dạng SpreadsheetML.
- Import đọc `.csv`, SpreadsheetML `.xls`, hoặc OOXML `.xlsx`.
- `normalizeItemRows()` nhận cấu trúc bốn cột chuẩn và định dạng hai block `Item/Name/Recipe` từ file nguồn.
- Dòng trùng hoàn toàn được bỏ qua bằng `itemExactExists()`.

File nguồn nghiệp vụ ban đầu:

`C:\Users\BinhDang\Desktop\Items Code.xlsx`

Không giả định file này luôn có trên máy khác. Sau khi import, nguồn chính là `STATE.itemCodes`, backup JSON và Firestore.

### Giới hạn kỹ thuật

- Không dùng SheetJS; parser XLSX dùng `DecompressionStream` và XML parser.
- Trình duyệt cũ thiếu `DecompressionStream` có thể không đọc được XLSX nén.
- `.xls` xuất ra là SpreadsheetML 2003, không phải binary BIFF.
- Các màn hình khác vẫn xuất CSV có BOM UTF-8 dù nút ghi “Xuất Excel”.

---

## 8. LocalStorage, backup và audit

```text
ild_sensory_v1             dữ liệu chính
ild_sensory_theme          giao diện sáng/tối
ild_sensory_cloud_auto     bật/tắt tự đồng bộ
ild_sensory_cloud_client   ID thiết bị để tránh nhận lại chính bản ghi vừa gửi
ild_sensory_cloud_sync_v2  bases, revision, outbox và conflict đang chờ xử lý
```

- `save()` là alias của `persist()`.
- `persist()` ghi LocalStorage trước, rồi gọi `cloudQueuePush()` nếu auto-sync bật.
- `persist(true)` chỉ lưu local, không xếp hàng push cloud.
- Audit log giữ tối đa 500 dòng.
- Backup JSON gồm Master Data, Items Code, Sets và audit.

---

## 9. Firebase / Firestore

### Project và client

```text
projectId: sensory-app-72a44
authDomain: sensory-app-72a44.firebaseapp.com
Web App: Sensory App Web
Firebase JS SDK: 12.19.0, import động từ gstatic
Allowed account: binhdangthanh94@gmail.com
```

Nếu email khác đăng nhập, client tự sign out. Đây chỉ là lớp UI; quyền thật do Firestore Rules bảo vệ.

### Vị trí dữ liệu

```text
Database: (default)
Location: asia-southeast3 (Bangkok)
Legacy document: ild_sensory/shared_state
Meta document: ild_sensory_v2/meta
Sets: ild_sensory_sets/{setId}
Scores: ild_sensory_scores/{setId}__{panellistId}
Schema version: 2
```

Document legacy được giữ tạm để migration/rollback. Bản v2 tách Meta, Set và Score để một thay đổi nhỏ không ghi lại toàn bộ STATE và không còn phụ thuộc giới hạn 1 MiB của một snapshot lớn.

### Firestore Rules đang Publish

```rules
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {
    function isSensoryOwner() {
      return request.auth != null
        && request.auth.token.email_verified == true
        && request.auth.token.email == 'binhdangthanh94@gmail.com';
    }
    match /ild_sensory/shared_state { allow read, write: if isSensoryOwner(); }
    match /ild_sensory_v2/{id} { allow read, write: if isSensoryOwner(); }
    match /ild_sensory_sets/{id} { allow read, write: if isSensoryOwner(); }
    match /ild_sensory_scores/{id} { allow read, write: if isSensoryOwner(); }
  }
}
```

Mọi document khác mặc định không được cấp quyền.

### Luồng đồng bộ

- `firebaseInit()` tải SDK và theo dõi Auth state.
- `cloudSignIn()` dùng Google popup với email gợi ý.
- `cloudPush(true)` tạo entity diff rồi gửi tuần tự từ outbox.
- Mỗi entity được ghi bằng Firestore transaction, kiểm tra `baseRevision` và tăng `revision`.
- Revision lệch tạo conflict; không tự động chọn bên thắng.
- `cloudPull()` ưu tiên schema v2 và tự đọc legacy nếu v2 chưa có.
- Listener theo dõi riêng Meta, Sets và Scores; thay đổi được hoãn nếu người dùng đang nhập.
- Web Locks API chỉ cho một tab làm tab chỉnh sửa chính.
- Retry dùng exponential backoff; outbox không bị xóa khi lỗi mạng.
- LocalStorage luôn được giữ làm fallback.

Giới hạn còn lại: audit vẫn do client tạo. Muốn audit bất biến cần Callable Cloud Function và thường phải dùng gói Blaze.

---

## 10. Cấu trúc code

Các vùng trong `index.html`:

```text
[STYLE] [PRINT] [CONFIG] [STATE] [STORAGE] [UTIL] [AUDIT]
[THEME] [NAV] [SCORING] [SETUP] [SUMMARY] [LOOKUP]
[MASTER] [ITEMS] [IMPEXP] [SENSORY] [GUIDE]
[CLOUD] [SETTINGS] [SEED] [BOOT]
```

Các hàm render:

```text
renderScoring · renderSetup · renderSummary · renderLookup
renderPanellists · renderProducts · renderItems · renderSensory
renderGuide · renderSettings
```

Không có framework và không có bước build.

---

## 11. Triển khai GitHub Pages

Repository public phục vụ tại `https://ngocthanhthien.github.io/Sensory-App/`.

1. Đưa `C:\Apps\Sensory App\index.html` lên root repository `Sensory-App`, thay file cũ.
2. Commit/push lên branch GitHub Pages đang dùng.
3. Chờ Pages deploy rồi hard refresh.
4. Vào **Cài đặt**; nếu chưa thấy card **Đồng bộ Firebase**, site vẫn cache hoặc chưa deploy bản mới.
5. Đăng nhập Google bằng `binhdangthanh94@gmail.com`.
6. Trên thiết bị đang có dữ liệu chuẩn, chọn **Đẩy dữ liệu lên Firebase** đúng một lần.
7. Trên thiết bị khác, chọn **Tải dữ liệu từ Firebase**, sau đó có thể bật tự động đồng bộ.

Không push nhầm dữ liệu demo từ một trình duyệt mới trước khi tải/khôi phục dữ liệu thật.

---

## 12. Kiểm thử bắt buộc sau thay đổi

### Cú pháp và offline

- Tách nội dung `<script>` ra file tạm và chạy `node --check`.
- Không có lỗi console làm hỏng app khi Firebase SDK tải thất bại hoặc thiết bị offline.

### Nghiệp vụ

- Tạo Set; kiểm tra chặn trùng tên trong cùng ngày/ca.
- Chấm đủ mẫu; Just In/Out phải có lỗi; nộp phiếu.
- Dữ liệu demo mẫu #3 phải cho 66,7% và Failed.
- Xuất CSV/PDF, backup JSON và restore.

### Items Code

- Tải template và mở bằng Excel.
- Import `.xlsx`, `.xls`, `.csv` có tiếng Việt.
- Dòng thiếu Item/Name/Recipe bị bỏ qua.
- Import lại cùng file không nhân đôi dòng trùng.
- Tìm theo Group, Items Code, Product Name và Recipe.
- Xuất dữ liệu rồi import lại vào STATE trống.

### Firebase

- Từ GitHub Pages, đăng nhập đúng email thành công.
- Email khác bị sign out và không đọc/ghi được Firestore.
- Push và kiểm tra `ild_sensory_v2/meta`, `ild_sensory_sets` và `ild_sensory_scores`.
- Mở hai thiết bị, sửa cùng entity từ cùng revision và kiểm tra conflict được giữ lại để người dùng quyết định.
- Thử offline, tạo thay đổi, kiểm tra outbox còn nguyên và tự gửi khi online.
- Pull trên thiết bị khác; so số Set, panellist và Items Code.
- Bật auto-sync, sửa dữ liệu nhỏ, xác nhận thiết bị còn lại nhận được.
- Thử offline: phần local vẫn dùng được và không mất dữ liệu.

---

## 13. Backlog ưu tiên

1. Publish `firestore.rules`, deploy `index.html`, tải legacy rồi push lần đầu để tạo schema v2.
2. Theo dõi migration và chỉ xóa `ild_sensory/shared_state` sau khi backup/đối chiếu hoàn tất.
3. Bổ sung Firebase App Check cho domain GitHub Pages.
4. Chuyển audit quan trọng sang Callable Cloud Function nếu nâng lên Blaze.
5. Liên kết Items Code trực tiếp với Product/Recipe và mẫu trong Set.
6. Xuất `.xlsx` thật cho mọi báo cáo nếu chấp nhận thêm thư viện hoặc tự viết workbook đầy đủ.
7. Xuất PDF giống form QA.F.072, có ô kiểm tra và chữ ký.
8. Thang 1–5 riêng cho Green Bean nếu nghiệp vụ phê duyệt.
9. Mẫu Spike ẩn và thống kê năng lực panellist.
10. Nếu mở cho nhiều tài khoản, đổi Rules và thiết kế role; không chỉ sửa `CLOUD_ALLOWED_EMAIL`.

---

## 14. Prompt bàn giao gợi ý

> Tôi có app một file `index.html` để nhập và tổng hợp kết quả nếm cảm quan tại ILD Coffee Vietnam. Hãy đọc `HANDOFF - iLD Sensory Entry App.md` trước khi sửa. Giữ tương thích LocalStorage key `ild_sensory_v1`, công thức Result%, tab Items Code và luồng Firebase hiện có. Không làm mất dữ liệu local/cloud. Việc cần làm: [mô tả yêu cầu]. Sau khi sửa, kiểm tra cú pháp JavaScript, Items Code, backup/restore và Firebase offline fallback.
