# Tài Liệu Kiến Trúc Đồng Bộ Apple Health — Heart Champ

> **Phiên bản**: 2.0 · **Ngày**: 20/03/2026 · **Team**: Heart Champ App

---

## 1. Tổng Quan Hệ Thống

Hệ thống đồng bộ Apple Health (HealthKit) của Heart Champ được thiết kế theo mô hình **Phased Syncing** (Đồng bộ phân giai đoạn) kết hợp **Sequential Batching** (Xử lý theo lô tuần tự). Toàn bộ kiến trúc chạy ở **tầng Dart (Flutter)**, sử dụng `WidgetsBindingObserver` để bắt sự kiện LifeCycle thay vì iOS native `BGTaskScheduler`.

### 1.1 Các file cốt lõi

| File | Vai trò |
|------|---------|
| `core/services/health_service.dart` | Giao tiếp trực tiếp với plugin `health` (HealthKit). Cung cấp các hàm static `fetch*` cho từng loại chỉ số. |
| `core/services/health_sync_service.dart` | Logic nghiệp vụ sync: chia giai đoạn, dispatch tuần tự 4 loại dữ liệu, dedup, lưu vào repository. |
| `core/services/health_sync_state.dart` | Enum `HealthSyncStatus` & `HealthSyncProgress` dùng cho UI layer. |
| `domain/models/health_sync_progress.dart` | Model `SyncProgress` — serializable (JSON) để persist trạng thái batching vào SharedPreferences. |
| `core/providers/health_sync_controller.dart` | Controller chính (ChangeNotifier + WidgetsBindingObserver): điều phối 3 phase, lắng nghe LifeCycle app. |
| `core/providers/health_sync_provider.dart` | Provider phụ (legacy): phục vụ trang Demo. |
| `presentation/.../onboarding_health_permission_page.dart` | Màn hình xin quyền HealthKit trong Onboarding. |
| `presentation/.../onboarding_sync_health_page.dart` | Màn hình hiển thị animation sync đang chạy (Phase 1 + trigger Phase 2). |
| `presentation/settings/pages/log_data_apple_health.dart` | Trang debug log — hiển thị raw data từ HealthKit. |

### 1.2 Dữ liệu được đồng bộ

| Chỉ số | HealthDataType | Đơn vị lưu DB | Ghi chú |
|--------|----------------|----------------|---------|
| Huyết áp | `BLOOD_PRESSURE_SYSTOLIC` + `BLOOD_PRESSURE_DIASTOLIC` | mmHg | Ghép cặp theo timestamp |
| Nhịp tim (Pulse) | `HEART_RATE` | bpm | Bổ trợ record huyết áp (±1 phút) |
| Đường huyết | `BLOOD_GLUCOSE` | mmol/L | Apple Health trả mg/dL → convert `mgDlToMmolL` |
| Cân nặng | `WEIGHT` | kg | Apple Health trả kg → lưu trực tiếp |
| Bước chân | `STEPS` | steps/ngày | Tổng hợp theo ngày, tính phụ: calories, distance, duration |
| Năng lượng tiêu hao | `ACTIVE_ENERGY_BURNED` | kcal | Dùng cho hiển thị Calories |
| Năng lượng cơ bản | `BASAL_ENERGY_BURNED` | kcal | Dùng cho tính TDEE |

---

## 2. Quy Trình Xin Quyền (Permission Flow)

### 2.1 Luồng thực thi

```
OnboardingHealthPermissionPage
  └─ Bấm "Allow Access"
       └─ HealthService.requestPermissions()
            └─ health.requestAuthorization(8 types, READ only, timeout: 30s)
                 ├── granted = true  → Lưu prefs + Thưởng 300 coin + Navigate → OnboardingSyncHealthPage
                 └── granted = false → Hiện CupertinoAlertDialog
                                         ├── "Skip for now"  → Navigate → OnboardingSyncHealthPage (không sync)
                                         └── "Open Settings" → openAppSettings()
```

