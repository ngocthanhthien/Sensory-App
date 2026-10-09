# HANDOFF — iLD Sensory Entry App

> Tài liệu bàn giao để tiếp tục phát triển app nhập & tổng hợp kết quả nếm cảm quan của **ILD Coffee Vietnam**.
> Dành cho người/mô hình AI tiếp nhận: đọc hết file này trước, rồi mở `index.html`.
>
> Cập nhật: **2026-10-06** · Chủ dự án: **Đặng Thanh Bình** — QA, ILD Coffee Vietnam

---

## 0. Tóm tắt nhanh (đọc trước)

| Mục | Giá trị |
|---|---|
| Kiến trúc | **1 file `index.html`** — HTML/CSS/JS thuần, không framework, không bước build |
| Repo | `C:\Users\BinhDang\Documents\GitHub\Sensory-App` (branch `main`) |
| App public | `https://ngocthanhthien.github.io/Sensory-App/` (GitHub Pages, deploy từ `main`) |
| Backend | Firebase project `sensory-app-72a44` (gói **Spark**): Auth + Cloud Firestore `(default)`, `asia-southeast3` |
| Firebase Console | `https://console.firebase.google.com/u/1/project/sensory-app-72a44/overview` (tài khoản `binhdangthanh94@gmail.com`) |
| Dữ liệu offline | LocalStorage key `ild_sensory_v1` (cache + dự phòng) |
| File trong repo | `index.html` · `firestore.rules` · file này · `.gitattributes` — không có file nào khác |

Biểu mẫu nghiệp vụ được số hóa: **QA.F.071** (phiếu chấm cá nhân) và **QA.F.072 v4** (Summary Tasting Result).

---

## 1. Nguyên tắc không được phá vỡ

1. Giữ app **một file `index.html`** (trừ khi chủ dự án yêu cầu khác).
2. Không làm mất/đổi nghĩa dữ liệu LocalStorage `ild_sensory_v1`; đổi data model phải có **migration tương thích ngược**.
3. **Không đổi công thức QA** (mục 5) khi chưa được duyệt nghiệp vụ.
4. Không đưa mật khẩu, service-account key, token vào HTML. Firebase Web API key trong HTML là định danh client — bảo mật thật nằm ở **Firestore Rules** + giới hạn tên miền của API key.
5. Tính năng mới phải chạy được khi offline; lỗi tải Firebase SDK không được làm hỏng phần local.
6. Thao tác thay thế/xóa dữ liệu (Reset, Phục hồi JSON, Tải đè từ Firebase, Xóa Set/phiếu) phải có xác nhận và chỉ Admin.
7. Ngày **lưu** dạng ISO `yyyy-mm-dd`, **hiển thị** `dd/mm/yyyy` (`fmtDate`, `fmtDateTime`, `parseDate`, ô nhập `dateField`).
8. Giao diện theo nhận diện **ILD Crafted** (mục 10); logo gốc luôn trên nền trắng, không đổi màu/invert.
9. Sau mỗi thay đổi: chạy kiểm thử ở mục 12 (ít nhất `node --check` + chạy thử trên trình duyệt).

---

## 2. Các tab & ai thấy

