# Gói đặc tả #4 — Bộ giải mã Goertzel (tổ S2 + R2)

## Phạm vi

Đặc tả 2 hàm trong `src/decode/`:
- `goertzel_power.m` — hàm lõi, tính công suất tại 1 bin DFT.
- `dtmf_decode_goertzel.m` — hàm bậc cao, dùng `goertzel_power` để giải mã cả chuỗi phím.

Cả hai đang ở trạng thái TODO trong repo (`error('...:notImplemented', ...)`), chữ ký hàm và ràng buộc `arguments` đã cố định theo `CONTRACTS.md` — đặc tả này KHÔNG được đổi chữ ký hàm, chỉ làm rõ nội dung Coder cần điền vào.

---

## A. `goertzel_power`

### 1. Dòng H1
```
%GOERTZEL_POWER Công suất tại bin DFT thứ k bằng thuật toán Goertzel.
```

### 2. Cú pháp gọi
```matlab
P = goertzel_power(x, k, N)
```
Tính `P = |X[k]|^2`, trong đó `X[k]` là hệ số DFT N điểm của khung `x`, bằng bộ lọc IIR bậc 2 (không cần số phức, không cần `fft`).

### 3. Đầu vào / Đầu ra

| Tên | Kích thước | Kiểu | Mô tả | Đơn vị |
|---|---|---|---|---|
| `x` | 1×M | `double` | một khung tín hiệu; chỉ dùng N mẫu đầu (M ≥ N) | biên độ |
| `k` | 1×1 | `double`, số nguyên không âm | chỉ số bin; tần số tương ứng `f_k = k*fs/N` | [Hz] (gián tiếp) |
| `N` | 1×1 | `double`, số nguyên dương | độ dài khung (số điểm DFT) | mẫu |
| `P` *(ra)* | 1×1 | `double`, ≥ 0 | công suất tại bin k | ≈ \|X[k]\|² |

### 4. Cơ sở lý thuyết
Chỉ số `n` tính từ 0 (`x[n]` ứng với `x(n+1)` trong MATLAB):

```
c    = 2*cos(2*pi*k/N)
s[n] = x[n] + c*s[n-1] - s[n-2],   n = 0..N-1,   s[-1] = s[-2] = 0
P    = s[N-1]^2 + s[N-2]^2 - c*s[N-1]*s[N-2]
```

Độ phức tạp O(N) cho mỗi bin, chỉ một phép nhân thực mỗi mẫu — hiệu quả hơn FFT (O(N log N) cho toàn phổ) khi chỉ cần một số ít bin (ở đây là 8 bin trong khi log₂(205) ≈ 7,7).

### 5. Ví dụ (kiểm chứng)
```matlab
n = 0:15; x = cos(2*pi*3*n/16);
P = goertzel_power(x, 3, 16);   % P = 64.000000 (sai số < 1e-9)
% Đối chiếu lý thuyết: X[3] = N/2 = 8  =>  |X[3]|^2 = 64.
```
Đã chạy thực tế bằng Octave, kết quả đúng khớp `P = 64.000000`.

### 6. Tham khảo
```
[1] G. Goertzel, "An algorithm for the evaluation of finite
    trigonometric series," Amer. Math. Monthly, vol. 65, no. 1,
    pp. 34–35, 1958.
[2] J. G. Proakis, D. G. Manolakis, Digital Signal Processing:
    Principles, Algorithms, and Applications, 4th ed., Pearson, 2007.
[3] Gói đặc tả #4 (tổ S2 + R2).
```

### 7. See also
`dtmf_decode_goertzel`, `goertzel`

---

## B. `dtmf_decode_goertzel`

### 1. Dòng H1
```
%DTMF_DECODE_GOERTZEL Giải mã DTMF bằng thuật toán Goertzel.
```

### 2. Cú pháp gọi
```matlab
[keys, info] = dtmf_decode_goertzel(y, opt)
```
Chữ ký hàm GIỐNG HỆT `dtmf_decode_fft` và `dtmf_decode_filterbank` (dùng chung để so sánh công bằng 3 phương pháp).

### 3. Đầu vào / Tham số / Đầu ra

| Tên | Kích thước | Kiểu | Mặc định | Mô tả |
|---|---|---|---|---|
| `y` | 1×N | `double` | — (bắt buộc) | tín hiệu cần giải mã |
| `opt.fs` | 1×1 | `double` | 8000 [Hz] | tần số lấy mẫu |
| `opt.frameN` | 1×1 | `double` | 205 [mẫu] | độ dài khung; Δf = fs/N = 8000/205 ≈ 39,02 Hz |
| `opt.hop` | 1×1 | `double` | 205 [mẫu] | bước nhảy giữa 2 khung liên tiếp |
| `keys` *(ra)* | 1×K | `char` | — | chuỗi phím giải mã được |
| `info.E` *(ra)* | 8×nFrame | `double` | — | công suất 8 bin mỗi khung (hàng 1-4: nhóm hàng, 5-7: nhóm cột, 8: hài bậc 2) |
| `info.rowIdx` / `info.colIdx` *(ra)* | 1×nFrame | `double` | — | chỉ số hàng/cột được chọn, 0 nếu khung bị từ chối |
| `info.conf` *(ra)* | 1×nFrame | `double` | — | độ tin cậy, 0–1 |
| `info.tFrame` *(ra)* | 1×nFrame | `double` | — | thời điểm bắt đầu mỗi khung [s] |
| `info.reject` *(ra)* | 1×nFrame | `cellstr` | — | `'twist'` \| `'level'` \| `'harmonic'` \| `'none'` |

