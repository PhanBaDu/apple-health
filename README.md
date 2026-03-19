# Tài Liệu Kiến Trúc Đồng Bộ Apple Health — Heart Champ

> **Phiên bản**: 1.0 · **Ngày**: 19/03/2026 · **Team**: Heart Champ App

---

## 1. Tổng Quan Hệ Thống

Hệ thống đồng bộ Apple Health (HealthKit) của Heart Champ được thiết kế theo mô hình **Phased Syncing** (Đồng bộ phân giai đoạn) kết hợp **Sequential Batching** (Xử lý theo lô tuần tự). Toàn bộ kiến trúc nằm ở **tầng Dart (Flutter)**, không tận dụng iOS native `BGTaskScheduler`.

### 1.1 Các file cốt lõi

| File | Vai trò |
|------|---------|
| `core/services/health_service.dart` | Giao tiếp trực tiếp với plugin `health` (HealthKit). Cung cấp các hàm static `fetch*` cho từng loại chỉ số. |
| `core/services/health_sync_service.dart` | Logic nghiệp vụ sync chính: chia giai đoạn, gọi song song 4 loại dữ liệu, xử lý dedup, lưu vào repository. |
| `core/services/health_sync_state.dart` | Enum `HealthSyncStatus` & `HealthSyncProgress` dùng để track trạng thái UI. |
| `domain/models/health_sync_progress.dart` | Model `SyncProgress` — serializable (JSON) để persist trạng thái batching vào SharedPreferences. |
| `core/providers/health_sync_controller.dart` | Controller chính (ChangeNotifier + WidgetsBindingObserver): điều phối 3 phase, lắng nghe LifeCycle app. |
| `core/providers/health_sync_provider.dart` | Provider phụ (legacy): cung cấp API tương tự nhưng **không persist progress**, dùng ở Demo page. |
| `presentation/custom_onboarding/pages/onboarding_health_permission_page.dart` | Màn hình xin quyền HealthKit trong Onboarding. |
| `presentation/custom_onboarding/pages/onboarding_sync_health_page.dart` | Màn hình hiển thị animation khi sync đang chạy (Phase 1 + trigger Phase 2). |
| `presentation/settings/pages/log_data_apple_health.dart` | Trang debug log hiển thị raw data từ HealthKit. |
| `presentation/settings/pages/health_sync_demo_page.dart` | Trang demo trạng thái sync (dùng `HealthSyncProvider` legacy). |

### 1.2 Dữ liệu được đồng bộ

| Chỉ số | HealthDataType | Đơn vị lưu DB | Ghi chú |
|--------|----------------|----------------|---------|
| Huyết áp | `BLOOD_PRESSURE_SYSTOLIC` + `BLOOD_PRESSURE_DIASTOLIC` | mmHg | Ghép cặp theo timestamp |
| Nhịp tim (Pulse) | `HEART_RATE` | bpm | Dùng bổ trợ cho record huyết áp (±1 phút) |
| Đường huyết | `BLOOD_GLUCOSE` | mmol/L | Apple Health trả về mg/dL → convert `mgDlToMmolL` |
| Cân nặng | `WEIGHT` | kg | Apple Health trả về kg → lưu trực tiếp |
| Bước chân | `STEPS` | steps/ngày | Tổng hợp theo ngày, tính phụ: calories, distance, duration |
| Năng lượng | `ACTIVE_ENERGY_BURNED` + `BASAL_ENERGY_BURNED` | kcal | Dùng cho TDEE, chưa sync vào DB |

---

## 2. Quy Trình Xin Quyền (Permission Flow)

### 2.1 Luồng thực thi

```
OnboardingHealthPermissionPage
  └─ Bấm "Allow Access"
       └─ HealthService.requestPermissions()
            └─ health.requestAuthorization(6 types, READ only)
                 ├── granted = true  → Lưu prefs + Thưởng 300 coin + Navigate → OnboardingSyncHealthPage
                 └── granted = false → Hiện Dialog CupertinoAlertDialog
                                         ├── "Skip for now"  → Navigate → OnboardingSyncHealthPage (không sync)
                                         └── "Open Settings" → openAppSettings()
```

### 2.2 Các quyền được yêu cầu (chỉ READ)