| Tab (`data-tab`) | Nội dung | Ai thấy |
|---|---|---|
| ✍️ Chấm nếm (`tab-scoring`) | Phiếu QA.F.071: chọn **người nếm trước** (ô tìm kiếm; tên được giữ khi đổi Set/ngày/tab, sau khi nộp và khi tải lại trang — nhớ ở `sessionStorage` khóa `ild_sensory_sc_person`, xoá khi Đăng xuất) → lọc **Ngày nếm** → Set; **All IN** nằm ở cuối phiếu (`.sc-head-tools`, sau phần Hướng dẫn); cụm **◀ Set n ▶** (`.sc-nav-group`) trên PC nằm cạnh All IN, còn ≤ 1180px nằm ở dòng dưới của thanh Discard / Nộp phiếu dính đáy (`#scBar`, con trực tiếp của `#scoringBody` để sticky bám trọn chiều cao); dòng cuối của thanh ghi nhỏ tên người nếm đang chấm (`.sc-bar-who`); ◀ ▶ chuyển nhanh giữa các Set đang mở cùng ngày + ca (`scNav`, giữ người nếm sau khi nộp); IN / Just In / Out, Remark, **All IN**, **Discard**, **Nộp phiếu** (thanh dính đáy trên tablet/phone) | Mọi người (User chung **chỉ** thấy tab này) |
| 🧪 Tạo Set (`tab-setup`) | Wizard 2 bước; Bước ② thêm mẫu tay / Excel / **Nhập Excel từ SAP**; Tên mã hóa (mẫu mù) | Quyền `createSet` |
| 📊 Báo cáo (`tab-summary`) | Ô chọn Set có lọc trước theo **Loại mẫu** và **Ngày** (`SUMF`, `sumFilter`); phiếu QA.F.072 theo Set, KPI, **Xác nhận kết quả** (Giám sát), Xuất Excel (CSV) / PDF | Tài khoản riêng |
| 🔍 Danh sách Set (`tab-lookup`, tên cũ: Tra cứu) | Sắp xếp + lọc theo từng cột (ô lọc ngay dưới tên cột; mặc định ngày gần nhất lên đầu — `LK`, `LK_COLS`, `lkList`, `lkPaint`): số người nếm, Passed/Failed, trạng thái, Giám sát; ✎ Sửa · 🔒 Đóng Set · Xóa | Tài khoản riêng (nút theo quyền) |
| 🗂️ Dữ liệu nộp phiếu (`tab-submissions`) | 1 dòng = 1 mẫu trong 1 phiếu; lọc Set/kết quả/ngày; xuất `.xls`; Sửa/Xóa dòng/Xóa phiếu | Tài khoản riêng (sửa/xóa theo `editSubmission`) |
| 👥 Panelist (`tab-panellist`) | Thêm/xóa, Nhập Excel (Employee code · Full name · Function · Dept.), **Xuất Excel** | Tài khoản riêng (sửa theo `editMaster`) |
| 📦 SP & Lô (`tab-product`) | Bảng liên tục Sản phẩm · Recipe · PO · Batch (`STATE.lots`), lọc/sắp xếp, nhập/xuất Excel | như trên |
| 🔗 Items Code (`tab-items`) | Loại (FGs/RW) · Item Code · Tên SP · Recipe (~3.100 dòng), lọc/sắp xếp/sửa tại ô, nhập/xuất | như trên |
| ☕ Sensory · 📖 Hướng dẫn | Kiến thức cảm quan; hướng dẫn trong app theo 4 bước Tạo Set → Chấm nếm → Đóng Set → Xác nhận (`renderGuide`, cập nhật 07/10/2026 — sửa khi thêm tính năng) | Tài khoản riêng |
| ⚙️ Cài đặt (`tab-settings`) | Trạng thái đồng bộ, Đẩy/Tải Firebase, auto-sync, Sao lưu/Phục hồi JSON, audit log, Reset | Tài khoản riêng (thao tác thay thế dữ liệu: Admin) |
| 🛡️ Người dùng (`tab-users`) | **Ma trận phân quyền**, mật khẩu User chung, tạo/sửa/khóa/xóa tài khoản | Admin |

Header (phải): Set đang mở · **nút đồng bộ** (✓ Đã đồng bộ / ↻ Cập nhật — bấm để gửi + nhận ngay, `cloudSyncNow`) · nút tài khoản (đăng nhập cấp 2) · 🚪 Đăng xuất · 🎨 Giao diện · 🌓 (ẩn trên điện thoại).

---

## 3. Đăng nhập, vai trò, phân quyền

### 3.1 Đăng nhập 2 cấp (Firebase Auth Email/Password + Google)
- **Cấp 1 — màn hình khóa:** chỉ hỏi **mật khẩu User chung** → đăng nhập tài khoản `staff@ild-sensory.local` (Admin tạo/đổi ở tab Người dùng, ≥ 6 ký tự). Mật khẩu **không** nằm trong mã nguồn.
- **Cấp 2 — nút tài khoản góc phải:** tài khoản riêng `tên-đăng-nhập` → email nội bộ `<tên>@ild-sensory.local`; hoặc **Google** của chủ dự án.
- **Chủ dự án** `binhdangthanh94@gmail.com` (Google, email verified) **luôn là Admin** (không cần hồ sơ).
- **Đăng xuất (🚪):** gửi nốt outbox lên Firebase → signOut → xóa mọi key `ild_sensory*` (trừ `ild_sensory_theme`), sessionStorage, Cache Storage, IndexedDB `firebaseLocalStorageDb` → reload. Cảnh báo nếu còn thay đổi chưa gửi/offline.
- Offline: hồ sơ đăng nhập gần nhất cache ở `ild_sensory_auth_cache` → vẫn dùng app khi không tải được Firebase.

