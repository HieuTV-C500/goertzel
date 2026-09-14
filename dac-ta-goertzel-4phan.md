# Bộ giải mã Goertzel

## I/O

```matlab
P = goertzel_power(x, k, N)
```

| Tên | Kích thước | Kiểu | Mô tả |
|---|---|---|---|
| `x` | 1×M | double | khung tín hiệu, chỉ dùng N mẫu đầu (M ≥ N) |
| `k` | 1×1 | double, nguyên ≥ 0 | chỉ số bin |
| `N` | 1×1 | double, nguyên > 0 | độ dài khung |
| `P` (ra) | 1×1 | double | công suất tại bin k, ≈ \|X[k]\|² |

```matlab
[keys, info] = dtmf_decode_goertzel(y, opt)
```

| Tên | Kích thước | Kiểu | Mặc định |
|---|---|---|---|
| `y` | 1×N | double | — |
| `opt.fs` | 1×1 | double | 8000 [Hz] |
| `opt.frameN` | 1×1 | double | 205 [mẫu] |
| `opt.hop` | 1×1 | double | 205 [mẫu] |
| `keys` (ra) | 1×K | char | chuỗi phím giải mã |
| `info.E` (ra) | 8×nFrame | double | công suất 8 bin (1-4 hàng, 5-7 cột, 8 hài bậc 2) |
| `info.rowIdx`/`info.colIdx` (ra) | 1×nFrame | double | 0 nếu khung bị từ chối |
| `info.reject` (ra) | 1×nFrame | cellstr | `'twist'\|'level'\|'harmonic'\|'none'` |

## Công thức

```
c    = 2*cos(2*pi*k/N)
s[n] = x[n] + c*s[n-1] - s[n-2],   n = 0..N-1,   s[-1] = s[-2] = 0
P    = s[N-1]^2 + s[N-2]^2 - c*s[N-1]*s[N-2]
```

**Bin chuẩn** (N=205, fs=8000): hàng 697→18, 770→20, 852→22, 941→24; cột 1209→31, 1336→34, 1477→38 — tính theo `k = round(N·f/fs)`.

**Bin hài bậc 2:** `kHarm = min(2*k_dominant, floor(N/2))`, trong đó `k_dominant` là bin có công suất lớn nhất trong 7 bin chuẩn của khung đang xét (`[~, dominant] = max(E(1:7))`). Chọn **động theo từng khung** — không cố định — vì mỗi khung có thể ứng với một phím khác nhau, hài bậc 2 chỉ có ý nghĩa so với đúng tần số đang trội. Nhân đôi vì hài bậc 2 = gấp đôi tần số cơ bản (k tỉ lệ thuận f); `min(..., floor(N/2))` để chặn không vượt tần số Nyquist, tránh bin bị gập ngược (aliasing).

## Ví dụ số kiểm chứng

```
N = 16, k = 3, x[n] = cos(2π·3n/16), n = 0..15
Lý thuyết: X[3] = N/2 = 8  ⇒  |X[3]|² = 64
goertzel_power(x, 3, 16) → P = 64,000000   (sai lệch < 1e-9)
```

```
E = [1, 8, 1, 1, 1, 9, 1, 0.1]ᵀ   (khung ứng với phím "5": 697Hz + 1336Hz)

Đỉnh hàng: 8 vs 1 → 9,03 dB ≥ 6 dB
Đỉnh cột:  9 vs 1 → 9,54 dB ≥ 6 dB
Twist:     10·log10(9/8) = 0,51 dB ∈ [−8, 4] dB
Năng lượng: (8+9)/22,1 ≈ 76,9% ≥ 70%
Hài bậc 2: 0,1 ≪ 0,5·min(8,9) = 4

→ rowIdx=1, colIdx=2, reject='none' ⇒ phím "5"
```

## Tiêu chí xong

- `goertzel_power(cos(2π·3n/16), 3, 16)` trả về `P = 64`, sai lệch < 1e-9.
- Khớp hàm `goertzel()` gốc của MATLAB trên tín hiệu ngẫu nhiên, sai số tương đối < 1e-10.
- Không gọi `fft`, `figure`, `plot`, `disp`, `sound`, `input`.
- Tín hiệu sạch: giải mã đúng 100% chuỗi kiểm thử ≥ 6 ký tự.
- SNR ≥ 15 dB: giải mã đúng ≥ 95% ký tự.
- Không phụ thuộc Signal Processing Toolbox.
- Chạy qua `tests/test_goertzel.m` trong `run_all_tests.m` không lỗi.
