# Gói đặc tả #2 — Bộ giải mã Goertzel

**Tổ Đặc Tả — phụ trách: Nguyễn Minh Hiếu (S2)**
**Đối chiếu lý thuyết: Nguyễn Tiến Mạnh (R2), mục 2.2.1–2.2.3 báo cáo**
**Người triển khai (Coder): dùng gói này để viết `goertzel_power.m` và `dtmf_decode_goertzel.m`**

---

## 0. Vì sao cần Goertzel (bối cảnh)

DFT đầy đủ (FFT) tính công suất tại **tất cả** N/2 bin tần số, tốn O(N log N). Nhưng để nhận diện một phím DTMF, ta chỉ cần công suất tại **đúng 8 tần số** (7 tần số chuẩn + 1 hài bậc 2 để loại nhiễu). Goertzel là một bộ lọc IIR bậc 2 tính **một bin DFT duy nhất** với chi phí O(N) — rẻ hơn FFT khi số bin cần tính nhỏ hơn log₂(N), đúng là trường hợp của DTMF (8 bin trong khi N = 205 → log₂(205) ≈ 7,7). Đây là lý do bài ra yêu cầu "khối lượng tính toán tối thiểu".

---

## 1. Đầu vào / Đầu ra (I/O)

### Hàm lõi: `goertzel_power`

```matlab
P = goertzel_power(x, k, N)
```

| Tên | Vai trò | Kích thước | Kiểu | Đơn vị / miền giá trị |
|---|---|---|---|---|
| `x` | khung tín hiệu đầu vào | 1×N (lấy N mẫu đầu nếu dài hơn) | `double` | biên độ, không đơn vị |
| `k` | chỉ số bin DFT cần tính | 1×1 | `double` (số nguyên) | 0 ≤ k ≤ N−1 |
| `N` | độ dài khung / cỡ DFT ảo | 1×1 | `double` (số nguyên) | mẫu |
| **`P`** *(ra)* | công suất tại bin k | 1×1 | `double` | ≈ \|X[k]\|² |

### Hàm bậc cao: `dtmf_decode_goertzel`

```matlab
[keys, info] = dtmf_decode_goertzel(y, opt)
```

| Tên | Vai trò | Kích thước | Kiểu | Mặc định |
|---|---|---|---|---|
| `y` | tín hiệu cần giải mã | 1×M | `double` | — (bắt buộc) |
| `opt.fs` | tần số lấy mẫu | 1×1 | `double` | 8000 [Hz] |
| `opt.frameN` | độ dài khung Goertzel | 1×1 | `double` | 205 [mẫu] |
| `opt.hop` | bước nhảy giữa 2 khung | 1×1 | `double` | 205 [mẫu] |
| **`keys`** *(ra)* | chuỗi phím giải mã được | 1×K | `char` | — |
| **`info.E`** *(ra)* | công suất 8 bin mỗi khung | 8×nFrame | `double` | hàng 1-4: nhóm hàng, 5-7: nhóm cột, 8: hài bậc 2 |
| **`info.rowIdx/colIdx`** *(ra)* | chỉ số hàng/cột được chọn | 1×nFrame | `double` | 0 nếu bị từ chối |
| **`info.reject`** *(ra)* | lý do từ chối mỗi khung | 1×nFrame | `cellstr` | `'twist'\|'level'\|'harmonic'\|'none'` |

**Ràng buộc bắt buộc (Coder không được đổi):** `goertzel_power` **không** được gọi `fft`, `figure`, `plot` — đúng luật cứng trong `CONTRACTS.md` (hàm trong `src/` chỉ tính toán thuần túy).

---

## 2. Công thức (cơ sở lý thuyết)

DFT tại bin k của khung N mẫu:

```
X[k] = Σ(n=0..N-1) x[n]·e^(-j2πkn/N)
```

Tính trực tiếp tổng này tốn O(N) phép nhân phức — nhưng ta có thể tránh số phức hoàn toàn bằng cách đưa về **bộ lọc đệ quy bậc 2** (đây chính là nội dung Mạnh đã dẫn giải ở mục 2.2.1–2.2.3). Đặt hệ số:

```
c = 2·cos(2πk/N)
```

Vòng lặp trạng thái (khởi tạo s1 = s2 = 0), chạy n = 1..N:

```
s0 = x[n] + c·s1 − s2
s2 ← s1
s1 ← s0
```

Sau N vòng lặp, công suất tại bin k (không cần khai triển số phức):

```
P = s1² + s2² − c·s1·s2      ≈ |X[k]|²
```

**Vì sao công thức này đúng (tóm tắt, chi tiết xem báo cáo của Mạnh):** đây là kết quả của việc phân tích bộ lọc IIR bậc 2 có 2 cực nằm trên vòng tròn đơn vị tại góc ±2πk/N — trạng thái (s1, s2) sau N mẫu chính là biến đổi Z của tín hiệu tại đúng tần số đó, và biểu thức trên là môđun bình phương suy ra từ định lý dư (không cần tính từng số phức trung gian).