### 3.2 Hồ sơ & vai trò — `ild_sensory_users/{uid}`
```js
{ username, email, displayName, role: 'user'|'qc'|'supervisor'|'admin', active: bool, shared: bool, createdAt, createdBy, updatedAt?, updatedBy? }
```
- Tài khoản hiện có (06/10/2026): `binh.dang` (admin), `qaline` (qc), `minh.thai` (supervisor), `staff` (user, **shared**). Trong Authentication còn `sensory@ild-sensory.local` **không có hồ sơ** (không vào được app) — có thể xóa ở Console.
- Tạo/đổi mật khẩu tài khoản khác dùng **Firebase app phụ (in-memory)** để không đăng xuất Admin. Gói Spark không xóa/đặt lại mật khẩu người khác được từ client → "Xóa" chỉ gỡ hồ sơ quyền; quên mật khẩu → xóa & tạo lại, hoặc đặt lại ở Console.

### 3.3 Ma trận phân quyền — `ild_sensory_v2/permissions`
```js
{ matrix: { user:{...}, qc:{...}, supervisor:{...} }, updatedAt, updatedBy }   // admin luôn toàn quyền
```
| Quyền (`perm`) | Mặc định |
|---|---|
| `createSet` Tạo Set | user ✔ · qc ✔ · supervisor ✔ |
| `editSet` Sửa Set | qc ✔ |
| `closeSet` Đóng Set | qc ✔ |
| `confirmSet` Xác nhận kết quả | supervisor ✔ |
| `deleteSet` Xóa Set | (chỉ admin) |
| `editSubmission` Sửa/xóa phiếu | (chỉ admin) |
| `editMaster` Sửa Master Data | user ✔ · qc ✔ · supervisor ✔ |

- Client: `can(perm)`; phần tử UI có class `need-<perm>` tự ẩn (CSS sinh động). Lưu cache `ild_sensory_perms`; đọc realtime (`permWatch`). Admin lưu bằng `permSave()`.
- **User chung (`shared: true`)**: `can()` luôn `false`, chỉ thấy tab Chấm nếm (`isSharedUser()`, body class `is-shared`).
- Cố định (không nằm trong ma trận): chấm & nộp phiếu, xem báo cáo (mọi tài khoản riêng); quản lý người dùng, Reset/Phục hồi/Tải đè (Admin).

### 3.4 Firestore Rules (`firestore.rules`)
- `isOwner`, `activeProfile`, `isMember`, `isAdmin`, `isShared()`, `can(perm)` đọc **cùng ma trận** `ild_sensory_v2/permissions`.
- Set: tạo `can('createSet')`; cập nhật: thành viên chỉ đổi `attendees` (+ khóa đồng bộ) · `can('confirmSet')` đổi `approval/status` · `can('closeSet')` đổi `status` · `can('editSet')` sửa khác (Set đã xác nhận chỉ Admin); xóa `can('deleteSet')`.
- Phiếu `ild_sensory_scores/{setId}__{panellistId}`: thành viên ghi khi Set chưa xác nhận; xóa `can('editSubmission')`.
- `ild_sensory_v2/log` và `ild_sensory_v2/tombstones` dùng chung luật `ild_sensory_v2/{documentId}` (thành viên đọc/ghi) → **không cần đổi/Publish Rules** cho thay đổi 07/10/2026.
- `ild_sensory_v2/meta`: đổi trường Master Data (`panellists, products, recipes, pos, batches, lots, itemCodes`) cần `can('editMaster')`.
- `ild_sensory_users`: tự đọc hồ sơ mình; Admin list/tạo/sửa/xóa; `role ∈ {user,qc,supervisor,admin}`.
- **Trạng thái Publish:** bản có ma trận phân quyền đã Publish **28/09/2026 16:18**. Bản trong repo có thêm `isShared()` (chặn tài khoản User chung ở server) — **cần kiểm tra đã Publish chưa** (Firestore → Rules → so nội dung với file).
- Ghi bị Rules từ chối (`permission-denied`) → app bỏ mục đó khỏi outbox, báo người dùng và tải lại bản Firebase (không kẹt hàng chờ).