### 2.2 Danh sách quyền yêu cầu (chỉ READ)

```dart
final allTypes = [
  HealthDataType.HEART_RATE,
  HealthDataType.STEPS,
  HealthDataType.ACTIVE_ENERGY_BURNED,
  HealthDataType.BASAL_ENERGY_BURNED,    // ✅ Đã bổ sung
  HealthDataType.BLOOD_PRESSURE_SYSTOLIC,
  HealthDataType.BLOOD_PRESSURE_DIASTOLIC,
  HealthDataType.BLOOD_GLUCOSE,
  HealthDataType.WEIGHT,                  // ✅ Đã bổ sung
];
```

Tất cả 8 loại dữ liệu mà app cần fetch đều nằm đầy đủ trong danh sách xin quyền. Timeout 30 giây đảm bảo không bao giờ bị treo vô hạn nếu iOS dialog bị block.

---

## 3. Kiến Trúc 3 Giai Đoạn Đồng Bộ (Phased Syncing)

### 3.1 Sơ đồ tổng quan

```
┌──────────────────────────────────────────────────────────────────────┐
│                        HealthSyncController                         │
│  (ChangeNotifier + WidgetsBindingObserver)                          │
│                                                                      │
│  ┌─────────────┐    ┌─────────────────────┐    ┌─────────────────┐  │
│  │  PHASE 1    │    │      PHASE 2        │    │    PHASE 3      │  │
│  │ Initial     │───▶│  Background         │    │  Incremental    │  │
│  │ Sync 30d    │    │  History Batching    │    │  Sync (Delta)   │  │
│  │ (Foreground)│    │  (unawaited Async)   │    │  (App Resume)   │  │
│  └─────────────┘    └─────────────────────┘    └─────────────────┘  │
│         │                    │                          │            │
│         ▼                    ▼                          ▼            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    HealthSyncService                         │   │
│  │  _syncRangeInternal(days, startFrom)                         │   │
│  │    └── Sequential: BP → Sugar → Weight → Steps               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      HealthService                          │    │
│  │  fetchBloodPressure / fetchBloodSugar / fetchWeight          │    │
│  │  fetchStepsRaw / fetchHeartRate                              │    │
│  │  (Plugin `health` → iOS HealthKit)                           │    │
│  └─────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

---

### 3.2 Phase 1 — Initial Sync (Đồng bộ khởi tạo)

**Mục đích**: Lấy nhanh 30 ngày dữ liệu để user thấy Dashboard có nội dung ngay lập tức.

**Trigger**: `OnboardingSyncHealthPage.initState()` → gọi `HealthSyncController.startInitialSync()`

**Code flow**:

```dart
// health_sync_controller.dart
Future<void> startInitialSync() async {
  _setStatus(SyncStatus.initialSyncing);
  
  // Phase 1: Foreground — chờ xong mới tiếp
  await _syncService.syncLast30Days();
  
  await _saveProgress(_progress.copyWith(synced30Days: true));
  await _prefs.setHealthInitialSyncDone(true);
  
  _setStatus(SyncStatus.backgroundSyncing);
  
  // Phase 2: Không await → UI không bị block
  unawaited(_syncHistoryInBackground());
}
```

**Bên trong `syncLast30Days()`**:

```dart
Future<void> syncLast30Days() async {
  if (_isHistorySyncing) return;  // Guard: chống gọi trùng
  _isHistorySyncing = true;
  try {
    await _syncRangeInternal(days: 30, startFrom: 0);
  } finally {
    _isHistorySyncing = false;
  }
}
```

**`_syncRangeInternal` — Dispatch tuần tự**:

Mỗi loại dữ liệu được fetch → xử lý → lưu DB xong hoàn toàn rồi mới chuyển sang loại tiếp theo. Điều này đảm bảo tại bất kỳ thời điểm nào, chỉ có **1 response từ HealthKit** được giữ trong bộ nhớ.

```dart
Future<void> _syncRangeInternal({required int days, required int startFrom}) async {
  await syncBloodPressure(days: days, startFrom: startFrom);
  await syncBloodSugar(days: days, startFrom: startFrom);
  await syncWeight(days: days, startFrom: startFrom);
  await syncSteps(days: days, startFrom: startFrom);
}
```

---

### 3.3 Phase 2 — Background History Sync (Batching)

**Mục đích**: Lấp đầy dữ liệu lịch sử (lên tới 5 năm) mà không gây đơ app hoặc tràn bộ nhớ.

**Trigger**: Tự động chạy sau Phase 1 thông qua `unawaited(_syncHistoryInBackground())`

**Cơ chế Batching — 7 lô tuần tự, tối đa 1 năm mỗi lô**:

```
Timeline (ngày trước hiện tại):
─────────────────────────────────────────────────────────────────────────────
│ 0  │ 30d │  90d  │ 180d  │ 365d  │ 730d │ 1095d │ 1460d │ 1825d (5y) │
─────────────────────────────────────────────────────────────────────────────
      Ph.1   Bat.1   Bat.2   Bat.3   Bat.4  Bat.5   Bat.6   Bat.7
      30d    60d     90d     185d    365d   365d    365d    365d
