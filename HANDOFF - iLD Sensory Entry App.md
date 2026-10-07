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
| ✍️ Chấm nếm (`tab-scoring`) | Phiếu QA.F.071: chọn Set + người nếm (ô tìm kiếm), IN / Just In / Out, Remark, **All IN**, **Discard**, **Nộp phiếu** (thanh dính đáy trên tablet/phone) | Mọi người (User chung **chỉ** thấy tab này) |
| 🧪 Tạo Set (`tab-setup`) | Wizard 2 bước; Bước ② thêm mẫu tay / Excel / **Nhập Excel từ SAP**; Tên mã hóa (mẫu mù) | Quyền `createSet` |
| 📊 Báo cáo (`tab-summary`) | Phiếu QA.F.072 theo Set, KPI, **Xác nhận kết quả** (Giám sát), Xuất Excel (CSV) / PDF | Tài khoản riêng |
| 🔍 Tra cứu (`tab-lookup`) | Danh sách Set: số người nếm, Passed/Failed, trạng thái, Giám sát; ✎ Sửa · 🔒 Đóng Set · Xóa | Tài khoản riêng (nút theo quyền) |
| 🗂️ Dữ liệu nộp phiếu (`tab-submissions`) | 1 dòng = 1 mẫu trong 1 phiếu; lọc Set/kết quả/ngày; xuất `.xls`; Sửa/Xóa dòng/Xóa phiếu | Tài khoản riêng (sửa/xóa theo `editSubmission`) |
| 👥 Panellist (`tab-panellist`) | Thêm/xóa, Nhập Excel (Employee code · Full name · Function · Dept.), **Xuất Excel** | Tài khoản riêng (sửa theo `editMaster`) |
| 📦 SP & Lô (`tab-product`) | Bảng liên tục Sản phẩm · Recipe · PO · Batch (`STATE.lots`), lọc/sắp xếp, nhập/xuất Excel | như trên |
| 🔗 Items Code (`tab-items`) | Loại (FGs/RW) · Item Code · Tên SP · Recipe (~3.100 dòng), lọc/sắp xếp/sửa tại ô, nhập/xuất | như trên |
| ☕ Sensory · 📖 Hướng dẫn | Kiến thức cảm quan; hướng dẫn trong app (**nội dung chưa cập nhật các tính năng mới** — xem backlog) | Tài khoản riêng |
| ⚙️ Cài đặt (`tab-settings`) | Trạng thái đồng bộ, Đẩy/Tải Firebase, auto-sync, Sao lưu/Phục hồi JSON, audit log, Reset | Tài khoản riêng (thao tác thay thế dữ liệu: Admin) |
| 🛡️ Người dùng (`tab-users`) | **Ma trận phân quyền**, mật khẩu User chung, tạo/sửa/khóa/xóa tài khoản | Admin |