### 3.5 Cấu hình Firebase/Google Cloud (đã kiểm tra 28/09/2026)
- Auth providers: Email/Password ✔, Google ✔; Enable create (sign-up) ✔; Email enumeration protection ✔.
- Authorized domains: `localhost`, `sensory-app-72a44.firebaseapp.com`, `sensory-app-72a44.web.app`, `ngocthanhthien.github.io`.
- **API key "Browser key (auto created by Firebase)"** giới hạn HTTP referrer: `https://ngocthanhthien.github.io/*`, `https://sensory-app-72a44.firebaseapp.com/*` (cái sau bắt buộc cho popup Google). ⇒ **chạy từ `localhost` không đăng nhập được** — thử Firebase thật trên GitHub Pages.

---

## 4. Data model (`STATE`, lưu ở `ild_sensory_v1`)

```js
{
  panellists: [{ id, code, name, func, dept }],          // code = Mã NV (Employee code); id chuẩn 'pnl_<mã nv thường>'
  products, recipes, pos, batches: [{ id, name }],         // legacy — UI dùng lots (giữ để tương thích)
  lots:      [{ id, product, recipe, po, batch }],         // bảng liên tục SP·Recipe·PO·Batch (migrateLots)
  itemCodes: [{ id, group, item, name, recipe }],          // group: FGs | RW
  sets: [{
    id, name, date /*yyyy-mm-dd*/, shift /*Ca 1|Ca 2|Ca 3 (`CONFIG.SHIFTS`); Set cũ: AM|PM*/, sampleType, creator, status /*open|closed*/,
    samples: [{ no, code /*Tên mã hóa – mẫu mù*/, name, sscc, batch }],
    scores: { [panellistId]: { [sampleNo]: { result /*IN|JUSTIN|OUT*/, appearance:[], taste:[], note, at /*ISO nộp*/, editedAt?, editedBy? } } },
    attendees: [panellistId],
    approval?: { by, byUid, at, panellists, passed, total }   // Giám sát xác nhận → Set khóa
  }],
  currentSetId /*đồng bộ qua document log*/, currentPanellistId /*chỉ phiên*/, audit: [{ t /*dd/mm/yyyy HH:mm:ss*/, role /*tên người*/, action }]  // tối đa 500
}
```
- `sampleCode(sm)` = `code` hoặc `no` (Set cũ chưa có mã).
- **Semi product:** khi sang Bước ② app nạp sẵn `CONFIG.SEMI_DEFAULT_SAMPLES` (City Water, RO Water, B27960, B27970, Foaming, Aroma, B26600) — người dùng điền SSCC/Hour + Batch, thêm/xoá được; nút "↺ Nạp mẫu mặc định" thêm lại mẫu còn thiếu (`wizSemiDefaults`).
- `SAMPLE_TYPES`: Green coffee · Freeze-dried · Semi product · KQ · Complaint · Other (mặc định Freeze-dried). **Green coffee: phiếu chỉ IN / OUT** (`isGreenCoffee`). Trên QA.F.072 (màn hình/PDF/Excel) cột Result của Green coffee ghi **IN / OUT** thay cho % (`sumResultText`: IN khi mẫu đạt ngưỡng 80% theo `calcSample`, OUT khi không đạt) — công thức không đổi, chỉ đổi cách hiển thị.
- Lỗi: `ERR_APPEARANCE` (màu sậm, hai màu, lumpy, hạt cháy, bụi, bề mặt bị rổ, vụn) · `ERR_TASTE` (chemical, overdried, less/more aroma, chua, đắng, cook). Just In/Out **bắt buộc** chọn lỗi hoặc ghi chú; IN được ghi chú (không bắt buộc).
- 49 panelist chuẩn nhúng sẵn `DEFAULT_PANELLISTS` (nạp 1 lần/thiết bị, cờ `ild_sensory_pnl_default_v1`); 4 tên demo (`pnl_a..d`) tự bị loại.
- Không lưu cứng % — `calcSample(set,no)` luôn tính lại.

---

## 5. Công thức QA (bất biến)

```text
Điểm: In = 0 · Just In = 0.5 · Out = 3
Defect  = (Out×3 + JustIn×0.5) / (Total×3)
Result% = (1 − Defect) × 100      → Passed ≥ 80 · Failed < 80
CONFIG: PASS_THRESHOLD 80 · SCORE {OUT:3, JUSTIN:0.5, IN:0} · MIN_PANELLIST 3 · MAX_IMPORT_MB 3
```
- **Đóng Set chỉ khi đủ ≥ MIN_PANELLIST người nếm** (`closeBlockedMsg`) — áp dụng ở Danh sách Set, Sửa Set và Tạo Set. Giám sát xác nhận vẫn được (có cảnh báo) và tự đóng Set.