```

| Batch | Khoảng thời gian | Dữ liệu kéo | Progress Flag |
|-------|------------------|--------------|---------------|
| Phase 1 (Init) | 0 → 30d | 30 ngày | `synced30Days` |
| Batch 1 | 30d → 90d | 60 ngày | `synced3Months` |
| Batch 2 | 90d → 180d | 90 ngày | `synced6Months` |
| Batch 3 | 180d → 365d | 185 ngày | `synced1Year` |
| Batch 4 | 365d → 730d | 365 ngày | `syncedFullHistory` (chung) |
| Batch 5 | 730d → 1095d | 365 ngày | ↑ |
| Batch 6 | 1095d → 1460d | 365 ngày | ↑ |
| Batch 7 | 1460d → 1825d | 365 ngày | ↑ |

Mỗi batch tối đa **365 ngày** — đảm bảo lượng DataPoint trong mỗi lần fetch không vượt ngưỡng gây tràn RAM, kể cả trên thiết bị cấu hình thấp.

**Code flow**:

```dart
Future<void> syncHistoryInBackground({
  required SyncProgress progress,
  required Future<void> Function(SyncProgress) onProgressSaved,
}) async {
  // Batch 1–3: Các mốc có progress flag riêng
  if (!progress.synced3Months) {
    await _syncRangeInternal(days: 90, startFrom: 30);
    progress = progress.copyWith(synced3Months: true);
    await onProgressSaved(progress);
  }
  // ... tương tự cho synced6Months, synced1Year ...
  
  // Batch 4–7: Chia nhỏ 4 năm thành 4 × 1 năm
  if (!progress.syncedFullHistory) {
    const yearlyBatches = [
      (days: 730,  startFrom: 365),   // Year 2
      (days: 1095, startFrom: 730),   // Year 3
      (days: 1460, startFrom: 1095),  // Year 4
      (days: 1825, startFrom: 1460),  // Year 5
    ];
    for (final batch in yearlyBatches) {
      await _syncRangeInternal(days: batch.days, startFrom: batch.startFrom);
    }
    progress = progress.copyWith(syncedFullHistory: true);
    await onProgressSaved(progress);
  }
}
```

**SyncProgress Model** — persist vào SharedPreferences dạng JSON:

```dart
class SyncProgress {
  final bool synced30Days;
  final bool synced3Months;
  final bool synced6Months;
  final bool synced1Year;
  final bool syncedFullHistory;
  final DateTime? lastSyncTime;
}
```

---

### 3.4 Phase 3 — Incremental Sync (Đồng bộ delta)

**Mục đích**: Cập nhật dữ liệu mới từ Apple Watch hoặc nhập thủ công khi user quay lại app.

**Trigger**: `HealthSyncController` implement `WidgetsBindingObserver`. Mỗi khi `AppLifecycleState.resumed` → gọi `_triggerIncrementalSync()`.

```dart
@override
void didChangeAppLifecycleState(AppLifecycleState state) {
  if (state == AppLifecycleState.resumed && _prefs.isHealthPermissionsGranted) {
    _triggerIncrementalSync();
  }
}