```dart
final allTypes = [
  HealthDataType.HEART_RATE,
  HealthDataType.STEPS,
  HealthDataType.ACTIVE_ENERGY_BURNED,
  HealthDataType.BLOOD_PRESSURE_SYSTOLIC,
  HealthDataType.BLOOD_PRESSURE_DIASTOLIC,
  HealthDataType.BLOOD_GLUCOSE,
];
```

> **Lưu ý**: Quyền `WEIGHT` và `BASAL_ENERGY_BURNED` **chưa** nằm trong danh sách `requestPermissions()` nhưng `HealthSyncService` vẫn gọi `fetchWeight()`. Trên iOS, nếu quyền chưa được cấp, `getHealthDataFromTypes` trả về mảng rỗng (không crash).

### 2.3 Đánh giá

| Tiêu chí | Đánh giá |
|----------|----------|
| Chỉ xin READ | ✅ Tốt — giảm thiểu lo ngại quyền riêng tư của user |
| Timeout 30s | ✅ Tốt — tránh treo vô hạn khi iOS dialog bị block |
| Thiếu quyền WEIGHT trong danh sách | ⚠️ Rủi ro — sync weight có thể luôn trả về rỗng nếu user không chủ động cấp quyền ở Settings |
| Thiếu quyền BASAL_ENERGY_BURNED | ⚠️ Tương tự — TDEE tính vẫn hoạt động nhưng Basal = 0 |
| Coin reward 300 | ✅ UX gamification tốt — khuyến khích user cấp quyền |

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
│  │    └── Future.wait([syncBP, syncSugar, syncWeight, syncSteps])│   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      HealthService                          │    │
│  │  fetchBloodPressure / fetchBloodSugar / fetchWeight          │    │
│  │  getStepsInRange / fetchHeartRate                            │    │
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
  
  // Phase 1: Foreground
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

**`_syncRangeInternal`** chạy **song song** 4 tác vụ sync:

```dart
await Future.wait([
  syncBloodPressure(days: days, startFrom: startFrom),
  syncBloodSugar(days: days, startFrom: startFrom),
  syncWeight(days: days, startFrom: startFrom),
  syncSteps(days: days, startFrom: startFrom),
]);
```

### Đánh giá Phase 1

| Tiêu chí | Đánh giá |
|----------|----------|
| Tốc độ phản hồi UI | ✅ Nhanh — 30 ngày dữ liệu nhẹ, chạy song song 4 task |
| Guard chống gọi trùng | ✅ Có `_isHistorySyncing` flag |
| Transition sang Phase 2 | ✅ `unawaited` — UI hoàn toàn không bị chặn |
| Error handling | ✅ try/catch bọc toàn bộ, set `SyncStatus.failed` nếu lỗi |

---

### 3.3 Phase 2 — Background History Sync (Batching)

**Mục đích**: Lấp đầy dữ liệu lịch sử (lên tới 5 năm) mà không gây đơ app.

**Trigger**: Tự động chạy sau Phase 1 thông qua `unawaited(_syncHistoryInBackground())`

**Cơ chế Batching — 4 lô tuần tự**:

```
Timeline (ngày trước hiện tại):
──────────────────────────────────────────────────────────────────
│ 0   │  30d  │   90d   │   180d   │   365d   │   1825d (5y)  │
──────────────────────────────────────────────────────────────────
       Phase1    Batch1     Batch2      Batch3      Batch4
        30d     30d→90d    90d→180d   180d→365d   365d→1825d
```

| Batch | Khoảng thời gian | Dữ liệu thực tế kéo | Progress Flag |
|-------|------------------|-----------------------|---------------|
| Phase 1 (Initial) | 0 → 30d trước | 30 ngày | `synced30Days` |
| Batch 1 | 30d → 90d trước | 60 ngày | `synced3Months` |
| Batch 2 | 90d → 180d trước | 90 ngày | `synced6Months` |
| Batch 3 | 180d → 365d trước | 185 ngày | `synced1Year` |
| Batch 4 | 365d → 1825d trước | 1460 ngày (~4 năm) | `syncedFullHistory` |

**Code flow (HealthSyncService)**:

```dart
Future<void> syncHistoryInBackground({
  required SyncProgress progress,
  required Future<void> Function(SyncProgress) onProgressSaved,
}) async {
  // Batch 1: 3 months
  if (!progress.synced3Months) {
    await _syncRangeInternal(days: 90, startFrom: 30);
    progress = progress.copyWith(synced3Months: true);
    await onProgressSaved(progress);
  }
  
  // Batch 2: 6 months
  if (!progress.synced6Months) {
    await _syncRangeInternal(days: 180, startFrom: 90);
    progress = progress.copyWith(synced6Months: true);
    await onProgressSaved(progress);
  }
  
  // Batch 3: 1 year
  if (!progress.synced1Year) {
    await _syncRangeInternal(days: 365, startFrom: 180);
    progress = progress.copyWith(synced1Year: true);
    await onProgressSaved(progress);
  }
  
  // Batch 4: Full history (~5 years)
  if (!progress.syncedFullHistory) {
    await _syncRangeInternal(days: 365 * 5, startFrom: 365);
    progress = progress.copyWith(syncedFullHistory: true);
    await onProgressSaved(progress);
  }
}
```

**SyncProgress Model** (persist vào SharedPreferences dạng JSON):

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

### Đánh giá Phase 2

| Tiêu chí | Đánh giá |
|----------|----------|
| Chia nhỏ thành batch | ✅ Tốt — tránh load toàn bộ lịch sử 1 lần |
| Persist progress | ✅ Mỗi batch xong → lưu JSON vào prefs → có thể resume nếu app bị kill |
| Chạy nền khi user ở MainScreen | ✅ `unawaited` giữ UI tự do |
| Batch 4 kích thước lớn (4 năm) | ⚠️ Batch cuối kéo 1460 ngày, nếu ngày nào user cũng có HR data → lượng DataPoint rất lớn. Kết hợp `Future.wait` 4 task song song nhân tải RAM lên x4 |
| Không phải OS-level background | ⚠️ Chạy trên Dart event loop khi app foreground. Kill app = dừng sync |

---

### 3.4 Phase 3 — Incremental Sync (Đồng bộ delta)

**Mục đích**: Cập nhật dữ liệu mới từ thiết bị đeo (Apple Watch, v.v.) hoặc nhập thủ công khi user quay lại app.

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
  
  await _syncRangeInternal(days: syncDays, startFrom: 0);
}
```

### Đánh giá Phase 3

| Tiêu chí | Đánh giá |
|----------|----------|
| Trigger tự động khi resume | ✅ Tốt — user không cần nhấn nút refresh |
| Tính delta chính xác | ✅ `daysSinceLastSync + 1` buffer tránh bỏ sót edge case nửa đêm |
| Guard chống trùng | ✅ `_isIncrementalSyncing` flag riêng biệt, không conflict với History Sync |
| Chỉ sync khi đã qua Initial | ✅ Check `isHealthInitialSyncDone` trước khi chạy |
| Fallback khi chưa có lastSyncTime | ✅ Default 1 ngày trước nếu null |

---

## 4. Cơ Chế Xử Lý Chi Tiết Từng Loại Dữ Liệu

### 4.1 Blood Pressure (Huyết Áp)

```
[HealthKit]                           [App DB]
  SYSTOLIC ─┐                         
  DIASTOLIC ┼──→ Ghép cặp theo ──→ Dedup timestamp ──→ bpRepo.addEntry()
  HEART_RATE┘    DateTime              (±5 phút)         BloodPressureEntry
```

**Chi tiết**:
- Lấy riêng `SYSTOLIC` + `DIASTOLIC` → ghép cặp theo `dateFrom` vào Map `bpPairs`.
- Lấy `HEART_RATE` riêng → match với timestamp BP (±1 phút) để lấy `pulse`.
- Nếu không tìm thấy HR match → default `pulse = 70`.
- Dedup: So sánh timestamp mới với tất cả existing entries trong khoảng, chênh lệch `<= 5 phút` → skip.

### 4.2 Blood Sugar (Đường Huyết)

```
[HealthKit]                              [App DB]
  BLOOD_GLUCOSE (mg/dL) ──→ Convert  ──→ Dedup ──→ sugarRepo.addEntry()
                              mgDl→mmolL   (±5ph)     BloodSugarEntry