---

## 6. Quy trình nghiệp vụ

1. **Tạo Set** (QC/có `createSet`): Bước ① Ca · Ngày · Tên Set (tự gợi ý số, không trùng trong cùng ca/ngày) · Loại mẫu → Bước ② mẫu: gõ tay / template Excel / **Nhập Excel từ SAP** (cột theo **tên**: ITEMNAME→Tên mẫu, SSCC→SSCC/Hour, BATCH→Batch; Tên mã hóa tự đánh; nếu cột Selection có "Checked" chỉ lấy dòng đó; bỏ SSCC trùng); đánh mã lại theo tiền tố (CHO 1, CHO 2…).
2. **Chấm nếm** (mọi người): chọn Set + tìm người nếm (tên không dấu/Mã NV) → IN/Just In/Out (All IN) → Nộp phiếu. Ghi `at`. Set đã xác nhận thì không nộp được.
3. **Đóng Set** (QC): khi đủ ≥ 3 người.
4. **Xác nhận** (Giám sát): tab Báo cáo → ✔ Xác nhận kết quả → `approval` + đóng & khóa; tên + giờ hiện ở ô **Check by** của QA.F.072 (màn hình/PDF/Excel). Hủy xác nhận để mở khóa.
5. Admin: sửa/xóa phiếu (Dữ liệu nộp phiếu), sửa/xóa Set (Danh sách Set) — bị chặn khi Set đã xác nhận.

---

## 7. Đồng bộ Firebase (schema v2)

```text
ild_sensory_v2/meta          panellists, products, recipes, pos, batches, lots, itemCodes (~3.100), schemaVersion 2   (chỉ đổi khi sửa Master Data)
ild_sensory_v2/log           audit (≤ 500), currentSetId   — tách khỏi meta 07/10/2026; gửi thưa (5 phút, User chung 30 phút)
ild_sensory_v2/tombstones    items: { "<set|score>:<id>": thời điểm xoá }   — báo cho máy khác biết document đã bị xoá (giữ 90 ngày)
ild_sensory_v2/permissions   ma trận phân quyền
ild_sensory_sets/{setId}     Set (không gồm scores)
ild_sensory_scores/{setId}__{panellistId}   { setId, panellistId, scores }
ild_sensory_users/{uid}      hồ sơ & vai trò
ild_sensory/shared_state     legacy (giữ để rollback, chỉ Admin)
```
- Mỗi document có `revision, updatedAt, updatedBy, clientId`; ghi bằng transaction kiểm tra `baseRevision`.
- Lệch revision → **gộp 3 chiều** `cloudMerge3(base, local, remote)` (mảng theo `id`/`no`, audit theo nội dung; cùng sửa giá trị đơn → bản máy thắng); gộp lỗi/quá 5 lần → conflict UI ở Cài đặt.
- Không tự nhận dữ liệu cloud đè lên máy khi còn outbox/conflict, **đang gõ chữ trong 20 giây gần nhất**, đang mở hộp thoại hoặc **đang chấm dở phiếu** → `pendingRemote`, áp dụng sau. Ô chọn (select) đang focus **không** chặn (`cloudUiBusy`) — trước 07/10/2026 nó chặn, làm tablet không thấy Set mới.
- Chống chậm/kẹt trên tablet & điện thoại (07/10/2026): nhịp 5 giây `cloudTick` áp dụng dữ liệu đang chờ; `cloudWake` mở lại kênh nghe khi màn hình sáng lại sau ≥ 30 giây; listener có xử lý lỗi và tự bật lại (lỗi liên tục → `cloudReconcile` tối đa 5 phút/lần); User chung tự gỡ xung đột (`cloudAutoResolveShared`).
- **Giảm Egress/Reads (07/10/2026) — không được quay lại kiểu "tải lại tất cả":**
  - **Nhận theo từng document:** 3 listener (`ild_sensory_v2`, `ild_sensory_sets`, `ild_sensory_scores`) dùng `where('updatedAt','>', watermark − 2 phút)`; dữ liệu lấy thẳng từ snapshot → `cloudOnSnap` xếp vào `CLOUD.incoming` → `cloudApplyIncoming` gộp từng thực thể (máy chưa sửa: lấy bản mới; máy có thay đổi chưa gửi: `cloudMerge3` rồi gửi lại). Không còn `getDocs` toàn bộ sau mỗi thay đổi.
  - **`watermark`** (lưu trong `ild_sensory_cloud_sync_v2`) = `updatedAt` lớn nhất đã áp dụng → mở lại app / thức dậy / có mạng lại chỉ đọc các document đổi trong lúc vắng. Mở lại listener (`cloudStartListener`) chính là thao tác "đồng bộ lại nhẹ".
  - **Tải đủ (`cloudFetchV2`)** chỉ còn ở: máy mới đăng nhập (`cloudFirstSync`), đối soát định kỳ 14 ngày / nâng cấp từ bản cũ (`cloudReconcile`), Admin bấm "Tải dữ liệu từ Firebase".
  - **Xoá:** `cloudWriteEntity` ghi thêm `tombstones` trong cùng transaction; máy khác thấy qua listener `ild_sensory_v2`. Document bị xoá rồi tạo lại (revision quay về 1) được nhận nhờ so thời gian (`replace`).
  - **meta không còn bị ghi lại sau mỗi thao tác:** audit + `currentSetId` nằm ở `log` (thực thể `log:main`), gửi thưa qua `cloudQueueEntities(force)`; nộp phiếu chỉ ghi document phiếu + Set.
  - Bị Rules từ chối → `cloudRevertEntity` trả thực thể về bản đã đồng bộ (không tải lại).
  - Bản app cũ (chưa tải lại trang) vẫn ghi audit vào `meta` và xoá không có tombstone → máy mới bỏ qua `meta.audit`, và phát hiện document bị xoá ở lần đối soát. **Sau khi deploy cần tải lại trang trên mọi thiết bị.**
  - Còn lại: máy mới/đăng xuất rồi đăng nhập lại vẫn tải đủ 1 lần (meta ~vài trăm KB + mọi Set + mọi phiếu) — muốn giảm tiếp thì cho User chung chỉ tải Set đang mở, và tách Items Code khỏi `meta`.