Future<void> _triggerIncrementalSync() async {
  if (!_prefs.isHealthInitialSyncDone) return;

  final lastSyncTime = _prefs.lastHealthSyncTime
      ?? DateTime.now().subtract(const Duration(days: 1));
  
  _setStatus(SyncStatus.incrementalSyncing);
  await _syncService.syncIncremental(lastSyncTime);
  await _prefs.setLastHealthSyncTime(DateTime.now());
  _setStatus(SyncStatus.completed);
}
```

**Bên trong `syncIncremental`**:

```dart
Future<void> syncIncremental(DateTime lastSyncTime) async {
  final daysSinceLastSync = DateTime.now().difference(lastSyncTime).inDays;
  final syncDays = daysSinceLastSync > 0 ? daysSinceLastSync + 1 : 1;
  // +1 buffer đảm bảo không bỏ sót edge case nửa đêm
  
  await _syncRangeInternal(days: syncDays, startFrom: 0);
}
```

---

## 4. Cơ Chế Xử Lý Chi Tiết Từng Loại Dữ Liệu

### 4.1 Blood Pressure (Huyết Áp)

```
[HealthKit]                           [App DB]
  SYSTOLIC ─┐                         
  DIASTOLIC ┼──→ Ghép cặp theo ──→ Dedup timestamp ──→ bpRepo.addEntry()
  HEART_RATE┘    DateTime              (±5 phút)         BloodPressureEntry
```

- Lấy riêng `SYSTOLIC` + `DIASTOLIC` → ghép cặp theo `dateFrom` vào Map `bpPairs`.
- Lấy `HEART_RATE` riêng → match với timestamp BP (±1 phút) để lấy `pulse`.
- Nếu không tìm thấy HR match → default `pulse = 70`.
- Dedup: So sánh timestamp mới với existing entries, chênh lệch `<= 5 phút` → skip.

### 4.2 Blood Sugar (Đường Huyết)

```
[HealthKit]                              [App DB]
  BLOOD_GLUCOSE (mg/dL) ──→ Convert  ──→ Dedup ──→ sugarRepo.addEntry()
                              mgDl→mmolL   (±5ph)     BloodSugarEntry
```

- Apple Health trả đường huyết đơn vị **mg/dL**.
- App lưu đơn vị **mmol/L** → gọi `UnitConverterService.mgDlToMmolL()`.
- Dedup ±5 phút.

### 4.3 Weight (Cân Nặng)

```
[HealthKit]                    [App DB]
  WEIGHT (kg) ──→ Dedup ──→ weightRepo.addEntry()
                   (±5ph)     WeightEntry(kg)
```

- Apple Health trả về kg → lưu trực tiếp.
- Dedup ±5 phút.

### 4.4 Steps (Bước Chân) — Tối ưu hóa Batch Fetch

```
[HealthKit]                                            [App DB]
  STEPS ──→ fetchStepsRaw (1 lần) ──→ Aggregate ──→ stepRepo.upsertForDate()
             toàn bộ range             theo ngày       StepEntry
```

Thay vì gọi `getStepsInRange()` cho **từng ngày** (N lần API call), hệ thống gọi `fetchStepsRaw()` **một lần duy nhất** cho toàn bộ khoảng thời gian, sau đó aggregate theo ngày trong bộ nhớ:

```dart
// 1 lần gọi HealthKit cho toàn bộ range
final stepsData = await HealthService.fetchStepsRaw(start, now);

