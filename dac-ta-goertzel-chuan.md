## Đặc tả: Bộ giải mã Goertzel

### Nguyên lý

DFT tại bin $k$ của khung $N$ mẫu:

$$X[k] = \sum_{n=0}^{N-1} x[n]\, e^{-j2\pi kn/N}$$

Thay vì tính trực tiếp (số phức, $O(N)$ phép nhân phức cho **mỗi** bin), Goertzel đưa về bộ lọc đệ quy bậc 2 với hệ số:

$$c = 2\cos(2\pi k/N)$$

Cập nhật trạng thái (khởi tạo $s_1 = s_2 = 0$), lặp $n = 1..N$:

$$s_0 = x[n] + c\,s_1 - s_2, \qquad s_2 \leftarrow s_1,\quad s_1 \leftarrow s_0$$

Công suất tại bin $k$ sau $N$ vòng lặp:

$$P = s_1^2 + s_2^2 - c\,s_1 s_2 \;\approx\; |X[k]|^2$$

Vì chỉ cần tính đúng 8 bin (7 tần số chuẩn + 1 hài bậc 2) thay vì toàn bộ phổ, chi phí $O(KN)$ với $K=8$ rẻ hơn FFT khi $K < \log_2 N$.

### Tham số

| Tham số | Giá trị | Ghi chú |
|---|---|---|
| $N$ (frameN) | 205 mẫu | $\Delta f = f_s/N \approx 39{,}02$ Hz |
| $f_s$ | 8000 Hz | — |
| Bin hàng $k$ | 697→18, 770→20, 852→22, 941→24 | $k = \mathrm{round}(N f/f_s)$ |
| Bin cột $k$ | 1209→31, 1336→34, 1477→38 | như trên |
| Bin hài bậc 2 | $k_{harm} = \min(2k_{dominant}, \lfloor N/2 \rfloor)$ | chọn động theo tần số trội nhất khung |

### I/O

```matlab
P = goertzel_power(x, k, N)
```

| Biến | Kích thước | Kiểu | Đơn vị |
|---|---|---|---|
| `x` | 1×N | double | biên độ |
| `k` | 1×1 | double (nguyên) | chỉ số bin, 0 ≤ k ≤ N−1 |
| `N` | 1×1 | double (nguyên) | mẫu |
| `P` (ra) | 1×1 | double | ≈ \|X[k]\|² |

```matlab
[keys, info] = dtmf_decode_goertzel(y, opt)
```

| Biến | Kích thước | Kiểu | Mặc định |
|---|---|---|---|
| `y` | 1×M | double | — |
| `opt.fs` | 1×1 | double | 8000 [Hz] |
| `opt.frameN` | 1×1 | double | 205 [mẫu] |
| `opt.hop` | 1×1 | double | 205 [mẫu] |
| `keys` (ra) | 1×K | char | — |
| `info.E` (ra) | 8×nFrame | double | hàng 1-4 nhóm hàng, 5-7 nhóm cột, 8 hài bậc 2 |
| `info.reject` (ra) | 1×nFrame | cellstr | `'twist'\|'level'\|'harmonic'\|'none'` |

### Ví dụ số kiểm chứng

```
N = 16, k = 3, x[n] = cos(2π·3n/16), n = 0..15
Lý thuyết: X[3] = N/2 = 8  ⇒  |X[3]|² = 64
goertzel_power(x, 3, 16) → P = 64,000000   (sai lệch < 1e-9)
```

Kiểm chứng luật quyết định — khung ứng với phím "5" (697 Hz + 1336 Hz):

```
E = [1, 8, 1, 1, 1, 9, 1, 0.1]ᵀ

Đỉnh nhóm hàng: 8 vs 1  → 10·log10(8/1) = 9,03 dB ≥ 6 dB
Đỉnh nhóm cột:  9 vs 1  → 10·log10(9/1) = 9,54 dB ≥ 6 dB
Twist:          10·log10(9/8) = 0,51 dB  ∈ [−8, 4] dB
Tỉ lệ năng lượng: (8+9)/Σ(E) = 17/22,1 ≈ 76,9% ≥ 70%
Hài bậc 2:      0,1 ≪ 0,5·min(8,9) = 4

→ rowIdx = 1, colIdx = 2, reject = 'none'  ⇒  phím "5"
```

### Tiêu chí xong

- `goertzel_power(cos(2π·3n/16), 3, 16)` trả về `P = 64` với sai lệch < 1e-9.
- Không gọi `fft`, `figure`, `plot`, `disp`, `sound`, `input`.
- Tín hiệu sạch: giải mã đúng 100% chuỗi kiểm thử ≥ 6 ký tự.
- SNR ≥ 15 dB: giải mã đúng ≥ 95% ký tự.
- Không phụ thuộc Signal Processing Toolbox.
- Có test case trong `tests/test_goertzel.m`, chạy qua `run_all_tests.m` không lỗi.