- Kiểm thử đồng bộ không cần Firebase thật: `.claude/sync_test.js` (không nằm trong git) mô phỏng Firestore + kịch bản PC/Tablet, đếm số lần đọc; chạy `python -m http.server`, mở app rồi `eval` file và gọi `runSyncTest()`.
- `cloudEntities()` trả **bản sao sâu** (tránh base trỏ chung STATE). Thiết bị chưa từng đồng bộ: cloud có dữ liệu → tải về (máy mới) hoặc hỏi; cloud trống → đẩy lên. Auto-sync mặc định **bật**. Web Locks: 1 tab chỉnh sửa chính.
- Dữ liệu thật đã có trên Firestore (kiểm tra 28/09/2026: 49 panelist, ~3.100 Items Code, 3 Set, 7 phiếu, 4 tài khoản).
- ⚠ `meta` chứa ~3.100 Items Code — Firestore giới hạn **1 MiB/document**; nếu Items Code tăng nhiều, cần tách ra document/collection riêng.

### Khóa LocalStorage
```text
ild_sensory_v1             dữ liệu chính (STATE)
ild_sensory_cloud_sync_v2  bases, outbox, conflicts, watermark, fullAt, logAt
ild_sensory_cloud_auto     '0' = tắt auto-sync (mặc định bật)
ild_sensory_cloud_client   ID thiết bị
ild_sensory_editor_lock    khóa tab
ild_sensory_auth_cache     hồ sơ đăng nhập cho offline
ild_sensory_perms          cache ma trận phân quyền
ild_sensory_pnl_default_v1 đã nạp danh sách panelist chuẩn
ild_sensory_theme          sáng/tối cũ (nút 🌓)
ild-crafted-appearance-v1  Theme ILD Crafted {v:1, mode, contrast, emphasis}
```

---

## 8. Nhập / xuất