// Aggregate theo ngày trong bộ nhớ
final Map<DateTime, int> dailySteps = {};
for (final data in stepsData) {
  if (data.value is NumericHealthValue) {
    final dayKey = DateTime(data.dateFrom.year, data.dateFrom.month, data.dateFrom.day);
    dailySteps[dayKey] = (dailySteps[dayKey] ?? 0) +
        (data.value as NumericHealthValue).numericValue.toInt();
  }
}
```

Công thức tính các giá trị phụ sinh từ profile user:
- **Stride** = `heightCm × factor` (Male=0.43, Female=0.415, Other=0.42)
- **Distance** = `steps × strideCm / 100` (m)
- **Duration** = `steps / 100` (phút, ~100 steps/phút)
- **Calories** = `weightKg × (distanceKm) × 0.7` (kcal)

Steps dùng `upsertForDate()` (ghi đè theo ngày) thay vì dedup timestamp vì bản chất dữ liệu là aggregate.

---

## 5. Cơ Chế Khử Trùng Dữ Liệu (Deduplication)

Áp dụng cho **Blood Pressure**, **Blood Sugar**, **Weight**:

```dart
// Lấy tất cả existing entries trong DB cho khoảng thời gian sync
final existingEntries = await repo.getEntriesInRange(start, now);
final existingTimestamps = <int>{};
for (final e in existingEntries) {
  existingTimestamps.add(e.recordedAt.millisecondsSinceEpoch);
}

// Với mỗi record từ HealthKit:
final isDuplicate = existingTimestamps.any((ts) =>
    (ts - timestamp.millisecondsSinceEpoch).abs() <= 5 * 60 * 1000);
if (isDuplicate) continue;  // Bỏ qua
```

| Tham số | Giá trị | Giải thích |
|---------|---------|------------|
| Threshold | 5 phút (300,000 ms) | Medical data hiếm khi đo 2 lần trong 5 phút. Đủ rộng để bắt duplicate nhưng không quá rộng gây mất data |
| Phương pháp | `Set<int>.any()` | Duyệt trên Set — nhanh vì chỉ compare int |
| Steps | `upsertForDate()` | Ghi đè theo ngày — đúng bản chất aggregate data |

---

## 6. Quản Lý Trạng Thái (State Management)

### 6.1 SyncStatus Enum

```dart
enum SyncStatus {
  idle,                // Chưa làm gì
  initialSyncing,      // Phase 1 đang chạy
  backgroundSyncing,   // Phase 2 đang chạy
  incrementalSyncing,  // Phase 3 đang chạy
  completed,           // Tất cả đã xong
  failed,              // Có lỗi xảy ra
}
```

### 6.2 SyncProgress — Persistent State

```dart
const SyncProgress.empty()
    : synced30Days = false,      // Phase 1
      synced3Months = false,     // Batch 1
      synced6Months = false,     // Batch 2
      synced1Year = false,       // Batch 3
      syncedFullHistory = false, // Batch 4–7
      lastSyncTime = null;