**Bin tần số dùng cho DTMF** (N = 205, fs = 8000 Hz, Δf = fs/N ≈ 39,02 Hz):

| Tần số [Hz] | 697 | 770 | 852 | 941 | 1209 | 1336 | 1477 |
|---|---|---|---|---|---|---|---|
| k = round(N·f/fs) | 18 | 20 | 22 | 24 | 31 | 34 | 38 |

Bin thứ 8 (hài bậc 2, để phát hiện méo/nhiễu hài): `kHarm = min(2·k_dominant, floor(N/2))`, trong đó `k_dominant` là bin có công suất lớn nhất trong 7 bin trên (chọn động theo từng khung, vì hài bậc 2 phụ thuộc tần số nào đang trội).

---

## 3. Ví dụ số kiểm chứng

### 3.1 Kiểm chứng `goertzel_power` (đối chiếu lý thuyết thuần túy)

```
N = 16, k = 3, x[n] = cos(2π·3n/16),  n = 0..15
```

Vì x[n] là đúng một hình sin thuần tại bin k = 3, theo lý thuyết DFT:

```
X[3] = N/2 = 8   ⇒   |X[3]|² = 64
```

Chạy `goertzel_power(x, 3, 16)` → **P = 64,000000** — khớp lý thuyết, sai lệch < 1e-9 (đã kiểm chứng bằng Octave, xem `tests/test_goertzel.m`).

### 3.2 Kiểm chứng `dtmf_decode_goertzel` (đối chiếu luật quyết định)

Giả sử tại một khung, tín hiệu là tổng đúng 697 Hz (hàng 1) + 1336 Hz (cột 2) — tức phím **"5"**. Vector công suất 8 bin đo được (đơn vị tương đối):

```
E = [1, 8, 1, 1, 1, 9, 1, 0.1]ᵀ
    ↑hàng 1-4↑      ↑cột 5-7↑ ↑hài↑
```

Áp luật `dtmf_decide`:
- Đỉnh nhóm hàng = 8 (bin 1) so với đỉnh nhì = 1 → chênh 10·log10(8/1) = 9,03 dB ≥ 6 dB ✓
- Đỉnh nhóm cột = 9 (bin 2 trong nhóm) so với đỉnh nhì = 1 → chênh 9,54 dB ≥ 6 dB ✓
- Twist = 10·log10(9/8) = 0,51 dB, nằm trong [−8, +4] dB ✓
- Tỉ lệ năng lượng: (8+9)/Σ(E) = 17/22,1 ≈ 76,9% ≥ 70% ✓
- Hài bậc 2 = 0,1 ≪ 0,5·min(8,9) = 4 → không phải hài ✓

**→ Kết quả mong đợi: rowIdx = 1, colIdx = 2 → phím "5", reject = 'none'.**

Đã chạy kiểm chứng thực tế bằng Octave trên toàn bộ pipeline (sinh tín hiệu → cộng nhiễu AWGN 20 dB → Goertzel → quyết định): giải mã đúng 100% chuỗi kiểm thử `'1590*#'`.

---

## 4. Tiêu chí xong (Definition of Done)

Gói này (và code dự bị đi kèm) được coi là **xong** khi:

- [ ] `goertzel_power(x, 3, 16)` với `x = cos(2π·3n/16)` trả về `P = 64` với sai lệch `< 1e-9`.
- [ ] `dtmf_decode_goertzel` không gọi bất kỳ hàm nào trong danh sách cấm (`figure`, `plot`, `disp`, `sound`, `input`, và **không dùng `fft`** — nếu dùng fft thì không còn là "Goertzel" nữa, mất hết ý nghĩa tối ưu).
- [ ] Với tín hiệu sạch (không nhiễu), giải mã đúng 100% một chuỗi kiểm thử ≥ 6 ký tự bất kỳ trong bộ `0-9,*,#`.
- [ ] Với tín hiệu có nhiễu AWGN, SNR ≥ 15 dB, giải mã đúng ≥ 95% ký tự (đo bằng `dtmf_metrics.acc`).
- [ ] Không có yêu cầu Toolbox nào ngoài MATLAB lõi (không `tukeywin`, không `hamming` của Signal Processing Toolbox bên trong hàm này — máy một số bạn trong nhóm không có toolbox này, đã xác nhận thực tế).
- [ ] Có ít nhất 1 test case trong `tests/test_goertzel.m` chạy qua `run_all_tests.m` không lỗi.

---

## 5. Code dự bị (backup implementation)

Đây là bản cài đặt tham chiếu, đã kiểm thử bằng Octave (khớp 100% ví dụ số ở mục 3). Coder có thể dùng trực tiếp hoặc đối chiếu với bản của mình.

### `src/decode/goertzel_power.m`