Header (phải): Set đang mở · nút tài khoản (đăng nhập cấp 2) · 🚪 Đăng xuất · 🎨 Giao diện · 🌓 (ẩn trên điện thoại).

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
    id, name, date /*yyyy-mm-dd*/, shift /*AM|PM*/, sampleType, creator, status /*open|closed*/,
    samples: [{ no, code /*Tên mã hóa – mẫu mù*/, name, sscc, batch }],
    scores: { [panellistId]: { [sampleNo]: { result /*IN|JUSTIN|OUT*/, appearance:[], taste:[], note, at /*ISO nộp*/, editedAt?, editedBy? } } },
    attendees: [panellistId],
    approval?: { by, byUid, at, panellists, passed, total }   // Giám sát xác nhận → Set khóa
  }],
  currentSetId, currentPanellistId /*chỉ phiên*/, audit: [{ t /*dd/mm/yyyy HH:mm:ss*/, role /*tên người*/, action }]  // tối đa 500
}
```
- `sampleCode(sm)` = `code` hoặc `no` (Set cũ chưa có mã).
- `SAMPLE_TYPES`: Green coffee · Freeze-dried · Semi product · KQ · Complaint · Other (mặc định Freeze-dried). **Green coffee: phiếu chỉ IN / OUT** (`isGreenCoffee`).
- Lỗi: `ERR_APPEARANCE` (màu sậm, hai màu, lumpy, hạt cháy, bụi, bề mặt bị rổ, vụn) · `ERR_TASTE` (chemical, overdried, less/more aroma, chua, đắng, cook). Just In/Out **bắt buộc** chọn lỗi hoặc ghi chú; IN được ghi chú (không bắt buộc).
- 49 panellist chuẩn nhúng sẵn `DEFAULT_PANELLISTS` (nạp 1 lần/thiết bị, cờ `ild_sensory_pnl_default_v1`); 4 tên demo (`pnl_a..d`) tự bị loại.
- Không lưu cứng % — `calcSample(set,no)` luôn tính lại.

---

## 5. Công thức QA (bất biến)

```text
Điểm: In = 0 · Just In = 0.5 · Out = 3
Defect  = (Out×3 + JustIn×0.5) / (Total×3)
Result% = (1 − Defect) × 100      → Passed ≥ 80 · Failed < 80
CONFIG: PASS_THRESHOLD 80 · SCORE {OUT:3, JUSTIN:0.5, IN:0} · MIN_PANELLIST 3 · MAX_IMPORT_MB 3
```
- **Đóng Set chỉ khi đủ ≥ MIN_PANELLIST người nếm** (`closeBlockedMsg`) — áp dụng ở Tra cứu, Sửa Set và Tạo Set. Giám sát xác nhận vẫn được (có cảnh báo) và tự đóng Set.

---

## 6. Quy trình nghiệp vụ

1. **Tạo Set** (QC/có `createSet`): Bước ① Ca · Ngày · Tên Set (tự gợi ý số, không trùng trong cùng ca/ngày) · Loại mẫu → Bước ② mẫu: gõ tay / template Excel / **Nhập Excel từ SAP** (cột theo **tên**: ITEMNAME→Tên mẫu, SSCC→SSCC/Hour, BATCH→Batch; Tên mã hóa tự đánh; nếu cột Selection có "Checked" chỉ lấy dòng đó; bỏ SSCC trùng); đánh mã lại theo tiền tố (CHO 1, CHO 2…).
2. **Chấm nếm** (mọi người): chọn Set + tìm người nếm (tên không dấu/Mã NV) → IN/Just In/Out (All IN) → Nộp phiếu. Ghi `at`. Set đã xác nhận thì không nộp được.
3. **Đóng Set** (QC): khi đủ ≥ 3 người.
4. **Xác nhận** (Giám sát): tab Báo cáo → ✔ Xác nhận kết quả → `approval` + đóng & khóa; tên + giờ hiện ở ô **Check by** của QA.F.072 (màn hình/PDF/Excel). Hủy xác nhận để mở khóa.
5. Admin: sửa/xóa phiếu (Dữ liệu nộp phiếu), sửa/xóa Set (Tra cứu) — bị chặn khi Set đã xác nhận.

---

## 7. Đồng bộ Firebase (schema v2)

```text
ild_sensory_v2/meta          panellists, products, recipes, pos, batches, lots, itemCodes (~3.100), currentSetId, audit, schemaVersion 2
ild_sensory_v2/permissions   ma trận phân quyền
ild_sensory_sets/{setId}     Set (không gồm scores)
ild_sensory_scores/{setId}__{panellistId}   { setId, panellistId, scores }
ild_sensory_users/{uid}      hồ sơ & vai trò
ild_sensory/shared_state     legacy (giữ để rollback, chỉ Admin)
```
- Mỗi document có `revision, updatedAt, updatedBy, clientId`; ghi bằng transaction kiểm tra `baseRevision`.
- Lệch revision → **gộp 3 chiều** `cloudMerge3(base, local, remote)` (mảng theo `id`/`no`, audit theo nội dung; cùng sửa giá trị đơn → bản máy thắng); gộp lỗi/quá 5 lần → conflict UI ở Cài đặt.
- Không tự nhận dữ liệu cloud đè lên máy khi còn outbox/conflict, đang gõ hoặc **đang chấm dở phiếu** → `pendingRemote`, áp dụng sau.
- `cloudEntities()` trả **bản sao sâu** (tránh base trỏ chung STATE). Thiết bị chưa từng đồng bộ: cloud có dữ liệu → tải về (máy mới) hoặc hỏi; cloud trống → đẩy lên. Auto-sync mặc định **bật**. Web Locks: 1 tab chỉnh sửa chính.
- Dữ liệu thật đã có trên Firestore (kiểm tra 28/09/2026: 49 panellist, ~3.100 Items Code, 3 Set, 7 phiếu, 4 tài khoản).
- ⚠ `meta` chứa ~3.100 Items Code — Firestore giới hạn **1 MiB/document**; nếu Items Code tăng nhiều, cần tách ra document/collection riêng.

### Khóa LocalStorage
```text
ild_sensory_v1             dữ liệu chính (STATE)
ild_sensory_cloud_sync_v2  bases, outbox, conflicts
ild_sensory_cloud_auto     '0' = tắt auto-sync (mặc định bật)
ild_sensory_cloud_client   ID thiết bị
ild_sensory_editor_lock    khóa tab
ild_sensory_auth_cache     hồ sơ đăng nhập cho offline
ild_sensory_perms          cache ma trận phân quyền
ild_sensory_pnl_default_v1 đã nạp danh sách panellist chuẩn
ild_sensory_theme          sáng/tối cũ (nút 🌓)
ild-crafted-appearance-v1  Theme ILD Crafted {v:1, mode, contrast, emphasis}
```

---

## 8. Nhập / xuất

| Nơi | Nhập | Xuất |
|---|---|---|
| Panellist | `.xlsx/.xls/.csv` (Employee code · Full name · Function · Dept.) — trùng mã/tên thì cập nhật | `iLD_Panellist_<ngày>.xls` (cùng cột, nhập lại được) |
| Tạo Set Bước ② | Excel mẫu (Tên mã hóa · Tên mẫu · SSCC/Hour · Batch) · **SAP** | `iLD_SetSamples_….xls`, template |
| SP & Lô | Sản phẩm · Recipe · PO · Batch | `.xls` |
| Items Code | 4 cột chuẩn hoặc 2 khối FGs/RW | `.xls` |
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
2. Cập nhật nội dung tab **📖 Hướng dẫn** trong app theo tính năng mới (vai trò, SAP, Green coffee, xác nhận…).
3. Xóa tài khoản thừa `sensory@ild-sensory.local` trong Authentication (nếu không dùng).
4. Tách Items Code khỏi `meta` khi gần giới hạn 1 MiB.
5. Firebase App Check cho GitHub Pages; audit bất biến qua Cloud Function (cần gói Blaze).
6. Liên kết Items Code / SP·Lô với mẫu trong Set (gợi ý tên mẫu khi tạo Set).
7. Xuất `.xlsx` thật thay CSV/SpreadsheetML nếu chấp nhận thư viện.
8. Thử trên iPad/điện thoại thật (mới thử bằng trình duyệt giả lập kích thước).
9. Thang điểm riêng Green Bean / mẫu Spike / thống kê năng lực panellist (chờ nghiệp vụ duyệt).

---

## 14. Prompt bàn giao gợi ý

> Tôi có app một file `index.html` (repo `Sensory-App`, deploy GitHub Pages, backend Firebase `sensory-app-72a44`) để nhập & tổng hợp kết quả nếm cảm quan tại ILD Coffee Vietnam. Hãy đọc `HANDOFF - iLD Sensory Entry App.md` trước khi sửa. Giữ: một file, LocalStorage `ild_sensory_v1` (có migration nếu đổi model), công thức Result%, luồng đồng bộ Firebase v2 + gộp 3 chiều, đăng nhập 2 cấp + ma trận phân quyền (và `firestore.rules` tương ứng), định dạng ngày dd/mm/yyyy, giao diện ILD Crafted. Việc cần làm: [mô tả]. Sau khi sửa: `node --check`, chạy thử trên trình duyệt (PC + tablet + phone cho tab Chấm nếm), và nếu đổi quyền thì cập nhật & Publish Rules.