```

- Serialize thành JSON → lưu vào `AppPreferences` (SharedPreferences).
- Mỗi khi 1 batch hoàn tất → cập nhật flag → lưu ngay lập tức.
- Khi app restart → load lại progress → skip các batch đã xong.

### 6.3 Guard Flags

```dart
bool _isHistorySyncing = false;    // Cho Phase 1 + 2
bool _isIncrementalSyncing = false; // Cho Phase 3
bool get isSyncing => _isHistorySyncing || _isIncrementalSyncing;
```

- Phase 1 và Phase 2 chia sẻ `_isHistorySyncing` → không thể chạy đồng thời.
- Phase 3 dùng flag riêng → có thể chạy khi Phase 2 còn đang chạy nền. Dedup xử lý trùng lặp nếu range overlap.

---

## 7. Tương Tác UI (User Interface)

### 7.1 Màn hình Onboarding Sync

UI hiển thị khi Phase 1 đang chạy:
- Animation **vòng tròn đồng tâm** + **dòng số chảy xuống** tạo cảm giác dữ liệu đang được truyền.
- Icon Apple Health ở trên + Logo Heart Champ ở dưới + **Loading spinner** nhỏ ở góc icon.
- Label trạng thái thay đổi theo `SyncStatus`:

| SyncStatus | Label hiển thị |
|------------|----------------|
| `initialSyncing` | "Pulling last 30 days..." |
| `backgroundSyncing` (chưa 3m) | "Loading 3 months of history..." |
| `backgroundSyncing` (chưa 6m) | "Loading 6 months of history..." |
| `backgroundSyncing` (chưa 1y) | "Loading 1 year of history..." |
| `backgroundSyncing` (khác) | "Loading full history..." |
| `completed` | "All data synced ✓" |
| `failed` | "Sync failed — will retry" |

- Nút "**Skip & Continue**" → cho phép user vào Main ngay khi Phase 1 chưa xong.
- Nút chuyển thành "**Continue**" khi Initial Sync hoàn tất.

### 7.2 Trang Debug / Settings

| Trang | Chức năng |
|-------|-----------|
| `HealthSyncDemoPage` | Card trạng thái sync (spinner / check icon) + message progress |
| `LogDataAppleHealthPage` | Console terminal — fetch & hiển thị raw data trực tiếp từ HealthKit |

---

## 8. Đánh Giá Toàn Diện Kiến Trúc Hiện Tại

### ✅ Điểm Mạnh

| # | Điểm mạnh | Chi tiết |
|---|-----------|----------|
| 1 | **UX phân giai đoạn** | User thấy dữ liệu ngay (30d), không chờ toàn bộ lịch sử. Pattern chuẩn industry (Fitbit, Samsung Health). |
| 2 | **Fault Tolerance** | `SyncProgress` persist qua JSON → app bị kill vẫn resume đúng batch. |
| 3 | **Batching chia nhỏ (tối đa 1 năm/lô)** | 7 lô tuần tự, mỗi lô tối đa 365 ngày — không có batch nào quá lớn gây tràn RAM. |
| 4 | **Sync tuần tự trong mỗi lô** | `_syncRangeInternal` chạy BP → Sugar → Weight → Steps **tuần tự** — tại mọi thời điểm chỉ 1 response HealthKit trong memory. |
| 5 | **Steps tối ưu 1 API call** | `fetchStepsRaw()` lấy toàn bộ data points 1 lần → aggregate theo ngày in-memory. Giảm từ N API calls xuống còn 1. |
| 6 | **Dedup thực chiến** | Threshold 5 phút phù hợp medical data. Steps dùng `upsert` — đúng bản chất aggregate. |
| 7 | **Guard chống race condition** | Flag riêng cho History vs Incremental; kiểm tra status trước mỗi hàm public. |
| 8 | **Notifier pattern** | Sau sync → notify listeners → UI chart tự cập nhật realtime. |
| 9 | **Quyền đầy đủ 8 types** | Tất cả types mà app fetch đều nằm trong `requestPermissions()` — không có silent empty results. |
| 10 | **Auto-trigger Incremental** | `WidgetsBindingObserver` bắt `resumed` → sync delta tự động, user không cần nhấn nút. |

### ⚠️ Điểm Cần Lưu Ý (Hiện trạng chấp nhận được)

| # | Lưu ý | Giải thích |
|---|-------|------------|
| 1 | **Không phải iOS BGTask** | Sync chạy trên Dart VM khi app foreground. Kill app = dừng sync. Nhưng nhờ persist progress, lần mở tiếp sẽ tiếp tục. |
| 2 | **Phase 2 + 3 có thể overlap** | Incremental dùng flag riêng → có thể chạy cùng lúc Background History. Dedup xử lý trùng lặp nếu xảy ra. |
| 3 | **Dual Controller / Provider** | Tồn tại 2 module (`HealthSyncController` production, `HealthSyncProvider` demo) — cùng gọi `HealthSyncService` nhưng logic khác nhau. Provider không persist progress. |
| 4 | **`Set.any()` dedup — O(n)** | Với mỗi record mới, `.any()` duyệt linear trên Set. Trong thực tế vẫn nhanh (compare int trên Set nhỏ) nhưng không O(1). |
