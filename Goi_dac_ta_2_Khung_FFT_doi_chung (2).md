# Gói đặc tả #2 — Khung FFT đối chứng

**Dự án:** Hệ thống phát và giải mã tín hiệu DTMF
**Tuần:** T1 — hạn 18/09
**Người soạn:** Nguyễn Minh Hiếu (D55.B9.AT.011), cùng Minh Tân
**Liên quan file mã nguồn:** `src/decode/dtmf_decode_fft.m`

---

## 1. Mục đích

Gói #2 mô tả cách bộ giải mã FFT chia khung tín hiệu và trích công suất phổ,
để dùng làm **chuẩn đối chứng** khi so sánh độ chính xác với hai bộ giải mã
còn lại (Goertzel, ngân hàng bộ lọc). Cả ba bộ giải mã dùng chung một hàm
chia khung (`dtmf_segment`) và một luật quyết định (`dtmf_decide`), nên phải
thống nhất tham số ngay từ gói đặc tả này.

## 2. Vì sao phải dùng cửa sổ

DFT ngầm giả định đoạn tín hiệu được lấy là tuần hoàn. Nếu độ dài khung N
không phải bội số nguyên của chu kỳ tín hiệu, khi nối khung này với chính
nó sẽ xuất hiện điểm gián đoạn biên độ, sinh ra các thành phần tần số giả
(không có trong tín hiệu gốc) — gọi là rò rỉ phổ (spectral leakage). Biên độ
gián đoạn càng lớn, rò rỉ càng nhiều.

Vì hai tần số DTMF trong một khung hiếm khi tạo đủ số chu kỳ nguyên trong
N = 256 mẫu, khung FFT đối chứng bắt buộc phải nhân cửa sổ trước khi biến
đổi Fourier, để giảm gián đoạn ở hai biên khung và giảm rò rỉ phổ.

## 3. Tham số khung đã chốt

| Tham số | Giá trị | Ý nghĩa |
|---|---|---|
| `frameN` (N) | 256 mẫu | Số điểm FFT; là lũy thừa của 2 để tận dụng thuật toán chia-để-trị, cho tốc độ O(N log N) |
| Cửa sổ | Hamming | Giảm rò rỉ phổ (spectral leakage) trước khi biến đổi Fourier |
| `hop` | 128 mẫu | Bước nhảy giữa hai khung liên tiếp, tức chồng lấp 50% |
| fs | 8000 Hz | Tần số lấy mẫu của toàn dự án |
| Δf = fs/N | 8000/256 = **31,25 Hz** | Độ phân giải tần số của phổ FFT |

## 4. Công thức

**Cửa sổ Hamming** (n = 0..N−1):

```
w[n] = 0.54 - 0.46*cos(2*pi*n/(N-1))
```

**Trích công suất tại mỗi khung:**

```
xw = frame .* w
X  = fft(xw, N)
E(j) = |X(k_j)|^2      % công suất tại bin k_j, j = 1..8
```

> **Lưu ý:** dùng công suất `|X[k]|^2` chứ **không** dùng biên độ `|X[k]|`, để
> cùng thang đo với `goertzel_power` — `dtmf_decide` giả định đầu vào `E` là
> công suất.

## 5. Bảng bin k và % sai lệch (đủ 7 tần số chuẩn)

Bin `k` là chỉ số nguyên gần nhất ứng với tần số chuẩn `f`, tính theo:

```
k = round(N * f / fs)
f_k_thuc_te = k * fs / N
sai_lech(%) = (f_k_thuc_te - f) / f * 100
```

| Nhóm | Tần số chuẩn (Hz) | k = round(N·f/fs) | f_k thực tế (Hz) | Sai lệch (%) |
|---|---:|---:|---:|---:|
| Hàng 1 | 697  | 22 | 687,500  | −1,36% |
| Hàng 2 | 770  | 25 | 781,250  | +1,46% |
| Hàng 3 | 852  | 27 | 843,750  | −0,97% |
| Hàng 4 | 941  | 30 | 937,500  | −0,37% |
| Cột 1  | 1209 | 39 | 1218,750 | +0,81% |
| Cột 2  | 1336 | 43 | 1343,750 | +0,58% |
| Cột 3  | 1477 | 47 | 1468,750 | −0,56% |

**Nhận xét:** sai lệch lớn nhất là **+1,46%** (tần số 770 Hz), vẫn nằm trong
dung sai **nhận ≤ ±1,5%** của ITU-T Q.24 → khung 256 điểm hợp lệ để dùng làm
đối chứng.

## 6. Bin hài bậc 2

Ngoài 7 bin chuẩn trên, `info.E` cần thêm 1 bin thứ 8 là hài bậc 2 của tần
số mạnh nhất trong khung (bin `2k` của tần số đang trội), dùng để loại khung
nghi là tiếng nói (`reject = 'harmonic'`). Bin này **không cố định** vì phụ
thuộc tần số trội của từng khung, nên không đưa vào bảng tĩnh ở Mục 5.

## 7. Việc cần làm khi cài đặt (`dtmf_decode_fft.m`)

1. `seg = dtmf_segment(y, 'fs',fs, 'frameN',256, 'hop',128);`
2. Với mỗi khung: nhân cửa sổ Hamming, `X = fft(w.*frame, 256)`.
3. Lấy `E(j,i) = |X(k_j)|^2` tại 8 bin (7 bin ở Mục 5 + 1 bin hài ở Mục 6).
4. `[rowIdx, colIdx, conf, reject] = dtmf_decide(E(:,i));`
5. Chống dội (debounce): gộp các khung liên tiếp cùng phím thành một ký tự.

---
*Deliverable Tuần 1 (hạn 18/09): bảng bin k cho đủ 7 tần số kèm % sai lệch —
xem Mục 5. Đã hoàn thành.*