| Nơi | Nhập | Xuất |
|---|---|---|
| Panelist | `.xlsx/.xls/.csv` (Employee code · Full name · Function · Dept.) — trùng mã/tên thì cập nhật | `iLD_Panelist_<ngày>.xls` (cùng cột, nhập lại được) |
| Tạo Set Bước ② | Excel mẫu (Tên mã hóa · Tên mẫu · SSCC/Hour · Batch) · **SAP** | `iLD_SetSamples_….xls`, template |
| SP & Lô | Sản phẩm · Recipe · PO · Batch | `.xls` |
| Items Code | 4 cột chuẩn hoặc 2 khối FGs/RW | `.xls` |
| Danh sách Set | — | `iLD_DanhSachSet_<ngày>.xls` (theo bộ lọc + thứ tự đang hiển thị, `exportLookup`) |
| Dữ liệu nộp phiếu | — | `iLD_DuLieuNopPhieu_<ngày>.xls` |
| Báo cáo | — | CSV (UTF-8 BOM) · PDF (in A4 ngang, form QA.F.072) |
| Cài đặt | Phục hồi JSON (Admin) | Sao lưu JSON |

Không dùng thư viện ngoài: XLSX đọc bằng `DecompressionStream` + DOMParser (`xlsxRows`); `.xls` xuất là SpreadsheetML 2003 (`itemsWorkbook`) — Excel có thể hỏi "định dạng không khớp", chọn Yes. Logo nhúng base64 (`ILD_LOGO.dark`).

---

## 9. Cấu trúc code `index.html`

```text
<head>  [ILD-THEME early script] <style>[STYLE]…[PRINT]</style> <style id="ild-theme-style">[ILD CRAFTED]</style> <style id="scoring-touch-style">[SCORING · TABLET/PHONE]</style>
<body>  header · nav.side (tab) · main (section.panel × 12) · footer · #authGate · #acctOverlay · #ild-theme-overlay · #cfOverlay · #toast · input file ẩn
<script> [CONFIG] [STATE] [STORAGE] [UTIL] [AUDIT] [THEME] [NAV] [SCORING] [SETUP] [SUMMARY] [LOOKUP] [SUBMISSIONS]
         [MASTER] [ITEMS] [IMPEXP] [PRINT] [SENSORY] [GUIDE] [CLOUD] [AUTH] [SETTINGS] [SEED] [BOOT]
<script> [ILD-THEME] mô-đun Theme độc lập
```
Hàm chính: `render*` (Scoring, Setup, Summary, Lookup, Submissions, Panellists, Products, Items, Sensory, Guide, Settings, Users) · `switchTab` · `save()/persist()` · `logAudit` · `calcSample` · `summaryFormHTML` · `can/isAdmin/isSharedUser` · `authApply` · `cloud*` · `approveSet/unapproveSet` · `lookupEdit/lookupCloseSet/lookupDelete` · `subEdit/subDeleteRow/subDeleteSheet` · `scAllIn/scDiscard/scSubmit`.

---

## 10. Giao diện ILD Crafted, Theme & responsive

- Nhận diện: nền kem `#F7F0E6`, espresso `#382E28`, bề mặt trắng, đỏ ILD `#E30613` (điểm nhấn nhỏ); tối dùng hệ nâu ấm `#1F1A17`. Token `--c-*` ánh xạ sang biến cũ (`--bg/--panel/--line/...`).
- Theme (🎨 Giao diện): 4 cấu hình mẫu (Crafted tiêu chuẩn/dịu/tối/rõ nét) · Sáng/Tối/Theo hệ thống · Tương phản Tiêu chuẩn/Cao · Độ đậm Nhẹ/Tiêu chuẩn/Đậm; thuộc tính `<html data-ild-appearance|scheme|contrast|emphasis>`; đồng bộ với nút 🌓 cũ qua `body[data-theme]`.
- Phiếu QA.F.072 (`.sf-paper`) và bản in luôn nền trắng.
- Cột **Remark** của phiếu chấm co theo nội dung (07/10→09/10/2026): dưới 1000px: chưa có ghi chú thì hẹp tối đa, cột "Mẫu" nhận phần còn lại; từ 1000px (PC/Laptop, tablet ngang): cột Mẫu và cột Remark cùng rộng 34% để nhóm Cupping results nằm giữa màn hình, ô ghi chú bên trong vẫn co theo chữ; ô ghi chú là `textarea.remark-note` tự rộng dần theo chữ (tối đa 44ch, tablet 36vw) rồi xuống dòng (`scNoteSize`). Bảng dùng `table-layout:auto` trên mọi cỡ màn hình; điện thoại vẫn là thẻ từng mẫu.
- Tab Chấm nếm tối ưu **tablet (601–1180px)** & **phone (≤600px)**: nút 50px, không cuộn ngang, thanh Nộp phiếu dính đáy; header gọn trên phone. PC (≥1181px) giữ bố cục cũ; các tab khác ưu tiên PC/Laptop.