### 4. Cơ sở lý thuyết

Bảng chỉ số bin chuẩn, `k = round(N*f/fs)` với N = 205, fs = 8000 Hz:

```
Nhóm hàng:  697 -> 18   770 -> 20   852 -> 22   941 -> 24
Nhóm cột:  1209 -> 31  1336 -> 34  1477 -> 38
```

Quy trình xử lý mỗi khung:
1. `seg = dtmf_segment(y, 'fs',fs, 'frameN',frameN, 'hop',hop)` — chia y thành các khung.
2. Với mỗi khung, tính `E(j,i) = goertzel_power(frame, k(j), frameN)` cho j = 1..7 (7 tần số chuẩn).
3. Xác định tần số trội nhất trong 7 bin (`[~, dominant] = max(E(1:7,i))`), tính bin hài bậc 2 **động**: `kHarm = min(2*k(dominant), floor(frameN/2))` — không cố định vì hài bậc 2 chỉ có ý nghĩa so với tần số nào đang trội trong khung đó.
4. `E(8,i) = goertzel_power(frame, kHarm, frameN)`.
5. `[rowIdx, colIdx, conf, reject] = dtmf_decide(E(:,i))` — áp luật quyết định chung của cả 3 bộ giải mã (xem `src/util/dtmf_decide.m`): đỉnh mỗi nhóm phải cao hơn đỉnh thứ nhì ≥ 6 dB; twist thuận ≤ 4 dB, nghịch ≤ 8 dB; tổng 8 bin ≥ 70% năng lượng khung; hài bậc 2 đủ nhỏ.
6. Chống dội (debounce): các khung liên tiếp cho cùng một phím chỉ tính là 1 ký tự trong `keys` (dùng biến `prevKey`, reset về rỗng khi gặp khung bị từ chối).

### 5. Ví dụ
Khung ứng với phím "5" (697 Hz + 1336 Hz), vector công suất đo được:
```matlab
E = [1, 8, 1, 1, 1, 9, 1, 0.1]';
[rowIdx, colIdx, conf, reject] = dtmf_decide(E);
% rowIdx = 1, colIdx = 2, reject = 'none'  -> phím "5"
```
Kiểm chứng cả pipeline: sinh tín hiệu sạch chuỗi `'1590*#'` → `dtmf_decode_goertzel` → giải mã đúng lại `'1590*#'`. Với AWGN SNR = 20 dB, vẫn giải mã đúng 100%.

### 6. Tham khảo
```
[1] G. Goertzel, "An algorithm for the evaluation of finite
    trigonometric series," Amer. Math. Monthly, vol. 65, no. 1,
    pp. 34–35, 1958.
[2] Gói đặc tả #4 (tổ S2 + R2).
```

### 7. See also
`goertzel_power`, `dtmf_segment`, `dtmf_decide`, `dtmf_decode_fft`

**Lưu ý bắt buộc:** không gọi `figure`/`plot`/`disp`/`sound`/`input` trong hàm này (luật cứng `CONTRACTS.md`) — vẽ và phát âm thanh chỉ được làm trong `app/ui/*.m`.

---

## Tiêu chí xong (dùng để chấm "hoàn thành", đối chiếu `tests/test_goertzel.m`)

- [ ] `goertzel_power(cos(2*pi*3*n/16), 3, 16)` (n=0:15) trả về `P = 64` với `AbsTol = 1e-9`.
- [ ] `goertzel_power` khớp hàm `goertzel()` gốc của MATLAB trên tín hiệu ngẫu nhiên (`rng(1); N=205; k=18; x=randn(1,N)`), sai số tương đối `RelTol = 1e-10` — lưu ý `goertzel()` dùng chỉ số bin 1-based (`k+1`).
- [ ] `dtmf_decode_goertzel` không dùng `fft` (mất ý nghĩa tối ưu nếu dùng) và không dùng `figure/plot/disp/sound/input`.
- [ ] Tín hiệu sạch: giải mã đúng 100% một chuỗi kiểm thử ≥ 6 ký tự bất kỳ trong `0-9,*,#`.
- [ ] Tín hiệu có nhiễu AWGN, SNR ≥ 15 dB: giải mã đúng ≥ 95% ký tự (đo bằng `dtmf_metrics.acc`).
- [ ] Không phụ thuộc Signal Processing Toolbox (không `hamming`, không `tukeywin`, không `fft`-based).
- [ ] `tests/test_goertzel.m` chạy qua `run_all_tests.m` không lỗi.