```matlab
function P = goertzel_power(x, k, N)
%GOERTZEL_POWER Cong suat tai bin DFT thu k bang thuat toan Goertzel.
%   P = GOERTZEL_POWER(X, K, N) = |X[k]|^2, cai dat theo cong thuc hoi
%   quy bac 2: s0 = x[n] + c*s1 - s2 (c = 2*cos(2*pi*k/N)), lap N lan,
%   P = s1^2 + s2^2 - c*s1*s2.
%
%   Da kiem chung: N=16, k=3, x=cos(2*pi*3*n/16) -> P=64.000000
%   (sai lech < 1e-9 so voi ly thuyet X[3]=N/2=8 => |X[3]|^2=64).
%
%   See also dtmf_decode_goertzel.

arguments
    x (1,:) double
    k (1,1) double {mustBeInteger, mustBeNonnegative}
    N (1,1) double {mustBeInteger, mustBePositive}
end

frame = x(1:N);
c = 2*cos(2*pi*k/N);
s1 = 0; s2 = 0;
for n = 1:N
    s0 = frame(n) + c*s1 - s2;
    s2 = s1;
    s1 = s0;
end
P = s1^2 + s2^2 - c*s1*s2;

end
```

### `src/decode/dtmf_decode_goertzel.m`

```matlab
function [keys, info] = dtmf_decode_goertzel(y, opt)
%DTMF_DECODE_GOERTZEL Giai ma DTMF bang thuat toan Goertzel.
%   Chia y thanh cac khung frameN mau, moi khung tinh cong suat 7 tan so
%   chuan (goertzel_power) + 1 bin hai bac 2 dong (theo tan so troi nhat
%   trong khung), roi ap dtmf_decide de ra phim. Debounce: cac khung lien
%   tiep cung phim chi tinh 1 lan.
%
%   [KEYS, INFO] = DTMF_DECODE_GOERTZEL(Y, 'fs',8000, 'frameN',205, 'hop',205)
%
%   See also goertzel_power, dtmf_segment, dtmf_decide.

arguments
    y (1,:) double
    opt.fs (1,1) double = 8000
    opt.frameN (1,1) double = 205
    opt.hop (1,1) double = 205
end

freqs = [697 770 852 941 1209 1336 1477];
k = round(opt.frameN * freqs / opt.fs);

seg = dtmf_segment(y, 'fs', opt.fs, 'frameN', opt.frameN, 'hop', opt.hop);
nFrame = numel(seg);

E = zeros(8, nFrame);
rowIdx = zeros(1, nFrame);
colIdx = zeros(1, nFrame);
conf = zeros(1, nFrame);
reject = cell(1, nFrame);
tFrame = zeros(1, nFrame);

T = dtmf_table();

for i = 1:nFrame
    frame = y(seg(i).idx(1):seg(i).idx(2));
    for j = 1:7
        E(j, i) = goertzel_power(frame, k(j), opt.frameN);
    end
    [~, dominant] = max(E(1:7, i));
    kHarm = min(2*k(dominant), floor(opt.frameN/2));
    E(8, i) = goertzel_power(frame, kHarm, opt.frameN);

    tFrame(i) = seg(i).tStart;
    [r, c, cf, rej] = dtmf_decide(E(:, i));
    rowIdx(i) = r; colIdx(i) = c; conf(i) = cf; reject{i} = rej;
end

keys = '';
prevKey = '';
for i = 1:nFrame
    if rowIdx(i) > 0 && colIdx(i) > 0
        thisKey = T.keys(rowIdx(i), colIdx(i));
        if ~strcmp(thisKey, prevKey)
            keys = [keys thisKey]; %#ok<AGROW>
        end
        prevKey = thisKey;
    else
        prevKey = '';
    end
end

info = struct('E', E, 'rowIdx', rowIdx, 'colIdx', colIdx, 'conf', conf, ...
              'tFrame', tFrame, 'reject', {reject});

end
```

---

## 6. Câu hỏi thường gặp khi bảo vệ (chuẩn bị trước cho buổi demo)

**Q: Vì sao không dùng FFT cho tất cả, viết Goertzel làm gì cho phức tạp?**
A: FFT tính toàn bộ N/2 bin dù chỉ cần 8 bin — lãng phí. Goertzel là lựa chọn kinh điển trong thực tế các chip giải mã DTMF phần cứng (điện thoại bàn, tổng đài) chính vì lý do này: chi phí tính toán thấp, phù hợp hệ thống nhúng tài nguyên hạn chế.

**Q: Tại sao công thức Goertzel không cần số phức mà vẫn ra đúng |X[k]|²?**
A: Vì bộ lọc đệ quy bậc 2 với 2 cực liên hợp phức trên vòng tròn đơn vị có đáp ứng tương đương phép chiếu tín hiệu lên đúng tần số k; trạng thái cuối (s1, s2) mã hóa đủ thông tin biên độ + pha, và biểu thức `s1²+s2²-c·s1·s2` chính là môđun bình phương rút gọn — không cần tính riêng phần thực/ảo.

**Q: Vì sao bin hài bậc 2 lại chọn động theo từng khung thay vì cố định?**
A: Vì hài bậc 2 chỉ có ý nghĩa so với tần số nào đang trội trong khung đó — nếu cố định một tần số hài, sẽ không phát hiện được méo hài khi tín hiệu chuyển từ phím này sang phím khác.