```

**Chi tiết**:
- Apple Health trả về đường huyết đơn vị **mg/dL**.
- App nội bộ lưu đơn vị **mmol/L** → gọi `UnitConverterService.mgDlToMmolL()`.
- Dedup tương tự: ±5 phút.

### 4.3 Weight (Cân Nặng)

```
[HealthKit]                    [App DB]
  WEIGHT (kg) ──→ Dedup ──→ weightRepo.addEntry()
                   (±5ph)     WeightEntry(kg)
```

**Chi tiết**:
- Apple Health trả về kg → lưu trực tiếp, không cần convert.
- Dedup ±5 phút.

### 4.4 Steps (Bước Chân)

```
[HealthKit]                                          [App DB]
  STEPS ──→ Tổng hợp theo ngày ──→ Tính phụ sinh ──→ stepRepo.upsertForDate()
             getStepsInRange()        calories          StepEntry
                                      distance
                                      duration
```

**Chi tiết**:
- Không dedup bằng timestamp. Thay vào đó dùng `upsertForDate()` — ghi đè record của ngày đó.
- Tính toán giá trị phụ sinh từ profile user:
  - **Stride** = `heightCm × factor` (Male=0.43, Female=0.415, Other=0.42)
  - **Distance** = `steps × strideCm / 100` (m)
  - **Duration** = `steps / 100` (phút, giả định 100 steps/phút)
  - **Calories** = `weightKg × (distanceMeters / 1000) × 0.7` (kcal)

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
| Threshold | 5 phút (300,000 ms) | Dữ liệu HealthKit thường bị trễ vài phút so với manual input. 5 phút đủ rộng để bắt duplicate nhưng không quá rộng gây mất data hợp lệ |
| Phương pháp | Linear scan `any()` | Duyệt qua toàn bộ existing timestamps |

### Đánh giá cơ chế Dedup

| Tiêu chí | Đánh giá |
|----------|----------|
| Threshold 5 phút | ✅ Hợp lý cho medical data — hiếm khi user đo huyết áp 2 lần trong 5 phút |
| Hiệu suất tìm kiếm | ⚠️ `existingTimestamps.any()` có O(n) cho mỗi record mới. Với 5 năm dữ liệu, Set `existingTimestamps` có thể rất lớn nhưng `.any()` trên `Set<int>` vẫn nhanh vì chỉ compare int |
| Steps không dùng dedup | ✅ Đúng thiết kế — Steps aggregrate theo ngày nên dùng `upsert` phù hợp hơn |

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
      syncedFullHistory = false, // Batch 4
      lastSyncTime = null;
```

- Serialize thành JSON → lưu vào `AppPreferences` (SharedPreferences).
- Mỗi khi 1 batch hoàn tất → cập nhật flag tương ứng → lưu ngay lập tức.
- Khi app restart → load lại progress → skip các batch đã xong.

### 6.3 Guard Flags (Mutex đơn giản)

```dart
bool _isHistorySyncing = false;   // Cho Phase 1 + 2
bool _isIncrementalSyncing = false; // Cho Phase 3
bool get isSyncing => _isHistorySyncing || _isIncrementalSyncing;
```

- Phase 1 và Phase 2 chia sẻ cùng flag `_isHistorySyncing` → không thể chạy đồng thời.
- Phase 3 dùng flag riêng `_isIncrementalSyncing` → có thể overlap với Phase 2 (vì Phase 2 chạy dài, user resume app giữa chừng).

### Đánh giá

| Tiêu chí | Đánh giá |
|----------|----------|
| Persist progress qua restart | ✅ JSON serialization → SharedPrefs |
| Resume batch sau kill | ✅ Skip các batch đã xong |
| Concurrency guard | ✅ Có flag riêng cho từng phase |
| Phase 2 + Phase 3 overlap | ⚠️ `_isIncrementalSyncing` riêng biệt → Incremental có thể chạy đồng thời với Background History. Cả hai cùng ghi vào Repository nhưng dedup sẽ xử lý trùng lặp |

---

## 7. Tương Tác UI (User Interface)

### 7.1 Màn hình Onboarding Sync (`OnboardingSyncHealthPage`)

UI hiển thị khi Phase 1 đang chạy:
- Animation **vòng tròn đồng tâm** (concentric circles) + **dòng số chảy xuống** (data flow) tạo cảm giác dữ liệu đang được truyền từ Apple Health vào app.
- Icon Apple Health ở trên + Logo Heart Champ ở dưới.
- Label trạng thái thay đổi theo `SyncStatus`:
  - `initialSyncing` → "Pulling last 30 days..."
  - `backgroundSyncing` → "Loading 3 months of history..." / "Loading 6 months..." / ...
  - `completed` → "All data synced ✓"
  - `failed` → "Sync failed — will retry"