---

## 11. Triển khai

1. Sửa `index.html` (và `firestore.rules` nếu đổi quyền) → commit/push `main` → GitHub Pages tự deploy → hard refresh (Ctrl+F5).
2. Đổi Rules: dán `firestore.rules` vào Firebase Console → Firestore → Rules → **Publish** (ghi lại ngày ở mục 3.4).
3. Thử Firebase thật trên GitHub Pages (localhost bị chặn bởi API key).
4. Tài liệu đào tạo: `Downloads\Huong dan su dung iLD Sensory Entry.pptx` (3 slide: Bìa · Tạo Set · Chấm nếm) — không nằm trong repo.

---

## 12. Kiểm thử sau thay đổi

- **Cú pháp:** gộp các `<script>` ra file tạm → `node --check`.
- **Chạy local:** `python -m http.server <port>` trong thư mục repo → mở `http://localhost:<port>`. Firebase không đăng nhập được ở localhost; để thử UI theo vai trò, gán tạm trong console: `AUTH.profile={uid:'x',displayName:'Test',role:'admin'|'qc'|'supervisor'|'user',shared:false}; authApply();`. Đồng bộ có thể thử bằng mock `CLOUD.m` (getDoc/getDocs/runTransaction…) như đã làm khi phát triển.
- **Nghiệp vụ:** tạo Set (chặn trùng tên ca/ngày), chấm & nộp (Just In/Out bắt buộc lỗi), Set Demo mẫu #3 = **66,7% Failed**, All IN/Discard, Green coffee chỉ IN/OUT, đóng Set khi đủ 3 người, xác nhận/hủy xác nhận + Check by, sửa/xóa phiếu & Set theo quyền, xuất PDF/Excel, sao lưu/phục hồi JSON.
- **Phân quyền:** User chung chỉ 1 tab; ma trận lưu và áp dụng ngay; Rules chặn tương ứng.
- **Responsive:** 375px, 768px, 1024px, ≥1280px — không cuộn ngang ở tab Chấm nếm.
- **Theme:** 4 cấu hình, sáng/tối/hệ thống, đổi theme không mất bản nháp đang chấm.
- **Offline:** tắt mạng → app vẫn dùng, outbox giữ nguyên, có mạng tự gửi.

---

## 13. Backlog / việc còn mở

1. **Kiểm tra & Publish** `firestore.rules` bản có `isShared()` (nếu chưa).
2. ~~Cập nhật nội dung tab 📖 Hướng dẫn~~ — xong 07/10/2026.
3. Xóa tài khoản thừa `sensory@ild-sensory.local` trong Authentication (nếu không dùng).
4. Tách Items Code khỏi `meta` khi gần giới hạn 1 MiB.
5. Firebase App Check cho GitHub Pages; audit bất biến qua Cloud Function (cần gói Blaze).
6. Liên kết Items Code / SP·Lô với mẫu trong Set (gợi ý tên mẫu khi tạo Set).
7. Xuất `.xlsx` thật thay CSV/SpreadsheetML nếu chấp nhận thư viện.
8. Thử trên iPad/điện thoại thật (mới thử bằng trình duyệt giả lập kích thước).
9. Thang điểm riêng Green Bean / mẫu Spike / thống kê năng lực panelist (chờ nghiệp vụ duyệt).

---

## 14. Prompt bàn giao gợi ý

> Tôi có app một file `index.html` (repo `Sensory-App`, deploy GitHub Pages, backend Firebase `sensory-app-72a44`) để nhập & tổng hợp kết quả nếm cảm quan tại ILD Coffee Vietnam. Hãy đọc `HANDOFF - iLD Sensory Entry App.md` trước khi sửa. Giữ: một file, LocalStorage `ild_sensory_v1` (có migration nếu đổi model), công thức Result%, luồng đồng bộ Firebase v2 + gộp 3 chiều, đăng nhập 2 cấp + ma trận phân quyền (và `firestore.rules` tương ứng), định dạng ngày dd/mm/yyyy, giao diện ILD Crafted. Việc cần làm: [mô tả]. Sau khi sửa: `node --check`, chạy thử trên trình duyệt (PC + tablet + phone cho tab Chấm nếm), và nếu đổi quyền thì cập nhật & Publish Rules.
