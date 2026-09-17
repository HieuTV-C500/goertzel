# Gói đặc tả #2 — Khung FFT đối chứng

---

## I/O

**Đầu vào**

| Tên | Kích thước | Kiểu | Ghi chú |
|---|---|---|---|
| `y` | 1×L | `double` | Tín hiệu cần giải mã (đơn vị biên độ tùy chuẩn hoá đầu vào) |
| `opt.fs` | 1×1 | `double` | Tần số lấy mẫu (mặc định 8000) [Hz] |
| `opt.frameN` | 1×1 | `double` | Độ dài khung = số điểm FFT (mặc định 256) [mẫu] |
| `opt.hop` | 1×1 | `double` | Bước nhảy giữa hai khung liên tiếp (mặc định 128) [mẫu], chồng lấp 50% |

**Đầu ra**

| Tên | Kích thước | Kiểu | Ghi chú |
|---|---|---|---|
| `keys` | 1×K | `char` | Chuỗi phím giải mã, theo thứ tự thời gian |
| `info.E` | 8×nFrame | `double` | Công suất tại 7 tần số chuẩn + 1 hài bậc 2, mỗi khung |
| `info.rowIdx` | 1×nFrame | `double` | Chỉ số hàng 1..4 (0: không quyết định) |
| `info.colIdx` | 1×nFrame | `double` | Chỉ số cột 1..3 (0: không quyết định) |
| `info.conf` | 1×nFrame | `double` | Độ tin cậy ∈ [0, 1] |
| `info.tFrame` | 1×nFrame | `double` | Thời điểm của khung [s] |
| `info.reject` | 1×nFrame | `cellstr` | `'none'` \| `'twist'` \| `'level'` \| `'harmonic'` |

Chữ ký hàm: `[keys, info] = dtmf_decode_fft(y, opt)` — giống hệt `dtmf_decode_goertzel` và `dtmf_decode_filterbank` (xem `CONTRACTS.md`).

---

## Công thức

**1) Cửa sổ Hamming**

$$
w[n] =
\begin{cases}
0.54 - 0.46\cos\left(\dfrac{2\pi n}{N-1}\right) & 0 \le n \le N-1 \\
0 & n \text{ còn lại}
\end{cases}
$$

```
        ⎧ 0.54 − 0.46·cos(2·π·n / (N−1))  ,  0 ≤ n ≤ N−1
w[n] =  ⎨
        ⎩ 0                                ,  n còn lại
```

**2) Phổ FFT của khung đã cửa sổ**

$$
X[k] = \mathrm{FFT}\big(w[n]\cdot x[n],\ N\big), \qquad k = 0,1,\dots,N-1
$$

```
X[k] = fft(w .* frame, N)   ,   k = 0, 1, ..., N-1
```

**3) Công suất tại bin k** (cùng thang đo với `goertzel_power`, dùng `|X|²` chứ không phải `|X|`)

$$
E[k] = \big|X[k]\big|^2
$$

```
E[k] = |X[k]|^2
```

**4) Độ phân giải tần số**

$$
\Delta f = \frac{f_s}{N} = \frac{8000}{256} = 31.25 \ \text{Hz}
$$

```
Δf = fs / N = 8000 / 256 = 31.25 Hz
```

**5) Bin gần nhất với tần số chuẩn f**

$$
k = \mathrm{round}\left(\frac{f \cdot N}{f_s}\right)
$$

```
k = round(f * N / fs)
```

**6) % sai lệch giữa bin thực tế và tần số chuẩn**

$$
f_k = k \cdot \Delta f, \qquad
\text{sai lệch } = \frac{f_k - f}{f}\times 100
$$

```
f_k        = k * Δf
sai_lech_% = (f_k - f) / f * 100
```



Dung sai theo `CONTRACTS.md`: **nhận** nếu |sai lệch| ≤ 1,5% · **từ chối** nếu |sai lệch| ≥ 3,5%.

---

## Ví dụ số kiểm chứng

Với `fs = 8000 Hz`, `N = 256` ⇒ `Δf = 31.25 Hz`. Bảng bin k cho đủ 7 tần số chuẩn DTMF, kèm % sai lệch đã tính sẵn:

| Nhóm | f chuẩn [Hz] | k = round(f·N/fs) | f_k = k·Δf [Hz] | Sai lệch [%] | Đánh giá |
|---|---:|---:|---:|---:|---|
| Hàng | 697 | 22 | 687,50 | −1,363% | Đạt (≤1,5%) |
| Hàng | 770 | 25 | 781,25 | +1,461% | Đạt (≤1,5%) |
| Hàng | 852 | 27 | 843,75 | −0,968% | Đạt (≤1,5%) |
| Hàng | 941 | 30 | 937,50 | −0,372% | Đạt (≤1,5%) |
| Cột | 1209 | 39 | 1218,75 | +0,806% | Đạt (≤1,5%) |
| Cột | 1336 | 43 | 1343,75 | +0,580% | Đạt (≤1,5%) |
| Cột | 1477 | 47 | 1468,75 | −0,559% | Đạt (≤1,5%) |

**Diễn giải ví dụ (tần số 770 Hz, hàng thứ 2):**

$$
k_{\text{exact}} = \frac{770 \times 256}{8000} = 24.640 \ \Rightarrow\ k = \mathrm{round}(24.640) = 25
$$

$$
f_{25} = 25 \times 31.25 = 781.25 \ \text{Hz}, \qquad
\text{sai lệch} = \frac{781.25 - 770}{770}\times 100 = +1.461\%
$$

```
k_exact = 770 * 256 / 8000 = 24,640
k       = round(24,640) = 25
f_25    = 25 * 31,25 = 781,25 Hz
sai_lech = (781,25 - 770) / 770 * 100 = +1,461%
```

→ 1,461% < 1,5% ⇒ bin k = 25 được **chấp nhận** làm đại diện cho tần số 770 Hz. Đây là mức sai lệch lớn nhất trong bảng (do 770 Hz nằm gần giữa hai bin FFT liên tiếp nhất), nhưng vẫn nằm trong dung sai nhận.

*Ghi chú:* index bin tính từ 0 (theo quy ước DFT `k = 0,1,...,N-1`, không phải MATLAB 1-based); khi lập trình cần cộng 1 khi truy xuất `X(k+1)` trong MATLAB.

---

## Tiêu chí xong

- [x] Đủ 4 phần bắt buộc: I/O · công thức · ví dụ số kiểm chứng · tiêu chí xong.
- [x] Bảng bin k có đủ 7 tần số chuẩn (4 hàng + 3 cột), mỗi dòng ghi k, f_k và % sai lệch.
- [x] Toàn bộ 7 sai lệch đều ≤ 1,5% (khớp dung sai "nhận" trong `CONTRACTS.md`); không có dòng nào cần cảnh báo hoặc bị từ chối.
- [x] Công thức Hamming và Δf viết đúng cú pháp MATLAB, chỉ rõ đơn vị `[Hz]`, `[mẫu]`.
- [x] Chữ ký hàm khớp `dtmf_decode_fft(y, opt)` trong `CONTRACTS.md` (cùng dạng với goertzel/filterbank).
- [x] Kích thước và kiểu của mọi biến I/O được ghi rõ (`1×N`, `double`, `char`…).
- [ ] Coder xác nhận bảng E (8×nFrame, gồm cả bin hài bậc 2) khớp với `dtmf_decide.m` trước khi cắm nối (ngoài phạm vi Gói #2, chỉ cần bàn giao đúng công thức + bảng k ở trên cho tổ Coder).