- Nút "Skip & Continue" cho phép user vào Main ngay khi Phase 1 chưa xong.
- Nút chuyển thành "Continue" khi Initial Sync hoàn tất.

### 7.2 Trạng thái trên Settings / Demo Page

| Trang | Chức năng |
|-------|-----------|
| `HealthSyncDemoPage` | Hiển thị card trạng thái sync (spinning indicator / check icon) + message tương ứng progress |
| `LogDataAppleHealthPage` | Console log dạng terminal — fetch raw data trực tiếp từ HealthKit, hiển thị tất cả records (debug purpose) |

---

## 8. Dual Provider / Dual Controller (Song Song Tồn Tại)

Hiện tại app có **2 bộ điều khiển** cùng tồn tại:

| Thành phần | File | Đặc điểm |
|------------|------|----------|
| **HealthSyncController** | `health_sync_controller.dart` | ✅ Có persist SyncProgress (JSON). Có WidgetsBindingObserver cho incremental. Dùng trong Onboarding. |
| **HealthSyncProvider** | `health_sync_provider.dart` | ⚠️ Không persist progress. Batching logic duplicate. Dùng trong Demo page. |

### Đánh giá

| Tiêu chí | Đánh giá |
|----------|----------|
| Code duplication | ⚠️ Cả hai đều gọi `HealthSyncService` với logic batching tương tự nhưng khác implementation |
| HealthSyncProvider batching | ⚠️ Chỉ đi đến 1 year, không có batch 5 năm. Không persist progress |
| Dùng ở đâu | Controller: Onboarding (production). Provider: Demo page (development) |
| Có lazy load | ✅ `HealthSyncProvider.loadMoreHistoryForRange()` — hook cho biểu đồ cuộn load thêm data theo range |

---

## 9. Tổng Đánh Giá Toàn Diện

### ✅ Điểm Mạnh Nổi Bật

1. **UX phân giai đoạn xuất sắc**: User thấy dữ liệu ngay lập tức (30d), không phải chờ toàn bộ lịch sử. Đây là pattern chuẩn industry (tương tự cách Fitbit, Samsung Health xử lý).

2. **Khả năng chịu lỗi (Fault Tolerance)**: SyncProgress persist qua JSON cho phép resume đúng batch bị gián đoạn. User không bao giờ phải sync lại từ đầu.

3. **Dedup thực chiến**: Threshold 5 phút phù hợp medical data. Steps dùng `upsert` thay vì dedup — đúng bản chất aggregate.

4. **Guard chống race condition**: Flag riêng cho History vs Incremental, check status trước mỗi hàm public.

5. **Notifier pattern**: Sau khi sync xong mỗi loại dữ liệu → notify listeners → UI chart tự cập nhật realtime mà không cần user refresh.

### ⚠️ Điểm Cần Lưu Ý

1. **Batch cuối quá lớn (4 năm)**: `Future.wait` 4 fetch song song × 4 năm dữ liệu → peak memory usage cao. Thiết bị cấu hình thấp có thể bị jetsam kill.

2. **Không phải iOS BGTask**: Toàn bộ sync chạy trên Dart VM khi app ở foreground. User gạt kill app → sync dừng. Đây là giới hạn có chủ đích (đơn giản, không cần native code) nhưng cần lưu ý.

3. **Dual Controller/Provider**: Tồn tại 2 module làm cùng việc → tiềm ẩn confusion khi maintain.

4. **Thiếu quyền WEIGHT + BASAL_ENERGY**: `requestPermissions()` không include 2 type này nhưng code vẫn fetch → phụ thuộc vào user tự grant ở iOS Settings.

5. **Steps chạy vòng for N ngày**: `syncSteps` dùng for-loop gọi `getStepsInRange` cho **từng ngày** → nếu sync 5 năm (1825 ngày) = 1825 lần gọi API HealthKit liên tiếp. Khác với BP/Sugar/Weight chỉ gọi 1 lần `getHealthDataFromTypes` cho toàn bộ range.
